# Master AI Build Prompt — Remito Transfer Pricing & eWire Desk

Use this document as the complete implementation prompt for an AI coding agent.

---

## 1. Product goal

Build a professional, browser-based remittance operations application named **Remito — Transfer Desk**. It is an internal tool for:

1. Calculating customer collection amounts, recipient amounts, agent fees, and operator margin.
2. Supporting bidirectional calculations: the operator can edit either **You Send** or **Recipient Gets**.
3. Using a live USD/CAD market rate plus a configurable operational spread of CAD 0.0300.
4. Managing country pricing, fee tiers, agents, Syria city rates, and Jordan-specific rates.
5. Turning a completed calculator quote into an eWire transaction.
6. Saving daily eWire transactions in the browser.
7. Searching saved senders and automatically finding beneficiaries previously linked to each sender.
8. Generating separate editable WhatsApp messages for agents and senders.
9. Exporting daily transactions to CSV, copying rows into Google Sheets, or posting them to a Google Apps Script endpoint.

The application must be suitable for static hosting on GitHub Pages. Separate the project into at least:

- `index.html`
- `styles.css`
- `app.js`
- `google-apps-script.gs`
- `README.md`

Do not require a build system or backend for the basic calculator and local eWire register.

---

## 2. Main navigation

Create a left sidebar with these tabs:

1. **Calculator**
2. **eWire Details**
3. **Settings & Rates**

The sidebar should also display the active calculation rate:

```text
1 USD = [Google Finance USD/CAD spot + 0.0300] CAD
```

Use a top bar that changes its title and status chip according to the active tab.

---

## 3. Visual design

Use a professional warm remittance design inspired by a modern international money-transfer website:

- Orange and peach gradient page background.
- Soft white radial glows and subtle texture.
- Dark espresso sidebar.
- Cream/white translucent cards.
- Large rounded corners, approximately 22–28 px.
- Orange primary buttons.
- Bold black headings.
- White rounded amount fields.
- Green-to-blue gradient rate banner.
- Real country flag images, not Unicode flag characters.
- Responsive desktop and mobile layouts.

Use accessible labels, visible focus states, keyboard-friendly menus, and sufficient contrast.

---

# PART A — TRANSFER CALCULATOR

## 4. Calculator fields

The calculator contains:

1. **Destination country** custom dropdown with actual flag images.
2. **You Send** amount field.
3. A clickable currency chip inside the You Send field:
   - CAD
   - USD
4. Editable USD/CAD calculation-rate field when CAD is being sent.
5. **Recipient Gets** amount field.
6. A clickable currency chip inside the Recipient Gets field.
7. Syria city selector, visible only when Syria/SYP is selected.
8. Results section.
9. **Create eWire Details** button after a valid calculation.

Both amount fields must remain editable. Never make either one permanently read-only.

### Calculation direction

Track the field most recently edited:

- If **Recipient Gets** was edited, calculate the required You Send amount.
- If **You Send** was edited, calculate the recipient amount.

Use this state:

```text
lastEditedField = "recipient" or "send"
```

### Supported currencies

Sending currencies for all destinations:

- CAD
- USD

Receiving currencies supported by current formulas:

- Syria: USD or SYP
- Jordan: JOD
- All other configured destinations: USD

Only show currencies that have an implemented formula.

---

## 5. Global rounding rule

Do not use `Math.ceil()` or `Math.floor()` for calculated money.

Round normally to the nearest two decimal places:

```js
function roundToTwo(amount) {
    return Math.round((amount + Number.EPSILON) * 100) / 100;
}
```

Display all CAD, USD, JOD, and SYP amounts with two decimal places.

---

## 6. Pricing data model

Each standard country has:

```js
{
  tiers: [
    { max: number, add: number }
  ],
  pctRate: decimal,
  usdPayPct: decimal,
  fees: [
    { max: number, amount: number }
  ],
  feePct: decimal
}
```

Meanings:

- `tiers`: recipient USD threshold and fixed amount added to the customer collection.
- `pctRate`: CAD percentage markup after all fixed tiers.
- `usdPayPct`: USD collection percentage markup after all fixed tiers.
- `fees`: agent fee tiers in USD.
- `feePct`: percentage agent fee after all fixed agent-fee tiers.

