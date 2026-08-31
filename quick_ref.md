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





