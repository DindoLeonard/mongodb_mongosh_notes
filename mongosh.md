# Mongodb indexes

---

look for "**IMPORTANT** " tag for topics that are important

---

- Why indexes?

  - No index will do a collection scan
  - If index, it will go to index scan instead of collection scan

- What is index?
  - Ordered list of all value of the fold document
  - Indexes don't come for free

---

## What and Why?

- Indexes allow you to retrieve data more efficiently (if used correctly) because your queries only have to look at a subset of all documents
- You can use:
  - single-field
  - compound
  - multi-key (array)
  - text indexes
- Indexes don't come for free, they will slow down your writes
  <br>
  <br>

---

## Queries & Sorting

- Indexes can be used for both queries and efficient sorting
- Compound indexes can be used as a whole or in a "left-to-right" (prefix) manner (e.g. only consider the "name-age" compound index)
  <br>
  <br>

---

<br>
<br>
<br>

# Getting started

### pre-requisites

- connect to mongoshell with mongosh
- create your own collection and documents
- then do this sample below

```javascript
// creates index
db.contacts.createIndex({ 'dob.age': 1 });

// explains index
db.contacts.explain('executionStats').find({ 'dob.age': { $gt: 60 } });

// getting index information
db.contact.getIndexes();
```

## Understanding Index Restriction

- Index would be slower if you'll return a big collection

## Creating Compound Indexes

```javascript
// will save index of dob.age_1_gender_1
db.contacts.createIndex({ 'dob.age': 1, gender: 1 });

// will look at "dob.age_1_gender_1" index
db.contacts.find({ 'dob.age': 100 });
db.contacts.find({ 'dog.age': 35, gender: 'male' });

// won't be able to index acan for second one
db.contacts.find({ gender: 'male' });
```

## Using Indexes for Sorting

- if not using indexes when do a sorting on large amount of document, it will time-out, hash threshold of 32mb of memory

```javascript
// will do an index scan for both
db.contacts.find({ 'dob.age': 35 }).sort({ gender: 1 });
```

## Configuring Indexes

- https://docs.mongodb.com/manual/indexes/

```javascript
db.contacts.find({<key>: <value>}, {<optionkey>: <value>})
```

## Understanding Partial filters

```js
// will create an index of age, but only for gender that is male
db.contacts.createIndex(
  { 'dob.age': 1 },
  { partialFilterExpression: { gender: 'male' } }
);
```

- good for only indexing what is only necessary

## Applying the Partial index

- if index is set but with no fields provided upon creating, it will error

```js
// will only elements on index where email field exist
// to avoid class with unique
// only if it exists
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { email: { $exist: true } } }
);
```

## Understanding the Time-To-Live (TTL) index

```js
// will only work on date field
// will expire after x seconds
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 10 });
db.sessions.insertOne({ data: 'exampleData', createdAt: new Date() });
```

## Query Diagnosis & Query Planning

- explain()
  - "queryPlanner"
    - Show Summary for Executed Query + Winning Plan
  - "executionStats"
    - Show detailed summary for executed query + winning plan + possibly rejected plans
  - "allPlansExecution"
    - Possibly

## Efficient Queries & Covered queries

- check for these fields

- To determine if the query is efficient
  - Check milliseconds process time
  - IXSCAN typically beats COLLSCAN
  - \# of keys (in index) examined
  - \# of documents examined
  - \# of documents returned

## Understanding Covered Queries

- Will not examine documents but will return entirely from inside the index
- Will return only those fields that are indexed

```js
// create customers
db.customers.insertMany([
  { name: 'Dindo', age: 29, salary: 3000 },
  { name: 'Manu', age: 29, salary: 4000 },
]);

// index name in accesin
db.customers.createIndex({ name: 1 });

// getting document with scanning the documents
db.customers.explain('executionStats').find({ name: 'Dindo' });
db.customers.find({ name: 'Dindo' });

// getting the document without scanning the documents and only getting index value
// db.<collection>.find(<filter>, <rejection>)
db.customers
  .explain('executionStats')
  .find({ name: 'Dindo' }, { _id: 0, name: 1 });
db.customers.find({ name: 'Dindo' }, { _id: 0, name: 1 });

// won't do covered query if you include field that are not indexed
db.customers
  .explains('executionStats')
  .find({ name }, { _id: 0, name: 1, salary: 1 });
db.customers.find({ name }, { _id: 0, name: 1, salary: 1 });
```

## How MongoDB rejects a plan

```js
db.customers.createIndex({ age: 1, name: 1 });

db.custoemrs.find({ name: 'Dindo', age: 29 });
```

- Winning Plans

  - Will do a race with indexes, with any index, the winning plan is the one that is executed
  - Rejected plans is going to be rejected

- Clearing the winning plan from cache
  - Stored forever? No
  - Removed from cache
    - Write threshold of (currently 1000)
    - Index is rebuilt
    - Other indexes are added or removed
    - mongodb server is restarted

# Using Multi-Key Indexes