---

## 7. Default country pricing

### Syria

```text
Collection tiers:
- Up to USD 400: add 15
- Up to USD 700: add 20
CAD percentage after tiers: 3%
USD collection percentage after tiers: 4%

Agent fee tiers:
- Up to USD 300: USD 5
- Up to USD 999: USD 7
Agent fee percentage after tiers: 0.7%
```

### Lebanon

```text
Collection tiers:
- Up to USD 650: add 20
CAD percentage: 3%
USD collection percentage: 4%

Agent fee tiers:
- Up to USD 2,500: USD 5
Agent fee percentage after tiers: 0.2%
```

### Lebanon OMT

```text
Collection tiers:
- Up to USD 700: add 25
CAD percentage: 3.5%
USD collection percentage: 4.5%

Agent fee tiers:
- Up to USD 300: USD 5
- Up to USD 999: USD 7
- Up to USD 1,500: USD 10
- Up to USD 2,000: USD 12
Agent fee percentage after tiers: 0.7%
```

### Gazah

```text
Collection tiers:
- Up to USD 800: add 20
CAD percentage: 2.5%
USD collection percentage: 3.5%

Agent fee tiers:
- Up to USD 1,000: USD 7
Agent fee percentage after tiers: 0.7%
```

### Dafeh

```text
Collection tiers:
- Up to USD 650: add 20
CAD percentage: 3%
USD collection percentage: 4%

Agent fee tiers:
- Up to USD 2,500: USD 10
Agent fee percentage after tiers: 0.4%
```

### Turkey

```text
Collection tiers:
- Up to USD 800: add 20
CAD percentage: 2.5%
USD collection percentage: 3.5%

Agent fee tiers:
- Up to USD 2,500: USD 5
Agent fee percentage after tiers: 0.2%
```

### Iraq

Use the same defaults as Turkey.

### Egypt

```text
Collection tiers:
- Up to USD 500: add 20
CAD percentage: 4%
USD collection percentage: 5%

Agent fee tiers:
- Up to USD 1,000: USD 7
Agent fee percentage after tiers: 0.7%
```

### Jordan

Jordan uses its own formula:

```text
JOD → USD rate: 1.4135
Threshold: USD 630
Flat customer fee: USD 20
Percentage customer fee above threshold: 3%

Agent fee tiers:
- Up to USD 2,500: USD 5
Agent fee percentage after tier: 0.2%
```

---

## 8. Standard CAD forward calculation

Use when:

- Sending currency is CAD.
- Recipient Gets was edited.
- Destination is not Jordan and not Syria/SYP.

Inputs:

```text
country
recipientUSD
usdCadRate
```

### Customer collection

For the first collection tier where:

```text
recipientUSD <= tier.max
```

calculate:

```text
customerPaysCAD = usdCadRate × (recipientUSD + tier.add)
```

If the amount exceeds every fixed tier:

```text
customerPaysCAD = usdCadRate × recipientUSD × (1 + pctRate)
```

Then apply normal two-decimal rounding.

### Agent fee

For the first agent-fee tier where:

```text
recipientUSD <= feeTier.max
```

use:

```text
agentFeeUSD = feeTier.amount
```

If it exceeds every fee tier:

```text
agentFeeUSD = recipientUSD × feePct
```

### Margin

```text
marginCAD = customerPaysCAD
            - (recipientUSD × usdCadRate)
            - (agentFeeUSD × usdCadRate)
```

---

## 9. Standard CAD reverse calculation

Use when:

- Sending currency is CAD.
- You Send was edited.
- Destination is not Jordan and not Syria/SYP.

Input:

```text
customerPaysCAD
```

Solve each fixed collection tier algebraically:

```text
candidateRecipientUSD = customerPaysCAD / usdCadRate - tier.add
```

The candidate is valid only when it is above the previous tier boundary and at or below the current tier maximum.

If no fixed tier is valid, solve the percentage tier:

```text
candidateRecipientUSD = customerPaysCAD / (usdCadRate × (1 + pctRate))
```

Use it only when it is above the final fixed-tier maximum.

Round the recipient amount normally to two decimals.

