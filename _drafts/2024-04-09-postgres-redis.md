---
layout: post
title: "Building a Postgres Plugin"
date: 2024-04-09
categories: databases
---

This project is simply a Postgres to Redis connector built as a Postgres extension. It intercepts all SELECT and UPDATE queries and inserts/updates relevant data into a Redis cache. The project repo can be found [here](https://github.com/systemEng-Learning/postgres-redis).

This project's purpose was to improve our understanding of Postgres Internals and Rust. We needed an excuse to dive into Postgres's source code to understand how well-designed and reliable software applications are built. This project provided that excuse. We are new users of Rust and we wanted something to build with it. This project provided that something. 

Since this project's purpose isn't to build a production ready connector, there are several limitations. These limitations are due to the goals and time constraints. Some of these limitations are:
* It doesn't support DELETE queries. This is important because a true connector should support delete operation. You don't want your cache to contain already deleted data. We didn't do this because of our assumptions of the complexity. In addition, we believe a delete operation will be better served with a WAL-based or TRIGGER-based design than our Postgres hooks-based design. 
* It only supports updates on a single row. Updates on multiple rows which are quite common are not supported here. In all honesty, here's where a WAL-based or TRIGGER-based design would be better. We went with a hooks-based design because it allows us to really dive deep into the design of Postgres. In this project, we didn't interface with Postgres as a black box.
* It only works for `WHERE` clauses with equal operator. `WHERE` clauses with other comparison operators do not work.
* Its config only allows for one table, one key column and one value column. All the columns must belong to that table. 
* No tests. That's enough reason to not use this plugin.

This project wouldn't have been done without the Postgres Internals [book](http://www.interdb.jp/pg/index.html), it's gold! The unofficial Postgres hooks [doc](https://github.com/taminomara/psql-hooks/blob/master/Detailed.md) was extremely helpful. The [pgrx](https://github.com/pgcentralfoundation/pgrx) crate is the foundation of the project. Without it, there's no Rust-based plugin.

The below progress log is a retrospective, so some of the details might not be as accurate due to memory failure :-).

### Log
* Jan 17 - Jan 20, 2024: We setup pgrx. The SELECT query handler simply printed an hello message with its operation number. The UPDATE query handler did something even cooler, it was able to print out the original query! The Postgres [source code](https://github.com/postgres/postgres/blob/27074bce08e994daf6b8fe9a84877ac257210fdd/src/include/executor/execdesc.h#L33) and pgrx [doc](https://docs.rs/pgrx/latest/pgrx/pg_sys/struct.QueryDesc.html) allowed us to understand the `QueryDesc` object and get relevant info from it.
* 