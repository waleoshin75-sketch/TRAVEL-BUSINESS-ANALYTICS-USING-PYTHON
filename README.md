# Travel Business Analytics Using Python

## Enterprise TMC Business Analytics Engine (2026)

### End-to-End Relational Data Pipeline, Revenue Leakage Auditing, and Workflow SLA Diagnostics Using Python and Pandas

*Optimizing omni-channel revenue, mitigating transactional leakage, and auditing operational SLA workflow velocity.*

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Dataset Overview](#2-dataset-overview)
3. [Data Quality Assessment](#3-data-quality-assessment)
4. [Data Transformation](#4-data-transformation)
5. [Data Modeling and Relational Schema](#5-data-modeling-and-relational-schema)
6. [Data Cleaning and Error Correction](#6-data-cleaning-and-error-correction)
7. [Feature Engineering](#7-feature-engineering)
8. [Core Business Analysis](#8-core-business-analysis)
   - [Objective 1: Optimize Omni-Channel Revenue and Margin Realization](#objective-1-optimize-omni-channel-revenue-and-margin-realization)
   - [Objective 2: Mitigate Transactional Revenue Leakage](#objective-2-mitigate-transactional-revenue-leakage)
   - [Objective 3: Audit Service SLA Adherence and Workflow Velocity](#objective-3-audit-service-sla-adherence-and-workflow-velocity)
   - [Objective 4: Uncover Customer Churn Triggers and Sentiment Drivers](#objective-4-uncover-customer-churn-triggers-and-sentiment-drivers)
   - [Objective 5: Enhance Industry Market Competitiveness](#objective-5-enhance-industry-market-competitiveness)
9. [Data Visualization and Interpretation](#9-data-visualization-and-interpretation)
10. [Key Findings and Insights](#10-key-findings-and-insights)
11. [Recommendations](#11-recommendations)
12. [Conclusion](#12-conclusion)

---

## 1. Project Overview

A travel management company earns money in more places than most people realize. Flights and hotels bring in the headline revenue, but visas, protocol services, and car hire quietly add up, and each one can leak cash if it is never billed. On top of that, corporate clients expect fast turnaround, and every missed deadline puts a contract at risk.

This project builds a full analytics pipeline in Python and Pandas to answer five business questions. It audits four raw tables, cleans and connects them, engineers new features, and then measures revenue, leakage, service speed, customer sentiment, and cross-sell reach. Every step below shows the exact code used.

**Tools used:** Python, Pandas, NumPy, Jupyter Notebook

```python
import pandas as pd
import numpy as np
```

---

## 2. Dataset Overview

The project draws on four source tables that together describe bookings, add-on services, support tickets, and the corporate clients behind them.

| Table | What It Holds | Key Columns |
| --- | --- | --- |
| fact_travel_bookings | Core flight and hotel bookings with fares, channels, and processing timestamps | booking_key, client_key, base_fare_usd, taxes_and_fees, booking_channel, request_timestamp, issuance_timestamp |
| fact_ancillary_services | Add-on services sold alongside a booking, such as visas and car hire | service_id, booking_key, service_type, quoted_price_usd, billed_status, service_date |
| fact_customer_service | Helpdesk tickets with complaint details and satisfaction ratings | ticket_id, client_key, complaint_category, resolution_status, customer_satisfaction_score, complaint_verdict_text |
| dim_corporate_clients | Master directory of corporate accounts | client_key, corporate_id, company_name, client_tier, preferred_channel, contract_sla_hours |

The three fact tables each connect back to the client directory, which sits at the center of the model.

---

## 3. Data Quality Assessment

Before changing anything, each table was audited on its own. Every audit checks size, exact duplicates, and missing values, then adds one check aimed at the specific risk in that table: text contamination for bookings, unbilled services for ancillaries, SLA breaches for support tickets, and messy casing for clients.

### Table 1: Travel Bookings

```python
print("Running Targeted Audit ONLY on Table: df_bookings\n")

# 1. Check dimensions and basic statistics
print(f"• Total Rows and Columns: {df_bookings.shape}")
print(f"• Total Exact Duplicate Rows: {df_bookings.duplicated().sum()}")

# 2. Check for missing values in this specific table
print(f"• Missing Values Count:\n{df_bookings.isnull().sum()}\n")

# 3. Print the data types of each column to expose text contamination
print("• Column Data Types (Checking for object/text contamination):")
print(df_bookings.dtypes)
```

Columns that should be numbers or dates but show up as text are the giveaway here. That usually means currency symbols or mixed date formats are hiding inside the data.

### Table 2: Ancillary Services

```python
print("TARGETED AUDIT: Table 2 - df_ancillaries\n")

# 1. Structural Dimensions & Duplicates Check
print(f"• Grid Dimensions (Rows, Columns): {df_ancillaries.shape}")
print(f"• Exact Duplicate Rows: {df_ancillaries.duplicated().sum()}")

# 2. Check for Missing Value Nodes
print(f"\n• Missing Values Check:\n{df_ancillaries.isnull().sum()}")

# 3. Expose Revenue Leakage in Billed Status
print("\n• Distribution of Billed Status (Checking for Revenue Leakage):")
print(df_ancillaries['billed_status'].value_counts())
```

The billed status breakdown gives an early read on how many services were delivered but never charged.

### Table 3: Customer Service Logs

```python
print("TARGETED AUDIT: Table 3 - df_cs (Customer Service Logs)\n")

# 1. Structural Dimensions & Duplicates Check
print(f"• Grid Dimensions (Rows, Columns): {df_cs.shape}")
print(f"• Exact Duplicate Rows: {df_cs.duplicated().sum()}")

# 2. Expose the Volume of Missing Satisfaction Ratings
print(f"\n• Missing Values Count:\n{df_cs.isnull().sum()}")

# 3. Audit SLA Breaches
print("\n• Helpdesk Ticket Status (Checking for SLA Breaches):")
print(df_cs['resolution_status'].value_counts())
```

This audit shows how many customers never left a rating and how tickets are split across resolved, pending, and breached states.

### Table 4: Corporate Master Directory

```python
print("TARGETED AUDIT: Table 4 - df_clients (Corporate Master Directory)\n")

# 1. Structural Dimensions & Duplicates Check
print(f"• Grid Dimensions (Rows, Columns): {df_clients.shape}")
print(f"• Exact Duplicate Rows: {df_clients.duplicated().sum()}")

# 2. Check for Text Casing Noise inside Corporate Names
print("\n• Raw Company Names (Exposing Mixed-Case Structural Issues):")
print(df_clients['company_name'].to_string(index=False))

# 3. Check for Inconsistent ID Channel Definitions
print("\n• Distribution of Preferred Booking Channels:")
print(df_clients['preferred_channel'].value_counts())
```

Mixed casing in company names and channel labels can split one client into several groups, so it is worth catching before any grouping happens.

---

## 4. Data Transformation

This phase turns the three supporting tables into clean production files. Each one starts from a protected copy of the raw data, so the original audit state is never overwritten. The bookings table is handled later in the cleaning phase, because it needs client details from the relational model first.

### Phase 1: Travel Booking Ledger

```python
print("PIPELINE PHASE 1: Processing Core Travel Bookings Transactions Ledger...\n")

# 1. Deep-copy raw memory state to protect original audit integrity
df_bookings_clean = df_bookings.copy()

# A. Regex Currency Scrubbing: Wipe out $, commas, text tags, and padding spaces
print("Stripping currency text flags from 'base_fare_usd'...")
df_bookings_clean['base_fare_usd_clean'] = (
    df_bookings_clean['base_fare_usd']
    .astype(str)
    .str.replace(r'[\$,\s]', '', regex=True)
    .str.replace('USD', '', case=False)
)
df_bookings_clean['base_fare_usd_clean'] = pd.to_numeric(df_bookings_clean['base_fare_usd_clean'], errors='coerce')

# B. Text Category Standardization: Enforce uniform Title Case across channels
print("Standardizing 'booking_channel' text casing splits...")
df_bookings_clean['booking_channel_clean'] = (
    df_bookings_clean['booking_channel']
    .astype(str)
    .str.strip()
    .str.lower()
    .str.replace('online cbt', 'Online CBT', case=False)
    .str.replace('online portal', 'Online Portal', case=False)
    .str.replace('offline walk-in', 'Offline Walk-In', case=False)
    .str.replace('offline', 'Offline Agent', case=False)
    .str.replace('email request', 'Email Request', case=False)
)

# C. Uniform Chronological Parsing: Force mixed datetime string structures to UTC format
print("Synchronizing mixed timestamp date profiles...")
df_bookings_clean['request_timestamp_clean'] = pd.to_datetime(df_bookings_clean['request_timestamp'], errors='coerce', utc=True)
df_bookings_clean['issuance_timestamp_clean'] = pd.to_datetime(df_bookings_clean['issuance_timestamp'], errors='coerce', utc=True)

# D. Feature Engineering: Calculate Workflow Turnaround Time (TAT Velocity Durations)
print("Calculating fulfillment turnaround times in hours (TAT)...")
df_bookings_clean['processing_duration_hours'] = (
    (df_bookings_clean['issuance_timestamp_clean'] - df_bookings_clean['request_timestamp_clean'])
    .dt.total_seconds() / 3600.0
)

# E. Finalize Production Schema Layer: Drop contaminated columns and rename clean ones
df_bookings_prod = df_bookings_clean.drop(
    columns=['base_fare_usd', 'booking_channel', 'request_timestamp', 'issuance_timestamp']
).rename(
    columns={
        'base_fare_usd_clean': 'base_fare_usd',
        'booking_channel_clean': 'booking_channel',
        'request_timestamp_clean': 'request_timestamp',
        'issuance_timestamp_clean': 'issuance_timestamp'
    }
)

# F. Production Migration Export: Preserving all headers seamlessly
output_file = 'prod_travel_bookings.csv'
df_bookings_prod.to_csv(output_file, index=False)

print(f"\nSUCCESS: Production dataset successfully generated and saved as '{output_file}'!")
print(f"Total Verified Clean Rows: {len(df_bookings_prod)}")

# Display data verification matrix
df_bookings_prod[['booking_id', 'booking_channel', 'base_fare_usd', 'processing_duration_hours']].head(5)
```
Stripping currency text flags from 'base_fare_usd', Standardizing 'booking_channel' text casing splits, Synchronizing mixed timestamp date profiles, Calculating fulfillment turnaround times in hours (TAT)

### Phase 2: Ancillary Operations Ledger

```python
print("PIPELINE PHASE 2: Processing Supplementary Ancillary Operations Ledger...\n")

# 1. Deep-copy raw memory state to protect original audit integrity
df_ancillaries_clean = df_ancillaries.copy()

# A. Uniform Chronological Parsing: Synchronize mixed datetime string structures into uniform UTC format
print("Synchronizing mixed timestamp date profiles for 'service_date'...")
df_ancillaries_clean['service_date_clean'] = pd.to_datetime(df_ancillaries_clean['service_date'], errors='coerce', utc=True)

# B. Finalize Production Schema Layer: Drop the old messy column and rename the clean one
df_ancillaries_prod = df_ancillaries_clean.drop(columns=['service_date']).rename(
    columns={'service_date_clean': 'service_date'}
)

# C. Production Migration Export: Preserving all headers seamlessly
output_file = 'prod_ancillary_services.csv'
df_ancillaries_prod.to_csv(output_file, index=False)

print(f"\nSUCCESS: Production dataset successfully generated and saved as '{output_file}'!")
print(f"Total Verified Clean Rows: {len(df_ancillaries_prod)}")

# Display data verification matrix
df_ancillaries_prod[['service_id', 'booking_key', 'service_type', 'quoted_price_usd', 'billed_status', 'service_date']].head(5)
```

Mixed date formats in the service date column are converted into one consistent timestamp format, and any value that cannot be read becomes a clean missing value instead of breaking the run.

### Phase 3: Customer Service Logs and Complaint Registry

```python
print("PIPELINE PHASE 3: Processing Customer Service Logs & Complaint Registry...\n")

# 1. Deep-copy raw memory state to protect original audit integrity
df_cs_clean = df_cs.copy()

# A. Missing Value Imputation: Standardize blank satisfaction entries to an explicit unrated flag (-1)
print("Imputing missing satisfaction metrics with an explicit flag (-1)...")
df_cs_clean['customer_satisfaction_score_clean'] = df_cs_clean['customer_satisfaction_score'].fillna(-1).astype(int)

# B. Text Normalization: Strip raw whitespaces and convert conversational text strings to Sentence case
print("Standardizing raw complaint verdict text and casing structures...")
df_cs_clean['complaint_verdict_text_clean'] = (
    df_cs_clean['complaint_verdict_text']
    .astype(str)
    .str.strip()
    .str.capitalize()
)

# C. Finalize Production Schema Layer: Drop noisy raw attributes and rename clean ones
df_cs_prod = df_cs_clean.drop(columns=['customer_satisfaction_score', 'complaint_verdict_text']).rename(
    columns={
        'customer_satisfaction_score_clean': 'customer_satisfaction_score',
        'complaint_verdict_text_clean': 'complaint_verdict_text'
    }
)

# D. Production Migration Export: Preserving all headers seamlessly
output_file = 'prod_customer_service.csv'
df_cs_prod.to_csv(output_file, index=False)

print(f"\nSUCCESS: Production dataset successfully generated and saved as '{output_file}'!")
print(f"Total Verified Clean Rows: {len(df_cs_prod)}")

# Display data verification matrix
df_cs_prod[['ticket_id', 'complaint_category', 'resolution_status', 'customer_satisfaction_score', 'complaint_verdict_text']].head(5)
```

Missing satisfaction ratings are replaced with a clear unrated flag of negative one, so blank entries are never confused with a real score. Complaint text is trimmed and set to sentence case.

### Phase 4: Corporate Clients Master Directory

```python
print("PIPELINE PHASE 4: Processing Corporate Clients Master Dimension Directory...\n")

# 1. Deep-copy raw memory state to protect original audit integrity
df_clients_clean = df_clients.copy()

# A. Casing Standardization: Transform scrambled name inputs into clean Title Case
print("Standardizing 'company_name' text casing into uniform Title Case...")
df_clients_clean['company_name_clean'] = df_clients_clean['company_name'].astype(str).str.strip().str.title()

# B. Finalize Production Schema Layer: Drop noisy raw attribute and rename clean one
df_clients_prod = df_clients_clean.drop(columns=['company_name']).rename(
    columns={'company_name_clean': 'company_name'}
)

# C. Production Migration Export: Preserving all headers seamlessly
output_file = 'prod_corporate_clients.csv'
df_clients_prod.to_csv(output_file, index=False)

print(f"\nSUCCESS: Production dimension directory successfully generated and saved as '{output_file}'!")
print(f"Total Corporate Profiles Processed: {len(df_clients_prod)}")

# Display data verification matrix
df_clients_prod[['corporate_id', 'company_name', 'client_tier', 'preferred_channel', 'contract_sla_hours']]
```

Company names are trimmed and converted to title case, so each client appears exactly once in every later grouping.

---

## 5. Data Modeling and Relational Schema

With the source tables understood, the next step was connecting them into a star schema. The client directory is the central dimension, and the bookings and customer service tables link to it through the client key. The ancillary table links to the bookings table through the booking key. This model is built from the raw source files so that cleaning can be applied cleanly on top of it.

```python
print("🔗 INITIALIZING UNCLEANED STAR SCHEMA: Mapping Relational Primary & Foreign Keys...")

# 1. Ensure we are reading directly from your raw, completely uncleaned source datasets
df_bookings_raw = pd.read_csv('fact_travel_bookings.csv')
df_clients_raw = pd.read_csv('dim_corporate_clients.csv')
df_ancillaries_raw = pd.read_csv('fact_ancillary_services.csv')
df_cs_raw = pd.read_csv('fact_customer_service.csv')

# =========================================================================
# SCHEMA RELATIONSHIP 1: Core Fact Bookings to Client Profile Dimension Lookup
# Primary Key [PK]: dim_corporate_clients.client_key
# Foreign Key [FK]: fact_travel_bookings.client_key
# =========================================================================
print("Establishing Relationship: fact_travel_bookings [FK] -> dim_corporate_clients [PK]...")
df_bookings_relational = pd.merge(
    df_bookings_raw,
    df_clients_raw,
    on='client_key',
    how='inner'
)

# =========================================================================
# SCHEMA RELATIONSHIP 2: Fact Ancillary Services to Core Fact Bookings Ledger
# Primary Key [PK]: fact_travel_bookings.booking_key
# Foreign Key [FK]: fact_ancillary_services.booking_key
# =========================================================================
print("Establishing Relationship: fact_ancillary_services [FK] -> fact_travel_bookings [PK]...")
df_ancillaries_relational = pd.merge(
    df_ancillaries_raw,
    df_bookings_raw,
    on='booking_key',
    how='inner',
    suffixes=('_ancillary', '_booking')
)

# =========================================================================
# SCHEMA RELATIONSHIP 3: Fact Customer Service Logs to Client Profile Dimension Lookup
# Primary Key [PK]: dim_corporate_clients.client_key
# Foreign Key [FK]: fact_customer_service.client_key
# =========================================================================
print("Establishing Relationship: fact_customer_service [FK] -> dim_corporate_clients [PK]...\n")
df_cs_relational = pd.merge(
    df_cs_raw,
    df_clients_raw,
    on='client_key',
    how='inner'
)

print("SUCCESS: Core Star Schema Data Model successfully mapped in memory!")
print(f"• Integrated Relational Bookings Table Structure Dimensions:  {df_bookings_relational.shape}")
print(f"• Integrated Relational Ancillaries Table Structure Dimensions: {df_ancillaries_relational.shape}")
print(f"• Integrated Relational Customer Service Table Dimensions:     {df_cs_relational.shape}")
```

Each link uses an inner merge, so a record only survives if it has a matching parent. That quietly removes orphaned rows and gives every downstream table client details such as company name and tier.

---

## 6. Data Cleaning and Error Correction

Cleaning now runs on the relational tables, so client details come along for the ride. Three fixes happen here: money stored as text becomes real numbers, text casing is standardized, and timestamps are parsed so processing time can be measured.

###

```python
print("REINSTATING TAXES: Generating the Perfect Clutter-Free Production Dataset...")

# 1. Reset completely using the original raw data source variables
df_bookings_production = df_bookings_relational.copy()

# A. Clean and format the base_fare_usd column in-place (No duplicate display columns)
df_bookings_production['base_fare_usd'] = (
    df_bookings_production['base_fare_usd']
    .astype(str)
    .str.replace(r'[\$,\s]', '', regex=True)
    .str.replace('USD', '', case=False)
)
df_bookings_production['base_fare_usd'] = pd.to_numeric(df_bookings_production['base_fare_usd'], errors='coerce')
df_bookings_production['base_fare_usd'] = df_bookings_production['base_fare_usd'].apply(lambda x: f"${x:,.2f}" if pd.notnull(x) else np.nan)

# B. REINSTATE & CLEAN: Parse taxes_and_fees and format in-place with a uniform '$' symbol
df_bookings_production['taxes_and_fees'] = pd.to_numeric(df_bookings_production['taxes_and_fees'], errors='coerce')
df_bookings_production['taxes_and_fees'] = df_bookings_production['taxes_and_fees'].apply(lambda x: f"${x:,.2f}" if pd.notnull(x) else np.nan)

# C. Normalize the booking channel text casing in-place
df_bookings_production['booking_channel'] = df_bookings_production['booking_channel'].astype(str).str.strip().str.title()

# D. Parse regional timestamp configurations and compute turnaround hours (TAT)
df_bookings_production['request_timestamp'] = pd.to_datetime(df_bookings_production['request_timestamp'], errors='coerce', utc=True)
df_bookings_production['issuance_timestamp'] = pd.to_datetime(df_bookings_production['issuance_timestamp'], errors='coerce', utc=True)
df_bookings_production['processing_duration_hours'] = (
    (df_bookings_production['issuance_timestamp'] - df_bookings_production['request_timestamp'])
    .dt.total_seconds() / 3600.0
)

# E. Save clean production asset directly to your local computer folder directory
output_file = 'prod_travel_bookings.csv'
df_bookings_production.to_csv(output_file, index=False)

print(f"\nSUCCESS: Reinstated asset generated and saved as '{output_file}'!")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Show exactly 10 rows to verify the perfect table structure
df_bookings_production.head(5)
```

This fixes corrupted currency formatting by stripping out messy symbols and spaces from the transaction values. It also standardizes inconsistent text casing and fixes chaotic regional time stamps to accurately calculate processing duration metrics.

### Travel Bookings

```python
print("RE-RUNNING PHASE 2: Reinstating Missing Client Descriptors to Table 1...")

# 1. Reset completely using the original raw relational cache variables
df_bookings_production = df_bookings_relational.copy()

# A. Clean and format the base_fare_usd column in-place (No duplicate display columns)
df_bookings_production['base_fare_usd'] = (
    df_bookings_production['base_fare_usd']
    .astype(str)
    .str.replace(r'[\$,\s]', '', regex=True)
    .str.replace('USD', '', case=False)
)
df_bookings_production['base_fare_usd'] = pd.to_numeric(df_bookings_production['base_fare_usd'], errors='coerce')
df_bookings_production['base_fare_usd'] = df_bookings_production['base_fare_usd'].apply(lambda x: f"${x:,.2f}" if pd.notnull(x) else np.nan)

# B. Clean and format taxes_and_fees in-place with a uniform '$' symbol
df_bookings_production['taxes_and_fees'] = (
    df_bookings_production['taxes_and_fees']
    .astype(str)
    .str.replace(r'[\$,\s]', '', regex=True)
)
df_bookings_production['taxes_and_fees'] = pd.to_numeric(df_bookings_production['taxes_and_fees'], errors='coerce')
df_bookings_production['taxes_and_fees'] = df_bookings_production['taxes_and_fees'].apply(lambda x: f"${x:,.2f}" if pd.notnull(x) else np.nan)

# C. Normalize the booking channel text casing in-place
df_bookings_production['booking_channel'] = df_bookings_production['booking_channel'].astype(str).str.strip().str.title()

# D. Parse regional timestamp configurations and compute turnaround hours (TAT)
df_bookings_production['request_timestamp'] = pd.to_datetime(df_bookings_production['request_timestamp'], errors='coerce', utc=True)
df_bookings_production['issuance_timestamp'] = pd.to_datetime(df_bookings_production['issuance_timestamp'], errors='coerce', utc=True)
df_bookings_production['processing_duration_hours'] = (
    (df_bookings_production['issuance_timestamp'] - df_bookings_production['request_timestamp'])
    .dt.total_seconds() / 3600.0
)

# E. CRITICAL PRESERVATION STEP: Explicitly retain client descriptors for analytical joins
# We drop ONLY uncleaned raw date parameters to keep the final template neat
df_bookings_production = df_bookings_production.drop(columns=['request_timestamp', 'issuance_timestamp'])

# F. Production Migration Export: Save directly to your local computer folder directory
output_file = 'prod_travel_bookings.csv'
df_bookings_production.to_csv(output_file, index=False)

print(f"\nSUCCESS: Pristine production asset generated and saved as '{output_file}'!")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Show exactly 10 rows to verify that company_name is safely present in the schema
df_bookings_production.head(5)
```

Currency symbols, commas, and the letters USD are stripped from fares and taxes, converted to numbers, then formatted uniformly. The gap between request and issuance is turned into processing hours, which powers the SLA analysis later.

### Customer Service Logs

```python
print("PIPELINE PHASE 3: Processing Customer Service Logs In-Place...")

# 1. Reset completely using the original raw data source variables
df_cs_production = df_cs_relational.copy()

# A. Missing Value Imputation In-Place: Convert satisfaction scores to integers and impute blanks with -1
df_cs_production['customer_satisfaction_score'] = df_cs_production['customer_satisfaction_score'].fillna(-1).astype(int)

# B. Text Normalization In-Place: Convert aggressive screaming customer comments to standard Sentence Case
df_cs_production['complaint_verdict_text'] = (
    df_cs_production['complaint_verdict_text']
    .astype(str)
    .str.strip()
    .str.capitalize()
)

# C. Normalize Structural Category Strings In-Place
df_cs_production['complaint_category'] = df_cs_production['complaint_category'].astype(str).str.strip().str.title()
df_cs_production['resolution_status'] = df_cs_production['resolution_status'].astype(str).str.strip().str.title()

# D. Normalize Inherited Dimension Profile Strings In-Place
df_cs_production['company_name'] = df_cs_production['company_name'].astype(str).str.strip().str.title()
df_cs_production['client_tier'] = df_cs_production['client_tier'].astype(str).str.strip().str.title()
df_cs_production['preferred_channel'] = df_cs_production['preferred_channel'].astype(str).str.strip().str.title()

# E. Production Migration Export: Save directly to your local workspace folder with headers preserved
output_file = 'prod_customer_service.csv'
df_cs_production.to_csv(output_file, index=False)

print(f"\nSUCCESS: Pristine production asset generated and saved as '{output_file}'!")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Display exactly the first 10 rows to verify a flawless table layout
df_cs_production.head(5)
```

This pass repeats the satisfaction and text fixes on the relational version of the table, and also standardizes the client details that came across from the directory, so every label is consistent.

### Corporate Clients Directory

```python
print("PIPELINE PHASE 4: 'dim_corporate_clients' Data Cleaning...")

# 1. Read directly from your raw, original source dataset file
df_clients_raw = pd.read_csv('dim_corporate_clients.csv')

# 2. Create an active development copy to ensure structural protection
df_clients_production = df_clients_raw.copy()

# A. Casing Standardization In-Place: Transform company names into crisp Title Case
df_clients_production['company_name'] = df_clients_production['company_name'].astype(str).str.strip().str.title()

# B. Normalize Category & Channel Operations Casing In-Place
df_clients_production['client_tier'] = df_clients_production['client_tier'].astype(str).str.strip().str.title()
df_clients_production['preferred_channel'] = df_clients_production['preferred_channel'].astype(str).str.strip().str.title()

# C. CRITICAL KEY VERIFICATION: Ensure primary identifiers are fully retained
# We explicitly keep: client_key, corporate_id, contract_sla_hours, and our clean fields
df_clients_production = df_clients_production[[
    'client_key', 
    'corporate_id', 
    'company_name', 
    'client_tier', 
    'preferred_channel', 
    'contract_sla_hours'
]]

# D. Production Migration Export: Overwrite your local production file with headers preserved
output_file = 'prod_corporate_clients.csv'
df_clients_production.to_csv(output_file, index=False)

print(f"\nSUCCESS: Pristine corporate directory generated and saved as '{output_file}'!")
print(f"Total Corporate Profiles Verified: {len(df_clients_production)}")

# Display the absolute full table layout to confirm keys are beautifully intact
df_clients_production.head(5)
```

The final pass locks in consistent casing and explicitly keeps the client key, corporate identifier, and SLA hours, so no identifier is lost before the analysis stage.

---

## 7. Feature Engineering

Cleaning fixes what is wrong. Feature engineering adds what was never there. Each production table gets two new columns that turn raw fields into flags and segments a business team can act on. The processing hours metric for bookings was already created during cleaning.

### Travel Bookings:

```python
print("EXECUTION MASTER PIPELINE: Upgrading Table 1 & Imputing Durations...\n")

# 1. Read your active production dataset directly from your local directory folder
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')

# FEATURE 1: Gross Financial Cost Realization (In-Place)
# Logic: Strips currency symbols, cleans numeric layers, and formats seamlessly
print("Engineering Feature 1: 'total_gross_cost'...")
base_num = pd.to_numeric(df_bookings_prod['base_fare_usd'].str.replace(r'[\$,]', '', regex=True), errors='coerce')
df_bookings_prod['total_gross_cost'] = base_num.apply(lambda x: f"${x:,.2f}" if pd.notnull(x) else np.nan)

# FEATURE 2: Structural Travel Segment Classification (In-Place)
# Logic: Maps category sub-types into explicit high-level business divisions
print("Engineering Feature 2: 'travel_classification'...")
df_bookings_prod['travel_classification'] = np.where(
    df_bookings_prod['travel_type'].str.contains('Flight', case=False, na=False),
    'Flight',
    'Lodging'
)

# FEATURE 3: Missing Value Imputation (In-Place)
# Logic: Calculates system median and completely overwrites NaN duration nodes
print("Checking for missing duration nodes...")
# Compute the mathematical median duration across all valid transaction records
median_duration = df_bookings_prod['processing_duration_hours'].median()
print(f"  └── Calculated Baseline System Median Processing Time: {median_duration:.1f} Hours")

# Fill missing NaN elements inside processing hours using your median baseline
df_bookings_prod['processing_duration_hours'] = df_bookings_prod['processing_duration_hours'].fillna(median_duration)

# FEATURE 4: Operational Contract SLA Status Profiling (In-Place Update)
# Logic: Dynamically re-evaluates and resets status mapping to fix errors
print("Engineering Feature 4: Recalculating 'sla_status' thresholds...\n")
df_bookings_prod['sla_status'] = np.where(
    df_bookings_prod['processing_duration_hours'] <= df_bookings_prod['contract_sla_hours'],
    'Within SLA',
    'SLA Breached'
)

# 2. In-Place Production Save: Overwrite file to incorporate your missing data fixes
output_file = 'prod_travel_bookings.csv'
df_bookings_prod.to_csv(output_file, index=False)

print(f"SUCCESS: Master engineered production asset updated and saved to '{output_file}'!")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Display exactly the first 10 rows to verify that all NaN nodes have been successfully removed
df_bookings_prod[['booking_id', 'travel_type', 'travel_classification', 'base_fare_usd', 'total_gross_cost', 'processing_duration_hours', 'sla_status']].head(5)
```

This fixes missing data holes and formatting errors by cleaning price strings and filling empty duration gaps with the system median.

### Ancillary Services: Leakage Flag and Value Tier

```python
print("FEATURE ENGINEERING PHASE: Upgrading Table 2 Production Schema In-Place...\n")

# 1. Load your clean production ancillary asset from disk
df_ancillaries_prod = pd.read_csv('prod_ancillary_services.csv')

# FEATURE 1: Revenue Leakage Operational Flagging
# Logic: Dynamically tags unbilled transactions to instantly isolate leakage vulnerabilities
print("Engineering Feature 1: 'revenue_leakage_flag'...")
df_ancillaries_prod['revenue_leakage_flag'] = np.where(
    df_ancillaries_prod['billed_status'].str.contains('Leakage', case=False, na=False),
    'Leakage Risk',
    'Secure'
)

# FEATURE 2: Operational Service Value Tier Segmentations
# Logic: Safely converts values to numeric floats depending on data state, then buckets them
print("Engineering Feature 2: 'service_priority_tier'...\n")

# Defensive Conversion: Check if the data type is string/text before applying string replacements
if df_ancillaries_prod['quoted_price_usd'].dtype == 'object':
    price_numeric = pd.to_numeric(df_ancillaries_prod['quoted_price_usd'].str.replace(r'[\$,]', '', regex=True), errors='coerce')
else:
    # If it is already floating numbers, directly use the column as-is without crashing
    price_numeric = pd.to_numeric(df_ancillaries_prod['quoted_price_usd'], errors='coerce')

df_ancillaries_prod['service_priority_tier'] = np.where(
    price_numeric >= 300.00,
    'High Value',
    'Standard Value'
)

# 2. In-Place Production Save: Overwrite file to fully incorporate engineered columns
output_file = 'prod_ancillary_services.csv'
df_ancillaries_prod.to_csv(output_file, index=False)

print(f"SUCCESS: Engineered production features successfully generated and saved to '{output_file}'!")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Display exactly the first 10 rows to verify the pristine, upgraded table structure
df_ancillaries_prod[['service_id', 'service_type', 'quoted_price_usd', 'billed_status', 'revenue_leakage_flag', 'service_priority_tier']].head(5)
```

Every service that shows a leakage status is tagged as a leakage risk, and every service quoted at three hundred dollars or more is tagged high value. Together they show where the biggest unbilled money sits.

### Customer Service: Satisfaction Segment and Escalation Flag

```python
print("FEATURE ENGINEERING PHASE: Upgrading Table 3 Production Schema In-Place...\n")

# 1. Load your clean production customer service asset from disk
df_cs_prod = pd.read_csv('prod_customer_service.csv')

# FEATURE 1: Strategic Customer Satisfaction Segmentation
# Logic: Segments satisfaction integers into distinct, boardroom-ready client sentiment blocks
print("Engineering Feature 1: 'satisfaction_segment'...")
conditions = [
    (df_cs_prod['customer_satisfaction_score'] >= 4),
    (df_cs_prod['customer_satisfaction_score'] >= 1) & (df_cs_prod['customer_satisfaction_score'] <= 2),
    (df_cs_prod['customer_satisfaction_score'] == 3),
    (df_cs_prod['customer_satisfaction_score'] == -1)
]
choices = ['Highly Satisfied', 'Dissatisfied', 'Neutral', 'Unrated']

df_cs_prod['satisfaction_segment'] = np.select(conditions, choices, default='Neutral')

# FEATURE 2: Text-Driven Critical Complaint Escalation Flagging
# Logic: Audits text strings for core liability indicators to prioritize support response queues
print("Engineering Feature 2: 'critical_complaint_flag'...\n")

# Build a regex matching pattern for high-risk corporate keyword triggers
leakage_keywords = r'overcharge|crash|missed|billing|error|fail'

# Combine category and comment text fields to scan for keywords globally
combined_text_layer = (df_cs_prod['complaint_category'].astype(str) + " " + df_cs_prod['complaint_verdict_text'].astype(str)).str.lower()

df_cs_prod['critical_complaint_flag'] = np.where(
    combined_text_layer.str.contains(leakage_keywords, regex=True, na=False),
    'High Priority Escalation',
    'Standard'
)

# 2. In-Place Production Save: Overwrite file to fully incorporate engineered columns
output_file = 'prod_customer_service.csv'
df_cs_prod.to_csv(output_file, index=False)

print(f"SUCCESS: Engineered production features successfully generated and saved to '{output_file}'!")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Display exactly the first 10 rows to verify the pristine, upgraded table structure
df_cs_prod[['ticket_id', 'complaint_category', 'customer_satisfaction_score', 'satisfaction_segment', 'critical_complaint_flag']].head(5)
```

Scores are grouped into four sentiment segments: highly satisfied, neutral, dissatisfied, and unrated. Complaints that mention words like overcharge, crash, missed, billing, error, or fail are flagged for priority escalation.

### Corporate Clients: Urgency Tier and Account Health

```python
print("FEATURE ENGINEERING PHASE: Upgrading Table 4 Production Schema In-Place...\n")

# 1. Load your clean production corporate clients directory asset from disk
df_clients_prod = pd.read_csv('prod_corporate_clients.csv')

# FEATURE 1: Operational Urgency Tier Segmentations
# Logic: Classifies accounts based on the speed required by their contract SLA windows
print("Engineering Feature 1: 'operational_urgency_tier'...")
df_clients_prod['operational_urgency_tier'] = np.where(
    df_clients_prod['contract_sla_hours'] <= 2,
    'High Urgency',
    'Standard Urgency'
)

# FEATURE 2: Corporate Account Health Index Structural Tags
# Logic: Blends client spend value tiers with operational urgency codes into a composite string flag
print("Engineering Feature 2: 'account_health_index'...\n")
df_clients_prod['account_health_index'] = (
    df_clients_prod['client_tier'].astype(str) + " - " + 
    df_clients_prod['operational_urgency_tier'].astype(str)
)

# 2. In-Place Production Save: Overwrite file to fully incorporate engineered columns
output_file = 'prod_corporate_clients.csv'
df_clients_prod.to_csv(output_file, index=False)

print(f"SUCCESS: Engineered production features successfully generated and saved to '{output_file}'!")
print(f"Total Corporate Account Dimensions Processed: {len(df_clients_prod)}")

# Override Jupyter column restrictions to display absolutely everything horizontally
pd.set_option('display.max_columns', None)

# Display the corporate lookup records to verify a flawless table layout
df_clients_prod.head(5)
```

Clients whose contract requires a response within two hours or less are marked high urgency. That urgency is then combined with the spend tier into a single account health label, so a big spender with a tight deadline stands out immediately.

---

## 8. Core Business Analysis

With clean, connected, and enriched tables in place, each of the five business objectives now runs as a short, repeatable script that ends in a formatted results matrix.

### Objective 1: Optimize Omni-Channel Revenue and Margin Realization

This analysis asks which booking channels and travel classes bring in the most money. It groups bookings by channel and travel class, then reports ticket volume, total revenue, average spend, and the lowest and highest prices found.

```python
print("INITIALIZING STRATEGIC BI PIPELINE: Objective 1 Analysis Framework...\n")

# 1. Read your engineered travel bookings production file from disk
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')

# 2. Pre-process currency text formatting in memory to perform mathematical math
df_bookings_prod['base_fare_numeric'] = pd.to_numeric(
    df_bookings_prod['base_fare_usd'].astype(str).str.replace(r'[\$,]', '', regex=True), 
    errors='coerce'
)

# CORE AGGREGATION: Channel & Sector Cross-Tabulation Matrix
# Calculates transactional volume, absolute revenue, and pricing spreads
obj1_pivot = df_bookings_prod.groupby(['booking_channel', 'travel_classification'])['base_fare_numeric'].agg(
    transaction_volume='count',
    total_revenue_usd='sum',
    average_ticket_spend='mean',
    min_ticket_price='min',
    max_ticket_price='max'
).reset_index()

# 3. Format numerical metrics cleanly into professional boardroom-ready currency text strings
obj1_pivot['total_revenue_usd'] = obj1_pivot['total_revenue_usd'].apply(lambda x: f"${x:,.2f}")
obj1_pivot['average_ticket_spend'] = obj1_pivot['average_ticket_spend'].apply(lambda x: f"${x:,.2f}")
obj1_pivot['min_ticket_price'] = obj1_pivot['min_ticket_price'].apply(lambda x: f"${x:,.2f}")
obj1_pivot['max_ticket_price'] = obj1_pivot['max_ticket_price'].apply(lambda x: f"${x:,.2f}")

# Rename column headers for clean structural readability
obj1_pivot.columns = [
    'Booking Channel', 'Travel Class', 'Ticket Volume', 
    'Total Revenue', 'Avg Ticket Spend', 'Min Price Found', 'Max Price Found'
]

print("OMNI-CHANNEL REVENUE & MARGIN REALIZATION PERFORMANCE MATRIX:")
print(obj1_pivot.to_string(index=False))
```

The fare column is converted back into numbers first, since it was formatted as text during cleaning. The resulting matrix shows at a glance which channel and class combinations carry the business.

### Objective 2: Mitigate Transactional Revenue Leakage

This analysis puts a dollar figure on services that were delivered but never billed, then shows which client accounts and service lines they come from.

```python
print("EXECUTION PIPELINE: Objective 2 Revenue Leakage Audit Dashboard...\n")

# 1. Read your production assets straight from disk
df_anc = pd.read_csv('prod_ancillary_services.csv')
df_bk = pd.read_csv('prod_travel_bookings.csv')
df_cli = pd.read_csv('prod_corporate_clients.csv')

# Print active column keys for immediate development transparency
print("Current Columns Stored on Disk:")
print(f"  ├── Ancillaries Columns: {list(df_anc.columns)}")
print(f"  ├── Bookings Columns:    {list(df_bk.columns)}")
print(f"  └── Clients Columns:     {list(df_cli.columns)}\n")

# 2. Convert price text formatting to numeric floats for exact financial calculations
df_anc['quoted_price_numeric'] = pd.to_numeric(
    df_anc['quoted_price_usd'].astype(str).str.replace(r'[\$,]', '', regex=True), 
    errors='coerce'
)

# 3. Standardize your revenue leakage status flags
df_anc['revenue_leakage_flag'] = np.where(
    df_anc['billed_status'].str.contains('Leakage', case=False, na=False),
    'Leakage Risk',
    'Secure'
)

# AUTOMATED KEY RESOLUTION JOIN BLOCK
# Programmatically determines the best columns to merge on to avoid KeyErrors
# Step 1: Link Ancillaries to Bookings on 'booking_key' (always shared)
df_master_merge = pd.merge(df_anc, df_bk, on='booking_key', how='inner', suffixes=('_anc', '_bk'))

# Step 2: Dynamically detect how to link to the Corporate Clients Directory
if 'client_key' in df_master_merge.columns and 'client_key' in df_cli.columns:
    df_leakage_final = pd.merge(df_master_merge, df_cli, on='client_key', how='inner', suffixes=('', '_dup'))
elif 'corporate_id' in df_master_merge.columns and 'corporate_id' in df_cli.columns:
    df_leakage_final = pd.merge(df_master_merge, df_cli, on='corporate_id', how='inner', suffixes=('', '_dup'))
elif 'company_name' in df_master_merge.columns and 'company_name' in df_cli.columns:
    df_leakage_final = pd.merge(df_master_merge, df_cli, on='company_name', how='inner', suffixes=('', '_dup'))
else:
    # Final fallback: link using the first column on both sides that contains 'key' or 'id'
    left_key = [c for c in df_master_merge.columns if 'key' in c or 'id' in c][0]
    right_key = [c for c in df_cli.columns if 'key' in c or 'id' in c][0]
    df_leakage_final = pd.merge(df_master_merge, df_cli, left_on=left_key, right_on=right_key, how='inner')

# METRIC CALCULATIONS & SUMMARY GRID DISPLAY
total_leakage_loss = df_leakage_final[df_leakage_final['revenue_leakage_flag'] == 'Leakage Risk']['quoted_price_numeric'].sum()
print(f"SYSTEM REVENUE RISK ANALYSIS SUMMARY:")
print(f"• Total Unbilled Revenue Leakage Amount: ${total_leakage_loss:,.2f}\n")

# Isolate columns for grouping dynamically based on what exists
name_field = 'company_name' if 'company_name' in df_leakage_final.columns else 'company_name_bk'
tier_field = 'client_tier' if 'client_tier' in df_leakage_final.columns else 'client_tier_bk'

leakage_breakdown = df_leakage_final[df_leakage_final['revenue_leakage_flag'] == 'Leakage Risk'].groupby(
    [name_field, tier_field, 'service_type']
)['quoted_price_numeric'].agg(
    incident_count='count',
    total_leaked_cash='sum'
).sort_values(by='total_leaked_cash', ascending=False).reset_index()

# Format numbers into clean, presentation-ready currency text strings
leakage_breakdown['total_leaked_cash'] = leakage_breakdown['total_leaked_cash'].apply(lambda x: f"${x:,.2f}")
leakage_breakdown.columns = ['Corporate Client Account', 'Account Tier', 'Leaked Service Line', 'Leakage Incidents', 'Total Leaked Cash']

print("GRANULAR REVENUE LEAKAGE AUDIT DIRECTORY:")
print(leakage_breakdown.to_string(index=False))
```

The script first prints a single headline number for total unbilled revenue, then breaks it down by client, tier, and service line, sorted from largest loss to smallest. The join block checks which shared key exists before merging, so the script keeps running even if column names change.

### Objective 3: Audit Service SLA Adherence and Workflow Velocity

This analysis checks whether tickets are being issued within the time each client contract allows, and where the slowdowns sit.

```python
print("INITIALIZING STRATEGIC BI PIPELINE: Objective 3 Service SLA Adherence Audit...\n")

# 1. Load the corrected, complete production bookings asset directly from disk storage
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')

# METRIC 1: Calculate Overall System SLA Breach Volume and Rates
total_tickets = len(df_bookings_prod)
total_breaches = (df_bookings_prod['sla_status'] == 'SLA Breached').sum()
breach_rate = (total_breaches / total_tickets) * 100

print(f"SYSTEM VELOCITY AUDIT SUMMARY HEADLINES:")
print(f"• Total Corporate Travel Tickets Evaluated: {total_tickets}")
print(f"• Total Contractual SLA Breaches Detected: {total_breaches}")
print(f"• Overall Global Workflow Breach Rate:      {breach_rate:.2f}%\n")

# METRIC 2: SLA & Turnaround Time (TAT) Aggregation by Corporate Client Account
# Parses volumes and average processing speeds across strict response windows
sla_matrix = df_bookings_prod.groupby(['company_name', 'client_tier', 'contract_sla_hours', 'sla_status']).agg(
    ticket_volume=('booking_id', 'count'),
    avg_turnaround_hours=('processing_duration_hours', 'mean')
).reset_index()

# Round processing hours to 2 decimal places for neat boardroom reporting matrices
sla_matrix['avg_turnaround_hours'] = sla_matrix['avg_turnaround_hours'].round(2)

# Sort the results matrix by ticket counts to expose high-traffic operational areas instantly
sla_matrix = sla_matrix.sort_values(by='ticket_volume', ascending=False)

# Rename column headers for clean structural scannability
sla_matrix.columns = [
    'Corporate Client Account', 'Account Spend Tier', 'Allowed SLA (Hours)', 
    'Fulfillment State', 'Ticket Count Volume', 'Avg Turnaround Time (Hrs)'
]

print("GRANULAR OPERATIONS VELOCITY MATRIX (BY CORPORATE SERVICE PROFILE):")
print(sla_matrix.to_string(index=False))
```

The breach rate is the number of breached tickets divided by all tickets, multiplied by one hundred. The matrix then shows, for each client, how many tickets landed in each state and how long processing actually took compared with the hours the contract allows.

### Objective 4: Uncover Customer Churn Triggers and Sentiment Drivers

This analysis looks for the complaint types that upset clients the most and the tickets most likely to push an account toward leaving.

```python
print("INITIALIZING STRATEGIC BI PIPELINE: Objective 4 Support Sentiment Analysis...\n")

# 1. Load your clean production datasets straight from your local drive storage
df_cs_prod = pd.read_csv('prod_customer_service.csv')
df_clients_prod = pd.read_csv('prod_corporate_clients.csv')

# RELATIONSHIP LAYOUT: Relational Table Join Matrix
# Securely link customer service logs to your corporate directory
# Programmatically determines the best join keys available on disk
if 'client_key' in df_cs_prod.columns and 'client_key' in df_clients_prod.columns:
    df_cs_master = pd.merge(df_cs_prod, df_clients_prod, on='client_key', how='inner', suffixes=('', '_cli'))
elif 'corporate_id' in df_cs_prod.columns and 'corporate_id' in df_clients_prod.columns:
    df_cs_master = pd.merge(df_cs_prod, df_clients_prod, on='corporate_id', how='inner', suffixes=('', '_cli'))
else:
    df_cs_master = pd.merge(df_cs_prod, df_clients_prod, left_index=True, right_index=True, suffixes=('', '_cli'))

# METRIC VALUE 1: Calculate Overall System Dissatisfaction & Breach Volume
total_tickets = len(df_cs_master)
dissatisfied_tickets = (df_cs_master['satisfaction_segment'] == 'Dissatisfied').sum()
unresolved_breaches = (df_cs_master['resolution_status'].str.contains('Breached', case=False, na=False)).sum()

print(f"HELPDESK SENTIMENT AUDIT SUMMARY HEADLINES:")
print(f"• Total Helpdesk Logs Evaluated:             {total_tickets}")
print(f"• Total Dissatisfied Client Ratings (1-2):   {dissatisfied_tickets}")
print(f"• Total Support Tickets Overdue/SLA Breached: {unresolved_breaches}\n")

# METRIC VALUE 2: Sentiment Drivers by Complaint Category & Escalation Status
# Parses active incident counts and highlights structural priority flags
name_field = 'company_name' if 'company_name' in df_cs_master.columns else 'corporate_id'
tier_field = 'client_tier' if 'client_tier' in df_cs_master.columns else 'client_tier_cli'

sentiment_matrix = df_cs_master.groupby(
    ['complaint_category', 'satisfaction_segment', 'critical_complaint_flag', 'resolution_status']
).size().reset_index(name='Incident Volume')

# Sort the results matrix by active volume to expose systemic pain points instantly
sentiment_matrix = sentiment_matrix.sort_values(by='Incident Volume', ascending=False).reset_index(drop=True)

# Rename column headers for clean structural scannability
sentiment_matrix.columns = [
    'Complaint Issue Category', 'Customer Sentiment Segment', 
    'System Priority Severity', 'Current Ticket Status', 'Active Incident Volume'
]

print("GRANULAR CLIENT SENTIMENT DRIVERS & CHURN RISK HOTSPOTS:")
print(sentiment_matrix.to_string(index=False))
```

The matrix lines up complaint category, sentiment, escalation priority, and ticket status, sorted by volume. The biggest rows at the top are the systemic pain points, and dissatisfied customers with high priority escalations are the clearest churn warnings.

### Objective 5: Enhance Industry Market Competitiveness

This analysis measures how much of each client's travel wallet the company actually captures. It compares flights and hotels against the higher margin add-ons: visas, protocol services, and car hire.

```python
print("INITIALIZING STRATEGIC BI PIPELINE: Objective 5 Market Competitiveness Matrix...\n")

# 1. Load your clean production datasets straight from your local drive storage
df_bookings_prod = pd.read_csv('prod_travel_bookings.csv')
df_ancillaries_prod = pd.read_csv('prod_ancillary_services.csv')
df_clients_prod = pd.read_csv('prod_corporate_clients.csv')

# STEP 1: Aggregate Core Bookings Product Offerings (Flights vs Hotels)
df_bookings_prod['is_flight'] = np.where(df_bookings_prod['travel_type'].str.contains('Flight', case=False, na=False), 1, 0)
df_bookings_prod['is_hotel'] = np.where(df_bookings_prod['travel_type'].str.contains('Hotel', case=False, na=False), 1, 0)

core_counts = df_bookings_prod.groupby('company_name').agg(
    flight_volume=('is_flight', 'sum'),
    hotel_volume=('is_hotel', 'sum')
).reset_index()

# STEP 2: Aggregate Lucrative Ancillary Offerings (Visas, Protocol, Car Hire)
# Mapping Logic: Connect Ancillary records back to Corporate Client Names via Bookings
df_anc_linked = pd.merge(df_ancillaries_prod, df_bookings_prod[['booking_key', 'company_name']], on='booking_key', how='inner')

df_anc_linked['is_visa'] = np.where(df_anc_linked['service_type'].str.contains('Visa', case=False, na=False), 1, 0)
df_anc_linked['is_protocol'] = np.where(df_anc_linked['service_type'].str.contains('Protocol', case=False, na=False), 1, 0)
df_anc_linked['is_car_hire'] = np.where(df_anc_linked['service_type'].str.contains('Car', case=False, na=False), 1, 0)

ancillary_counts = df_anc_linked.groupby('company_name').agg(
    visa_volume=('is_visa', 'sum'),
    protocol_volume=('is_protocol', 'sum'),
    car_hire_volume=('is_car_hire', 'sum')
).reset_index()

# STEP 3: Synthesize the Final Market Penetration Wallet Share Matrix
# Merge our core bookings and ancillary counts sequentially
df_matrix_build = pd.merge(core_counts, ancillary_counts, on='company_name', how='left').fillna(0)

# Bring in Client Tier profiles from your Master Dimension Directory
competitiveness_matrix = pd.merge(df_clients_prod[['company_name', 'client_tier']], df_matrix_build, on='company_name', how='inner')

# Convert volume trackers to integers for crisp visual presentation
vol_cols = ['flight_volume', 'hotel_volume', 'visa_volume', 'protocol_volume', 'car_hire_volume']
competitiveness_matrix[vol_cols] = competitiveness_matrix[vol_cols].astype(int)

# Sort by core flight activity to arrange your largest accounts at the top
competitiveness_matrix = competitiveness_matrix.sort_values(by='flight_volume', ascending=False).reset_index(drop=True)

# Rename column headers for clean boardroom-ready scannability
competitiveness_matrix.columns = [
    'Corporate Account Name', 'Spend Tier', 'Flights Issued', 
    'Hotels Booked', 'Visa Requests', 'Protocol Cleared', 'Car Hires Logged'
]

print("ACCOUNT PORTFOLIO CROSS-SELL PENETRATION MATRIX (WALLET SHARE):")
print(competitiveness_matrix.to_string(index=False))
```

Each client gets a row showing how many flights, hotels, visas, protocol services, and car hires they bought. Large accounts with plenty of flights but almost no add-ons are the clearest cross-sell opportunities.

---

## 9. Data Visualization and Interpretation

This project delivers its results as formatted matrices rather than charts, and each one is built to be read like a report table. Here is how to read them.

| Matrix | What to Look For |
| --- | --- |
| Omni-channel revenue matrix | The channel and travel class rows with the highest total revenue and average ticket spend |
| Revenue leakage directory | The top rows, since they are sorted from the largest unbilled amount downward |
| SLA velocity matrix | Rows in the breached state where average turnaround exceeds the allowed contract hours |
| Sentiment drivers matrix | High volume rows combining dissatisfied ratings with high priority escalation |
| Cross-sell penetration matrix | Large accounts with strong flight counts but low visa, protocol, or car hire counts |

---

## 10. Key Findings and Insights

The five analyses are designed to expose a consistent set of patterns.

Revenue is rarely spread evenly across channels. The channel and class matrix pinpoints which booking routes carry the business and which underperform on average ticket value.

Leakage concentrates in specific accounts and service lines. Ranking unbilled cash from largest to smallest points finance straight to where recovery effort pays off most.

SLA breaches cluster around particular clients and contract windows. Comparing actual turnaround against contracted hours separates genuine slowdowns from tight contracts that were always hard to meet.

Customer frustration follows recognizable triggers. Billing errors and missed commitments show up repeatedly in the escalation flag, and the sentiment matrix shows which complaint types drive dissatisfaction.

Cross-sell reach varies widely by account. Some large clients buy flights only, which signals untapped add-on revenue that the company is already positioned to serve.

---

## 11. Recommendations

Start recovery with the accounts and service lines at the top of the leakage directory, and add a billing check at service delivery so an unbilled service is caught the same day.

Review staffing and workflow for any client where average turnaround regularly exceeds the contract window, and revisit unrealistic SLA terms at renewal.

Route every high priority escalation to a dedicated response queue, since these complaints carry the highest churn risk.

Give account managers the cross-sell matrix and target large flight-only accounts first with visa, protocol, and car hire offers.

Schedule the whole pipeline to run regularly, so these five views stay current instead of becoming one-time reports.

---

## 12. Conclusion

Four raw tables became a clean, connected, and enriched analytics pipeline that answers five real business questions on demand. Every data issue found in the audit was traced to a specific fix, every engineered feature feeds a specific analysis, and every objective now produces a clear matrix a manager can act on. The real value is not any single number. It is that the whole chain, from raw data to decision, can be rerun and trusted every time.
