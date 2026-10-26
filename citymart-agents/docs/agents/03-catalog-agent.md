# THE CATALOG AGENT

## 1. Concept 

Imagine a library where the librarian knows every book not just by title, but by meaning. If you ask for "a sad story about a robot," they find The Iron Giant even though the word "sad" isn't in the title.

The **Catalog Agent** is that librarian for the store.

- **It understands Intent:** If a user searches for "headache stuff," it returns *Ibuprofen*, not a book about headaches.

- **It enforces Safety:** If a user is under 21, the Catalog Agent physically hides alcohol from the search results. It doesn't just say "You can't buy this"; it pretends the item doesn't exist for that user.

- **It is the "Source of Truth":** It defines what the product *is* (Gluten-Free, 500g, Spicy). It does *not* know if it is in stock (Inventory's job) or how much it costs today (Pricing's job).

## 2. Technical Architecture Specification

**For Engineers & Architects**

| **Attribute**        | **Specification**                                                    |
|----------------------|----------------------------------------------------------------------|
| **Primary Function** | Semantic Search & Attribute Retrieval                                |
| **Latency Budget**   | **\< 150ms** (Must feel instant for type-ahead search)               |
| **Consistency**      | **Eventual**. Updates (e.g., new product images) can lag by minutes. |
| **Dependencies**     | None. It is a self-contained read-replica of the Master PIM.         |

### 2.1 Critical Operational Constraints

1.  **Inventory Agnostic:** The Catalog Agent must **NEVER** output "In Stock" or "Out of Stock." It returns a valid_product record. The Orchestrator must combine this with Inventory data later.

2.  **Safety Filtering:** If the user_context indicates age \< 21 or allergy = PEANUTS, the Agent must apply a **Hard Filter** at the database level to exclude restricted items from the result set entirely.

3.  **Variant Collapse:** To prevent search pollution, the Agent must group variants. Searching for "T-Shirt" should return one result ("Cotton Tee") with a metadata flag has_variants:, rather than returning 50 separate rows for every size and color.

## 3. Input & Output Specifications

### A. Search Request (Input)

- **Source:** Orchestrator (User Query).

**Schema:**

```jsonc
{
  "intent": "FIND_PRODUCT",
  "query_string": "something spicy for dinner",
  "user_constraints": {
    "age_verified": false,
    "dietary_flags":
  },
  "limit": 5
}
```

### B. Product Object (Output)

- **Description:** The "Digital Twin" of the physical item.

**Schema:**

```jsonc
{
  "products":
},
```

"relevance_score": 0.95

}

\]

> }

**SYSTEM PROMPT**  
  
IDENTITY & CORE DIRECTIVE

You are the Catalog Agent, the semantic librarian of the CityMart ecosystem.

Your role is to bridge the gap between vague human intent (e.g., "I want a snack") and precise internal SKUs (e.g., CHIPS-BBQ-04). You are the sole authority on product attributes, descriptions, images, and safety classifications.

YOUR PRIME DIRECTIVE:

Accuracy & Safety. You must ensure that users find exactly what they are looking for, but you must NEVER show a restricted item (Alcohol, Tobacco) to an unverified user. You prioritize semantic relevance over keyword matching.

# SYSTEM CONTEXT & BOUNDARIES

- **What you DO:** Search, Filter, Classify, Describe.

- **What you do NOT do:**

  - You do **NOT** know the Price. (That is the Pricing Agent).

  - You do **NOT** know the Stock Level. (That is the Inventory Agent).

  - You do **NOT** know the Location. (That is the Navigation Agent).

- **Interaction:** You receive queries from the Orchestrator and return JSON lists of SKUs.

# INTERNAL LOGIC & SAFETY RULES

1.  **Semantic Expansion:** You must expand queries conceptually. If a user asks for "Coke", search for Coca-Cola, Soda, Beverages.

2.  **The Safety Hard-Filter:**

    - Check the user_constraints input field.

    - IF age_verified == false -\> **EXCLUDE** all items where category == ALCOHOL \| TOBACCO.

    - IF allergy == PEANUTS -\> **EXCLUDE** all items where allergens contains PEANUT.

    - *Constraint:* It is better to return 0 results than to return a dangerous product.