Calculate the agent fee using the resulting recipient USD amount.

```text
marginCAD = customerPaysCAD
            - (recipientUSD × usdCadRate)
            - (agentFeeUSD × usdCadRate)
```

If no positive solution exists, display a clear error.

---

## 10. Standard USD forward calculation

Use when:

- Sending currency is USD.
- Recipient Gets was edited.
- Destination is not Jordan and not Syria/SYP.

Both sides are USD.

For the first fixed tier:

```text
customerPaysUSD = recipientUSD + tier.add
```

After all fixed tiers:

```text
customerPaysUSD = recipientUSD × (1 + usdPayPct)
```

Then:

```text
marginUSD = customerPaysUSD - recipientUSD - agentFeeUSD
```

---

## 11. Standard USD reverse calculation

Use when:

- Sending currency is USD.
- You Send was edited.

For each fixed tier:

```text
candidateRecipientUSD = customerPaysUSD - tier.add
```

Validate against that tier’s boundaries.

For the percentage tier:

```text
candidateRecipientUSD = customerPaysUSD / (1 + usdPayPct)
```

Use the customer-paid USD amount as the fee-tier basis in this reverse mode:

```text
agentFeeUSD = agentFeeUSD(country, customerPaysUSD)
marginUSD = customerPaysUSD - recipientUSD - agentFeeUSD
```

---

# SYRIA SYP CALCULATIONS

## 12. Syria city-rate model

Store daily rates by city:

```js
{
  lastUpdated: "YYYY-MM-DD",
  cities: {
    cityName: {
      usdSyp: number,
      cadSyp: number
    }
  }
}
```

Default cities:

- دمشق( مرجة )
- دمشق ( سمان )
- حلب
- حماة
- حمص
- درعا
- اللاذقية
- طرطوس
- الهرم
- شام كاش

Default example rates:

```text
Damascus/Marjeh, Damascus/Samman, Aleppo, Hama:
- USD/SYP: 129.50
- CAD/SYP: 86.30

Homs, Daraa, Latakia, Tartous:
- USD/SYP: 129.00
- CAD/SYP: 86.00

Al-Haram, Sham Cash:
- USD/SYP: 128.50
- CAD/SYP: 85.70
```

Before allowing a Syria/SYP calculation, require:

```text
lastUpdated === today in local YYYY-MM-DD format
```

Settings must have an **Update Today** confirmation button.

USD sending mode is not supported for SYP recipient calculations. Show an error and require CAD.

---

## 13. Syria SYP forward calculation

Use when Recipient Gets SYP was edited.

```text
sypReceived = entered recipient amount
cadNet = sypReceived / city.cadSyp
customerFeeCAD = cadNet < 500 ? 10 : 0
customerPaysCAD = roundToTwo(cadNet + customerFeeCAD)
agentPrincipalUSD = sypReceived / city.usdSyp
agentFeeUSD = 5
marginCAD = customerPaysCAD
            - ((agentPrincipalUSD + agentFeeUSD) × usdCadRate)
```

---

## 14. Syria SYP reverse calculation

Use when You Send CAD was edited.

```text
customerPaysCAD = entered amount
customerFeeCAD = customerPaysCAD < 500 ? 10 : 0
cadNet = customerPaysCAD - customerFeeCAD
```

If `cadNet <= 0`, show an error.

```text
sypReceived = roundToTwo(cadNet × city.cadSyp)
agentPrincipalUSD = sypReceived / city.usdSyp
agentFeeUSD = 5
marginCAD = customerPaysCAD
            - ((agentPrincipalUSD + agentFeeUSD) × usdCadRate)
```

For Syria/SYP, show only one fee row:

```text
Total Fee Applied = customerFeeCAD
```

Do not duplicate it with a second Customer Fee row.

```text
Below CAD 500: CAD 10.00
At or above CAD 500: CAD 0.00
```

---

# JORDAN CALCULATIONS

## 15. Jordan CAD forward

Recipient enters JOD.

```text
jodAmount = recipient amount
usdNet = jodAmount × jodRate
```

Customer-fee rule:

```text
if usdNet <= threshold:
    grossUSD = usdNet + flatFee
else:
    grossUSD = usdNet × (1 + pctFee)
```

