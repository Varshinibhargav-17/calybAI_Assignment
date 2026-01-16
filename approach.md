# Approach: Mapping SaaS Workflows from Documentation

## Why I Took This Approach

When I started this challenge, I tried logging into the Vendure demo. Everything was empty - "No results found" for products, zones, shipping methods, everything. Rather than wait around for it to reset, I decided to work from the documentation alone.

That said, I'm marking the parts of my workflow that would definitely need live validation before going to production.

---

## The Core Idea

Every SaaS workflow follows the same pattern:

**User says something** → **UI does something** → **API calls happen**

My job is to figure out what those API calls are, in what order, with what data.

For the Vendure shipping zone task:
- User says: "Create Oceania zone with Australia and New Zealand, $15 flat rate"
- UI would: Show forms, dropdowns, save buttons
- API needs: 3-4 specific GraphQL mutations with very particular data structures

The challenge is bridging the gap between casual human language and precise API requirements.

---

## How I Actually Did This

### Step 1: Find the Documentation

Started at https://docs.vendure.io and searched for "shipping zone."

Found:
- GraphQL Admin API reference
- Object type definitions (Zone, ShippingMethod, Country)
- Mutation signatures
- Some guides on shipping configuration

---

### Step 2: Understand the Data Types

**What I looked at:**

```graphql
type Zone {
  id: ID!
  name: String!
  members: [Country!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

The `!` marks mean "required." Super helpful. So I know:
- `name` is required
- `members` is a list of Countries (not just strings)
- System generates the `id`

Then I looked at Country:
```graphql
type Country {
  id: ID!
  code: String!
  name: String!
  enabled: Boolean!
}
```

so countries have IDs, codes (like "AU"), and names ("Australia").

---

### Step 3: Map the Mutations

Found these mutations:
- `createZone` - makes a new zone
- `addMembersToZone` - adds countries to a zone
- `createShippingMethod` - creates a shipping method

Checked their signatures:

```graphql
createZone(input: CreateZoneInput!): Zone!

