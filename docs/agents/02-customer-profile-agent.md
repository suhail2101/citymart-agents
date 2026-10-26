# AGENT 02: THE CUSTOMER PROFILE AGENT (CPA)

**"The Privacy Guardian & Loyalty Engine"**

## 1. Concept 

In most systems, customer data is scattered everywhere—shipping has the address, marketing has the email, and sales has the credit card. This creates a security nightmare.

The **CPA** is the solution to that chaos. Think of it as a **Bank Vault**.

- It is the *only* agent allowed to see the customer's real name, home address, and phone number.

- When other agents (like the Delivery Agent) need to know where to ship a package, they don't get the customer's full life history; they get *only* the shipping address, and nothing else.

- It also manages "Status." It knows if you are a VIP (Gold Tier) or a new user, and it tells the Pricing Agent to give you a discount without revealing *why* you got it.

**The Golden Rule:** "Need to Know." The CPA hides personal details from everyone—even the Orchestrator—unless absolutely necessary for the transaction.

## 2. Technical Architecture Specification

**For Engineers & Architects**

| **Attribute**      | **Specification**                                        |
|--------------------|----------------------------------------------------------|
| **Classification** | Level 5 Identity Authority / Secure Enclave              |
| **Model Backbone** | GPT-4o (Optimized for JSON & PII Detection)              |
| **Data Store**     | **Customer DB** (PostgreSQL with Row-Level Encryption)   |
| **Standards**      | GDPR (Europe), CCPA (California), NIST Digital Identity  |
| **Latency Budget** | \< 100ms (Must be extremely fast for session validation) |

### 2.1 Critical Architectural Constraints

1.  **The PII Firewall:** This agent is the *only* entity in the ecosystem authorized to read/write columns flagged as SENSITIVE_PII (e.g., email_address, home_address_line1).

2.  **Opaque References:** When communicating with other agents, the CPA never sends raw PII. It sends a customer_id (UUID).

    - *Bad:* Sending john.doe@gmail.com to the Pricing Agent.

    - *Good:* Sending CUST-8829-X to the Pricing Agent.

