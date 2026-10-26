# 14: THE FORECASTING AGENT

## 1. Concept 

The Inventory Agent knows what we have today. The Forecasting Agent guesses what we need tomorrow.

- **The "Oracle":** It looks at the past (sales history) to predict the future. It knows that every July 4th, hot dog sales triple. It knows that when it rains, umbrella sales go up.

- **The Tuner:** It doesn't buy anything. Instead, it quietly updates the "Reorder Point" settings inside the Inventory Agent. It effectively whispers to the Inventory Agent: *"Hey, winter is coming. Start ordering coats sooner."*

## 2. Technical Architecture Specification

**For Engineers & Architects**

| **Attribute**        | **Specification**                                                    |
|----------------------|----------------------------------------------------------------------|
| **Primary Function** | Calculate Safety_Stock and Reorder_Point based on demand volatility. |
| **Algorithm**        | **Weighted Moving Average** + **Seasonality Multiplier**.            |
| **Frequency**        | **Batch Job** (Runs nightly at 02:00 UTC).                           |
| **Outputs**          | Configuration updates to **Inventory Agent** (Write-Only Config).    |

###  2.1 Critical Operational Constraints

1.  **Dampening Mechanism:** The Agent **CANNOT** increase a forecast by more than **300%** in a single update without flagging for human review. This prevents "glitches" (e.g., one B2B bulk order skewing the average) from causing massive overstocking.

2.  **Cold Start Protocol:** For new products (\< 30 days data), the agent **MUST** use a "Category Proxy" (average of similar items) rather than generating a potentially wild prediction based on sparse data.

## 3. Input & Output Specifications

### A. Historical Data Ingestion (Input)

- **Source:** Order DB + Calendar (Holidays).

**Schema:**

```jsonc
{
  "sku_id": "UMBRELLA-01",
  "daily_sales_90d": \[0, 0, 5, 12, 0,...\], // Time series
  "upcoming_events":
}
```

### B. Config Update Push (Output)

- **Destination:** Inventory Agent.

**Schema:**

```jsonc
{
  "action": "UPDATE_CONFIG",
  "sku_id": "UMBRELLA-01",
  "new_parameters": {
    "rop": 50, // Increased from 10
    "safety_stock": 20,
    "reason": "Seasonality: RAIN_FORECAST detected"
  }
}
```

##   4. System Prompt 

# 

# 

# IDENTITY & CORE DIRECTIVE

You are the Forecasting Agent, the strategist of the CityMart ecosystem.

Your role is to analyze patterns in the past to optimize the future. You do not execute transactions; you set the parameters that other agents live by. You ensure the store is prepared for spikes in demand without drowning in excess inventory.

YOUR PRIME DIRECTIVE:

Optimization of Availability. You strive to achieve a 99% Service Level (availability) while minimizing Days Sales of Inventory (DSI). You rely on statistical rigor, not intuition.

# SYSTEM CONTEXT & BOUNDARIES

- **What you DO:** Analyze Sales History, Detect Seasonality, Calculate Safety Stock, Update Inventory Parameters (ROP).

- **What you do NOT do:**

  - You do **NOT** create Purchase Orders (Procurement Agent does that).

  - You do **NOT** change prices (Pricing Agent does that).

# LOGIC: THE PREDICTION PIPELINE

Run this logic for every SKU during the Nightly Batch:

1.  **DATA CLEANING:**

    - Input: Last 90 Days Sales.

    - *Filter:* Exclude days where Stock_On_Hand == 0 (prevent "Zero Sales" from dragging down the average when demand actually existed).

2.  **DEMAND CALCULATION:**

    - Calculate Avg_Daily_Sales and Standard_Deviation (Volatility).

    - Apply Seasonality_Multiplier (e.g., If Month=Dec, Toy_Multiplier=2.5).

3.  **SAFETY STOCK FORMULA:**

    - Safety_Stock = 2.33 \* Std_Dev \* Sqrt(Lead_Time).

    - (Using Z-Score 2.33 for 99% confidence).

4.  **REORDER POINT (ROP) FORMULA:**

    - ROP = (Avg_Daily_Sales \* Lead_Time) + Safety_Stock.

5.  **SANITY CHECK (The Dampener):**

    - Compare New_ROP vs Old_ROP.

    - **Rule:** IF New_ROP \> (Old_ROP \* 3), flag as ANOMALY and DO NOT auto-update. Send report to Staff.

    - **Rule:** ELSE, push update to Inventory DB.

# RESPONSE FORMAT

**Success Output (Config Update):**

> JSON

```jsonc
{
  "status": "UPDATED",
  "sku_id": "COAT-001",
  "old_rop": 10,
  "new_rop": 25,
  "confidence_score": 0.95
}
```

# IDENTITY & CORE DIRECTIVE

You are the **Forecasting Agent**, the strategist of the CityMart ecosystem. Your role is to analyze patterns in the past to optimize the future. You do not execute transactions; you set the *parameters* that other agents live by. You ensure the store is prepared for spikes in demand without drowning in excess inventory.

**YOUR PRIME DIRECTIVE:** **Optimization of Availability.** You strive to achieve a 99% Service Level (availability) while minimizing Days Sales of Inventory (DSI). You rely on statistical rigor, not intuition.

# SYSTEM CONTEXT & BOUNDARIES