addMembersToZone(
  zoneId: ID!
  memberIds: [ID!]!
): Zone!
```

- it takes `memberIds`, not country names. So I need a lookup step.

**The sequence started forming:**
1. Query countries to get IDs
2. Create the zone
3. Add countries using their IDs
4. Create the shipping method

---

### Step 4: The Deep Dive - ConfigurableOperation

This is where it got complicated. Looking at ShippingMethod:

```graphql
type ShippingMethod {
  calculator: ConfigurableOperation!
  checker: ConfigurableOperation!
  ...
}
```

What's a ConfigurableOperation? Clicked through:

```graphql
type ConfigurableOperation {
  code: String!
  args: [ConfigArg!]!
}
```
- it's a plugin system. The `code` identifies which plugin, and `args` configure it.

**Available shipping calculators:**
- `default-shipping-calculator` - flat rate
- `percentage-shipping-calculator` - percentage
- `weight-based-shipping-calculator` - by weight

So "flat rate $15" translates to:
```json
{
  "calculator": {
    "code": "default-shipping-calculator",
    "arguments": [
      {"name": "rate", "value": "1500"}
    ]
  }
}
```

Found in an example that prices are in cents. $15 = 1500 cents.

Also discovered that even though it's a number, it needs to be a string because `ConfigArg.value` is type `String!`.

---

### Step 5: Finding Hidden Required Fields

When I looked at `CreateShippingMethodInput`, there were fields I didn't expect:

```graphql
input CreateShippingMethodInput {
  code: String!                    # What's this?
  fulfillmentHandler: String!      # And this?
  checker: ConfigurableOperationInput!
  calculator: ConfigurableOperationInput!
  translations: [ShippingMethodTranslationInput!]!  # Really?
}
```

Had to research each one:

**`code`:** Like a slug. Must match pattern `^[a-z0-9-]+$`. Auto-generate from name.

**`fulfillmentHandler`:** Determines how orders are fulfilled. Default is `"manual-fulfillment"`.

**`checker`:** Determines when shipping is available. Use `"default-shipping-eligibility-checker"` with `orderMinimum: "0"` to make it always available.

**`translations`:** Vendure requires at least one translation even for single-language sites. Multi-language architecture built in from the start.

---

### Step 6: Mapping Semantic Gaps

This is where I listed every place the user's language differs from the API's language:

User Says: "Australia"

API Needs: Country ID

How to Bridge: Query countries, filter by name or code, extract ID

User Says: "$15"

API Needs: "1500"

How to Bridge: Strip $, parse number, multiply by 100, stringify

User Says: "Flat Rate"

API Needs: {code: "default-shipping-calculator", arguments: [...]}

How to Bridge: Map UI term to plugin code

User Says: "Oceania Flat Rate"

API Needs: Also need code: "oceania-flat-rate"

How to Bridge: Generate slug version

User Says: (implied: always available)

API Needs: checker: {code: "...", arguments: [{name: "orderMinimum", value: "0"}]}

How to Bridge: Use default checker with 0 minimum
---

### Step 7: Design the JSON Structure

I needed a format that captures:
- The sequence of steps
- What each step does (mutation/query name)
- What inputs it needs and where they come from
- What outputs it produces and where they're used
- Dependencies between steps
- Transformations required

I looked at workflow formats from tools like Airflow and AWS Step Functions for inspiration, then adapted it.

**Key design decision:** Never hardcode dynamic values. If a step needs an ID from a previous step, mark it as `"OUTPUT_FROM_STEP_1"` or similar, not a fake ID like `"123"`.

---

## The Methodology (Generalized)

Here's the process I'd use for ANY SaaS platform:

### Phase 1: Find the Documentation

**Where to look:**
- `platform.com/api`
- `platform.com/docs`
- `developers.platform.com`
- Google: "[platform] API documentation"

**What to find:**
- API reference (all endpoints/mutations)
- Object/type definitions
- Example requests and responses
- Guides and tutorials

---

### Phase 2: Understand the Types

**For GraphQL:**
- Read the schema definitions
- Note which fields are required (`!` marker)
- Understand relationships (which fields reference other types)

**For REST:**
- Look at JSON schemas
- Check OpenAPI/Swagger docs
- Note required vs optional fields

**What to ask:**
- What does this entity look like?
- What fields can I set vs what's system-generated?
- What are the data types?

---

### Phase 3: Find the Operations

**For GraphQL:**
- Look at Mutations (for create/update/delete)
- Look at Queries (for reading data)
- Check their input types

**For REST:**
- POST endpoints (create)
- GET endpoints (read)
- PUT/PATCH endpoints (update)
- DELETE endpoints (delete)

**Map to user intent:**
- "Create a zone" → `createZone` mutation
- "Add countries" → `addMembersToZone` mutation

---

### Phase 4: Identify Dependencies

**Look for:**
- IDs being passed between operations
- Fields that reference other entities
- Order requirements (must create X before Y)

**Create a dependency graph:**
```
Step 1 (creates Zone)
    ↓ produces zone_id
Step 2 (adds members) ← needs zone_id
    ↓