3.  **Variant Logic:** Do not return 10 rows for "Nike Shirt (S)", "Nike Shirt (M)". Return **ONE** Parent SKU and list the available variants in the metadata.

# RESPONSE FORMAT

You must return a strictly formatted JSON object. Do not include conversational filler ("Here is what I found").

**Success Output:**

> JSON

```jsonc
{
  "status": "SUCCESS",
  "matches_found": 1,
  "products": [ ... ],
  "is_restricted": false
}
```

\]  
}

**No Results Output:**

> JSON

```jsonc
{
  "status": "NO_MATCH",
  "suggestion": "Did you mean 'Almond Milk'?"
}
```

# FEW-SHOT REASONING

Input: Query: "Headache medicine", User: {Age: 16}

Thinking:

1.  Concept "Headache" maps to Category PAIN_RELIEF.

2.  Target items: Ibuprofen, Acetaminophen, Aspirin.

3.  Safety Check: Aspirin is age-restricted in some regions, but generally safe. Alcohol is strictly forbidden.

4.  Result: Return Ibuprofen SKUs.  
    > Output: JSON list of Ibuprofen products.

Input: Query: "Whiskey", User: {Age: 16}

Thinking:

1.  Concept "Whiskey" is Category ALCOHOL.

2.  Safety Check: User is 16 (Under 21).

3.  Action: HARD BLOCK.  
    > Output: { "status": "NO_MATCH", "message": "No results found for your criteria." } (Do not explain why if it reveals the existence of the item, simply hide it).

# AGENT 02: THE CATALOG AGENT

Classification: Level 3: Domain Expert (Information Retrieval)

Architecture Style: Read-Heavy / Vector-Enhanced

Primary Data Store: Product Master Database (SQL) + Vector Index (Pinecone/Milvus)

### 1. Agent Profile & Mandate

The Catalog Agent serves as the **Semantic Librarian** of the ecosystem. It is responsible for bridging the gap between vague human intent (e.g., "something for a headache") and precise stock keeping units (e.g., SKU-998: Ibuprofen 200mg).

- **Primary Goal:** Discovery & Accuracy. Ensure users find the correct product, and that product metadata (Allergens, Safety Flags) is immutable and accurate.

- **Key Constraint:** **Stateless regarding Inventory.** The Catalog Agent does not know stock levels. It will return a product even if it is sold out (it relies on the Inventory Agent to apply the "Out of Stock" badge later).

- **Authority:** It is the *sole* owner of product descriptions, images, taxonomy, and safety attributes.

### 2. Input Specifications (The Stimuli)

The Catalog Agent accepts requests primarily from the **Orchestrator** (User Search) and occasionally from the **Delivery Agent** (Physical Dimensions lookup).

#### A. Search & Discovery Input

- **search_query** *(String)*: Raw text from the user.

  - *Example:* "gluten free snacks for kids"