## **IMPORTANT**

- good for multiple key indexes

```js
db.contacts.insertOne({
  name: 'Dindo',
  hobbies: ['Programming', 'Gaming'],
  addresses: [{ street: 'Guadalajara Street' }, { street: 'Private Street' }],
});

/**
 * ARRAY
 * Will create index on hobbies array
 */
db.createIndex({ hobbies: 1 });
// Will create multi-key index based on the array
db.explain('executionStats').find({ hobbies: 'Programming' });

/**
 * OBJECT
 * Will create index and hold an object
 */
db.createIndex({ addresses: 1 });
// Will do IX scan and look for the object
db.explain('exec');

// WIll do COL scan instead of IX, because it doesn't hold 'addresses.street'
db.explain('execustionStats').find({ 'addresses.street': 'Guadaljara street' });

// Cannot index parallel arrays
db.contacts.creacteIndex({ addresses: 1, hobbies: 1 });
```

# Understanding "text" indexes

## **IMPORTANT**

- really powerful and much faster than regular expressions (regex)

```js
db.products.insertMany([
  {
    title: 'A Book',
    description: 'This is an awesome book about a young artists',
  },
  {
    title: 'Red T-Shirt',
    description: "This T-Shirt is red and it's pretty awesome",
  },
]);

// adding text index
// it will breakdown the words, remove all stop words, and store keyword in an array
db.products.createIndex({ description: 'text' });

// It will look for the description text indexes
db.products.find({ $text: { $search: 'book' } }); // result A Book
db.products.find({ $text: { $search: 'awesome' } }); // result both
db.products.find({ $text: { $search: 'red' } }); // result Red T-Shirt
db.prodcuts.find({ $text: { $search: 'red book' } }); // result both
```

# Text Indexes & Sorting

- for sorting searches base on search

```js
db.products.insertMany([
  {
    title: 'A Book',
    description: 'This is an awesome book about a young artists',
  },
  {
    title: 'Red T-Shirt',
    description: "This T-Shirt is red and it's pretty awesome",
  },
]);

// This query
db.products.find({ $text: { $search: 'awesome t-shirt' } });
```

Will result to

```json
[
  {
    "_id": "example-id-1",
    "title": "A Book",
    "description": "This is an awesome book about a young artists"
  },
  {
    "_id": "example-id-2",
    " title": "Red T-Shirt",
    "description": "This T-Shirt is red and it's pretty awesome"
  }
]
```

```js
// to see the textScore for matches example
// it will be added in the field
// db.<collection>.find(${text: {$search: <string>}}, {<variable-for-score>: {$meta: "textScore"}})
db.products.find(
  { $text: { $search: 'awesome t-shirt' } },
  { score: { $meta: 'textScore' } }
);

// to sort base on the textScore
db.products
  .find(
    { $text: { $search: 'awesome t-shirt' } },
    { score: { $meta: 'textScore' } }
  )
  .sort({ score: { $meta: 'textScore' } });
```

# Creating Combined Text Indexes

## **IMPORTANT**

- you can't have 2 text index fields
- you need to drop current index to add another combined index
- you can merge 2 field and created a combined text index instead

```js
// to check the current text index on the collection
db.products.getIndexes();

// the result will be an objects of indexes, find the text index and get it's name and drop the index
db.products.dropIndexe('description_text');

// check if index is registered
db.products.getIndexes();

db.products.insertOne({ title: 'A ship', description: 'Floats perfectly' });

// Search for the ship title and can also search for description
db.products.find({ $text: { $search: 'ship' } });
db.products.find({ $text: { $search: 'perfect' } });
```

# Using Text Indexes To Exclude Words

- can also rule out words
- add "-" on the string

```js
// it will look for words that has awesome that doesn't have the word shirt
db.products.find({ $text: { $search: 'awesome -shirt' } });
```

# Setting the Default Language & Using Weights

Will put a weight on the field of the document for its score

```js
// will put more weight on description 10 times more than title
db.products.createIndex(
  { title: 'text', description: 'text' },
  {
    default_language: 'english',
    weights: {
      title: 1,
      description: 10,
    },
  }
);

// to check indexes
db.products.getIndexes()[
  // will result to
  ({ v: 2, key: { _id: 1 }, name: '_id_' },
  {
    v: 2,
    key: { _fts: 'text', _ftsx: 1 },
    name: 'title_text_description_text',
    weights: { description: 10, title: 1 },
    default_language: 'english',
    language_override: 'language',
    textIndexVersion: 3,
  })
];
```

# Building indexes

## **IMPORTANT**

- Foreground
  - not good for production as it will block the query process
  - the collection will be locked
- Background
  - good for production database
  - will not lock the collection while creating index in the background

```js
// foreground
db.ratings.createIndex({ age: 1 });

// will execute only after the foreground process is done
db.ratings.findOne();
```

```js
// background
db.ratings.createIndex({ age: 1 }, { background: true });

// this will execute immedietely while the background index is still ongoing
db.ratings.findOne();
```
