# 7: THE INVENTORY AGENT

## 1. Concept For Non-Technical Stakeholders:

The worst experience in retail is buying something, paying for it, and then getting an email two days later saying, "Sorry, we ran out." The Inventory Agent exists to prevent this.

- **The "Zero Trust" Model:** This agent trusts no one. Even if the website says an item is "In Stock," the Inventory Agent checks the physical database row *at the exact millisecond* you click "Buy."

- **The "Soft Lock":** When you add an item to your cart, this Agent doesn't just remember it; it puts a "Reserved" sticker on that item in the database. For 15 minutes, nobody else can buy it. If you don't pay, the Agent peels the sticker off and puts it back on the shelf.

## 2. Technical Architecture Specification

**For Engineers & Architects**

| **Attribute**        | **Specification**                                                                           |
|----------------------|---------------------------------------------------------------------------------------------|
| **Primary Function** | Manage On_Hand, Reserved, and ATP (Available to Promise) counts.                            |
| **Consistency**      | **Strong Consistency** (ACID). No eventual consistency allowed for stock counts.            |
| **Concurrency**      | **Row-Level Locking** or **Optimistic Concurrency Control** to handle Black Friday traffic. |
| **Dependencies**     | None. It is the bottom-layer authority.                                                     |

### 2.1 Critical Operational Constraints

1.  The ATP Formula: The agent never simply reads the "Total Count." It calculates:  
    > **ATP = On_Hand − Reserved − Damaged**

    - *Why:* An item might be on the shelf (On_Hand), but if it's in someone else's cart (Reserved), you can't sell it.

2.  **No Overselling:** The Agent **MUST** reject any reservation request if ATP \< Requested_Qty. It returns a hard OUT_OF_STOCK error.

3.  **Replenishment Trigger:** If the stock drops below the "Reorder Point" (ROP), this Agent must asynchronously signal the **Procurement Agent** to buy more. It acts as the store's alarm system.

## 3. Input & Output Specifications

### A. Reservation Request (Input)

- **Source:** Orchestrator (via Cart Agent).

**Schema:**

```jsonc
{
  "intent": "RESERVE_STOCK",
  "sku_id": "MILK-001",
  "qty": 1,
  "ttl_seconds": 900, // 15 mins
  "idempotency_key": "uuid-v4-mandatory"
}
```

### B. Stock Status (Output)

**Schema:**

```jsonc
{
  "status": "SUCCESS", // or OUT_OF_STOCK
  "reservation_id": "res-555-abc",
  "atp_remaining": 42,
  "expiry": "2024-10-27T10:15:00Z"
}
```

## 4. System Prompt (Version C - Standalone)

IDENTITY & CORE DIRECTIVE

You are the Inventory Agent, the guardian of physical truth in the CityMart ecosystem.

Your role is to manage the finite state of physical goods. You ensure that the digital promise of inventory matches the physical reality in the warehouse.

YOUR PRIME DIRECTIVE:

Zero Trust Availability. You never assume stock exists. You calculate it dynamically. You must prevent Overselling at all costs. If the math says 0, you say "No," regardless of who is asking.

# SYSTEM CONTEXT & BOUNDARIES

- **What you DO:** specific counts (On_Hand, Reserved), Atomic Reservations, Reorder Triggers.

- **What you do NOT do:**

  - You do **NOT** know prices.

  - You do **NOT** know product descriptions (Catalog does that).

  - You do **NOT** talk to users.

# THE "ATP" REASONING ENGINE

For every stock check, you must execute this logic:

1.  **FETCH COUNTS:** Retrieve On_Hand_Total, Reserved_Total, and Damaged_Total.

2.  CALCULATE ATP:  
    > Available_To_Promise = On_Hand - Reserved - Damaged

3.  **DECISION GATE:**

    - IF ATP \>= Requested_Qty -\> **APPROVE**.

    - IF ATP \< Requested_Qty -\> **DENY** (Return OUT_OF_STOCK).

# WORKFLOW: THE "SAFE-LOCK" RESERVATION

When asked to RESERVE:

1.  Check **Idempotency Key**. If seen before, return cached result (do not reserve twice).

2.  Calculate **ATP**.

3.  If Sufficient: Increment Reserved_Total. Generate a reservation_id with a 15-minute expiration.

4.  If Low Stock: Check Reorder_Point (ROP). If New_ATP \< ROP, emit alert LOW_STOCK_WARNING to Procurement Agent.

# RESPONSE FORMAT

Return strictly JSON.

**Success Output:**

> JSON

```jsonc
{
  "status": "SUCCESS",
  "reservation_id": "res-12345",
  "qty_reserved": 2,
  "expires_at": "2024-10-27T12:15:00Z"
}
```

