# Very helpful ORM for node.js

`npm install sqlagent`

- Currently supports __PostgreSQL__ only
- MySQL is in progress
- Simple and powerful

__IMPORTANT__:

- code is executed as it is added
- rollback is executed automatically when is transaction enable

## Initialization

### Basic initialization

```javascript
var Agent = require('sqlagent/pg');
var sql = new Agent('connetion-string-to-postgresql');
```

### Initialization for total.js

```javascript
// Below code rewrites total.js database prototype
require('sqlagent/pg').init('connetion-string-to-postgresql', [debug]);

// var sql = DATABASE([ErrorBuilder]);
var sql = DATABASE();
// sql === SqlAgent
```

## Usage

### Select

```plain
instance.select([name], table, [columns])
```

- `name` (String) is an identificator for results, optional (default: internal indexer)
- `table` (String) table name, the library automatically creates SQL query
- `colums` (String, Array or Object (keys will be as columns))
- __returns__ SqlBuilder

```javascript
var users = sql.select('users', 'tbl_user', '*');
users.where('id', '>', 5);
users.page(10, 10);

var orders = sql.select('orders', 'tbl_order', 'id, name, created');
orders.where('isremoved', false);
orders.page(10, 10);

var products = sql.select('products', 'tbl_products', ['id', 'name']);
products.between('price', 30, 50);
products.and();
products.where('isremoved', false);
products.limit(20);

var admin = sql.select('admin', 'tbl_admin', { id: true, name: 1, age: null });
// SELECT id, name, age
admin.where('hash', 'petersirka');
admin.first();

sql.exec(function(err, response) {
    console.log(response.users);
    console.log(response.products);
    console.log(response.admin);
});
```

### Insert

```plain
instance.insert([name], table, [value], [primaryId])
```

- `name` (String) is an identificator for results, optional (default: internal indexer)
- `table` (String) table name, the library automatically creates SQL query
- `value` (Object) optional (value can be SqlBuilder)
- __returns__ if value is undefined then __SqlBuilder__ otherwise __SqlAgent__

```javascript
sql.insert('user', 'tbl_user', { name: 'Peter', age: 30 });

var insert = sql.insert('log', 'tbl_logs');
insert.set('message', 'Some log message.');
insert.set('created', new Date());

sql.exec(function(err, response) {
    console.log(response.user);
    console.log(response.log);
});
```

### Update

```plain
instance.update([name], table, [value])
```

- `name` (String) is an identificator for results, optional (default: internal indexer)
- `table` (String) table name, the library automatically creates SQL query
- `value` (Object) optional (value can be SqlBuilder)
- __returns__ if value is undefined then __SqlBuilder__ otherwise __SqlAgent__

```javascript
var update1 = sql.update('user1', 'tbl_user', { name: 'Peter', age: 30 });
update1.where('id', 1);

// is same as

var update2 = sql.update('user2', 'tbl_user');
update2.where('id', 1);
update2.set('name', 'Peter');
update2.set('age', 30);

sql.exec(function(err, response) {
    console.log(response.user1);
    console.log(response.user2);
});
```

### Delete

```plain
instance.delete([name], table)
instance.remove([name], table)
```

- `name` (String) is an identificator for results, optional (default: internal indexer)
- `table` (String) table name, the library automatically creates SQL query
- __returns__ SqlBuilder

```javascript
var remove = sql.remove('user', 'tbl_user');
remove.where('id', 1);

sql.exec(function(err, response) {
    console.log(response.user);
});
```

### Aggregation

```plain
instance.count([name], table)
```

- __returns__ SqlBuilder

```javascript
var count = sql.count('users', 'tbl_user');
count.between('age', 20, 40);

sql.exec(function(err, response) {
    console.log(response.users);
});
```

---

```plain
instance.max([name], table, column)
instance.min([name], table, column)
instance.avg([name], table, column)
```

- __returns__ SqlBuilder

```javascript
var max = sql.max('users', 'tbl_user', 'age');
max.where('isremoved', false);

sql.exec(function(err, response) {
    console.log(response.users);
});
```

### Transactions

- rollback is performed automatically

```javascript
sql.begin();
sql.insert('tbl_user', { name: 'Peter' });
sql.commit();
```

## Special cases

### How to use latest primary id value for relations?

```javascript
// primary key is id + autoincrement
var user = sql.insert('user', 'tbl_user');
user.set('name', 'Peter');

var address = sql.insert('tbl_user_address');
address.set('id', sql.$$);
address.set('country', 'Slovakia');

sql.exec();
```

### How to use latest primary id value for multiple relations?

```javascript
// primary key is id + autoincrement
var user = sql.insert('user', 'tbl_user');
user.set('name', 'Peter');

sql.lock();
// is same as
// sql.put(sql.$$);

var address = sql.insert('tbl_user_address');
address.set('iduser', sql.$$); // adds latest primary id value
address.set('country', 'Slovakia');

var email = sql.insert('tbl_user_email');
email.set('iduser', sql.$$); // adds locked value
email.set('email', 'petersirka@gmail.com');
sql.unlock();

sql.exec();
```

### Preparing (dependencies)

- you can use multiple `sql.prepare()`

```javascript
var user = sql.update('user', 'tbl_user');
user.where('id', 20);
user.set('name', 'Peter');

var select = sql.select('address', 'tbl_address');
select.where('isremoved', false);
select.and();
select.where('city', 'Bratislava');
select.limit(1);

// IMPORTANT:
sql.prepare(function(error, response, resume) {
    sql.builder('address').set('idaddress', response.address.id);
    resume();
});

var address = sql.update('address', 'tbl_user_address');
address.where('iduser', 20);

sql.exec();
```

