---
title:  "Leveraging the Power of JavaScript Generator Functions: Keep Database Batch Operations clean and maintainable"
date:   2023-06-14 22:00:00 +0200
excerpt: "In Node.js applications, querying a database in batches can be a common requirement when dealing with large datasets. Generator functions can streamline the implementation to keep the code clean and maintainable."
---


**In Node.js applications, querying a database in batches can be a common requirement when dealing with large datasets. In this blog post, we'll explore how generator functions can streamline the implementation to keep the code clean and maintainable.**

The blog post is divided into 4 steps:

1. [Simple Querying Without Batch Operations](#1-simple-querying-without-batch-operations)
1. [Implementing Batch Queries](#2-implementing-batch-queries)
1. [ Using Generator Functions for Batch Queries](#3-using-generator-functions-for-batch-queries)
1. [Clean Code: Extract the batch logic into its own function](#4-clean-code-extract-the-batch-logic-into-its-own-function)


## 1. Simple Querying Without Batch Operations

Suppose we want to send a notification to all our users, we would need to retrieve all users from the database using TypeORM, iterate over each individual user and send the actual notification. The code is clean and split up into the more general function `sendOfferToUsers()`, whereas the database query logic is encapsulated into `findAllConfirmedUsers()`


```typescript
async function sendOfferToUsers() {
  for (const user of await findAllConfirmedUsers()) {
    const message = `Special Offer just for ${user.name}`;
    await notifyUser(user.email, message);
  }
}

async function findAllConfirmedUsers() {
  return await userRepository.find({
    where: { confirmed: true }
  });
}

async function notifyUser(email: string, message: string) {
  // omitted for the brevity
}
```

While this approach works fine for smaller datasets, it becomes inefficient and memory-intensive when dealing with larger ones. To address this issue, we need to query the database in batches.



## 2. Implementing Batch Queries

To efficiently query large datasets from a database, it's necessary to retrieve the data in subsets rather than all at once. This is achieved through batch operations, which involve querying the database multiple times and limiting the returned rows to a specific number while skipping the previously fetched rows.

TypeORM allows us to set `take` and `skip` attributes for batch querying, but it requires explicit configuration. Here's an example of implementing batch operations:


```typescript
async function sendOfferToUsers() {
  const batchSize = 1000;
  let batchIndex = 0;
  let users: User[];

  do {
    users = await findAllConfirmedUsers(batchSize, batchIndex);
    for (const user of users) {
      const message = `Special Offer just for ${user.name}`;
      await notifyUser(user.email, message);
    }
    batchIndex++;
  } while (users.length === batchSize);

}

async function findAllConfirmedUsers(batchSize: number, batchIndex: number) {
  return await userRepository.find({
    where: { confirmed: true },
    take: batchSize,
    skip: batchIndex * batchSize
  });
}
```

Obviously, the code is less readable than the previous example due to littering the batch operation from the database-specific method `findAllConfirmedUsers()` into `sendOfferToUsers()`. However, I want to enforce a stricter separation of concerns and keep all database-related code in `findAllConfirmedUsers()`.

To address this requirement, we can employ a generator function.


## 3. Using Generator Functions for Batch Queries

Let's begin by examining the refactored code, where we consolidate all database-related logic into `findAllConfirmedUsers()` and transform it into a generator function.

```typescript
async function sendOfferToUsers() {
  for await (const user of findAllConfirmedUsers()) {
    const message = `Special Offer just for ${user.name}`;
    await notifyUser(user.email, message);
  }
}

async function* findAllConfirmedUsers() {
  const batchSize = 1000;
  let batchIndex = 0;
  let users: User[];

  do {
    users = await userRepository.find({
      where: { confirmed: true },
      take: batchSize,
      skip: batchIndex * batchSize
    });
    for (const user of users) {
      yield user;
    }
    batchIndex++;
  } while (users.length === batchSize);
}
```

The signature of `findAllConfirmedUsers()` has changed from `function` to `function*` (a star at the end) and includes the `yield` keyword in the function body. Beside from these syntactical changes, the function itself works much different. It's a special type of function that can be paused and resumed during execution, allowing for the generation of a sequence of values. `findAllConfirmedUsers()` retrieves users from the database in batches and yields one user at a time. The `for await...of` loop in the `sendOfferToUsers()` function then iterates over the generated values, providing a clean and readable way to process each user without loading the entire dataset into memory. The code becomes more efficient and memory-friendly when dealing with large datasets.

Using a generator function makes the code more readable and maintainable. The `sendOfferToUsers()` function remains focused on its main purpose, while the details of batch querying are handled by the `findAllConfirmedUsers()` generator function.


## 4. Clean Code: Extract the batch logic into its own function

To enhance the code further, we can extract the batch logic into its own function to make it reusable. This function can be used to retrieve arbitrary entities in batches, eliminating the need to duplicate the batch logic when fetching different types of entities.

```typescript
async function sendOfferToUsers() {
  for await (const user of findAllConfirmedUsers()) {
    const message = `Special Offer just for ${user.name}`;
    await notifyUser(user.email, message);
  }
}

async function* findAllConfirmedUsers() {
  yield* findInBatches(userRepository, { where: { confirmed: true } });
}

async function* findInBatches<T extends ObjectLiteral>(repository: Repository<T>, findOptions: FindManyOptions<T>) {
  const batchSize = 1000;
  let batchIndex = 0;
  let entities: T[];

  do {
    entities = await repository.find({
      take: batchSize,
      skip: batchIndex * batchSize,
      ...findOptions
    });
    for (const entity of entities) {
      yield entity;
    }
    batchIndex++;
  } while (entities.length === batchSize);
}
```

The `yield*` keyword is used to pass on the iteration control from one generator function to another. In the `findAllConfirmedUsers()` function, `yield*` is employed to transfer the iteration of the batched users to the `findInBatches()` generator function. This helps to maintain a clear separation of concerns: the `findAllConfirmedUsers()` function concentrates on fetching users from the database, while the `findInBatches()` function handles the logic for querying in batches.


## Conclusion

In Node.js applications, querying a database in batches is essential for handling large datasets efficiently. By utilizing generator functions, we can simplify the code, improve readability, and maintainability. The extracted batch logic also promotes clean code organization and reusability. Generator functions are a powerful language feature that I just learned about a few weeks ago.
