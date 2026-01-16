# Archaeology Log: Vendure Shipping Zone Workflow
 
**Platform:** Vendure Headless Commerce  
**Task:** Map the "Oceania Shipping Zone" workflow (Track B)  
**Approach:** Documentation analysis (demo wasn't accessible)  

---

## Why I Couldn't Use the Demo

Right off the bat - I tried logging into the Vendure demo (admin/admin) but everything showed "No results found." The catalog was empty, shipping methods were empty, zones were empty. It looked like either the demo was reset recently or having issues.

So I pivoted to a documentation-first approach. Honestly, this might've been a blessing in disguise because it forced me to really understand the API structure rather than just clicking around.

---

## The Documentation Journey

### Finding My Way Around

Started at https://docs.vendure.io and immediately searched for "shipping zone." Found a few things:
- The GraphQL Admin API reference
- Some guides on shipping configuration  
- Type definitions for Zone and ShippingMethod

The docs are pretty solid actually. GraphQL schema is well documented with all the field types marked.

### First Pass: Understanding Zones

Looked at the Zone type definition:

```graphql
type Zone {
  id: ID!
  name: String!
  members: [Country!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

Pretty straightforward. The `!` marks mean required fields, which is helpful. So I know:
- `name` is required when creating
- `members` is a list of Country objects
- System generates the `id`

Then I checked the mutation:

```graphql
createZone(input: CreateZoneInput!): Zone!
```

And the input type:

```graphql
input CreateZoneInput {
  name: String!
}
```

This should be step 1.

---

## Discovery #1: The Country ID Problem

Looking at the Zone type, I see:

```graphql
members: [Country!]!
```

So it returns full Country objects. But then I looked at how you ADD countries:

```graphql
addMembersToZone(
  zoneId: ID!
  memberIds: [ID!]!
): Zone!
```

- `memberIds`, not `members`. And it's just IDs, not country names or codes.

So if the user says "add Australia and New Zealand," I can't just send:
```json
["Australia", "New Zealand"]  //  won't work
```

Or even:
```json
["AU", "NZ"]  //  also won't work
```

I need the actual database IDs for these countries. Which means there's a prerequisite step: look up the countries first.

Found the query:
```graphql
query {
  countries {
    items {
      id
      code  
      name
    }
  }
}
```

So the workflow needs to:
1. Query countries
2. Filter to find Australia (probably where code == "AU")
3. Extract the ID
4. Same for New Zealand
5. Then use those IDs in addMembersToZone

**The gap:** User says "Australia" → API needs some ID (not sure of format yet - could be "T_1" or UUID or just "1")

This is a classic semantic gap. The UI dropdown probably does this lookup behind the scenes.

---

## Discovery #2: ConfigurableOperation - The Confusing Part

I was looking at ShippingMethod and saw:

```graphql
type ShippingMethod {
  calculator: ConfigurableOperation!
  checker: ConfigurableOperation!
  ...
}
```


Clicked through to the type definition:

```graphql
type ConfigurableOperation {
  code: String!
  args: [ConfigArg!]!
}

type ConfigArg {
  name: String!
  value: String!
}
```


**Available shipping calculators:**
- `default-shipping-calculator` - flat rate
- `percentage-shipping-calculator` - percentage of order total
- `weight-based-shipping-calculator` - based on weight

So when the user says "flat rate $15," I need to translate that to:

```json
{
  "calculator": {
    "code": "default-shipping-calculator",
    "arguments": [
      {"name": "rate", "value": "???"}
    ]
  }
}
```

## Discovery #3: Prices Are in Cents

Found an example in the shipping calculator docs:

```json
{
  "arguments": [
    {"name": "rate", "value": "1500"}  // $15.00
  ]
}
```

So $15 becomes 1500. It's in cents, as an integer.

This makes sense - I've seen this in Stripe and other payment systems. Avoids floating point issues. If you store $19.99 as a float you might get 19.9899999, but 1999 cents is exact.

**Transformation needed:**
- User input: "$15" or "15"
- Strip dollar sign: "15"  
- Convert to decimal: 15.0
- Multiply by 100: 1500
- Use as string in argument: "1500"


Yeah, the ConfigArg type shows:
```graphql
value: String!
```

So even though it's a number conceptually, the API expects a string. Makes sense for a generic plugin argument structure.

---

## Discovery #4: The Checker Configuration

So there's also a `checker` field that's required:

```graphql
checker: ConfigurableOperation!
```

This determines when the shipping method is available. Found in the docs:

**Common checker:** `default-shipping-eligibility-checker`

It has arguments too:
```json
{
  "checker": {
    "code": "default-shipping-eligibility-checker",
    "arguments": [
      {"name": "orderMinimum", "value": "0"}
    ]
  }
}
```

The `orderMinimum` is the minimum order value to qualify. Setting it to "0" means it's available for all orders.

- we want the Oceania flat rate available to everyone.

---

## Discovery #5: Hidden Required Fields

Looking at the CreateShippingMethodInput, there are fields the user never mentioned:

```graphql
input CreateShippingMethodInput {
  code: String!                    # ← What's this?
  fulfillmentHandler: String!      # ← And this?
  checker: ConfigurableOperationInput!
  calculator: ConfigurableOperationInput!
  translations: [ShippingMethodTranslationInput!]!  # ← And this??
}
```

### The `code` field

Separate from `name`. It's like a slug or API identifier. Found a note in docs:

> Must match pattern: ^[a-z0-9-]+$

So only lowercase, numbers, and hyphens. No spaces, no underscores.

For "Oceania Flat Rate" I'd generate: `oceania-flat-rate`

### The `fulfillmentHandler` field

Searched for this. Found:

> Determines how orders are fulfilled. Default: "manual-fulfillment"

Okay so just use "manual-fulfillment" as the standard option.

### The `translations` field


 Vendure is built for multi-language from the ground up. You have to provide at least one translation:

```json
{
  "translations": [
    {
      "languageCode": "en",
      "name": "Oceania Flat Rate",
      "description": "Flat rate shipping for Oceania region"
    }
  ]
}
```

This is actually pretty smart for internationalization, but means more boilerplate for simple cases.

---

## Discovery #6: The Missing Link

After creating:
1. Zone (Oceania)
2. Countries added to zone (AU, NZ)  
3. Shipping method (flat rate $15)

I searched for "assign shipping method" and found a mention of `assignShippingMethodsToChannel` but the docs weren't super clear on this.

**Possibilities:**
- Maybe it's automatic through the channel system?
- Maybe you select the zone when creating the shipping method? (didn't see that in the input type)
- Maybe there's a separate assignment step?

This is where having a live system would help. I'd create the method and see if there's a zone selector somewhere, or check the admin UI to understand the relationship.

---

## Discovery #7: The Code Pattern Gotcha

When testing my knowledge, I realized the `code` field has strict rules. From the docs:

> Pattern: ^[a-z0-9-]+$

What this means:
-  `oceania-flat-rate` (good)
-  `Oceania Flat Rate` (has spaces and capital letters)
-  `oceania_flat_rate` (underscores not allowed)
-  `océania-flat-rate` (accented chars not allowed)

The generation logic:
```javascript
function makeCode(name) {
  return name
    .toLowerCase()
    .replace(/\s+/g, '-')      // spaces to hyphens
    .replace(/[^a-z0-9-]/g, '') // remove bad chars
    .replace(/-+/g, '-');       // collapse multiple hyphens
}
```

## What The Full Workflow Looks Like

Here's the sequence:

**Step 0 (Prerequisite):** Query countries to get IDs
```graphql
query {
  countries {
    items { id code name }
  }
}
```
Extract IDs where code = "AU" and code = "NZ"

**Step 1:** Create zone
```graphql
mutation {
  createZone(input: { name: "Oceania" }) {
    id
    name
  }
}
```
Save the returned zone ID

**Step 2:** Add countries to zone
```graphql  
mutation {
  addMembersToZone(
    zoneId: "ZONE_ID_FROM_STEP_1"
    memberIds: ["AU_ID", "NZ_ID"]
  ) {
    id
    members { name }
  }
}
```

**Step 3:** Create shipping method
```graphql
mutation {
  createShippingMethod(input: {
    code: "oceania-flat-rate"
    fulfillmentHandler: "manual-fulfillment"
    checker: {
      code: "default-shipping-eligibility-checker"
      arguments: [{ name: "orderMinimum", value: "0" }]
    }
    calculator: {
      code: "default-shipping-calculator"  
      arguments: [
        { name: "rate", value: "1500" }
        { name: "taxRate", value: "0" }
      ]
    }
    translations: [{
      languageCode: "en"
      name: "Oceania Flat Rate"
      description: "Flat rate shipping for Oceania region"
    }]
  }) {
    id
    code
  }
}
```

**Step 4 (Maybe?):** Assign to zone/channel
*Not sure about this step - needs verification*

---

## Semantic Gaps Summary

All the places where user language differs from API requirements:

User Says: "Australia"

API Needs: Country ID (unknown format)

Why: Names aren't unique, IDs are

User Says: "$15"

API Needs: "1500" as string

Why: Cents, not dollars, and must be string for ConfigArg

User Says: "Flat Rate"

API Needs: {code: "default-shipping-calculator", arguments: [...]}

Why: It's a plugin system

User Says: "Oceania Flat Rate" (as name)

API Needs: Also need code: "oceania-flat-rate"

Why: Separate slug vs display name

User Says: Implicitly "available to all"

API Needs: checker: {code: "...", arguments: [{name: "orderMinimum", value: "0"}]}

Why: Eligibility is configurable

User Says: Nothing about fulfillment

API Needs: fulfillmentHandler: "manual-fulfillment"

Why: Required but uses sensible default

User Says: Nothing about language

API Needs: translations: [{languageCode: "en", ...}]

Why: Multi-language architecture
---


## What I'd Test First With Live Access

If I could access a working Vendure instance:

1. **Country ID format** - Are they numeric? UUIDs? The T_ pattern I've seen mentioned?
2. **The zone-method relationship** - How do they actually get linked?
3. **ConfigArg validation** - What happens if I send wrong argument names?
4. **Error messages** - What if the code already exists? Or zone name is duplicate?
5. **Tax rate handling** - Can taxRate be "0" or does it need actual tax config?

---

## Assumptions I Made

Things I assumed without being 100% certain:

1. There's a default channel that's being used implicitly
2. Country IDs are stable (won't change between queries)  
3. taxRate can be "0" (saw it in an example but not confirmed)
4. "manual-fulfillment" is the right default fulfillment handler
5. The zone-method relationship might be automatic via channel config
6. Description in translations can be a simple string (didn't see validation rules)

All of these would need verification with a live system before going to production.

---

## How This Compares to Other Platforms

**GraphQL platforms I've used:**
- Shopify: Similar plugin-based approach for shipping, also has the input/output type separation
- GitHub API: Simpler, fewer nested configurations

**REST platforms:**
- Salesforce: Uses Apex classes for customization, similar plugin concept but different implementation
- Stripe: Also uses cents for amounts, same pattern

**The ConfigurableOperation pattern** is new to me but makes sense. It's like dependency injection for shipping logic.

---

## Final Thoughts

---

**Status:** Ready to validate with live testing  
**Next step:** Get access to working Vendure instance and verify assumptions  

---
