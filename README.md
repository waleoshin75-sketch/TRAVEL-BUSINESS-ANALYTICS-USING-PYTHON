# Travel Business Analytics Using Python

**Python | Pandas | NumPy | Star schema | Revenue leakage, SLA adherence and customer sentiment analytics**

Built an end-to-end Pandas pipeline that audits, cleans and models four corporate travel datasets, then answers five business objectives on revenue, leakage, SLA performance, churn drivers and cross-sell.

| | |
|---|---|
| **Tools** | Python, Pandas, NumPy, Jupyter |
| **Data** | 4 files: bookings, ancillary services, customer service logs, corporate clients |
| **Deliverable** | Production-ready CSVs and 5 analytical frameworks |

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Dataset Overview](#2-dataset-overview)
3. [Data Quality Assessment](#3-data-quality-assessment)
4. [Data Cleaning and Error Correction](#4-data-cleaning-and-error-correction)
5. [Feature Engineering](#5-feature-engineering)
6. [Data Modeling](#6-data-modeling)
7. [Business Objectives](#7-business-objectives)
8. [Data_Visualization_and_Interpretation](#8-Data_visualization-interpretation)
9. [Key Findings_and_Insight](#9-key-findings_insight)
10. [Recommendations](#10-recommendations)
11. [Conclusion](#11-conclusion)

---

## 1. Project Overview
A corporate travel management company needs clean, connected data to protect revenue and meet client SLAs. This project turns four raw files into a relational Python pipeline and answers five objectives:

1. Optimize omni-channel revenue and margin realization
2. Mitigate transactional revenue leakage
3. Audit SLA adherence and workflow velocity
4. Uncover customer churn triggers and sentiment drivers
5. Enhance market competitiveness through cross-sell

## 2. Dataset Overview

| File | Table | Type | Contents |
|---|---|---|---|
| `fact_travel_bookings.csv` | `df_bookings` | Fact | Fares, taxes, channel, travel type, request and issuance timestamps |
| `fact_ancillary_services.csv` | `df_ancillaries` | Fact | Service type, quoted price, billed status, service date |
| `fact_customer_service.csv` | `df_cs` | Fact | Complaint category, resolution status, satisfaction score, comments |
| `dim_corporate_clients.csv` | `df_clients` | Dimension | Company, client tier, preferred channel, contract SLA hours |

## 3. Data Quality Assessment

Each table was profiled for shape, duplicates, missing values and data types.

```python
print(f"Rows and columns: {df_bookings.shape}")
print(f"Exact duplicate rows: {df_bookings.duplicated().sum()}")
print(df_bookings.isnull().sum())
print(df_bookings.dtypes)
```

| Table | Issue | Impact |
|---|---|---|
| Bookings | Currency symbols and text in fare columns | Fares cannot be summed |
| Bookings | Mixed timestamp formats, inconsistent channel casing | Wrong turnaround times, split channels |
| Ancillaries | Mixed date strings; unbilled services in `billed_status` | Date errors; hidden revenue leakage |
| Customer service | Missing satisfaction scores, inconsistent text casing | Biased ratings, split categories |
| Clients | Mixed-case company names, channels and tiers | Duplicate-looking accounts |

## 4. Data Cleaning and Error Correction

Each cleaned table is saved as a `prod_*.csv` file, and the raw data is left untouched.

**Bookings:** strip currency symbols, convert to numeric, standardize channel text and compute turnaround hours.
```python
df_bookings_production = df_bookings_relational.copy()

for col in ['base_fare_usd', 'taxes_and_fees']:
    df_bookings_production[col] = (
        df_bookings_production[col]
        .astype(str)
        .str.replace(r'[\$,\s]', '', regex=True)
        .str.replace('USD', '', case=False)
    )
    df_bookings_production[col] = pd.to_numeric(df_bookings_production[col], errors='coerce')

df_bookings_production['booking_channel'] = (
    df_bookings_production['booking_channel'].astype(str).str.strip().str.title()
)

df_bookings_production['request_timestamp'] = pd.to_datetime(
    df_bookings_production['request_timestamp'], errors='coerce', utc=True)
df_bookings_production['issuance_timestamp'] = pd.to_datetime(
    df_bookings_production['issuance_timestamp'], errors='coerce', utc=True)

df_bookings_production['processing_duration_hours'] = (
    (df_bookings_production['issuance_timestamp'] - df_bookings_production['request_timestamp'])
    .dt.total_seconds() / 3600.0
)
```

**Ancillary services:** unify mixed date formats into one UTC format.
```python
df_ancillaries_prod['service_date'] = pd.to_datetime(
    df_ancillaries['service_date'], errors='coerce', utc=True)
```

**Customer service:** flag missing scores as `-1` (unrated) and normalize text.
```python
df_cs_production = df_cs_relational.copy()

df_cs_production['customer_satisfaction_score'] = (
    df_cs_production['customer_satisfaction_score'].fillna(-1).astype(int)
)
df_cs_production['complaint_verdict_text'] = (
    df_cs_production['complaint_verdict_text'].astype(str).str.strip().str.capitalize()
)
for col in ['complaint_category', 'resolution_status', 'company_name',
            'client_tier', 'preferred_channel']:
    df_cs_production[col] = df_cs_production[col].astype(str).str.strip().str.title()
```

**Corporate clients:** standardize to Title Case and keep the key columns.
```python
for col in ['company_name', 'client_tier', 'preferred_channel']:
    df_clients_production[col] = df_clients_production[col].astype(str).str.strip().str.title()

df_clients_production = df_clients_production[
    ['client_key', 'corporate_id', 'company_name', 'client_tier',
     'preferred_channel', 'contract_sla_hours']
]
```

## 5. Feature Engineering

| Table | Feature | Logic |
|---|---|---|
| Ancillaries | `revenue_leakage_flag` | "Leakage Risk" if `billed_status` contains "Leakage", else "Secure" |
| Ancillaries | `service_priority_tier` | "High Value" if quoted price is at least 300.00, else "Standard Value" |
| Customer service | `satisfaction_segment` | Highly Satisfied (4 and above), Neutral (3), Dissatisfied (1 to 2), Unrated (-1) |
| Customer service | `critical_complaint_flag` | "High Priority Escalation" if text matches risk keywords |
| Clients | `operational_urgency_tier` | "High Urgency" if contract SLA is 2 hours or less |
| Clients | `account_health_index` | Client tier combined with urgency tier |
| Bookings | `sla_status` | "SLA Breached" if processing hours exceed `contract_sla_hours` (see note) |

```python
# Revenue leakage and service value tier
df_ancillaries_prod['revenue_leakage_flag'] = np.where(
    df_ancillaries_prod['billed_status'].str.contains('Leakage', case=False, na=False),
    'Leakage Risk', 'Secure')

price_numeric = pd.to_numeric(
    df_ancillaries_prod['quoted_price_usd'].astype(str).str.replace(r'[\$,]', '', regex=True),
    errors='coerce')
df_ancillaries_prod['service_priority_tier'] = np.where(
    price_numeric >= 300.00, 'High Value', 'Standard Value')

# Satisfaction segment
conditions = [
    df_cs_prod['customer_satisfaction_score'] >= 4,
    (df_cs_prod['customer_satisfaction_score'] >= 1) & (df_cs_prod['customer_satisfaction_score'] <= 2),
    df_cs_prod['customer_satisfaction_score'] == 3,
    df_cs_prod['customer_satisfaction_score'] == -1,
]
choices = ['Highly Satisfied', 'Dissatisfied', 'Neutral', 'Unrated']
df_cs_prod['satisfaction_segment'] = np.select(conditions, choices, default='Neutral')

# Critical complaint escalation
leakage_keywords = r'overcharge|crash|missed|billing|error|fail'
combined_text = (df_cs_prod['complaint_category'].astype(str) + " "
                 + df_cs_prod['complaint_verdict_text'].astype(str)).str.lower()
df_cs_prod['critical_complaint_flag'] = np.where(
    combined_text.str.contains(leakage_keywords, regex=True, na=False),
    'High Priority Escalation', 'Standard')

# Client urgency and account health
df_clients_prod['operational_urgency_tier'] = np.where(
    df_clients_prod['contract_sla_hours'] <= 2, 'High Urgency', 'Standard Urgency')
df_clients_prod['account_health_index'] = (
    df_clients_prod['client_tier'].astype(str) + " - "
    + df_clients_prod['operational_urgency_tier'].astype(str))

# SLA status (processing hours vs contract SLA)
df_bookings_prod['sla_status'] = np.where(
    df_bookings_prod['processing_duration_hours'] > df_bookings_prod['contract_sla_hours'],
    'SLA Breached', 'SLA Met')
```

> Note: the `sla_status` rule is documented here as the logic the SLA objective relies on. Confirm it matches the definition used when you ran the analysis.

## 6. Data Modeling

Star schema with `dim_corporate_clients` and `fact_travel_bookings` as the anchors. Tables are joined with inner merges on key columns.

| Fact (foreign key) | Dimension or parent (primary key) | Join column |
|---|---|---|
| `fact_travel_bookings` | `dim_corporate_clients` | `client_key` |
| `fact_ancillary_services` | `fact_travel_bookings` | `booking_key` |
| `fact_customer_service` | `dim_corporate_clients` | `client_key` |

```python
df_bookings_relational = pd.merge(df_bookings_raw, df_clients_raw, on='client_key', how='inner')

df_ancillaries_relational = pd.merge(
    df_ancillaries_raw, df_bookings_raw, on='booking_key', how='inner',
    suffixes=('_ancillary', '_booking'))

df_cs_relational = pd.merge(df_cs_raw, df_clients_raw, on='client_key', how='inner')
```

## 7. Business Objectives

| # | Objective | Technique |
|---|---|---|
| 1 | Omni-channel revenue | `groupby` on channel and travel class with count, sum, mean, min, max |
| 2 | Revenue leakage | Merge ancillaries, bookings and clients; filter "Leakage Risk" |
| 3 | SLA adherence | Breach rate and average turnaround by client |
| 4 | Churn and sentiment | Incident volume by category, sentiment and escalation flag |
| 5 | Competitiveness | Cross-sell matrix of flights, hotels, visas, protocol and car hire |

<details>
<summary><b>Objective 1: Optimize omni-channel revenue and margin realization</b></summary>

```python
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')
df_bookings_prod['base_fare_numeric'] = pd.to_numeric(
    df_bookings_prod['base_fare_usd'].astype(str).str.replace(r'[\$,]', '', regex=True),
    errors='coerce')

obj1_pivot = (
    df_bookings_prod
    .groupby(['booking_channel', 'travel_classification'])['base_fare_numeric']
    .agg(transaction_volume='count', total_revenue_usd='sum',
         average_ticket_spend='mean', min_ticket_price='min', max_ticket_price='max')
    .reset_index()
)
print(obj1_pivot.to_string(index=False))
```
</details>

<details>
<summary><b>Objective 2: Mitigate transactional revenue leakage</b></summary>

```python
df_anc = pd.read_csv('prod_ancillary_services.csv')
df_bk = pd.read_csv('prod_travel_bookings.csv')
df_cli = pd.read_csv('prod_corporate_clients.csv')

df_anc['quoted_price_numeric'] = pd.to_numeric(
    df_anc['quoted_price_usd'].astype(str).str.replace(r'[\$,]', '', regex=True),
    errors='coerce')

df_master = pd.merge(df_anc, df_bk, on='booking_key', how='inner', suffixes=('_anc', '_bk'))
df_leakage = pd.merge(df_master, df_cli, on='client_key', how='inner', suffixes=('', '_dup'))

leaked = df_leakage[df_leakage['revenue_leakage_flag'] == 'Leakage Risk']
print(f"Total unbilled revenue leakage: ${leaked['quoted_price_numeric'].sum():,.2f}")

leakage_breakdown = (
    leaked.groupby(['company_name', 'client_tier', 'service_type'])['quoted_price_numeric']
    .agg(incident_count='count', total_leaked_cash='sum')
    .sort_values('total_leaked_cash', ascending=False)
    .reset_index()
)
print(leakage_breakdown.to_string(index=False))
```
</details>

<details>
<summary><b>Objective 3: Audit service SLA adherence and workflow velocity</b></summary>

```python
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')

total_tickets = len(df_bookings_prod)
total_breaches = (df_bookings_prod['sla_status'] == 'SLA Breached').sum()
print(f"Breach rate: {(total_breaches / total_tickets) * 100:.2f}%")

sla_matrix = (
    df_bookings_prod
    .groupby(['company_name', 'client_tier', 'contract_sla_hours', 'sla_status'])
    .agg(ticket_volume=('booking_key', 'count'),
         avg_turnaround_hours=('processing_duration_hours', 'mean'))
    .reset_index()
)
sla_matrix['avg_turnaround_hours'] = sla_matrix['avg_turnaround_hours'].round(2)
print(sla_matrix.sort_values('ticket_volume', ascending=False).to_string(index=False))
```
</details>

<details>
<summary><b>Objective 4: Uncover customer churn triggers and sentiment drivers</b></summary>

```python
df_cs_prod = pd.read_csv('prod_customer_service.csv')
df_clients_prod = pd.read_csv('prod_corporate_clients.csv')

df_cs_master = pd.merge(df_cs_prod, df_clients_prod, on='client_key', how='inner', suffixes=('', '_cli'))

dissatisfied = (df_cs_master['satisfaction_segment'] == 'Dissatisfied').sum()
breached = df_cs_master['resolution_status'].str.contains('Breached', case=False, na=False).sum()
print(f"Dissatisfied ratings: {dissatisfied} | SLA breached tickets: {breached}")

sentiment_matrix = (
    df_cs_master
    .groupby(['complaint_category', 'satisfaction_segment',
              'critical_complaint_flag', 'resolution_status'])
    .size().reset_index(name='incident_volume')
    .sort_values('incident_volume', ascending=False)
)
print(sentiment_matrix.to_string(index=False))
```
</details>

<details>
<summary><b>Objective 5: Enhance industry market competitiveness</b></summary>

```python
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')
df_ancillaries_prod = pd.read_csv('prod_ancillary_services.csv')
df_clients_prod = pd.read_csv('prod_corporate_clients.csv')

df_bookings_prod['is_flight'] = np.where(df_bookings_prod['travel_type'].str.contains('Flight', case=False, na=False), 1, 0)
df_bookings_prod['is_hotel'] = np.where(df_bookings_prod['travel_type'].str.contains('Hotel', case=False, na=False), 1, 0)
core_counts = df_bookings_prod.groupby('company_name').agg(
    flight_volume=('is_flight', 'sum'), hotel_volume=('is_hotel', 'sum')).reset_index()

df_anc_linked = pd.merge(df_ancillaries_prod,
                         df_bookings_prod[['booking_key', 'company_name']],
                         on='booking_key', how='inner')
df_anc_linked['is_visa'] = np.where(df_anc_linked['service_type'].str.contains('Visa', case=False, na=False), 1, 0)
df_anc_linked['is_protocol'] = np.where(df_anc_linked['service_type'].str.contains('Protocol', case=False, na=False), 1, 0)
df_anc_linked['is_car_hire'] = np.where(df_anc_linked['service_type'].str.contains('Car', case=False, na=False), 1, 0)
ancillary_counts = df_anc_linked.groupby('company_name').agg(
    visa_volume=('is_visa', 'sum'),
    protocol_volume=('is_protocol', 'sum'),
    car_hire_volume=('is_car_hire', 'sum')).reset_index()

matrix = pd.merge(core_counts, ancillary_counts, on='company_name', how='left').fillna(0)
competitiveness = pd.merge(df_clients_prod[['company_name', 'client_tier']], matrix,
                           on='company_name', how='inner')
vol_cols = ['flight_volume', 'hotel_volume', 'visa_volume', 'protocol_volume', 'car_hire_volume']
competitiveness[vol_cols] = competitiveness[vol_cols].astype(int)
print(competitiveness.sort_values('flight_volume', ascending=False).to_string(index=False))
```
</details>

## 8. Data Visualization and Interpretation

Each business objective is shown as a pair of graphs side by side.

### Objective 1: Omni-channel revenue
<img width="3000" height="1156" alt="objective-1-revenue (1)" src="https://github.com/user-attachments/assets/2555ccbf-7c9a-4b23-9453-4857b6c60d53" />

### Objective 2: Revenue leakage
<img width="3000" height="1156" alt="objective-2-leakage (1)" src="https://github.com/user-attachments/assets/88180451-9e2b-4582-9159-5cfe3af58c8e" />
Unbilled ancillary services total about $55.1K. Platinum clients account for roughly $32.9K of it, and Airport Protocol is the largest service line at about $20.9K.

### Objective 3: SLA adherence
<img width="3000" height="1156" alt="objective-3-sla (1)" src="https://github.com/user-attachments/assets/5bc6ac06-6e35-4310-a436-d8fb00d7fd64" />
Gamma Pharma Inc. logs the most SLA breaches (57). Theta Ventures has the longest average turnaround on breached tickets, at about 42 hours against a 24-hour contract window.

### Objective 4: Churn triggers and sentiment
<img width="3000" height="1156" alt="objective-4-sentiment (1)" src="https://github.com/user-attachments/assets/15a808aa-9bac-49ab-8c7f-39ff0e48074e" />
Flight Delay Protocol Failure (46) and Overbilling Request Dispute (42) generate the most tickets. Average satisfaction is similar across categories, between 2.9 and 3.1 out of 5.

### Objective 5: Market competitiveness
<img width="3000" height="1156" alt="objective-5-competitiveness (2)" src="https://github.com/user-attachments/assets/d7301f86-3433-45d7-8f8a-692daaef9390" />

## 9. Key Findings and Insight

Outputs depend on the loaded data. Run the notebook to produce current figures. The analysis is built to reveal:

| Objective | Insight delivered |
|---|---|
| Revenue | Ticket volume, total revenue and average spend by channel and travel class |
| Leakage | Total unbilled ancillary revenue and the clients and service lines behind it |
| SLA | Overall breach rate and turnaround time by client and contract window |
| Sentiment | Complaint categories driving dissatisfaction and high-priority escalations |
| Cross-sell | Accounts that buy flights but not hotels, visas, protocol or car hire |

## 10. Recommendations
1. **Close the billing gap:** invoice every ancillary service and review the accounts with the most leakage incidents.
2. **Shift volume to high-yield channels:** use the revenue matrix to promote the channels with the best average spend.
3. **Protect high-urgency accounts:** prioritize processing for clients with contract SLAs of 2 hours or less.
4. **Act on escalation keywords:** route billing, overcharge and failure complaints to a priority queue.
5. **Cross-sell ancillaries:** target flight-heavy accounts with hotel, visa and car hire offers.

## 11. Conclusion
Cleaning four raw files, connecting them through a star schema and engineering business-ready features produced a repeatable Python pipeline for tracking revenue, leakage, service levels and client sentiment.

