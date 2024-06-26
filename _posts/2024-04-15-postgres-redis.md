---
title: "Building a Postgres Plugin"
last_modified_at: 2024-04-15
excerpt_separator: "<!--more-->"
categories:
  - Database
tags:
  - rust
  - database
---

This project is simply a Postgres to Redis connector built as a Postgres extension. It intercepts all SELECT and UPDATE queries and inserts/updates relevant data into a Redis cache. <!--more--> The project repo can be found [here](https://github.com/systemEng-Learning/postgres-redis).

This project's purpose was to improve our understanding of Postgres Internals and Rust. We needed an excuse to dive into Postgres's source code to understand how well-designed and reliable software applications are built. This project provided that excuse. We are new users of Rust and we wanted something to build with it. This project provided that something. 

Since this project's purpose isn't to build a production-ready connector, there are several limitations. These limitations are due to the goals and time constraints. Some of these limitations are:
* It doesn't support DELETE queries. This is important because a true redis connector should support delete operation. You don't want your cache to contain already deleted data. We didn't do this because of our assumptions of the complexity. In addition, we believe a delete operation will be better served with a WAL-based or TRIGGER-based design than our Postgres hooks-based design. 
* It only supports updates on a single row. Updates on multiple rows which are quite common are not supported here. In all honesty, here's where a WAL-based or TRIGGER-based design would be better. We went with a hooks-based design because it allows us to really dive deep into the design of Postgres. In this project, we didn't interface with Postgres as a black box.
* It only works for `WHERE` clauses with equal operator. `WHERE` clauses with other comparison operators do not work.
* Its config only allows for one table, one key column and one value column. All the columns must belong to that table. 
* No tests. That's enough reason to not use this plugin.

This project wouldn't have been done without the Postgres Internals [book](http://www.interdb.jp/pg/index.html), it's gold! The unofficial Postgres hooks [doc](https://github.com/taminomara/psql-hooks/blob/master/Detailed.md) were extremely helpful. The [pgrx](https://github.com/pgcentralfoundation/pgrx) crate is the foundation of the project. Without it, there's no Rust-based plugin.

The below progress log is retrospective, so some of the details might not be as accurate due to memory failure :-).

### Log
* Jan 17 - Jan 21, 2024: We setup pgrx. The SELECT query handler simply printed an hello message with its operation number. The UPDATE query handler did something even cooler, it was able to print out the original query (which the SELECT query handler stole)! The Postgres [source code](https://github.com/postgres/postgres/blob/27074bce08e994daf6b8fe9a84877ac257210fdd/src/include/executor/execdesc.h#L33) and pgrx [doc](https://docs.rs/pgrx/latest/pgrx/pg_sys/struct.QueryDesc.html) allowed us to understand the `QueryDesc` object and get relevant info from it.
* Jan 22 - Jan 28, 2024: Figured out that the custom enums we used for detecting query type wasn't needed since pgrx [had](https://docs.rs/pgrx/latest/pgrx/pg_sys/constant.CmdType_CMD_SELECT.html) [it](https://docs.rs/pgrx/latest/pgrx/pg_sys/constant.CmdType_CMD_UPDATE.html). Our goal was to print out messages if specific tables were queried or if a single row was queried/updated. The UPDATE query handler did the first by using a regex approach. The SELECT query handler did both via the [`PlannedStmt`](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/include/nodes/plannodes.h#L46) and [`EState`](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/include/nodes/execnodes.h#L618) classes. This is one of those problems that Postgres Internals really [helped](https://www.interdb.jp/pg/pgsql03/01.html#314-planner-and-executor).
* Jan 28 - Feb 4, 2024: We wanted to be able to print out a specific subset(column) of rows that were selected and updated. This was an exercise in being able to get the data. The SELECT query handler used a custom interceptor approach because we figured out that Postgres [sends](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/backend/executor/execMain.c#L1677) each selected tuple to a [`DestReceiver`](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/include/tcop/dest.h#L113). We created our own custom [DestReceiver](https://github.com/systemEng-Learning/postgres-redis/blob/05ec5172157932635c1f773fd49d8b61dd13a948/src/select.rs#L19) and used it. We were able to do this by [looking](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/backend/tcop/dest.c#L75) at [some](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/backend/access/common/printsimple.c) of Postgres [examples](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/backend/access/common/printtup.c#L422).
We also tried, to fetch updated tuples, but was very difficult for some reason to fetch updated tuples in PG13, though before now, we've had success with that in PG16, but our aim was also to support lower PG version and since we could not get it to work in PG13, we tried PG14 and it works smoothly.
* Feb 5 - Feb 11, 2024: Extensions should not carry out long-running operations in the hook functions. This is because the query executor needs to quickly return a result to the client and carrying out long-running operations could cause the query executor to have unacceptably long latency. Since the selected/updated data needs to be sent to a redis server, it can't happen in the hook functions. This necessitates the use of a background worker and a way to pass data to this worker. Thankfully, Postgres provides a way for an extension to use both [background workers](https://www.postgresql.org/docs/current/bgworker.html) and [shared memory](https://pgpedia.info/s/shmem_request_hook.html). We were able to use these in pgrx with the help of [its](https://github.com/pgcentralfoundation/pgrx/tree/develop/pgrx-examples/bgworker) [examples](https://github.com/pgcentralfoundation/pgrx/tree/develop/pgrx-examples/shmem). At this point, we just printed out data to the console and log.
* Feb 12 - Feb 18, 2024: Some refactoring was done on the codebase.
* Feb 19 - Feb 24, 2024: We needed to process the WHERE clause part of the query. Turns out this was the hardest part of the project. Postgres WHERE clause handling is very very complex. One of us went down the rabbit hole of trying to understand it and came up with nothing.
* Feb 25 - Mar 3, 2024: One of us discovered the [pg_qualstats](https://github.com/powa-team/pg_qualstats) extension which went a long way in helping him understand the WHERE clause relevant structs. He tried porting the relevant part of the extension to Rust. You can see his incomplete work [here](https://github.com/systemEng-Learning/postgres-redis/blob/whereclause-expr/src/utils.rs).
* Mar 4, 2024 - Mar 10, 2024: While one of us was porting, his project partner was able to get something done that worked. That was the breakthrough we needed!
* Mar 11, 2024 - Mar 17, 2024: The working solution only worked for numbers comparison and didn't support ORs and ANDs in the WHERE clause. The porting task was abandoned and the working solution was refactored to make it work on a wider range of WHERE clause types. This was where the abandoned porting task really helped because we added some of its code to the working solution. After this was done, we added the redis functionality. We were able to confirm that the extension worked, Yay!
* Mar 18, 2024 - Mar 24, 2024: We wanted to make the extension configurable. Thankfully, pgrx provides some [structs](https://docs.rs/pgrx/latest/pgrx/guc/index.html) to achieve this. In addition, looking at [zombodb codebase](https://github.com/zombodb/zombodb/blob/1416c99a4885f1cfe5e7bd86b935e89e6d6ba431/src/gucs/mod.rs) helped us figured out how to use them. We went with a simple string format for redis url and a json string for the table-related stuffs. To be honest, we went with json as an excuse to use serde :-).
* Apr 1, 2024 - Apr 7, 2024: Thankfully wiser heads prevailed and we went with a simple configuration option instead. At least our "use serde library" itch was scratched.
* Apr 8, 2024 - Apr 15, 2024: Clean up the codebase, add comments, add a license, and write this post.

### Participants' Writeups
* [Goodness](https://goodyduru.github.io/database/2024/04/15/what-i-learned-from-building-a-postgres-extension-in-rust.html)
* [Steve](https://steveoni.github.io/database/2024/04/15/postgres-redis-extension-in-rust.html)

### Conclusion
This admittedly limited project was a massive success. It really improved our understanding of Postgres and Rust. Being forced to dive into Postgres source code schooled us on world class software design. Looking at the pgrx source code and writing this project taught us a lot on how to use Rust. We are glad to have done this project.
