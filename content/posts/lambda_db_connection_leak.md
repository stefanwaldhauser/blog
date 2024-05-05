---
title: "Application Side Database Connection Pooling can lead to a Connection Leak in an AWS Lambda Application"
date: "2024-05-05"
description: "How it happens and what you can do about it."
summary: "How it happens and what you can do about it."
tags: ["AWS", "AWS Lambda", "Database"]
ShowToc: true
TocOpen: true
---

### TL;DR

In an AWS Lambda setup using application-side database connection pooling, there's a risk of connection leaks due to frozen lambda processes not cleaning up idle database connections. To address this, configure the PostgreSQL `idle_session_timeout` parameter slightly higher than the application-side pool's `idleTimeoutMillis`. Also, manually check if your application-side pool contains any idle connections when the lambda process is unfrozen and another execution occurs, to ensure you are not using a database connection that has been terminated by the database while the process was frozen.

### Project Setup

The project where the problem occured uses [NodeJs](https://nodejs.org/en) based [AWS Lambdas](https://docs.aws.amazon.com/lambda/latest/dg/lambda-nodejs.html) that connect to a [PostgreSQL AWS RDS](https://aws.amazon.com/rds/postgresql/) database. The main library used for the database handling is the object–relational mapping tool [mikroOrm](https://github.com/mikro-orm/mikro-orm) which uses the SQL query builder [knex](https://github.com/knex/knex) which in turn uses [pg](https://github.com/brianc/node-postgres) as PostgreSQL client and [tarn](https://github.com/vincit/tarn.js/) for application side connection pooling.

As [recommended by AWS](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) the mikroOrm instance and thus the tarn based database connection pool are created outside of the lambda handler:

> Initialize SDK clients and database connections outside of the function handler (...) Subsequent invocations processed by the same instance of your function can reuse these resources. This saves cost by reducing function run time.

The idea is that variables declared at the global scope within a Node.js module are retained in memory throughout the process's lifespan. Consequently, they can be utilized for multiple invocations of the same lambda instance, persisting even after the handler function concludes its execution. These global variables remain in memory and are not subject to garbage collection.

When the database connection pool is defined as a global variable outside the lambda handler, it persists across invocations of the same lambda instance. This enables reusing database connections within the pool for multiple invocations of the same lambda instance. To avoid keeping connections open indefinitely, the pool allows setting an `idleTimeoutMillis`. Unused connections are destroyed after the specified duration.

### Problem

It was observed that the database connections opened per lambda instance sometimes persisted longer than the expected `idleSessionTimeout` value set on the application connection pool. This resulted in quickly depleting the number of maximum database connections permitted by our RDS instance.

The issue stems from the way tarn (and potentially other pooling libraries) handle the pool reaping process. Reaping involves detecting and clearing idle resources, such as database connections, that remain idle for longer than the specified timeout period.

The implementation of reaping in tarn can be seen [here](https://github.com/Vincit/tarn.js/blob/e33d223831367a264db33e35d4c9381a089e539c/src/Pool.ts):

```ts {linenos=true,hl_lines=[4,17]}
  _startReaping() {
    if (!this.interval) {
      this._executeEventHandlers('startReaping');
      this.interval = setInterval(() => this.check(), this.reapIntervalMillis);
    }
  }

  check() {
    const timestamp = now();
    const newFree: Resource<T>[] = [];
    const minKeep = this.min - this.used.length;
    const maxDestroy = this.free.length - minKeep;
    let numDestroyed = 0;

    this.free.forEach(free => {
      if (
        duration(timestamp, free.timestamp) >= this.idleTimeoutMillis &&
        numDestroyed < maxDestroy
      ) {
        numDestroyed++;
        this._destroy(free.resource);
      } else {
        newFree.push(free);
      }
    });

    this.free = newFree;

    // Pool is completely empty, stop reaping.
    // Next .acquire will start reaping interval again.
    if (this.isEmpty()) {
      this._stopReaping();
    }
  }
```
An [interval in Node.js](https://nodejs.org/api/timers.html) is initiated to check every `reapIntervalMillis` whether any of the idle resources surpass `idleTimeoutMillis` and subsequently destroys them, thus closing the database connection.

However, the problem arises when a lambda execution concludes. What happens is well explained in this [blog post by Dynatrace](https://www.dynatrace.com/news/blog/a-look-behind-the-scenes-of-aws-lambda-and-our-new-lambda-monitoring-extension/):

>After each execution, AWS Lambda puts the instance to sleep. In other words, the instance freezes (similar to a laptop in hibernate mode). The virtual CPU is turned off. This frees up resources on the worker node. The overhead from waking up such a function is negligible.

**Consequently, the `setInterval` ceases to run once the execution concludes, and the lambda process enters a frozen state. Therefore, database connections that remain open in the application side pool are not cleaned up after `idleTimeoutMillis` since the reaping interval is not running. A case of a database connection leak, which [degrades database performance](https://aws.amazon.com/blogs/database/resources-consumed-by-idle-postgresql-connections/).**


### Solution

The project implements a solution that involves **combining application-side pooling with configuring database-side cleanup of idle connections.**

To implement this solution, follow these steps:

- Set the [`idle_session_timeout`](https://www.postgresql.org/docs/current/runtime-config-client.html) parameter of PostgreSQL to a value slightly higher than the `idleTimeoutMillis` of your application-side pool. For example, if `idleTimeoutMillis` is set to `500ms`, set `idle_session_timeout` to `510ms`. This ensures that connections missed by the application-side pool while the lambda process was frozen are caught and cleaned up by the database. To ensure connections are always controlled by the application side pool and not unexpectedly terminated by the database while the lambda process is hot, it's crucial to configure the database-level timeout higher than the application sidee timeout.

- When using the `idle_session_timeout` parameter of the database, it's vital to perform an additional manual pool reaping check within the lambda handler before using any connection from the pool. This extra check ensures detection and cleanup of any idle connections within the pool that may have been terminated by the database while the lambda process was frozen. Relying solely on the usual check that runs on a fixed interval isn't sufficient. There's a risk that the reaping check, operating on the interval, kicks in after the first attempt to use a database connection that has already been terminated by the server. With tarn, you can run a manual reaping check by accessing the tarn pool instance and invoking the [`.check()`](https://github.com/Vincit/tarn.js/blob/e33d223831367a264db33e35d4c9381a089e539c/src/Pool.ts#L244) method. In the project `.check()` is simply called first thing in the lambda handler before doing anything else, ensuring the application side pool is in sync and does not contain any dead connections.