```text
customerPaysCAD = roundToTwo(grossUSD × usdCadRate)
agentFeeUSD = agentFeeUSD(country, grossUSD)
marginCAD = customerPaysCAD
            - (usdNet × usdCadRate)
            - (agentFeeUSD × usdCadRate)
```

---

## 16. Jordan CAD reverse

```text
customerPaysCAD = entered amount
grossUSD = customerPaysCAD / usdCadRate
```

Reverse customer-fee rule:

```text
if grossUSD <= threshold:
    usdNet = grossUSD - flatFee
else:
    usdNet = grossUSD / (1 + pctFee)
```

If `usdNet <= 0`, display an error.

```text
jodAmount = roundToTwo(usdNet / jodRate)
agentFeeUSD = agentFeeUSD(country, grossUSD)
marginCAD = customerPaysCAD
            - (jodAmount × jodRate × usdCadRate)
            - (agentFeeUSD × usdCadRate)
```

---

## 17. Jordan USD forward

```text
jodAmount = recipient amount
usdNet = jodAmount × jodRate

if usdNet <= threshold:
    customerPaysUSD = usdNet + flatFee
else:
    customerPaysUSD = usdNet × (1 + pctFee)

customerPaysUSD = roundToTwo(customerPaysUSD)
agentFeeUSD = agentFeeUSD(country, customerPaysUSD)
marginUSD = customerPaysUSD - usdNet - agentFeeUSD
```

---

## 18. Jordan USD reverse

```text
customerPaysUSD = entered amount
grossUSD = customerPaysUSD

if grossUSD <= threshold:
    usdNet = grossUSD - flatFee
else:
    usdNet = grossUSD / (1 + pctFee)

jodAmount = roundToTwo(usdNet / jodRate)
agentFeeUSD = agentFeeUSD(country, grossUSD)
marginUSD = customerPaysUSD
            - (jodAmount × jodRate)
            - agentFeeUSD
```

---

## 19. Result rows

For every successful calculation show:

1. Agent Fee
2. Total Fee Applied
3. Your Margin

For non-Syria/SYP routes:

```text
If customer paid CAD:
    totalFeeAppliedCAD = marginCAD + agentFeeUSD × usdCadRate

If customer paid USD:
    totalFeeAppliedUSD = marginUSD + agentFeeUSD
```

For Syria/SYP, use the explicit CAD 10/0 customer fee instead.

---

# PART B — LIVE USD/CAD RATE

## 20. Google Finance market rate

The Settings page must display the raw Google Finance market indication.

The calculator must use:

```text
calculationRate = googleFinanceUsdCadSpot + 0.0300 CAD
```

Do not show the raw spot inside the calculator as though it were the calculation rate.

### Google quote direction

Read Google Finance’s public CAD/USD quote and invert it:

```text
USD/CAD = 1 / CAD/USD
```

Validate that the resulting USD/CAD value is within a realistic range, for example 1.0–2.0.

### Static-host reliability strategy

Google Finance does not provide an official browser API and blocks normal cross-origin browser requests. For a GitHub Pages implementation:

1. Attempt multiple CORS-compatible readers in parallel.
2. Use a request timeout of approximately nine seconds.
3. Retry a failed round once.
4. Save the last successful spot and calculation rate in `localStorage`.
5. Display the cached rate immediately at startup.
6. Refresh in the background.
7. Refresh every five minutes.
8. Refresh when the browser returns online.
9. Refresh when the tab becomes visible.
10. If live retrieval fails, keep using the last successful saved rate instead of displaying a disruptive unavailable state.

If the operator manually edits the calculation rate, background refreshes must not overwrite it. Clicking the explicit Refresh button should return the field to live-update mode.

---

# PART C — SETTINGS

## 21. Pricing settings

Allow the operator to:

- Select a destination country.
- Edit fixed collection tiers.
- Edit CAD percentage markup.
- Edit USD percentage markup.
- Edit agent-fee tiers.
- Edit percentage agent fee.
- Save changes to local storage.
- Reset defaults.
- Add a new country.

New-country template:

```text
No fixed tiers
CAD percentage: 3%
USD collection percentage: 4%
No fixed agent-fee tiers
Agent fee percentage: 1%
```

