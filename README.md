# SQL Functional Programming

This repo is dedicated to pursuing the following goals:
- to embrace the **power of the SQL query language**
> to move beyond simple SELECT and INSERT statements mixed with a pile of procedural code and instead write complex queries that give the answer directly. ... how a few lines (or a few dozen lines) of SQL can replace thousands and thousands of lines of procedural code. Doing so reduces the number of bugs (which are roughly proportional to the number of lines of code) and also often result in faster solutions as well. [Quote source](https://www.sqlite.org/forum/forumpost/44c8bf23fa?t=h)
- to identify **what hinders programmers** from benefitting from this power
- to **propose useful and pragmatic changes** to SQL to address these issues

See the discussion section for dialogs on two specific topics
- [Functional Programming with SQL Query lambdas, functions and pipelines](https://github.com/DavidPratten/sql-fp/discussions/6)
- [Composable Recursive Relational Queries](https://github.com/DavidPratten/sql-fp/discussions/8)

This work is based on a few assumptions about SQL:
- SQL is a (relational) pure functional query language with strictly controlled effects,
- SQL recursive and fixed-point queries are under utilised,
- [SQLite](https://sqlite.org/index.html) is an approachable and powerful testbed for exploratory ideas in SQL language design.

David Pratten
