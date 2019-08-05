## A light weight library to access dynamodb tables

It provides the following common methods for each dynamodb table / indexes:

- create
- delete
- get
- getAll
- query
- update

### Installation

```
npm install --save aqua-dynamo
```

### Prerequisite

- Have your [aws credentials configured](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-credentials-node.html) properly.
  For example, you can configure it in `~/.aws/credentials` file.

- Have your [aws region](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-region.html) configured using **environment variable**. Here is an example to set the environment variable at run time,

  Mac:

  - `AWS_REGION=us-east-1 node app.js`

  Windows:

  - Install [cross-env](https://www.npmjs.com/package/cross-env) npm package

    `npm install --save-dev cross-env`

  - Run

    `cross-env AWS_REGION=us-east-1 node app.js`

- This package is intended to be used in services such as AWS Lambda, which has aws-sdk installed by default in the environment. Therefore, aws-sdk is not listed as a dependency for this package to reduce package size. During development, make sure you have aws-sdk installed globally if you haven't done so already `npm install aws-sdk -g`. If you are working with an environment that doesn't have aws-sdk installed, make sure you install the aws-sdk pacakge in your project that uses `aqua-dynamo`.


### Usage Example:

Assume you have a simple Dynamodb table, whose table name is `Users`, and hashKey `userId`.

```javascript
const TableModel = require("aqua-dynamo");

const userTable = new TableModel({ name: "Users", hashKey: "userId" });

await userTable.create({
  userId: "357",
  username: "Juan Guzman"
});

const result = await userTable.get({ userId: "357" }); // returns the user
```

#### Update:

```javascript
const key = {
  userId: "357"
};
const newFields = {
  username: "c.lee"
};
const result = await userTable.update(key, newFields);
console.log(result.Attributes.username); // c.lee
```

#### Delete:

```javascript
const key = {
  userId: "357"
};
const result = await userTable.delete(key);
console.log(result.Attributes.key); // 357
```

#### getAll:

Get all your users in the table, library will send as many requests as needed to get all the data.

```javascript
const result = await userTable.getAll({
  pagination: false
});
console.log(result.Items);
```

In case you want pagination,

```javascript
let result = await userTable.getAll();
console.log(result.Items); // Users with pagination. If LastEvaluatedKey exist, it means there are more data to get

if (result.LastEvaluatedKey) {
  const result = await userTable.getAll({}, { ExclusiveStartKey: result.LastEvaluatedKey });
  console.log(result.Items); // The next batch of users
}
```

### Extends Table model:

Example of a complex table model with **global secondary indexes** and **customized methods**:

```javascript
const TableModel = require("aqua-dynamo");

const UserBalanceHistoricalTable = {
  name: "UserBalanceHistorical",
  hashKey: "usernameSymbol",
  sortKey: "itemDateTime"
};
const UserBalanceIndexUsername = { name: "UsernameIndex", hashKey: "username" };

class UserBalance extends TableModel {
  constructor() {
    super({
      ...UserBalanceTable,
      indexes: [UserBalanceIndexUsername]
    });
  }

  // Define you customized method of the model
  getUserBalancesByUsername(username) {
    return this.query({
      tableName: this.tableName,
      indexName: UserBalanceIndexUsername.name,
      hashKey: UserBalanceIndexUsername.hashKey,
      hashKeyValue: username
    });
  }
}

module.exports = UserBalance;
```

Make use of the UserBalance table, query with variant dynamodb parameters such as Limit and FilterExpression.

```javascript
const UserBalances = require("./UserBalance");

const UserBalancesTable = new UserBalances();

console.error = jest.fn();

describe("Method that will grab queried items from  a given table", () => {
  test("Grab query by indexName", async () => {
    const result = await UserBalancesTable.queryByUsername({ username: "aleung" });
    expect(result).not.toBeNull();
    expect(result.Items).not.toBeNull();
    expect(result.ScannedCount).toBeDefined();
  });

  test("Using options, Limit the amount of items present(small limit)", async () => {
    const result = await UserBalancesTable.queryByUsername({ username: "aleung" }, { Limit: 2 });
    expect(result).not.toBeNull();
    expect(result.Count).toBe(2);
    expect(result.LastEvaluatedKey).toBeDefined();
  });
  test("Using Projection Expression options, only show certain attributes of an item", async () => {
    const result = await UserBalancesTable.queryByUsername(
      { username: "aleung" },
      { ProjectionExpression: "availableBalance, symbol" }
    );
    expect(result).not.toBeNull();
    const attributes = Object.keys(result.Items[0]);
    expect(attributes).toEqual(expect.arrayContaining(["availableBalance", "symbol"]));
    expect(attributes).not.toEqual(
      expect.arrayContaining(["pendingTransfer", "totalBalance", "depositAddress"])
    );
  });
  /**
   * FilterExpression still queries over the whole table and filters from there, not more efficient
   */
  test("Grab info using filterExpression", async () => {
    const result = await UserBalancesTable.queryByUsername(
      { username: "aleung" },
      {
        FilterExpression: "availableBalance > :availableBalance",
        ExpressionAttributeValues: { ":availableBalance": 0 }
      }
    );
    console.log(result);
  });
  /**
   * Consistent Read does not work on secondary index
   */
  test("Use consistent read on items", async () => {
    const result = await UserBalancesTable.query(
      { hashKeyValue: "aleung_BTC" },
      { ConsistentRead: true }
    );
    console.log(result);
  });
  test("Empty Items when a username you search for does not exist", async () => {
    expect.assertions(1);
    const data = await UserBalancesTable.queryByUsername({ username: "qleung" });
    expect(data.Items.length).toBe(0);
  });
});
```

### Sparse Index - Remove attributes

Sparse index is a way to design your secondary indexes so that only a small portion of the items will be stored in the index. It is used for performing more efficient query and scans. [More info here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-indexes-general-sparse-indexes.html).

When an item doesn't belong to the sparse index anymore, you'll need to remove the attribute that is used as the hashKey/sortKey of an item to take the item out from the index.

To do that, you can simply assign the attribute to `null` in the `update` method, and this library will handle the attribute removal for you.

For example, if we have some `address` items that can be assigned to users, when users signs up, we want to query for an available `address` efficiently, we can design the model as follows

```javascript
class Address extends TableModel {
  constructor() {
    super({
      ...DepositTable,
      indexes: [DepositIndexAvailable] // <-- Sparse Index
    });
  }

  create(item) {
    if (!item.symbol) {
      throw new Error("param symbol is required!");
    }
    return super.create({
      ...item,
      symbolTime: `${item.symbol}_${Date.now()}`,
      status: "available"
    });
  }

  queryOneAvailableAddress(symbol) {
    return super.query(
      {
        indexName: DepositIndexAvailable.name,
        hashKeyValue: "available", // <-- Use the Sparse Index - DepositIndexAvailable
        sortKeyOperator: "begins_with",
        sortKeyValue: symbol
      },
      {
        Limit: 1
      }
    );
  }

  useAddress(address) {
    return super.update({ address }, { status: null }); // <-- remove the element from the Sparse Index
  }
}
```

### Supported dynamodb operators

<!-- | Operators | =   | <   | <=  | >   | >=  | begins_with (or beginsWith ) | between |
| ---------- | --- | --- | --- | --- | --- | ---------------------------- | ------- | -->

|          Operators           |
| :--------------------------: |
|              =               |
|              <               |
|              <=              |
|              >               |
|              >=              |
|           between            |
| begins_with (or beginsWith ) |

### Known Issues:

When querying with the options `Select: "COUNT"`, throws an error due to the Items being undefined. This is a rare use case, but one should keep that in mind.

```javascript
const result = await UserBalancesTable.queryByUsername({ username: "aleung" }, { Select: "COUNT" });
```