---

## 22. Automatic country flags

Support all ISO countries and territories.

Recognize:

- Common names
- Official names
- ISO alpha-2
- ISO alpha-3
- FIFA/IOC abbreviations
- Alternative spellings
- Common aliases such as KSA, SAU, Saudi Arabia, UAE, UK, and USA

Use an embedded country-name-to-ISO lookup so detection does not depend on a runtime API.

For built-in countries, embed SVG flag data. For newly added countries, a stable flag URL such as FlagCDN may be used. Save custom country-code mappings in local storage.

---

## 23. Agent Directory

Settings must include an Agent Directory.

Each agent record contains:

```js
{
  id,
  country,
  name,
  phone
}
```

The phone is the international WhatsApp number.

Features:

- Add agent.
- Assign agent to country.
- Remove agent.
- Filter eWire agent choices by selected country.
- Save agents in local storage.

---

# PART D — CALCULATOR TO eWIRE

## 24. Quote snapshot

After every valid calculation, store a clean internal snapshot:

```js
{
  capturedAt,
  country,
  city,
  receiveAmount,
  receiveCurrency,
  agentFees,
  agentFeesCurrency: "USD",
  paidAmount,
  paidCurrency: "CAD" or "USD",
  calculationRate,
  mode
}
```

Clear the snapshot whenever the current calculator input is invalid.

### Create eWire Details button

Show this button only with successful results.

When clicked:

1. Open the eWire Details tab.
2. Initialize a new form.
3. Generate the current date/time.
4. Generate the next reference.
5. Transfer country.
6. Transfer Syria city when applicable.
7. Transfer receive amount.
8. Transfer receiving currency.
9. Transfer agent fee as USD.
10. Put customer paid amount into Paid CAD or Paid USD according to sending currency.
11. Leave customer identity, beneficiary identity, agent, payment methods, pickup location, and teller available for completion.

---

# PART E — eWIRE DETAILS

## 25. eWire fields

Create an eWire form containing:

1. Date and time
2. Reference number
3. Beneficiary Name
4. Beneficiary Phone
5. Receive Amount
6. CCY
7. Agent Fees (USD)
8. Country
9. City
10. Agent
11. Agent Payment Method
12. Pick-Up Location
13. Sender Name
14. Sender Phone (Canadian or US)
15. Paid Amount CAD
16. Paid Amount USD
17. Sender Payment Method
18. Teller

Required fields:

- Beneficiary name
- Beneficiary phone
- Receive amount
- CCY
- Country
- City
- Agent
- Sender name
- Sender phone
- Sender payment method
- Teller

Require at least one positive customer paid amount: CAD or USD.

Validate Canadian/US sender phone as:

- Ten digits, or
- Eleven digits beginning with 1

Store it normalized with leading country code 1.

---

## 26. Automatic date/time

On every new form:

```text
Date = current local date and time
```

Use a `datetime-local` compatible value and make it read-only unless an explicit override is desired.

---

## 27. Automatic reference

Format:

```text
SETRF_YYMMDD-NN
```

Examples:

```text
SETRF_260904-01
SETRF_260904-02
SETRF_260904-21
```

Rules:

1. Use local date.
2. Find the largest serial already stored for that date.
3. Add one.
4. Pad to at least two digits.
5. Reset the sequence each new day.

---

## 28. Daily transaction storage

Store eWire transactions in local storage.

Each transaction requires a unique internal ID in addition to the visible reference.

Show today’s records in an HTML table with:

- Reference
- Date
- Beneficiary
- Receive amount
- Country / agent
- Sender
- Paid amount
- Teller
- Sync status
- Actions

Actions:

- Open editable messages
- Quick Agent WhatsApp
- Quick Sender WhatsApp
- Delete transaction

Show pending/synced status.

Allow clearing today’s table only after confirmation.

---

# PART F — SENDER & BENEFICIARY DIRECTORY

## 29. Contact data model

Store sender profiles with nested beneficiary relationships:

```js
{
  id,
  name,
  phone,
  createdAt,
  lastUsedAt,
  beneficiaries: [
    {
      id,
      name,
      phone,
      country,
      city,
      agentName,
      lastUsedAt
    }
  ]
}
```

