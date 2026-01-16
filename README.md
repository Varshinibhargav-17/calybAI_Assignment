# Calyb Engineering Challenge - Track B Submission

**Challenge:** Map the "Oceania Shipping Zone" workflow for Vendure

**Approach:** Documentation-driven workflow mapping

**Why this approach:** Vendure demo was empty when accessed, so I worked from 
the GraphQL API documentation to map the complete workflow.

---

## Files

- **workflow_map.json** - Complete workflow specification with dependency tracking
- **approach.md** - Methodology and generalization strategy
- **archaeology_log.md** - Detailed discovery process and findings

---

## Key Highlights

### Semantic Mappings
Documented all transformations between user intent and API requirements:
- Country names → IDs (lookup required)
- "$15" → "1500" (currency to cents)
- "Flat Rate" → ConfigurableOperation plugin structure

### Dependency Chain
```
Step 0: Lookup country IDs
  ↓
Step 1: Create zone → zone_id
  ↓  
Step 2: Add countries (needs zone_id + country_ids)
  ↓
Step 3: Create shipping method
  ↓
Step 4: Zone assignment (needs verification)
```