### Index
Explain command run chesinappudu stage value `COLLSCAN` vunte index lenattu. 
```JavaScript
db.customers.find({email: "alice@example.com"}).explain("executionStats")
```
Output lo important point:
```
stage: 'COLLSCAN'
nReturned: 0
totalKeysExamined: 0
totalDocsExamined: 3
```
Meaning:
- **COLLSCAN** → collection motham scan chesindi
- **totalDocsExamined: 3** → 3 documents check chesindi
- **totalKeysExamined: 0** → index use cheyyaledu
- **nReturned: 0** → `alice@example.com` ane email mana data lo ledu
Production lo millions of documents unte **COLLSCAN** expensive avvachu.

#### Create index on Email
```JavaScript
db.customers.createIndex({email: 1})
```
Expected output approximately: `email_1`
Then verify: (Get list of indexes)
```JavaScript
db.customers.getIndexes()
```
**Same query malli run cheddaam**
```JavaScript
db.customers.find({email: "alice@example.com"}).explain("executionStats")
```
Ee sari output value `stage: 'IXSCAN'`. 


#### Unique Index
Example: Customer email should be unique
Currently mana index `{ email: 1 }` but `isUnique: false`. So theoretically it allows duplicate emails.
First remove current index
```JavaScript
db.customers.dropIndex("email_1")
```
Expected result: `{ nIndexesWas: 2, ok: 1 }`
Then verify: `db.customers.getIndexes()`. Ikkada `_id_` maatrame vuntundhi.

#### Create Unique Index
```JavaScript
db.customers.createIndex(
  { email: 1 },
  { unique: true }
)
```
Expected: `email_1`
Then verify:
```JavaScript
ecommerce> db.customers.getIndexes()
[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { email: 1 }, name: 'email_1', unique: true }
]
```
#### Test duplicate email insert
```JavaScript
db.customers.insertOne({
  customerId: "CUST004",
  name: "Test Duplicate",
  email: "ravi@example.com",
  city: "Berlin",
  status: "active",
  createdAt: new Date()
})
```
Expected error roughly: `MongoServerError: E11000 duplicate key error collection: ecommerce.customers index: email_1 dup key:....`

#### Compound Index
Example:
```JavaScript
db.customers.find({
  city: "Berlin",
  status: "active"
})
```
Imagine application frequently asks:
| "Berlin lo active customers evaru?"
Ikkada compound index useful:
```JavaScript
{ city: 1, status: 1 }
```
But before creating it, we should understand index field order.
MongoDB compound index: `{ city: 1, status: 1 }`. Means `city → status`
So, index is primarily ordered by `city`, and with in each city by `status`.

**ESR rule**
MongoDB compound index design chesetappudu commonly:
E = Equality, S = Sort, R = Range

Create `{city: 1, status: 1}` compound index.
```JavaScript
db.customers.createIndex({ city: 1, status: 1 })
```
Get the index list
```JavaScript
db.customers.getIndexes()
```
Run explain
```JavaScript
db.customers.find(
  { city: "Berlin", status: "active" }
).explain("executionStats")
```