---

## 30. Sender search workflow

In the eWire form:

1. Search sender by name or phone.
2. Show matching sender profiles.
3. Show each sender’s phone and number of known beneficiaries.
4. Selecting a sender fills sender name and phone.
5. Immediately display that sender’s saved beneficiaries.
6. Selecting a beneficiary fills:
   - Beneficiary name
   - Beneficiary phone
   - Country
   - City
   - Matching country agent, when found

Support English and Arabic search. Normalize:

- Case
- Repeated whitespace
- Latin accents
- Arabic diacritics
- Tatweel
- Common punctuation

Do not merge unrelated senders solely because they share a generic phone number. Match using normalized name plus normalized phone.

---

## 31. Automatic contact learning

Whenever an eWire transaction is saved:

1. Find or create the sender.
2. Find or create the beneficiary under that sender.
3. Update phone, country, city, and agent.
4. Update last-used timestamps.
5. Make the relationship immediately searchable next time.

This must happen without a separate Save Contact step.

---

## 32. Contact CSV import/export

Settings must have **Sender & Beneficiary Directory** with:

- Sender count
- Beneficiary relationship count
- Import Contacts CSV
- Export Directory CSV

Expected CSV columns:

```text
Sender
Sender Phone
Beneficiary
Beneficiary Phone
Country
City
Agent
```

Import requirements:

- Parse quoted commas and multiline CSV fields.
- Accept UTF-8 Arabic.
- Merge repeated sender/beneficiary relationships.
- Skip rows containing literal `????` corruption.
- Skip empty and unusable records.
- Report imported and skipped counts.

Important: literal `????` is not an encoding-display issue; the original characters have already been destroyed. Do not import such placeholders as real names.

---

# PART G — WHATSAPP MESSAGES

## 33. Message composer

Each transaction must have two independent editable messages:

1. Agent message
2. Sender message

The Messages action opens a modal with:

- Agent textarea
- Sender textarea
- Copy Agent
- Copy Sender
- Send to Agent
- Send to Sender
- Reset defaults
- Save changes

Save custom text with the transaction. Quick-send actions must use the latest saved custom message or the current default template.

---

## 34. Agent WhatsApp template

Use this structure:

```text
🟢 *TRANSFER AUTHORIZATION* 🟢
🔢 *Ref:* `SETRF_260904-01` | 📅 *Date:* 2026-09-04

*📥 RECEIVER DETAILS (SYRIA)*
👤 Name: Obada Hiber
📞 Mobile: +12892302289
📍 City/Area: Damascus - Bab Musala
📱 Payout: Mobile Wallet
💵 *Amount to Pay:* *$1,200.00 USD*

*📤 SENDER & FEE DETAILS*
👤 Name: Samer Hiber
💳 Method: Interac e-Transfer
🏷️ *Agent Fees:* $10.00 USD (Teller: 02)
📌 Pick-Up Location: [only when provided]
```

Privacy requirement:

**Never show the agent how much the sender paid.**

The agent message must not contain:

- Paid Amount CAD
- Paid Amount USD
- Total collected from sender
- Operator margin

It may contain only the beneficiary payout amount, operational payout details, agent fee, sender name, payment-method label, and teller.

---

## 35. Sender WhatsApp template

Use an emoji-based customer confirmation:

```text
🟠 *TRANSFER CONFIRMATION* 🟠
🔢 *Ref:* `SETRF_260904-01` | 📅 *Date:* 2026-09-04

👋 Hello [Sender Name],
✅ Your transfer has been recorded successfully.

*📥 RECEIVER DETAILS*
👤 Name: [Beneficiary]
📞 Mobile: [Beneficiary Phone]
🌍 Destination: [City], [Country]
📱 Payout: [Agent Payment Method]
💵 *Receive Amount:* *[Amount] [CCY]*
📌 Pick-Up Location: [when provided]

*📤 PAYMENT DETAILS*
💵 *Paid Amount:* [CAD amount when positive]
💵 *Paid Amount:* [USD amount when positive]
💳 Method: [Sender Payment Method]
🧾 Teller: [Teller]

🔐 Please keep reference `[Reference]` for your records.
🙏 Thank you for choosing Remito.
```

