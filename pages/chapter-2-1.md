## Agenda - Full Hour Session

### Part 1: Presentation
1. Introduction: Why Data Models Matter
2. Relational Model vs Document Model
3. Query Languages
4. Graph-Like Data Models

### Part 2: Discussion
5. Structured Q&A and Group Reflection

---

## Why Data Models Matter

> "Data models are perhaps the most important part of developing software, because they have such a profound effect on what the software above them can and can't do."
>
> — Martin Kleppmann

---

# Part 1: Relational Model vs Document Model

---

## The Evolution: Relational to NoSQL

**1970: Edgar Codd's Relational Model**
- Tables & SQL - dominated for 40+ years
- Replaced hierarchical (IMS) & network (CODASYL) models

**2010: NoSQL Movement**
- **Drivers**: Scale (billions of users), flexibility, specialized queries
- **Pioneers**: Google BigTable, Amazon Dynamo, Facebook Cassandra, LinkedIn Voldemort
- **Reality**: Not replacement - **polyglot persistence** (right tool for each job)

---

## Object-Relational Mismatch

**The Impedance Mismatch Problem**

```
Application Code (OOP)          Relational Database
┌─────────────────┐            ┌──────────────────┐
│   User Object   │            │   users table    │
│  ┌──────────┐   │            │  ┌────────────┐  │
│  │ name     │   │  ◄────►    │  │ user_id    │  │
│  │ email    │   │            │  │ name       │  │
│  │ positions│   │            │  │ email      │  │
│  └──────────┘   │            │  └────────────┘  │
└─────────────────┘            └──────────────────┘
                               ┌──────────────────┐
                               │ positions table  │
                               │  ┌────────────┐  │
                               │  │ user_id    │  │
                               │  │ job_title  │  │
                               │  │ org        │  │
                               │  └────────────┘  │
                               └──────────────────┘
```

**Solution**: ORMs (Object-Relational Mapping) or... document databases

---

## Document Model: MongoDB/CouchDB Style

**LinkedIn Profile Example**

```json
{
  "user_id": 123,
  "first_name": "Martin",
  "last_name": "Kleppmann",
  "positions": [
    {
      "job_title": "Software Engineer",
      "organization": "LinkedIn",
      "start_date": "2015-01-01"
    }
  ],
  "education": [
    {
      "school": "Cambridge University",
      "degree": "PhD in Computer Science"
    }
  ]
}
```

**Advantages**:
- Better locality
- Schema flexibility
- Closer to application data structures

<style>
pre[class*='language-'],
code[class*='language-'],
div[class*='language-'] {
    display: block !important;
    line-height: 1.3 !important;
    font-size: 0.7em !important;
}
</style>

---

## Key Trade-offs: Document vs Relational

| Aspect | Document | Relational |
|--------|----------|------------|
| **Schema** | Schema-on-read (flexible) | Schema-on-write (guarantees) |
| **Joins** | Weak support | First-class, optimized |
| **Locality** | Excellent (if need all) | Selective (if need parts) |
| **Relationships** | One-to-many ✅ Many-to-many ❌ | All relationships ✅ |

**The Normalization Problem**:
- **Many-to-one** (user → region): Relational uses foreign keys. Document needs denormalization (inconsistent) or app-side joins (slow).
- **Many-to-many** (LinkedIn endorsements): Relational uses junction tables. Document struggles - denormalize (update anomalies), app joins (N+1 queries), or avoid feature.

**Bottom line**: Document = tree-shaped data. Relational = interconnected data.

---

## When to Use Document Databases

✅ **Good fit**:
- Document-like structure (tree of one-to-many relationships)
- No need for many-to-many relationships
- Application-driven data access patterns

❌ **Poor fit**:
- Highly interconnected data
- Need for complex joins
- Many-to-many relationships
- Need for strong consistency across documents

---

## Schema Flexibility & Evolution

**Schema-on-read** (Document):
```javascript
// No migration needed
if (user.name) {
  return user.name;
} else {
  return user.first_name + " " + user.last_name;
}
```

**Schema-on-write** (Relational):
```sql
-- Migration required
ALTER TABLE users ADD COLUMN full_name TEXT;
UPDATE users SET full_name = first_name || ' ' || last_name;
ALTER TABLE users DROP COLUMN first_name, DROP COLUMN last_name;
```

**Reality**: Schema-on-read ≠ schemaless! Still need versioning in code. Modern tools help both: pt-online-schema-change, MongoDB validation.

---

## Convergence: Hybrid Approaches

**Relational databases adding document features**:
- PostgreSQL: JSON/JSONB columns
- MySQL: JSON data type
- SQL:2016 standard: JSON functions

**Document databases adding relational features**:
- MongoDB: Multi-document transactions (4.0+)
- MongoDB: $lookup for joins
- CouchDB: Views with map-reduce

**The trend**: Best of both worlds

---

# Part 2: Query Languages

---

## Query Languages: Declarative Wins

**Imperative** (How): Loop, check, push → 8 lines
```javascript
var sharks = [];
for (var i = 0; i < animals.length; i++) {
  if (animals[i].family === "Sharks") sharks.push(animals[i]);
}
```

**Declarative** (What): Just say what you want → 1 line
```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

**Why declarative wins**: Concise, hides implementation, parallelizable, future-proof (add index → automatically faster!)

**Modern trend**: MongoDB moved from MapReduce (imperative functions) to Aggregation Pipeline (declarative stages). Everyone moving toward declarative: Spark SQL.

**Examples**: SQL (databases), CSS (browsers), XPath (XML), GraphQL (APIs)
