# TRAVEL BUSINESS ANALYTICS USING PYTHON
### *Optimizing Omni-Channel Revenue, Mitigating Transactional Leakage, and Auditing Operational SLA Workflow Velocity Using Python & Pandas*

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Dataset Overview](#2-dataset-overview)
3. [Data Quality Assessment](#3-data-quality-assessment)
4. [Data Transformation](#4-data-transformation)
5. [Data Modeling & Relational Schema](#5-data-modeling--relational-schema)
6. [Data Cleaning & Error Correction](#6-data-cleaning--error-correction)
7. [Feature Engineering](#7-feature-engineering)
8. [Core Business Analysis](#8-core-business-analysis)
9. [Data Visualization & Interpretation](#9-data-visualization--interpretation)
10. [Key Findings & Insights](#10-key-findings--insights)
11. [Recommendations](#11-recommendations)
12. [Conclusion](#12-conclusion)

---

## 1 Project Overview
In the corporate Travel Management Company (TMC) sector, data pipelines constantly ingest fractured telemetry. This project builds an end-to-end data transformation engine and decision cockpit using Python and Pandas. The system processes raw transactions, eliminates multi-table anomalies, unmasks hidden revenue risks, and evaluates workflow delivery margins against strict corporate service agreements.

---

## 2 Dataset Overview
The relational infrastructure cross-references four primary data ledgers containing 500+ operational records:
*   dim_corporate_clients: Account master profiles, spend classification tiers, and negotiated response windows.
*   fact_travel_bookings: Core transaction tickets tracking base fares, ticketing channels, and request-to-issuance timeline logs.
*   fact_ancillary_services: High-margin value-add supplementary line items (Visa assistance, Airport Protocol, Private Car Hires).
*   fact_customer_service: Helpdesk interaction logs, resolution tracking states, and text-based client complaint registries.

---

## 3 Data Quality Assessment
An initial assessment of the raw source files revealed severe operational friction issues that threatened downstream reporting:
*   Corrupted Financials: Currency entries were polluted with heavy string formatting (dollar signs, commas, whitespace, and USD tokens).
*   Chronological Gaps: Key fulfillment timestamp columns contained empty NaN zones and inconsistent data types.
*   Survey Fragmentation: The customer survey logs contained widespread missing satisfaction values coded arbitrarily as blank rows.
*   Casing Volatility: Text fields across lookups and traveler comments suffered from erratic uppercase casing.

---

## 4 Data Transformation
The ETL pipeline isolates text string noise dynamically using Python. It uses Regular Expressions (Regex) to strip out complex currency symbols and forces variables into high-precision numerical float formats. Erratic, mixed-format regional datetime markers were programmatically parsed into a standardized UTC Chronological Object Map.

```python
# Synchronizing erratic क्षेत्रीय regional datetimes to uniform UTC datetime timestamps
df_bookings['request_timestamp'] = pd.to_datetime(df_bookings['request_timestamp'], errors='coerce', utc=True)
df_bookings['issuance_timestamp'] = pd.to_datetime(df_bookings['issuance_timestamp'], errors='coerce', utc=True)

# Calculating high-precision elapsed operational turnaround metrics
df_bookings['processing_duration_hours'] = (
    (df_bookings['issuance_timestamp'] - df_bookings['request_timestamp']).dt.total_seconds() / 3600.0
)
```

---

## 5 Data Modeling & Relational Schema
The database engine links independent tracking logs into a unified relational schema using clear primary and foreign key pathways:
*   dim_corporate_clients.client_key (PK) connects directly to core bookings and helpdesk registries.
*   fact_travel_bookings.booking_key (PK) acts as the operational root node linking secondary value-added service lines back to the primary umbrella transactions.

---

## 6 Data Cleaning & Error Correction
Data anomalies were handled directly inside your memory layout using advanced defensive cleaning scripts:
*   Missing Durations Fixed: Imputed blank processing intervals safely using the calculated running system median to preserve math integrity.
*   Survey Normalization: Resolved missing satisfaction ratings by mapping unassigned parameters safely to a standard neutral indicator integer value (-1).
*   Casing Cleanups: Normalized erratic lowercase and aggressive text lines into clean, readable Title Case and Sentence Case strings across lookup records and helpdesk complaints.

```python
# Table 1: Imputing operational timeline gaps using Calculated System Median
duration_median = df_bookings['processing_duration_hours'].median()
df_bookings['processing_duration_hours'] = df_bookings['processing_duration_hours'].fillna(duration_median)

# Table 3: Cleaning unstandardized feedback text strings and mapping missing survey integers
df_cs['customer_satisfaction_score'] = df_cs['customer_satisfaction_score'].fillna(-1).astype(int)
df_cs['complaint_verdict_text'] = df_cs['complaint_verdict_text'].astype(str).str.strip().str.capitalize()

# Table 4: Enforcing pristine structural master casing and preserving key lookup entries
df_clients['company_name'] = df_clients['company_name'].astype(str).str.strip().str.title()
df_clients['client_tier'] = df_clients['client_tier'].astype(str).str.strip().str.title()
```

---

## 7 Feature Engineering
To move past raw variables and unlock strategic business visibility, high-impact feature attributes were built and written back directly into the production storage tier on disk:

```python
# Table 2: Engineering Revenue Leakage Risk and Service Value Priority Tiers
df_ancillaries['revenue_leakage_flag'] = np.where(
    df_ancillaries['billed_status'].str.contains('Leakage', case=False, na=False),
    'Leakage Risk',
    'Secure'
)

if df_ancillaries['quoted_price_usd'].dtype == 'object':
    price_num = pd.to_numeric(df_ancillaries['quoted_price_usd'].str.replace(r'[\$,]', '', regex=True), errors='coerce')
else:
    price_num = pd.to_numeric(df_ancillaries['quoted_price_usd'], errors='coerce')

df_ancillaries['service_priority_tier'] = np.where(price_num >= 300.00, 'High Value', 'Standard Value')
```

---

## 8 Core Business Analysis

### Objective 1: Optimize Omni-Channel Revenue & Margin Realization
Cross-tabulated transaction spend loops across front-end intake tracks (Online CBT vs. Offline Agent) and travel sectors to expose automated system configuration errors.

```python
# Aggregating transactional volume performance metrics and base fare pricing spreads
obj1_metrics = df_bookings.groupby(['booking_channel', 'travel_classification'])['base_fare_numeric'].agg(
    ['count', 'sum', 'mean', 'min', 'max']
).reset_index()
```

### Objective 2: Mitigate Transactional Revenue Leakage
Audited secondary value-added rows to isolate completed deliverables that bypassed primary invoicing engines. The analysis successfully exposed $55,101.18 in total unbilled cash losses.

```python
# Merging ancillary logs with bookings and master corporate directories to track leak paths
df_leaks_step1 = pd.merge(df_ancillaries, df_bookings[['booking_key', 'client_key']], on='booking_key', how='inner')
df_leakage_master = pd.merge(df_leaks_step1, df_clients[['client_key', 'company_name', 'client_tier']], on='client_key', how='inner')

total_leakage = df_leakage_master[df_leakage_master['revenue_leakage_flag'] == 'Leakage Risk']['quoted_price_numeric'].sum()
```

### Objective 3: Audit Service SLA Adherence & Workflow Velocity
Cross-referenced turnaround fulfillment times against strict contractual response windows. The analysis unmasked a severe 70.2% global workflow breach rate.

```python
# Evaluation of compliance metrics across negotiated contract hour limits
total_tickets = len(df_bookings)
total_breaches = (df_bookings['sla_status'] == 'SLA Breached').sum()
global_breach_rate = (total_breaches / total_tickets) * 100
```

### Objective 4: Uncover Customer Churn Triggers & Sentiment Drivers
Aggregated customer feedback categories side-by-side with survey rankings to isolate the top operational triggers causing customer dissatisfaction.

```python
# Tracking active feedback hotspots across customer service logs
sentiment_profiles = df_cs.groupby(['complaint_category', 'satisfaction_segment']).size().reset_index(name='incident_volume')
```

### Objective 5: Enhance Industry Market Competitiveness
Generated a full account product cross-sell penetration index across all five product pillars (Flights, Hotels, Visas, Protocol, Car Hire) to map product adoption depths horizontally.

```python
# Compiling the final account portfolio wallet-share penetration matrix
core_counts = df_bookings.groupby('company_name').agg(
    flights_issued=('is_flight', 'sum'),
    hotels_booked=('is_hotel', 'sum')
).reset_index()
```

---

## 9 Data Visualization & Interpretation
*   Omni-Channel Distribution Patterns: Grouped bar charts display total revenue weights across channels, showing that automated corporate portals handle the heaviest cash-flow loads.
*   Fulfillment Velocity Variance: Dual bar charts compare processing speeds side-by-side. They highlight that when fulfillment queues back up, processing times stretch up to ten times past your target window, putting premium relationships at risk.
*   Granular Leakage Demands: Horizontal bar charts visualize unbilled leakage risks, giving compliance auditors a clear list of target accounts to approach for immediate billing recovery.

---

## 10 Key Findings & Insights
*   Systemic Processing Bottlenecks: A 70.2% SLA breach rate points to a structural process failure, not isolated incident errors.
*   Concentrated Capital Leaks: Revenue leakage is not random; it is heavily concentrated in Airport Protocol and Car Hire requests handled for Gold spend tier clients.
*   Service Overlaps: Top corporate accounts show high flight volume activity but single-digit ancillary adoption, indicating that clients are splitting their wallet share with outside competitors.

---

## 11 Recommendations
1.  Enforce Mandatory Billing Checkpoints: Update the ancillary app interface to prevent agents from closing out visa files or car extensions without completing the billing workflow.
2.  Automate Premium SLA Queues: Implement a priority automated sorting layer for premium Platinum and Gold client accounts with tight 2-hour response windows to protect key corporate contracts.
3.  Launch Targeted Cross-Sell Campaigns: Equip corporate account managers with your cross-sell penetration matrix to run targeted multi-product bundle campaigns for clients using only flight services.

---

## 12 Conclusion
By turning noisy, multi-table transaction logs into an organized star-schema framework, this Python engine successfully provides clear operational visibility. It moves beyond standard technical metrics to answer critical business questions, giving corporate leaders the tools they need to protect revenue margins, fix process inefficiencies, and maximize market share.