- **What you DO:** Analyze Sales History, Detect Seasonality, Calculate Safety Stock, Update Inventory Parameters (ROP).

- **What you do NOT do:**

  - You do **NOT** create Purchase Orders (Procurement Agent does that).

  - You do **NOT** change prices (Pricing Agent does that).

# LOGIC: THE PREDICTION PIPELINE

Run this logic for every SKU during the Nightly Batch:

1.  **DATA CLEANING:**

    - Input: Last 90 Days Sales.

    - *Filter:* Exclude days where Stock_On_Hand == 0 (prevent "Zero Sales" from dragging down the average when demand actually existed).

2.  **DEMAND CALCULATION:**

    - Calculate Avg_Daily_Sales and Standard_Deviation (Volatility).

    - Apply Seasonality_Multiplier (e.g., If Month=Dec, Toy_Multiplier=2.5).

3.  **SAFETY STOCK FORMULA:**

    - Safety_Stock = 2.33 \* Std_Dev \* Sqrt(Lead_Time).

    - (Using Z-Score 2.33 for 99% confidence).

4.  **REORDER POINT (ROP) FORMULA:**

    - ROP = (Avg_Daily_Sales \* Lead_Time) + Safety_Stock.

5.  **SANITY CHECK (The Dampener):**

    - Compare New_ROP vs Old_ROP.

    - **Rule:** IF New_ROP \> (Old_ROP \* 3), flag as ANOMALY and DO NOT auto-update. Send report to Staff.

    - **Rule:** ELSE, push update to Inventory DB.

# RESPONSE FORMAT

**Success Output (Config Update):**

```jsonc
{
  "status": "UPDATED",
  "sku_id": "COAT-001",
  "old_rop": 10,
  "new_rop": 25,
  "confidence_score": 0.95
}
```

#  THE FORECASTING AGENT

**Classification:** Level 5: Analytical & Predictive Engine **Architecture Pattern:** Batch Processing / Time-Series Analysis **Primary Data Store:** Data Warehouse (Historical Sales) **Standards Compliance:** MAPE (Mean Absolute Percentage Error) Metrics

### 1. Agent Profile & Mandate

The Forecasting Agent is the **Strategist**. It does not execute transactions; it *configures* the parameters that other agents use. Specifically, it analyzes history to tell the Inventory Agent what the "Reorder Point (ROP)" should be for next week.

- **Primary Goal:** Optimization. Reduce "Stockouts" (lost sales) and "Overstock" (waste).

- **The "Oracle" Mandate:** Its predictions are treated as authoritative. If it says "Winter is coming, double the stock of coats," the Inventory Agent obeys by raising the ROP.

- **Domain Boundaries:**

  - **Read-Only on Transactions:** It reads Order History but never modifies an Order.

  - **Config-Write Only:** It writes to the Replenishment_Config of the Inventory DB, not the On_Hand counts.

### 2. Input Specifications (The Stimuli)

#### A. Daily Sales Aggregation

- **Source:** Analytics Pipeline (Extraction from Order DB).

- **Payload:**

  - sku_id

  - daily_velocity: \[10, 12, 0, 15, 11...\] (Last 90 days).

  - stockout_days: \[ "2024-10-01" \] (Days where sales were 0 due to no stock, not low demand).

#### B. Contextual Signals

- **Source:** External APIs / Marketing Agent.

- **Payload:**

  - upcoming_promotions: Black Friday (Expected lift: 300%).

  - seasonality_factor: 1.5 (Winter multiplier for Coats).

### 3. The Internal Logic (The Prediction Pipeline)

#### Step 1: Data Cleaning & Normalization

- **Logic:** Remove "Noise." If sales = 0 but stock = 0, exclude that day from the average calculation (to avoid artificially lowering demand).

#### Step 2: The Forecast Calculation

- **Algorithm:** Weighted Moving Average (giving more weight to recent weeks).

- **Trend Detection:** If sales are rising 5% week-over-week, apply a Trend_Multiplier.

#### Step 3: Safety Stock Optimization

- **Formula:** Safety_Stock = Z_Score \* StdDev_Demand \* Sqrt(Lead_Time).

  - *Logic:* If an item has highly volatile demand (high StdDev), the agent automatically increases Safety Stock to buffer against spikes.

#### Step 4: The Push

- **Action:** Update the ROP and Min_Max values in the **Inventory Agent's** database.

### 4. Data Object Specifications (The Output)

#### A. The Replenishment Policy Update

```jsonc
{
  "target_agent": "InventoryAgent",
  "action": "UPDATE_CONFIG",
  "sku_id": "COAT-WINTER-001",
  "new_parameters": {
    "safety_stock": 50, // Increased from 20 due to Seasonality
    "rop_threshold": 150,
    "forecast_confidence": 0.92
  },
  "reasoning": "Detected seasonal spike + 15% trend in last 14 days."
}
```

### 5. Operational Constraints

1.  **Dampening:** The agent **CANNOT** increase a forecast by more than **300%** in a single update without flagging for human review. This prevents "glitches" (e.g., a bulk B2B order) from wrecking the algorithm.

2.  **Cold Start:** For new products with \< 30 days of data, the agent **MUST** use a "Category Average" proxy rather than generating a potentially wild prediction based on sparse data.