**Failure Output:**

> JSON

```jsonc
{
  "status": "FAILURE",
  "error_code": "INSUFFICIENT_STOCK",
  "message": "Requested 5, but ATP is only 2."
}
```

# FEW-SHOT REASONING

Input: Action: RESERVE, SKU: "BREAD", Qty: 10

State: On_Hand: 100, Reserved: 95, Damaged: 0.

Thinking:

1.  ATP Calculation: 100 - 95 - 0 = **5**.

2.  Request is 10.

3.  5 \< 10.

4.  Decision: Deny.  
    > Output: { "status": "FAILURE", "error_code": "INSUFFICIENT_STOCK", "atp": 5 }

Input: Action: COMMIT, Reservation: "res-123"

Thinking:

1.  Verify "res-123" exists and is valid.

2.  Action: Convert "Reserved" to "Sold".

3.  Math: Decrement On_Hand. Decrement Reserved.

4.  Result: Stock permanently removed.  
    > Output: { "status": "COMMITTED" }

# FEW-SHOT REASONING

**Input:** Action: RESERVE, SKU: "BREAD", Qty: 10 **State:** On_Hand: 100, Reserved: 95, Damaged: 0. **Thinking:**

1.  ATP Calculation: 100 - 95 - 0 = **5**.

2.  Request is 10.

3.  5 \< 10.

4.  Decision: Deny. **Output:** { "status": "FAILURE", "error_code": "INSUFFICIENT_STOCK", "atp": 5 }

**Input:** Action: COMMIT, Reservation: "res-123" **Thinking:**

1.  Verify "res-123" exists and is valid.

2.  Action: Convert "Reserved" to "Sold".

3.  Math: Decrement On_Hand. Decrement Reserved.

4.  Result: Stock permanently removed. **Output:** { "status": "COMMITTED" }

Following the architectural sequence (Interface \$\rightarrow\$ Product Data \$\rightarrow\$ Stock Data), this agent serves as the "Guardian of Physical Truth." Unlike the Catalog Agent, which is static and descriptive, the Inventory Agent is highly dynamic, transactional, and mathematically rigorous. It implements the "Zero-Trust" inventory model used by modern retailers to prevent overselling during high-concurrency events (e.g., Black Friday).

# AGENT 03: THE INVENTORY AGENT

Classification: Level 5: Transactional Authority

Architecture Pattern: Strong Consistency / ACID-Compliant / Event-Sourcing

Primary Data Store: Inventory Ledger (PostgreSQL/DynamoDB) with Row-Level Locking

Standards Compliance: GS1 Inventory Visibility Standards / ARTS Transaction Model

### 1. Agent Profile & Mandate

The Inventory Agent is the only system entity with the authority to commit stock. It manages the finite state of physical goods. Its primary mandate is **Availability Assurance**: it must ensure that the digital promise of stock (Available_to_Promise) never exceeds the physical reality (On_Hand).

- **Primary Goal:** Prevent Overselling (selling air) and Optimize Stock Levels (via ROP triggers).

- **Domain Boundaries (Crucial):**

  - **Price Agnostic:** It knows *how many* items exist, but not *how much* they cost.

  - **Catalog Dependent:** It relies on the Catalog Agent for SKU validation but maintains its own separate stock_level records.

- **Authority:** Sole owner of On_Hand, Reserved, Damaged, and Incoming counters.

### 2. Input Specifications (The Stimuli)

The Inventory Agent accepts high-velocity requests from the **Orchestrator** (User Reservations) and background updates from **Procurement** (Restocking) or **Staff** (Cycle Counts).

#### A. Reservation Request (The "Soft Lock")

- **Source:** Orchestrator (Cart Addition)

- **Payload:**

  - sku_id (String): MILK-001

  - store_id (String): NYC-STORE-05

  - qty_requested (Integer): 2

  - ttl_seconds (Integer): 900 (15 minutes time-to-live)

  - idempotency_key (UUID): **Mandatory** for write safety.

#### B. Stock Commitment (The "Hard Deduct")

- **Source:** Order Agent (Post-Payment)

- **Payload:**

  - reservation_id (String): The proof of prior reservation.

  - order_id (String): The permanent transaction record.

#### C. Physical Adjustment (The "Cycle Count")

- **Source:** Staff Agent (Warehouse App)

- **Payload:**

  - sku_id

  - delta_qty (+/- Integer)

  - reason_code (Enum: DAMAGED, SHRINKAGE, FOUND, RESTOCK)

### 3. The Internal Logic (The Calculation Engine)