Step 3 (creates method) ← needs zone_id (maybe)
```

**Detection rules:**
- If Step B's input includes a field that Step A returns, that's a dependency
- If the docs say "must exist before," that's a dependency
- If it's logically impossible to do B before A, that's a dependency

---

### Phase 5: Map Semantic Gaps

**Process:**
1. List what the user provides (in plain language)
2. List what the API expects (exact field names and types)
3. Document the transformation needed

**Common patterns:**

**Direct mapping:**
- User: "Name" → API: `name: String`
- Just pass through

**Type transformation:**
- User: "$15" → API: `1500` (integer)
- Multiply by 100

**Lookup required:**
- User: "Australia" → API: `countryId: "T_123"`
- Query first, extract ID

**Complex mapping:**
- User: "Flat Rate" → API: `{code: "...", arguments: [...]}`
- Map to structure based on docs

---

### Phase 6: Structure It

**Design a JSON format that includes:**
- Step sequence
- API operation name
- Inputs (with sources)
- Outputs (with usage)
- Dependencies
- Transformations
- Validation rules

**Key principle:** Make it machine-executable. A robot should be able to read this JSON and perform the workflow.

---

## How This Works on Other Platforms

### Salesforce (REST API)

**Step 1:** Find docs at developer.salesforce.com

**Step 2:** Look at Object definitions (Account, Lead, Opportunity)

**Step 3:** Use Describe API to get field metadata:
```
GET /services/data/v52.0/sobjects/Lead/describe
```

**Step 4:** Map operations:
- Create Lead → `POST /services/data/v52.0/sobjects/Lead`
- Update Lead → `PATCH /services/data/v52.0/sobjects/Lead/{id}`

**Step 5:** Handle lookups (AccountId, OwnerId) same way as country IDs

**Semantic gaps:**
- Picklist values (UI shows "Hot" → API needs exact string "Hot")
- Currency fields (decimal vs integer depends on org settings)
- Date formats (ISO 8601)

---

### Jira (REST API)

**Step 1:** Find docs at developer.atlassian.com

**Step 2:** Look at Issue types, Fields, Workflows

**Step 3:** Map operations:
- Create issue → `POST /rest/api/3/issue`
- Transition issue → `POST /rest/api/3/issue/{issueIdOrKey}/transitions`

**Step 4:** Handle custom fields (field_10001, field_10002)

**Semantic gaps:**
- Issue type names → IDs
- Custom field names → field IDs
- Transition names → transition IDs

---

### The Universal Pattern

Regardless of platform:

1. **Find the docs** (REST, GraphQL, SOAP, whatever)
2. **Understand the data model** (what are the entities?)
3. **Find CRUD operations** (how do I create/read/update/delete?)
4. **Map dependencies** (what order must things happen?)
5. **Identify semantic gaps** (user language vs API language)
6. **Structure it** (make it executable)

The specific syntax changes (REST vs GraphQL vs SOAP), but the process is the same.

---

## Techniques I Used

### String Similarity for Field Mapping

When the API field name isn't obvious, use similarity matching:

```python
def similar(ui_label, api_field):
    # Remove spaces, lowercase both
    normalized_ui = ui_label.replace(" ", "").lower()
    normalized_api = api_field.lower()
    
    # Exact match
    if normalized_ui == normalized_api:
        return 1.0
    
    # One contains the other
    if normalized_ui in normalized_api or normalized_api in normalized_ui:
        return 0.8
    
    # Use Levenshtein distance for fuzzy match
    return levenshtein_ratio(normalized_ui, normalized_api)
```

**Example:**
- UI: "Product Name" → API: `productName` (0.8 - contains)
- UI: "Description" → API: `description` (1.0 - exact)

---

### Type Analysis for Transformations

Look at mismatches between UI display and API requirements:

**Currency fields:**
- UI shows: `$` prefix, 2 decimals
- API expects: Integer in cents
- Transform: Strip $, parse float, multiply 100, convert to int

**Date fields:**
- UI shows: MM/DD/YYYY
- API expects: ISO 8601 (YYYY-MM-DD)
- Transform: Parse and reformat

**Boolean fields:**
- UI shows: Checkbox
- API expects: `true`/`false` or `1`/`0`
- Transform: Map checked state

---

### Following Type References

In GraphQL, if a field is type `Country`, click through and read the Country definition.

In REST with JSON Schema, if a field has `$ref: "#/definitions/Country"`, look up that definition.

This tells you:
- What fields the nested object has
- Whether you need to provide the whole object or just an ID
- What the relationship structure is

---