3.  **Consent Gate:** Before returning *any* data, the agent checks the consent_ledger. If marketing_opt_in = false, the email field must return null or \`\` to the Marketing Agent.

## 3. Data Object Specifications

The CPA manages the **"Golden Record"**—the single source of truth for a human's identity.

### A. The Customer Profile Schema

```jsonc
{
  "customer_profile": {
    "customer_id": "CUST-998877", // Public ID (Safe to share)
    "identity": {
      "first_name": "Jane",
      "last_name": "Doe",
      "email_masked": "j\*\*\*e@citymart.com", // Default output
      "phone_masked": "+1-555-\*\*\*-9988",
      "kyc_verified": true // Required for Alcohol/Meds
    },
    "loyalty": {
      "tier": "GOLD", // Used by Pricing Agent
      "points_balance": 1500,
      "lifetime_spend": 4500.50
    },
    "preferences": {
      "dietary_flags": [ ... ], // Used by Catalog
      "communication_channel": "SMS"
    },
    "addresses":
  }
}
```

## 4. The Agent Workflows (Internal Logic)

The CPA does not just "read data"; it enforces policy.

### Workflow A: Profile Lookup (The "Who am I?" Request)

- **Trigger:** Orchestrator receives "Login" or Pricing Agent needs "Loyalty Tier."

- **Input:** { "auth_token": "jwt_xyz", "field_mask": \["loyalty", "preferences"\] }

- **Logic:**

  1.  **Authenticate:** Validate the token signature.

  2.  **Scope Check:** Does the requestor (e.g., Pricing Agent) have permission to see loyalty? **Yes.**

  3.  **Scope Check:** Does the requestor have permission to see home_address? **No.**

  4.  **Filter:** Strip out all fields *not* in the field_mask.

- **Output:** Returns partial JSON containing *only* Loyalty and Preferences.

### Workflow B: Address Normalization (The "Clean Up" Request)

- **Trigger:** User enters "123 main st, ny" during checkout.

- **Input:** { "raw_address": "123 main st, ny", "customer_id": "CUST-99" }

- **Logic:**

  1.  **Geocoding:** Call internal/external Geocoder API.

  2.  **Correction:** "ny" \$\to\$ "New York, NY". Zip code 10001 appended.

  3.  **Deduplication:** Check if this address already exists in the user's profile.

  4.  **Persist:** Save as addr_new_ID.

- **Output:** { "status": "SUCCESS", "normalized": "123 Main St, New York, NY 10001", "address_id": "addr_02" }

### Workflow C: The "Right to Forget" (GDPR Compliance)

- **Trigger:** User requests account deletion.

- **Logic:**

  1.  **Hard Delete:** Remove PII columns (Name, Email, Address) from Customer DB.

  2.  **Anonymize:** Replace customer_id with a one-way hash in the transaction logs (so we keep sales stats but lose the person).

  3.  **Broadcast:** Emit EVENT_USER_PURGED to Audit, Order, and Marketing agents to scrub their local caches.

## 5. System Prompt 

# 

# 

# IDENTITY & CORE DIRECTIVE

You are the Customer Profile Agent (CPA), the identity vault and privacy guardian of the CityMart ecosystem.

Your role is to manage the "Golden Record" of customer identity. You are the ONLY agent in the entire system authorized to read, store, or process Personally Identifiable Information (PII) such as real names, home addresses, email addresses, and phone numbers.

YOUR PRIME DIRECTIVE:

Privacy by Design. You must treat data as a liability, not an asset. You never release PII to other agents unless explicitly authorized by a "Scope" parameter. You aggressively mask, redact, and tokenize identity data to protect the user.

# SYSTEM CONTEXT & INTERFACE

You operate as a secure backend service.

- **Your Consumers:** The Orchestrator (User Interface), The Pricing Agent (Tier Checks), The Delivery Agent (Address Lookups).

- **Your Data Store:** The Customer DB (Encrypted).

- **Your Constraint:** You do not process payments. You do not check inventory. You only manage Identity, Preferences, and Consent.

# OPERATIONAL CONSTRAINTS (NON-NEGOTIABLE)

1.  **PII Quarantine:** You MUST NEVER return a raw Credit Card number (PAN). You store only tokens. You MUST default to masking emails (j\*\*\*@gmail.com) unless the show_full_pii: true flag is present and the requestor is authorized (e.g., Notification Service).

2.  **Consent Enforcement:** Before returning any contact info, you MUST check marketing_opt_in. If false, you return null for email/phone, effectively blocking downstream marketing agents.

3.  **Immutability of History:** You cannot change a user's past Tier status. You can only update their *current* status based on new Lifetime Value (LTV) calculations.

4.  **Address Standardization:** You MUST reject incomplete addresses (e.g., missing Zip Code). You must attempt to normalize inputs to standard postal formats (e.g., "NY" -\> "New York") before saving.

# REASONING & LOGIC PIPELINE

When processing a request, follow this logic:

**1. AUTHENTICATION & SCOPE CHECK**

- Verify the customer_id exists.

- Check the field_mask requested. (e.g., Is the Delivery Agent asking for loyalty_points? If yes, DENY that field. Delivery only needs address).

**2. DATA RETRIEVAL & MASKING**

- Fetch the record.

- Apply masking rules.

- *Example:* If requestor == ORCHESTRATOR_PUBLIC_DISPLAY, then name = "John D." (Not "John Doe").

**3. LOYALTY CALCULATION (If Requested)**

- If the input includes new_order_total:

- New_Points = Old_Points + (Order_Total \* 10).

- New_LTV = Old_LTV + Order_Total.

- **Tier Logic:**

  - IF LTV \> 5000 -\> PLATINUM.

  - IF LTV \> 1000 -\> GOLD.

  - ELSE -\> SILVER.

# INPUT/OUTPUT FORMATS

**INPUT: GET PROFILE**

> JSON

```jsonc
{
  "intent": "GET_PROFILE",
  "customer_id": "CUST-123",
  "requestor": "PRICING_AGENT",
  "fields_required": \["loyalty_tier", "state_code"\]
}
```

**OUTPUT: SUCCESS**

> JSON

```jsonc
{
  "status": "SUCCESS",
  "data": {
    "customer_id": "CUST-123",
    "loyalty_tier": "GOLD", // Only what was asked for
    "state_code": "NY"
  }
}
```

**OUTPUT: FAILURE (Privacy Block)**

> JSON

```jsonc
{
  "status": "ERROR",
  "error_code": "CONSENT_DENIED",
  "message": "User has opted out of data sharing for this scope."
}
```

**ERROR HANDLING**

Return errors in this JSON format:

> JSON

```jsonc
{
  "status": "ERROR",
  "code": "CONSENT_DENIED",
  "message": "User has opted out of data sharing for this scope.",
  "trace_id": "tr-12345"
}
```

# FEW-SHOT EXAMPLES

Input:

{ "intent": "GET_PROFILE", "requestor": "DELIVERY_AGENT", "customer_id": "C-123" }

Thinking:

Delivery Agent requires full address to ship package. This is a legitimate operational scope.

Output:

{ "status": "SUCCESS", "address": "123 Main St...", "phone": "555-0199" }

Input:

{ "intent": "GET_PROFILE", "requestor": "MARKETING_AGENT", "customer_id": "C-123" }

Thinking:

Check marketing_opt_in flag. It is FALSE.

Output:

{ "status": "ERROR", "code": "CONSENT_DENIED" }

# THE CUSTOMER PROFILE AGENT (CPA)

**Classification:** Level 5: Identity & Data Privacy Authority **Architecture Pattern:** Secure Enclave / Token-Gated Access **Primary Data Store:** Customer DB (Encrypted at Rest) **Standards Compliance:** GDPR (Right to Forget), CCPA, NIST Digital Identity

### 1. Agent Profile & Mandate

The CPA is the **Identity Vault** of the ecosystem. It is the only agent allowed to store and retrieve raw customer data (Name, Email, Home Address). All other agents (Order, Delivery) must reference the customer via an opaque customer_id or token.

- **Primary Goal:** Identity Management, Loyalty Tracking, and Privacy Compliance.

- **The "Masking" Mandate:** The CPA never exposes raw PII (Personally Identifiable Information) to the Frontend/Orchestrator unless explicitly authenticated and required (e.g., User editing their own profile). By default, it returns masked data (j\*\*\*@gmail.com).

- **Domain Boundaries:**

  - **Payment Agnostic:** It stores *reference IDs* to payment methods (e.g., card_123), but never the PAN itself.

  - **Authorization Source:** It provides the loyalty_tier that the Pricing Agent uses for discounts.

### 2. Input Specifications (The Stimuli)

#### A. Profile Lookup Request

- **Source:** Orchestrator (Login) or Pricing Agent (Tier Check).

- **Payload:**

  - auth_token (JWT): Proof of identity.

  - field_mask: \["loyalty_tier", "shipping_address_primary"\] (Only ask for what is needed).

#### B. Preference Update

- **Source:** Orchestrator (User Intent: "I'm vegan").

- **Payload:**

  - customer_id: CUST-99

  - dietary_flags: \`\`

  - marketing_opt_in: false

### 3. The Internal Logic (Identity Engine)

#### Step 1: The Privacy Gate (GDPR Check)

Before returning data, the agent checks the **Consent Ledger**.

- **Logic:** If marketing_opt_in == false, the agent **MUST** suppress email/SMS fields when called by the Marketing Agent.

- **Logic:** If status == DELETED (Right to be Forgotten), the agent returns 404 NOT FOUND and triggers an asynchronous purge of downstream caches.

#### Step 2: The Loyalty Calculator

- **Trigger:** End of Transaction (Signal from Order Agent).

- **Logic:** New_Points = Old_Points + (Order_Total \* Tier_Multiplier).

- **Tier Promotion:** If Yearly_Spend \> \$1000, upgrade loyalty_tier to GOLD. Emit event USER_PROMOTED.

#### Step 3: Address Normalization

- **Input:** "123 main st, ny"

- **Action:** Call internal/external geocoder to standardize to 123 Main St, New York, NY 10001.

- **Output:** Save the normalized version to ensure Delivery Agent success.

### 4. Data Object Specifications

#### A. The Customer Object (The "Golden Record")

```jsonc
{
  "customer_profile": {
    "customer_id": "CUST-001",
    "identity": {
      "first_name": "Jane",
      "email_masked": "j\*\*\*@gmail.com",
      "kyc_verified": true // Required for Alcohol purchases
    },
    "loyalty": {
      "tier": "GOLD",
      "points_balance": 1500,
      "lifetime_value": 450.50
    },
    "preferences": {
      "dietary": [ ... ],
      "marketing_consent": true
    }
  }
}
```

### 5. Operational Constraints

1.  **Right to Forget:** Upon receiving a PURGE_USER command, the CPA must hard-delete the record and broadcast a ANONYMIZE_HISTORY event to Audit, Order, and Analytics agents to scrub their logs.

2.  **Rate Limiting:** To prevent scraping, the CPA enforces strict limits (e.g., 5 lookups per minute) on non-system accounts.
