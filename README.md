# Corporate Actions & Asset Servicing – Power BI Dashboard

🔒 Disclaimer

All data in this project is synthetic and generated purely for learning and demonstration purposes.
It does not represent any real client, trade, or production system.

A Power BI dashboard simulating a real **Asset Servicing / Corporate Actions Workbench** for an investment bank.

It covers the full flow from **corporate action events → client entitlements → payments → FX bookings → SWIFT messages (MT566 / MT202)** with drillthrough investigation screens.

---

## 🧱 Project Structure

- `CorporateActionsDashboard.pbix` – main Power BI report
- `data/asset_events_expanded.csv` – corporate action events (Dividend, Rights, Tender, Bonus, FX Repatriation)
- `data/asset_entitlements_expanded.csv` – client-level entitlements per event
- `data/asset_payments_expanded.csv` – cash payments, statuses, MT202 references
- `data/asset_fx_bookings_expanded.csv` – FX deals linked to events (MT300-style)
- `data/asset_swift_messages_expanded.csv` – SWIFT messages (MT566 / MT202) per event

> Note: All data is synthetic and generated for demo/training purposes only.

---

## 📊 Data Model

The model follows a simple star-ish structure:

- **Events** (EventID, EventType, ExDate, PayDate, Country, Currency, FXRate, EntitledQty…)
- **Entitlements** (EntitlementID, EventID, ClientAccount, Country, SecurityISIN, PositionQty, EntitledQty, GrossAmountLCY, NetAmountLCY, Currency)
- **Payments** (PaymentID, EntitlementID, PayDate, PaymentAmountLCY, PaymentAmountUSD, PaymentStatus, PaymentChannel, MT202Ref)
- **FXBookings** (FXId, EventID, TradeDate, ValueDate, FromCurrency, ToCurrency, DealRate, NotionalFrom, NotionalTo, MT300Ref)
- **SwiftMessages** (SwiftID, EventID, MessageType, Status, Reference, RelatedReference, CreatedDateTime)
- **DimDate** (Date, Year, Month, Quarter, YearMonth) – related to Events/PayDate

Key relationships:

- `Events[EventID]` → `Entitlements[EventID]`
- `Entitlements[EntitlementID]` → `Payments[EntitlementID]`
- `Events[EventID]` → `FXBookings[EventID]`
- `Events[EventID]` → `SwiftMessages[EventID]`
- `DimDate[Date]` → `Events[PayDate]`

---

## 📈 Report Pages

### 1. Corporate Actions Overview

High-level view of the corporate actions book:

- KPIs:
  - Total Events
  - Total Entitled Quantity
  - Total Payment (USD)
  - Paid %
- Visuals:
  - Total Payment USD by **EventType**
  - Total Payment USD by **Country**
  - Total Payment USD by **YearMonth** (trend line)
- Slicers:
  - YearMonth
  - Country
  - PaymentStatus
  - EventType

This page answers:
> How much are we paying by event type / country, and how is that trending over time?

---

### 2. Client Entitlements & Payments

Client / account focused view:

- **Top 10 Accounts by Payment USD** (bar chart)
- **Problematic Payments (Not Released)** – table filtered for `PaymentStatus <> Released`
- **Matrix by ClientAccount and EventType**:
  - Total Entitled Qty
  - Total Payment LCY
  - Unpaid Amount LCY (Entitlement – Payments)

This page answers:
> Which clients have the largest corporate action cash flows, and where do we have pending or failed payments?

---

### 3. FX & SWIFT Monitoring

Operational monitoring for FX and messaging:

- KPIs:
  - Total FX Notional From (LCY)
  - Total FX Notional To USD
  - FX Coverage % (vs payments)
  - MT202 messages
  - MT566 messages
  - Events with missing MT202
- Visuals:
  - FX Notional From by EventID
  - Count of SWIFT MessageType by EventID
  - Table of **Events with Missing MT202** (by EventType / Country / EventID)

This page answers:
> Are FX deals booked to cover our LCY exposures, and do all events with released payments have corresponding MT202 messages?

---

### 4. Event Details (Drillthrough)

Investigative drillthrough page for a single EventID:

- Selected Event card (EventID)
- Event details (EventType, PayDate, Country, Gross/Net amounts)
- Entitlements by ClientAccount
- Payments table (amounts, status, channel, MT202Ref)
- SWIFT messages table (MT566 / MT202, status, references)

Accessed via **drillthrough** from other pages (e.g., from FX & SWIFT Monitoring or Client Entitlements).

This page answers:
> For a specific event, who is entitled, what has been paid, and what SWIFT messages/FX trades exist?

---

## 🧮 Key DAX Measures (examples)

Some of the core measures used in the report:

```DAX
Total Events =
DISTINCTCOUNT( Events[EventID] )

Total Payment LCY =
SUM( Payments[PaymentAmountLCY] )

Total Payment USD =
SUM( Payments[PaymentAmountUSD] )

Total Entitled Qty =
SUM( Entitlements[EntitledQty] )

Paid % =
DIVIDE( [Total Payment LCY], [Total Entitlement LCY] )

Total FX Notional From =
SUM( FXBookings[NotionalFrom] )

Total FX Notional To USD =
SUM( FXBookings[NotionalTo] )

FX Coverage % =
DIVIDE( [Total FX Notional From], [Total Payment LCY] )

# MT202 Messages =
CALCULATE(
    COUNTROWS( SwiftMessages ),
    SwiftMessages[MessageType] = "MT202"
)

# Events Missing MT202 =
[Events With Released Payments] - [Events With MT202]