- **active_filters** *(Object)*:

  - category_id (String \| Null): cat_snacks

  - price_range (Object): { "max": 20.00 }

  - dietary_flags (Array): \`\`

- **user_context** *(Object)*:

  - personalization_vector: User's affinity (e.g., usually buys Organic).

#### B. Direct Lookup Input

- **sku_list** *(Array)*: \`\`

- **requested_fields** *(Enum)*: \["weight_kg", "dimensions", "hazmat_class"\] (Used by Logistics).

### 3. The Internal Logic (The Search Pipeline)

The Catalog Agent executes a **3-Stage Retrieval Process** to ensure relevance and safety.

#### Stage 1: Semantic Parsing & Expansion

Unlike a basic database LIKE %query% search, the Agent uses vector embeddings to understand intent.

- **Input:** "Something for a headache."

- **Expansion Logic:**

  1.  Map "Headache" \$\rightarrow\$ Category: PAIN_RELIEF.

  2.  Expand synonyms: \[Aspirin, Ibuprofen, Acetaminophen, Paracetamol\].

  3.  **Negative Filtering:** If query contains "Baby," exclude "Adult Strength" products.

#### Stage 2: The Filtering Sieve (Hard Constraints)

The Agent applies rigid filters that cannot be fuzzy-matched.

- **Dietary Logic:** If filter="VEGAN", strictly exclude any SKU where tags does not contain VEGAN. Do not guess.

- **Safety Logic:** If user_context.age \< 21, strictly exclude SKUs with flag: ALCOHOL. Return a sanitized list (or a "Restricted" placeholder).

#### Stage 3: Variant Collapse & Ranking

To prevent search result clutter (e.g., showing 50 rows of the same T-Shirt in different sizes), the Agent groups results.

- **Logic:** Identify Parent_SKU (The Style) vs Child_SKU (The Variant).

- **Action:** Return the Parent SKU with a variant_count: 5 indicator.

- **Ranking:** Boost scores based on user_context (e.g., if User buys "Store Brand," boost Store Brand items).

### 4. Data Object Specifications (The "Virtual" Schema)

The Catalog Agent outputs strictly formatted JSON objects. These are the "Truth" used by all other agents.

#### A. The Product Object (Standard)

```jsonc
{
  "product_summary": {
    "sku_id": "CK-OAT-CRUNCH-01",
    "name": "CityMart Oat Crunch Cereal",
    "brand": "CityMart Organics",
    "parent_sku": null, // Is a standalone item
    "category_path": [ ... ],
    "attributes": {
      "weight_g": 500,
      "is_perishable": true,
      "tax_code": "FOOD_ESSENTIAL" // Critical for Pricing Agent
    },
    "safety_flags": {
      "is_alcohol": false,
      "allergens": [ ... ],
      "choking_hazard": false
    },
    "media": {
      "thumbnail_url": "https://img.citymart.com/t/ck-oat.jpg",
      "ar_model_url": null
    }
  }
}
```

#### B. The Search Result Object (List)

```jsonc
{
  "search_meta": {
    "total_results": 14,
    "did_you_mean": "Cereal", // Auto-correction from "Cerel"
    "query_interpretation": "Category: Breakfast OR Snack"
  },
  "results":
}
```

### 5. Operational Constraints

1.  **Read-Only Authority:** The Catalog Agent *never* creates products on the fly. It can only read from the Master Data set approved by the Merchandising Team.

2.  **Price Agnostic:** The Catalog Agent stores a MSRP (Manufacturer Suggested Retail Price) for display sorting, but it **MUST NOT** be treated as the final transaction price. (The Pricing Agent applies discounts/dynamic pricing).

    - *Constraint:* The Output must label price as estimated_base_price to avoid user confusion.

3.  **Truth in Labeling:** If a product has a contains_peanuts flag in the DB, the Agent **CANNOT** hide or suppress this field, even if the user explicitly asks for "Peanut free items" and the search result is empty. It is better to return 0 results than to return an unsafe product.

### 6. Error Handling & Edge Cases

| **Scenario**                                  | **Agent Reaction**                                          | **Output Message**                                                            |
|-----------------------------------------------|-------------------------------------------------------------|-------------------------------------------------------------------------------|
| **Zero Results**                              | Trigger **Semantic Fallback**. Look for related categories. | "We couldn't find 'Gatorade', but here are our 'Sports Drinks'."              |
| **Ambiguous Query** (e.g., "Apple")           | Trigger **Disambiguation**. Return categories, not items.   | "Did you mean Fresh Apples or Apple Electronics?"                             |
| **Conflicting Filters** (e.g., "Vegan Steak") | Trigger **Logic Explanation**.                              | "There are no items that match 'Steak' and 'Vegan'. Removing 'Steak' filter." |
| **Database Timeout**                          | Return **Cached Top Sellers**.                              | "Search is momentarily down. Here are today's popular items."                 |

### 7. System Interdependencies

- **Upstream:**

  - **Admin/Merchandising System:** Feeds new SKUs into the Catalog Agent.

- **Downstream:**

  - **Inventory Agent:** Uses sku_id from Catalog to check stock.

  - **Pricing Agent:** Uses tax_code from Catalog to calculate tax.

  - **Delivery Agent:** Uses weight_g and dimensions from Catalog to calculate shipping costs.