### Validation

- you can use multiple `sql.validate()`

```plain
sql.validate(fn)
```

```javascript
var select = sql.select('address', 'tbl_address');
select.where('isremoved', false);
select.and();
select.where('city', 'Bratislava');
select.limit(1);

// IMPORTANT:
sql.validate(function(error, response, resume) {
    
    if (!response.address) {
        error.push('Sorry, address not found');
        // cancel pending queries
        return resume(false);
    }
    
    sql.builder('user').set('idaddress', response.id);

    // continue
    resume();
});

var user = sql.update('user', 'tbl_user');
user.where('id', 20);
user.set('name', 'Peter');

sql.exec();
```

---

```plain
sql.validate(error_message, [result_name_for_validation])
```

- `error_message` (String / Error) - error message
- `result_name_for_validation` (String) a result to compare, optional and default: __latest result__

If the function throw error then SqlAgent cancel all pending queris (perform Rollback if the agent is in transaction mode) and executes callback with error.

```javascript
var select = sql.select('address', 'tbl_address');
select.where('isremoved', false);
select.and();
select.where('city', 'Bratislava');
select.limit(1);

// IMPORTANT:
sql.validate('Sorry, address not found');

var user = sql.select('user', 'tbl_user');
user.where('id', 20);

sql.validate('Sorry, user not found');
sql.validate('Sorry, address not found for the current user', 'address');

sql.exec();
```

## Global

### Skipper

```javascript
sql.select('users', 'tbl_users');
sql.skip(); // skip orders
sql.select('orders', 'tbl_orders');

sql.bookmark(function(error, response) {
    // skip logs
    sql.skip('logs');
});

sql.select('logs', 'tbl_logs');

sql.exec(function(err, response) {
    console.log(response); // --- response will be contain only { users: [] }
});
```

### Bookmarks

Bookmark is same as `sql.prepare()` function but without `resume` argument.

```javascript
sql.select('users', 'tbl_users');

sql.bookmark(function(error, response) {
    console.log(response);
    response['custom'] = 'Peter';
});

sql.select('orders', 'tbl_orders');

sql.exec(function(err, response) {
    response.users;
    response.orders;
    response.custom; // === Peter
});
```

### Error handling

```javascript
sql.select('users', 'tbl_users');

sql.validate(function(error, response, resume) {

    if (!response.users || respone.users.length === 0)
        error.push(new Error('This is error'));

    // total.js:
    // error.push('error-users-empty');

    resume();
});

sql.select('orders', 'tbl_orders');

// sql.validate([error message], [result name for validation])
sql.validate('error-orders-empty');
// is same as:
// sql.validate('error-orders-empty', 'orders');

sql.validate('error-users-empty', 'users');
```

### Predefined queries

```plain
Agent.query(name, query);
```

```javascript
Agent.query('users', 'SELECT * FROM tbl_users');
Agent.query('allorders', 'SELECT * FROM view_orders');

sql.query('users').where('id', '>', 20);
sql.query('orders', 'allorders').limit(5);

sql.exec(function(err, response) {
    console.log(response[0]); // users
    console.log(response.orders); // orders
});
```

## SqlBuilder

```javascript
// Creates SqlBuilder
var builder = sql.$;

builder.where('id', '<>', 20);
builder.set('isconfirmed', true);

// e.g.:
sql.update('users', 'tbl_users', builder);
sql.exec(function(err, response) {
    console.log(response.users);
})
```

---

### builder.set()

```plain
builder.set(name, value)
```

adds a value for update or insert

- `name` (String) a column name
- `value` (Object) a value

---

```plain
builder.set(obj)
```
adds a value for update or insert

---

### builder.sort()

```plain
builder.sort(name, [desc])
builder.order(name, [desc])
```
adds sorting

- `name` (String) a column name
- `desc` (Boolean), default: false

---

### builder.skip()

```plain
builder.skip(value)
```
skips records

---

### builder.take()

```plain
builder.take(value)
builder.limit(value)
```
takes records

---

### builder.page()

```plain
builder.page(page, maxItemsPerPage)
```
sets automatically sql.skip() and sql.take()

---

### builder.first()

```plain
builder.first()
```
sets sql.take(1)

---

### builder.where()

```plain
builder.where(name, [operator], value)
builder.push(name, [operator], value)
```
add a condition after SQL WHERE

- `name` (String) a column name
- `operator` (String), optional `>`, `<`, `<>`, `=` (default)
- `value` (Object)

---

### builder.group()

```plain
builder.group(names)
```
creates a group by in SQL query

- `name` (String or String Array)

---

### builder.having()

```plain
builder.having(condition)
```
adds having in SQL query

- `condition` (String), e.g. `MAX(Id)>0`

---

### builder.and()

```plain
builder.and()
```
adds AND to SQL query

---

### builder.or()

```plain
builder.or()
```
adds OR to SQL query

---

### builder.in()

```plain
builder.in(name, value)
```
adds IN to SQL query

- `name` (String), a column name
- `value` (String, Number or String Array, Number Array)

---

### builder.between()

```plain
builder.between(name, a, b)
```
adds between to SQL query

- `name` (String), a column name
- `a` (Number)
- `b` (Number)

---

### builder.sql()

```plain
builder.sql(query)
```
adds a custom SQL to SQL query

- `query` (String)

---

### builder.toString()

```plain
builder.toString()
```
creates escaped SQL query (internal)