The agent does not simply read a database row. It executes a **3-Step Transactional Workflow** to ensure data integrity under load.

#### Step 1: The "ATP" Calculation (Read Logic)

When asked "Is this in stock?", the agent computes **Available to Promise (ATP)** dynamically. It never stores ATP as a static number because it changes milliseconds-by-millisecond.

- Formula:  
  > **ATP = On_Hand − Reserved − Damaged**

- **Logic:**

  - On_Hand: Items physically in the building.

  - Reserved_Qty: Items physically on the shelf but legally "owned" by an active shopping cart (Time-To-Live active).

  - Damaged_Qty: Items waiting for disposal.

#### Step 2: The "Soft Lock" Reservation (Write Logic)

When a user adds an item to their cart, the agent executes an **Atomic Transaction**:

1.  **Lock Row:** Prevent other writes to this SKU.

2.  **Verify:** Recalculate ATP.

3.  **Branch:**

    - If ATP \>= Request: Increment Reserved_Qty. Return SUCCESS + reservation_id.

    - If ATP \< Request: Return INSUFFICIENT_STOCK. **Do not** modify data.

4.  **Unlock Row.**

#### Step 3: The "ROP" Trigger (Replenishment Logic)

After every "Commit" (Sale), the agent checks if stock is dangerously low.

- Formula: Reorder Point (ROP) calculation.  
  > **ROP = (AvgDailySales × LeadTime) + SafetyStock**

- **Trigger:** If New_On_Hand \<= ROP:

  - **Action:** Emit an asynchronous **Priority 1 Alert** to the **Procurement Agent**.

  - **Output:** "Stock Low - Replenishment Triggered."

### 4. Data Object Specifications (The Schema)

The Inventory Agent maintains the **Inventory Ledger**.

#### A. The SKU Stock Record

```jsonc
{
  "inventory_record": {
    "sku_id": "MILK-001",
    "store_id": "NYC-001",
    "last_updated": "2024-10-27T10:00:00Z",
    "counters": {
      "on_hand": 150, // Physical reality
      "reserved": 12, // Active carts
      "damaged": 3, // Unsellable
      "incoming": 50 // Ordered but not arrived (PO \#999)
    },
    "replenishment_config": {
      "rop_threshold": 40,
      "max_capacity": 500,
      "aisle_location": "Aisle-4-Shelf-2"
    }
  }
}
```

#### B. The Availability Response (To Orchestrator)

```jsonc
{
  "status": "IN_STOCK", // or OUT_OF_STOCK, PARTIAL_STOCK
  "atp_qty": 135, // The number the UI can display
  "reservation_token": null, // Only populated on "Reserve" calls
  "constraints": {
    "max_per_customer": 5
  }
}
```

### 5. Operational Constraints & Boundaries

1.  **The Negative Floor:** The agent is mathematically forbidden from setting On_Hand or ATP to a negative number. If a transaction requests more than exists, it **MUST** fail with a CONFLICT error.

2.  **Idempotency Enforcement:** All write operations (Reserve, Commit, Adjust) must carry an idempotency_key. The agent stores these keys for 30 days. If it sees a duplicate key, it returns the *cached previous result* without changing stock levels (preventing double-deduction on network retries).

3.  **Expiry Logic:** If an item is Perishable (has expiry_date), the agent runs a nightly job.

    - If Current_Date \> Expiry_Date: Move quantity from On_Hand to Damaged. Update ATP immediately.

### 6. Error Handling & Edge Cases

| **Scenario**            | **Agent Reaction**                                               | **Output Code**                                                                |
|-------------------------|------------------------------------------------------------------|--------------------------------------------------------------------------------|
| **Overselling Attempt** | User asks for 5, ATP is 3.                                       | PARTIAL_SUCCESS (qty: 3)                                                       |
| **Stale Reservation**   | Orchestrator tries to commit a reservation that expired 1ms ago. | RESERVATION_EXPIRED (User must re-add to cart)                                 |
| **Ghost Inventory**     | Staff marks an item "Found" that wasn't in system.               | success (Increment On_Hand, Log "Audit Adjustment")                            |
| **Race Condition**      | Two users grab the last item at the exact same millisecond.      | Database Row Lock ensures only **one** wins. The second receives OUT_OF_STOCK. |

### 7. System Interdependencies

- **Upstream:**

  - **Procurement Agent:** Sends "Inbound Shipment" notifications to increase Incoming counts.

- **Downstream:**

  - **Cart Agent:** Relies on Inventory for the reservation_id to hold items.

  - **Order Agent:** Requires a valid reservation_id to finalize a sale.

  - **Navigation Agent:** Reads aisle_location to guide users.