---

# PART H — GOOGLE SHEETS & EXPORTS

## 36. Daily export methods

Provide all three:

1. Copy for Google Sheets as tab-separated values.
2. Download UTF-8 CSV with BOM.
3. Send pending rows to a Google Apps Script Web App.

Export columns:

```text
Date
Reference #
Benef. Name
Benef. Phone #
Receive Amount
CCY
Agent Fees (USD)
Country
City
Agent
Agent Phone
Agent Payment Method
Pick-Up Location
Sender Name
Sender Phone #
Paid Amount (CAD)
Paid Amount (USD)
Sender Payment Method
Teller
Agent WhatsApp Message
Sender WhatsApp Message
Synced
```

---

## 37. Google Apps Script receiver

Provide a `google-apps-script.gs` file.

Deployment instructions:

1. Open the destination Google Sheet.
2. Extensions → Apps Script.
3. Paste script.
4. Deploy as Web App.
5. Execute as the sheet owner.
6. Allow access to Anyone.
7. Paste the `/exec` URL into the eWire tab.

The script must:

- Accept JSON POST data.
- Create an `eWire Transactions` sheet when absent.
- Create and freeze the header row.
- Append transaction rows.
- Store the internal record ID.
- Prevent duplicate rows by record ID.
- Store both WhatsApp messages.

The browser should preserve CSV and copy-to-Sheets as backups because `no-cors` Web App posting cannot fully verify the Google response.

---

# PART I — LOCAL STORAGE

## 38. Suggested keys

Use stable local-storage keys such as:

```text
fx_pricing
syria_rates
last_google_usdcad
custom_country_flags
remito_ewire_transactions
remito_ewire_agents
remito_ewire_contacts
remito_ewire_sheet_url
```

Handle malformed or missing stored JSON without crashing. Fall back to defaults.

---

# PART J — ACCEPTANCE TESTS

## 39. Calculator tests

1. Editing recipient amount calculates send amount.
2. Editing send amount calculates recipient amount.
3. Switching CAD/USD changes formula mode without disabling either amount field.
4. No money result is always rounded up to a whole number.
5. Results use two decimal places.
6. Syria SYP requires today’s confirmed city rates.
7. Syria SYP below CAD 500 displays one CAD 10 fee row.
8. Syria SYP at or above CAD 500 displays CAD 0.
9. Jordan JOD forward and reverse work in CAD and USD modes.
10. Agent fee, total fee, and margin use the formulas above.

## 40. Live-rate tests

1. Settings shows raw Google spot.
2. Calculator uses raw spot + 0.0300.
3. Cached rate appears immediately on reload.
4. Temporary proxy failure does not erase the current rate.
5. Manual rate edits survive background refresh.
6. Explicit Refresh restores automatic updating.

## 41. eWire tests

1. Date/time is current.
2. Daily serial increments correctly.
3. Calculator quote populates a fresh eWire form.
4. CAD payment fills only Paid CAD.
5. USD payment fills only Paid USD.
6. Agent fee transfers as USD.
7. Sender phone validation accepts valid CA/US numbers.
8. Saving displays the transaction in today’s table.
9. Reload preserves the table.
10. CSV and copy exports contain all fields.

## 42. Contact tests

1. Saving a new sender and beneficiary creates a relationship.
2. Searching the sender later displays that beneficiary.
3. Selecting the beneficiary fills details.
4. Multiple beneficiaries can belong to one sender.
5. Same beneficiary name may belong to different senders.
6. UTF-8 Arabic CSV imports correctly.
7. `????` rows are skipped.

## 43. WhatsApp tests

1. Agent link uses agent phone.
2. Sender link uses sender phone.
3. Both messages can be edited and saved.
4. Agent message never contains Paid CAD or Paid USD.
5. Sender message includes positive paid amounts.
6. Custom messages are included in Google Sheet/CSV exports.

---

## 44. Final delivery requirements

Deliver a complete, working static web application. Do not provide only mockups. Preserve data across browser refreshes. Keep the calculator, eWire register, settings, contact directory, messages, and exports integrated into one coherent application.
