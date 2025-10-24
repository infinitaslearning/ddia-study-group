
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

## The Layering of Abstractions

Each layer hides complexity by providing a clean data model:

1. **Application**: Objects, data structures, APIs
2. **Database**: JSON, XML, tables, graphs
3. **Database internals**: Bytes in memory, disk, network
4. **Hardware**: Electrical currents, magnetic fields, photons

**Key insight**: Each model represents how we think about the problem

---

# Part 1: Relational Model vs Document Model

---

## The Birth of Relational Databases (1970s)

**Edgar Codd's Relational Model (1970)**

- Data organized in **relations** (tables)
- Each relation is a collection of **tuples** (rows)
- Dominated data storage for 30+ years

**Main competitors (now historical curiosities)**:
- Network model (CODASYL)
- Hierarchical model (IMS)

---

## The NoSQL Movement (2010s)

**Why "Not Only SQL"?**

1. **Scalability**: Need for large-scale datasets & high write throughput
2. **Open Source**: Preference for free/open software
3. **Specialized queries**: Operations not well supported by relational model
4. **Flexibility**: Desire for more dynamic/expressive data models

**Not a replacement**: Polyglot persistence is the reality

---

## The Driving Forces Behind NoSQL

**Real-world examples**:
- **Google**: BigTable for web indexing
- **Amazon**: Dynamo for shopping cart
- **Facebook**: Cassandra for inbox search
- **LinkedIn**: Voldemort for data storage

**Key realization**: "One size fits all" doesn't work at scale

---

## Network Model vs Hierarchical Model (Historical Context)

**Before Relational (1960s-1970s)**:

**Hierarchical Model** (IBM's IMS):
- Tree structure, parent-child relationships only
- Had to navigate using pointers
- Difficult to model many-to-many relationships

**Network Model** (CODASYL):
- Graph-like structure with records and links
- Manual pointer navigation
- Query: "Follow this link, then that link..."

**Codd's Innovation**: Hide implementation details behind declarative interface

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

## Document vs Relational: Key Differences

| Aspect | Document Model | Relational Model |
|--------|----------------|------------------|
| **Schema** | Schema-on-read (implicit) | Schema-on-write (explicit) |
| **Joins** | Limited, poor support | Powerful, well-supported |
| **Data locality** | Excellent for reads | Can require multiple queries |
| **Many-to-one** | Requires joins or denormalization | Natural with foreign keys |
| **Many-to-many** | Difficult | Natural with junction tables |

---

## Data Locality: A Closer Look

**Document databases** store related data together:
```javascript
// One network request, all data together
db.users.findOne({_id: 123})
// Returns entire document with positions, education, etc.
```

**Relational databases** may require multiple queries:
```sql
SELECT * FROM users WHERE id = 123;
SELECT * FROM positions WHERE user_id = 123;
SELECT * FROM education WHERE user_id = 123;
-- Or one query with JOINs, but still scattered on disk
```

**Trade-off**: Locality is great if you need the whole document, wasteful if you need one field

---
layout: two-cols-header
---

## Normalization vs Denormalization

**The Normalization Principle**: Store each fact in one place

**Example**: User's region

::left::

**Normalized (Relational)**:
```
users: id | name | region_id
regions: id | name
```
✅ Change "Greater Seattle" once, affects all users<br>
✅ Consistent spelling and naming<br>
✅ Easy to query all users in a region

::right::

**Denormalized (Document)**:
```json
{"name": "Martin", "region": "Greater Seattle Area"}
{"name": "Sarah", "region": "Greater Seattle area"}
```
❌ Inconsistent spelling<br>
❌ Updates require changing multiple documents<br>
✅ Faster reads (no joins)

<style>
.two-cols-header {
  column-gap: 20px;
}
</style>

---

## The Many-to-One Problem

**Real example from the book**:
- LinkedIn profiles
- User lives in "Greater Seattle Area"
- Should this be a string or an ID?

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 40px; margin-top: 20px;">

<div>

**String (denormalized)**:
- ✅ Simple
- ❌ Inconsistent ("Greater Seattle", "Seattle Area", "Seattle, WA")
- ❌ Hard to update
- ❌ No localization

</div>

<div>

**ID (normalized, many-to-one)**:
- ✅ Consistent
- ✅ Easy to update
- ✅ Can localize
- ❌ Requires join

</div>

</div>

<div style="margin-top: 30px; text-align: center;">

**Kleppmann's point**: Many-to-one relationships don't fit well in document model

</div>

---

## The Many-to-Many Challenge

**Example**: Resume/CV database

- Users can endorse other users' skills
- Users can work at the same organization
- Organizations have multiple users
- Users have multiple skills

```
users ←→ endorsements ←→ users
users ←→ employment ←→ organizations
users ←→ user_skills ←→ skills
```

**In relational databases**: Junction tables, straightforward

**In document databases**:
- Denormalize (data duplication, update anomalies)
- Application-side joins (slow, complex)
- Avoid the feature (limiting)

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

## Schema Flexibility

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

**Trade-off**: Flexibility vs. Guarantees

---

## Schema Changes: The Reality

**Schema-on-read is NOT schema-less**:
- Schema exists, but implicit (in application code)
- Still need to handle old and new formats
- Multiple versions of code must coexist

**Schema-on-write challenges**:
- Migrations can be slow on large tables
- Requires downtime or complex online schema changes
- But: database enforces correctness

**Best practice**: Document your schema even in "schemaless" databases!

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

## Declarative vs Imperative

**Imperative** (How to do it):
```javascript
function getSharkSpecies() {
  var sharks = [];
  for (var i = 0; i < animals.length; i++) {
    if (animals[i].family === "Sharks") {
      sharks.push(animals[i]);
    }
  }
  return sharks;
}
```

**Declarative** (What you want):
```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

---

## Benefits of Declarative Languages

1. **Concise and easier to work with**
2. **Hides implementation details**: Database can optimize
3. **Parallel execution**: No specified order
4. **Future-proof**: Can add indexes without changing queries

**Examples**: SQL, CSS, XPath

---

## Declarative Languages Enable Optimization

**Example**: Query optimizer in action

```sql
SELECT * FROM users WHERE age > 18 AND country = 'Italy';
```

**Database can choose**:
- Scan `age` index first, then filter by country
- Scan `country` index first, then filter by age
- Use composite index on (country, age)
- Full table scan if no index

**You don't care HOW** - only WHAT result you want

**In imperative code**: You'd have to rewrite the loop to optimize

---

## CSS: Another Declarative Language

**CSS Example**:
```css
li.selected > p {
  background-color: blue;
}
```

**Imperative equivalent (JavaScript)**:
```javascript
var liElements = document.getElementsByTagName("li");
for (var i = 0; i < liElements.length; i++) {
  if (liElements[i].className === "selected") {
    var children = liElements[i].childNodes;
    for (var j = 0; j < children.length; j++) {
      if (children[j].nodeName === "P") {
        children[j].style.backgroundColor = "blue";
      }
    }
  }
}
```

**Browser can optimize CSS**: Different rendering strategies, parallel rendering, GPU acceleration

---

## MapReduce Querying

**MongoDB MapReduce Example**:

```javascript
db.observations.mapReduce(
  // Map function
  function map() {
    var year = this.observationTimestamp.getFullYear();
    var month = this.observationTimestamp.getMonth() + 1;
    emit(year + "-" + month, this.numAnimals);
  },
  // Reduce function
  function reduce(key, values) {
    return Array.sum(values);
  },
  {
    query: { family: "Sharks" },
    out: "monthlySharkReport"
  }
);
```

**Middle ground**: Neither fully declarative nor imperative

---

## MongoDB Aggregation Pipeline

**More declarative alternative**:

```javascript
db.observations.aggregate([
  { $match: { family: "Sharks" } },
  { $group: {
      _id: {
        year: { $year: "$observationTimestamp" },
        month: { $month: "$observationTimestamp" }
      },
      totalAnimals: { $sum: "$numAnimals" }
  }}
]);
```

**Trend**: Moving toward more declarative interfaces
