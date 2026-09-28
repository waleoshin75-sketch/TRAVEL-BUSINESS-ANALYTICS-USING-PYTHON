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
*   Survey Fragmentation: The customer survey logs contained widespread missing satisfaction values coded arbitrarily.
*   Casing Volatility: Text fields across lookups and traveler comments suffered from erratic uppercase casing.

---

## 4 Data Transformation
The ETL pipeline isolates text string noise dynamically using Python. It uses Regular Expressions (Regex) to strip out complex currency symbols and forces variables into high-precision numerical float formats. Erratic, mixed-format regional datetime markers were programmatically parsed into a standardized UTC Chronological Object Map to establish a reliable baseline for time-intelligence reporting.

---

## 5 Data Modeling & Relational Schema
The database engine links independent tracking logs into a unified relational schema using clear primary and foreign key pathways:
*   dim_corporate_clients.client_key (PK) connects directly to core bookings and helpdesk registries.
*   fact_travel_bookings.booking_key (PK) acts as the operational root node linking secondary value-added service lines back to the primary umbrella transactions.

---

## 6 Data Cleaning & Error Correction
Data anomalies were handled directly inside your memory layout using advanced defensive cleaning scripts:
*   Missing Durations Fixed: Found NaN fields inside fulfillment timeline arrays. These gaps were resolved by calculating the system baseline median processing duration and applying an in-place imputation step to protect calculation accuracy.
*   Survey Normalization: Resolved missing satisfaction ratings by mapping unassigned parameters safely to a standard neutral indicator integer value (-1).
*   Casing Cleanups: Normalized erratic lowercase and aggressive text lines into clean, readable Title Case and Sentence Case strings across lookup records and helpdesk complaints.

---

## 7 Feature Engineering
To move past raw variables and unlock strategic business visibility, four high-impact feature attributes were built:
*   total_gross_cost: A clean, structured currency variable representing true inventory expenses.
*   revenue_leakage_flag: Evaluates unbilled workflows to instantly tag items as Leakage Risk or Secure.
*   operational_urgency_tier: Evaluates negotiated service limits to group client files into High Urgency vs. Standard queues.
*   account_health_index: Combines corporate tiers and urgency indicators into a combined account management tracker flag.

---

## 8 Core Business Analysis

### Objective 1: Optimize Omni-Channel Revenue & Margin Realization
Cross-tabulated transaction spend loops across your front-end intake tracks (Online CBT vs. Offline Agent) and travel sectors. The analysis unmasked key distribution fields, pinpointing exactly where corporate bookings are heavily concentrated and highlighting system pricing configuration anomalies.

### Objective 2: Mitigate Transactional Revenue Leakage
Audited secondary value-added rows to isolate completed deliverables that bypassed primary invoicing engines. The analysis successfully exposed $55,101.18 in total unbilled cash losses, with Zeta Financial Services ranking as the highest-risk account holding outstanding capital.

### Objective 3: Audit Service SLA Adherence & Workflow Velocity
Cross-referenced turnaround fulfillment times against strict contractual response windows. The analysis unmasked a severe 70.2% global workflow breach rate, proving that delayed or bottlenecked ticketing desks routinely slide to over 18 to 20+ processing hours.

### Objective 4: Uncover Customer Churn Triggers & Sentiment Drivers
Aggregated customer feedback categories side-by-side with survey rankings. The script isolated Flight Delay Protocol Failure (46 active logs) and Overbilling Request Dispute (42 active logs) as the top operational triggers causing customer dissatisfaction.

### Objective 5: Enhance Industry Market Competitiveness
Generated a full account product cross-sell penetration index across all five product pillars (Flights, Hotels, Visas, Protocol, Car Hire). This matrix maps product adoption rates horizontally to help sales teams spot multi-product cross-sell white spaces.

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
