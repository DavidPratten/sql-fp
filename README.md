# Functional Programming with SQL Query functions, lambdas, and pipelines
This article argues firstly that SQL is a useful functional programming language with optional, and strictly controlled, effects. And secondly, that the extension of SQL with functional programming query functions, lambdas and pipelines would:
- improve the language ergonomics for functional programming, and
- be implementable with O(0) cost.

The article builds the argument with the following outline:
- define what is meant by a query function,
- argue that SQL is, at its heart, a functional programming language,
- motivate the proposal using a non-trivial data transformation scenario,
- show some reasons which drive programmers to abandon SQL's functional language,
- apply query functions to the data transformation scenario,
- introduce SQL query lambdas and pipelines and apply them to the scenario,
- show the simplicity of the implementation of query functions, lambda's and pipelines, and
- outline a next experiment to build on this work.

### Query Functions
Let's begin by proposing query functions as a syntactic extension to SQL. Later, query lambdas and pipelines will build on this foundation.

A query function:

- accepts one or more tables or subqueries as input
- and returns a relation.

To a good approximation, query functions may be understood as ```VIEW```s with tables or sub-queries as parameters. Crucially, as with ```VIEW```, query functions do not change the semantics of SQL's ```SELECT```, and may be optimised by the query planner/optimiser just like any ```SELECT``` query.

The proposed syntax for creating a query function is adding a ```GIVEN``` clause to the form of the ```CREATE VIEW``` statement:

#### Extending ```VIEW``` form with a ```GIVEN``` clause
```sql
CREATE QUERYFUNCTION qfname(column, ...) 
        GIVEN (local_table_name(column, ...), ...) 
        AS SELECT ...;
```

which may be read as follows:

- given the input relation(s) ```local_table_name(column, ...), ...```
- the function returns the relation ```qfname(column, ...)```
- by evaluating a ```SELECT``` query.

The only semantic change proposed is that the ```SELECT``` in body of a query function may only refer to 
```local_table_name```s declared in the ```GIVEN``` phrase. 

#### Query Function Invocation

Just like ```VIEW```s (and CTEs), query functions stand in a ```SELECT``` query in the place of a table or  subquery. Table or subquery parameters follow the ```qfname``` surrounded by brackets. A simple case of  invoking a query function would be:

```sql
select * from qf1(table1, (SELECT col1, col2 FROM table2));
```

Before looking at a motivating scenario, let's look at SQL's claim to be a functional programming language.

### SQL is a functional programming platform

With a thirty plus year history, the heart of SQL is a pure functional language, with optional mechanisms to apply side effects in a controlled fashion.

#### Functional subset of SQL

The ```SELECT``` query of SQL combined with Common Table Expressions (CTE) ```WITH .... SELECT```is a complete and expressive pure functional language.

#### Pure functions

Grounded in the Relational Algebra, SQL's Data Query Language (DQL) (aka ```SELECT```) expresses a transformation, or computation and is:

- declarative,
- composable,
- side effect free,
- and optionally, turing-complete.

```SELECT``` queries also:

- operate over immutable data, and
- are referentially transparent.

#### Higher-Order functions

```SELECT``` queries support higher order functions. Examples are shown in the following table:

| Higher Order Function | SQL Equivalent                         |
|-----------------------|----------------------------------------|
| ```map()```           | ```SELECT```                           |
| ```reduce()```        | ```GROUP BY``` and aggregate functions |
| ```filter()```        | ```WHERE``` clause                     |    
| ```sort()```          | ```ORDER BY``` clause                  |

The SQL ```SELECT``` can "hold it's head high" among the ranks of functional programming languages, lets now look at its approach
to side effects.

#### Encapsulation of common code

SQL has two mechanisms to factor out common code and build purely functional expressions:

- the CTE mentioned already, and
- reusable ```VIEW```s.

#### Key role of the execution environment
The relational query planner/optimiser plays a crucial role in the functional nature of SQL.  It is this part of the execution environment that enables: 
- queries to be reordered and elided,
- materialised, indexed,
- lazily evaluated and/or
- parallelised.

The query planner may be built into a database like [SQLite's Query Planner](https://www.sqlite.org/queryplanner.html), or be a standalone product like 
[Calcite](https://calcite.apache.org/docs/algebra.html).

#### Optional and controlled side effects

If updating state ***is important for the programmer***, SQL provides for controlled side effects through the Data Manipulation Language (DML). The following ***non-composable*** forms consume the results of the purely functional ```SELECT``` transformation and apply them to the current state:

- ```INSERT INTO ... SELECT ...```
- ```UPDATE ... FROM (SELECT ...)```
- ```DELETE FROM ... WHERE (SELECT ...)```

In summary, SQL's ```SELECT``` has the attributes of a composable pure functional programming language and may optionally be partnered with non-composable statements that provide controlled side effects.

Before we explore an opportunity to improve SQL's appeal as a functional programming language, lets now look at a non-trivial data transformation scenario.

### Scenario: Summarising data over multiple hierarchies

The Theta organisation has a hierarchy of departments and a portfolio of projects. The scenario explored here is to write SQL queries that turn an```Org``` table representing the hierarchy of departments and a ```Projects``` tables into ```Cost_Summary``` and ```Risk_Summary``` relations. It is a requirement that the reporting level for each summary is data-driven and can be moved up and down the hierarchy without changing the queries.

Theta's departments are organised in a tree. Team-level departments are the lowest level in the hierarchy and are shown as rectangles.

![Theta Overview](https://user-images.githubusercontent.com/1667631/147862816-193d2280-ff60-4e79-8d3c-98e8b6ec3330.png)


<details>
<summary><em>Click to view Graphviz Code of this diagram.</em></summary>

```
digraph G {
rankdir=BT;
node [ color=gray, fontname=arial];

Theta[label="Theta"]
Operations[label=<Operations>]
Finance[ shape=box, ]
People_and_Culture[shape=box, label="People and Culture"]
Eastern_Operations[label=<Eastern Operations>]
Western_Operations[label="Western Operations"]
Northern_Operations[shape=box, label="Northern Operations"]
Western_Operations[shape=box, label="Western Operations"]
South_Eastern_Operations[shape=box, label="South-Eastern Operations"]
North_Eastern_Operations[shape=box, label="North-Eastern Operations"]

Operations->Theta[dir=none]
Finance->Theta[dir=none]		
People_and_Culture->Theta[dir=none]		
Eastern_Operations->	Operations[dir=none]
Western_Operations->	Operations[dir=none]		
Northern_Operations->	Operations[dir=none]		
South_Eastern_Operations->	Eastern_Operations[dir=none]		
North_Eastern_Operations->	Eastern_Operations[dir=none]		

}

```

</details>

#### Projects Table

Each project is linked to a team-level department. Each project also has an "Estimated cost to completion" and "Value at Risk" recorded.

| project\_id              | estimate\_to\_completion | value\_at\_risk | dept\_id |
|:-------------------------|-------------------------:|----------------:| :--- |
| Sales                    |                  720000 |          128000 | Finance |
| Portal                   |                  130000 |           50000 | People and Culture |
| Operational Effectiveness |                  130000 |           28000 | Western Operations |
| Tooling Business         |                  580000 |          150000 | Northern Operations |
| Medical Devices          |                  440000 |          120000 | South Eastern Operations |
| Delivery Services        |                  100000 |           24000 | North Eastern Operations |

#### Summary Reports

Within the organisational hierarchy, 100% of project task costs and at-risk amounts are reported, but by two distinct levels across the full width of the hierarchy:

**Cost Reporting** is at the CxO level ie Finance, Operations and People and Culture:

| summary\_dept\_id  | estimate\_to\_completion |
|:-------------------|-------------------------:|
| Finance            |                   720000 |
| Operations         |                  1150000 |
| People and Culture |                   130000 |

**Risk Reporting** is finer-grained and is by the direct reports of the Chief Operating Officer.

| summary\_dept\_id | value\_at\_risk |
| :--- |----------------:|
| Eastern Operations |          144000 |
| Finance |          128000 |
| Northern Operations |          150000 |
| People and Culture |           50000 |
| Western Operations |           28000 |

These two breakdowns are illustrated in the following diagram, the green departments show the summarisation level for costs and the red bordered departments show the summarisation level for risks.

![Theta Cost and Risk reporting lines](https://user-images.githubusercontent.com/1667631/147862827-b6ba3b68-9a90-48e9-9bfb-3d229e7f94bb.png)

<details>
<summary><em>Click to view Graphviz code for this diagram!</em></summary>

```
digraph G {
rankdir=BT;
node [ color=gray, fontname=arial, fontsize=8];

Theta[label="Theta"]
Operations[label=<Operations<br/><font color="green2">(Cost Report Flag)</font>>, fillcolor="green4", style="filled"]
Finance[ shape=box, color="red3", penwidth=2, fillcolor="green4", style="filled"]
People_and_Culture[shape=box, label="People and Culture", color="red3",penwidth=2,  fillcolor="green4", style="filled"]
Eastern_Operations[label=<Eastern Operations<br/><font color="red2">(Risk Report Flag)</font>>, color="red3",penwidth=2 ]
Western_Operations[label="Western Operations", penwidth=2, color="red3"]
Northern_Operations[shape=box, label="Northern Operations", penwidth=2, color="red3"]
Western_Operations[shape=box, label="Western Operations"]
South_Eastern_Operations[shape=box, label="South-Eastern Operations"]
North_Eastern_Operations[shape=box, label="North-Eastern Operations"]



Operations->Theta[dir=none]
Finance->Theta[dir=none]		
People_and_Culture->Theta[dir=none]		
Eastern_Operations->	Operations[dir=none]
Western_Operations->	Operations[dir=none]		
Northern_Operations->	Operations[dir=none]		
South_Eastern_Operations->	Eastern_Operations[dir=none]		
North_Eastern_Operations->	Eastern_Operations[dir=none]		

}
```

</details>

#### Org table

The Org table identifies each department in the "dept_id" column and captures the hierarchy through the "parent_dept_id" column. The organisation as a whole may be represented by a department "Theta" which is it's own parent.

Based on the diagram above, the "Cost Report Flag" on the "Operations" department says
"report all descendent department's costs here". Like wise the "Risk Report Flag" on the "Eastern Operations" department says " report all descendent department's risks here".

Adding the reporting level flags for the Cost and Risk reporting levels we get the complete Org table with data-driven summarisation levels:

| dept\_id | parent\_dept\_id | cost\_report | risk\_report |
| :--- | :--- | :--- | :--- |
| Theta | Theta | NULL | NULL |
| Operations | Theta | 1 | NULL |
| Finance | Theta | NULL | NULL |
| People and Culture | Theta | NULL | NULL |
| Eastern Operations | Operations | NULL | 1 |
| Western Operations | Operations | NULL | NULL |
| Northern Operations | Operations | NULL | NULL |
| South Eastern Operations | Eastern Operations | NULL | NULL |
| North Eastern Operations | Eastern Operations | NULL | NULL |

Now that we have the scenario mapped out, lets look at the SQL Query which can be used to create the required summaries at the required levels.

### Generate the ```Cost_Summary``` using SQL

This is an SQLite query that will transform the ```org``` and ```project``` tables into the
```Cost_Summary``` relation:

```sql
WITH map_cost_to_summary (summary_dept_id, leaf_dept_id)
AS (
    SELECT CASE 
            WHEN cost_report OR (
                    NOT EXISTS (
                        SELECT dept_id
                        FROM org org3
                        WHERE org3.parent_dept_id = org.dept_id
                        ) OR 1 = (
                        SELECT count(*)
                        FROM org
                        )
                    )
                THEN dept_id
            END, dept_id
    FROM org
    WHERE dept_id = parent_dept_id
    
    UNION
    
    SELECT CASE 
            WHEN map_cost_to_summary.summary_dept_id IS NOT NULL
                THEN map_cost_to_summary.summary_dept_id
            WHEN org2.cost_report OR NOT EXISTS (
                    SELECT dept_id
                    FROM org org3
                    WHERE org3.parent_dept_id = org2.dept_id
                    )
                THEN org2.dept_id
            END, org2.dept_id
    FROM map_cost_to_summary
    JOIN org org2 ON org2.parent_dept_id = map_cost_to_summary.leaf_dept_id AND org2.dept_id <> org2.parent_dept_id
    )

SELECT map.summary_dept_id AS summary_dept_id, sum(p.estimate_to_completion) AS estimate_to_completion
FROM projects p
JOIN map_cost_to_summary map ON p.dept_id = map.leaf_dept_id
GROUP BY map.summary_dept_id;
```

The query has two parts:

- a sub-query ```map_cost_to_summary```, which is a tree-walking recursive Common Table Expression (CTE), followed by
- the main ```SELECT``` that sums the ```estimate_to_completion``` amounts and creates the result.

The last four lines of the query create the final result, but the 'heavy lifting' in this solution is being performed by the ```map_cost_to_summary``` sub-query. This sub-query creates a mapping from every department to its corresponding summary department based on the "cost_report" flag.

| summary\_dept\_id | leaf\_dept\_id |
| :--- | :--- |
| NULL | Theta |
| Finance | Finance |
| Operations | Operations |
| People and Culture | People and Culture |
| Operations | Eastern Operations |
| Operations | Northern Operations |
| Operations | Western Operations |
| Operations | North Eastern Operations |
| Operations | South Eastern Operations |

However, the SQL query code has limited reuse potential as the transformation is **hardcoded** to the specific tables and columns that are required by the use case.

#### Generate the ```Risk_Summary``` using SQL

To create the ```Risk_Summary``` we will have to manually copy and alter this code because neither ```VIEW```s nor CTEs are able to abstract out the tranformation from the references to tables and columns in global namespace.
<details>
<summary><em>Click to show Risk_Summary SQL query</em></summary>

```sql
WITH map_risk_to_summary (summary_dept_id, leaf_dept_id)
         AS (
        SELECT CASE
                   WHEN risk_report OR (
                           NOT EXISTS(
                                   SELECT dept_id
                                   FROM org org3
                                   WHERE org3.parent_dept_id = org.dept_id
                               ) OR 1 = (SELECT count(*) FROM org)
                       )
                       THEN dept_id
                   END,
               dept_id
        FROM org
        WHERE dept_id = parent_dept_id

        UNION

        SELECT CASE
                   WHEN map_risk_to_summary.summary_dept_id IS NOT NULL
                       THEN map_risk_to_summary.summary_dept_id
                   WHEN org2.risk_report OR NOT EXISTS(
                           SELECT dept_id
                           FROM org org3
                           WHERE org3.parent_dept_id = org2.dept_id
                       )
                       THEN org2.dept_id
                   END,
               org2.dept_id
        FROM map_risk_to_summary
                 JOIN org org2 ON org2.parent_dept_id = map_risk_to_summary.leaf_dept_id AND org2.dept_id <> org2.parent_dept_id
    )
SELECT map.summary_dept_id AS summary_dept_id, sum(p.value_at_risk) AS value_at_risk
FROM projects p
         JOIN map_risk_to_summary map ON p.dept_id = map.leaf_dept_id
GROUP BY map.summary_dept_id;
```

</details>

### Current SQL features only partially address the functional code reuse issues

For the developer, transformations like this have to be hand-coded for each use-case which is error-prone and motivates programmers to cross the "impedance barrier" to alternative technologies to reduce or eliminate code duplication.

As well, from the point of view of the development team, maintaining multiple hand-coded transformation queries is hard to 
justify. The lack of clean separation between data sources and transformations and business rules drives teams to use other technologies to capture reusable transformations and business rules.

From the point of view of the broader SQL community, hand coding every solution means that there are no reusable libraries of SQL 
transformations that can become a shared resource and built on. Technologies with communities of practice building reusable libraries grow through virtuous cycles of contribution and benefit.

Why aren't the available SQL features adequate to address separation of concerns and code duplication issues?

#### VIEWs don't help (enough)

SQL ```VIEW```s can remove code duplication so long as every use case refers to the same tables and columns in the global name 
space. In our example above, the two use cases (Cost vs Risk) are distinguished by references to different columns, and
therefore require separate ```VIEW```s to capture the two use cases.

#### Common Table Expressions don't help (enough)

SQL Common Table Expressions are identical to ```VIEWS```s, in this regard, in that they can remove duplication so long as every
use case refers to the same tables and columns.

##### ````CREATE FUNCTION```` and "Table Valued Functions" don't help enough either

SQL ````CREATE FUNCTION```` functions are not able to remove the duplicated functional code because to the extent that they are
functional they have the same expressive power as ```VIEW```s and CTEs.

Here is a quick survey of some major vendor's offerings.  Greater code reuse is enabled **only at the cost of leaving the 
functional programming environment** and entering a procedural environment such as C or PL/SQL.

| Vendor     | Supported Feature                                                                                                                                                              | Support for SQL ```SELECT``` code reuse                                                                                                                                 |
|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| SQLite     | [Table Valued Functions ](https://www.sqlite.org/vtab.html)                                                                                                                    | None. SQLite TVFs are coded in general purpose languages (e.g. C)                                                                                                       |      
| MS SQL     | [Table Valued Functions](https://docs.microsoft.<br/>com/en-us/sql/relational-databases/user-defined-functions/create-user-defined-functions-database-engine?view=sql-server-ver15) | SQL-only query expressivity is equivalent to a ```VIEW``` or CTE. If we use the inbuilt procedural language (PL/SQL) then 100% code reuse is achievable. |
| PostgreSQL | [SQL Functions Returning TABLE](https://www.postgresql.org/docs/current/xfunc-sql.html#XFUNC-SQL-FUNCTIONS-RETURNING-TABLE)                                                    | SQL-only query expressivity is equivalent to a ```VIEW``` or CTE. If we use the inbuilt procedural language (PL/SQL) then 100% code reuse is achievable.     |
| Oracle     | [Table Functions](https://docs.oracle.com/cd/B19306_01/appdev.102/b14289/dcitblfns.htm#CHDIIFEG)                                                                               | SQL-only query expressivity is equivalent to a ```VIEW``` or CTE. If we use the inbuilt procedural language (PL/SQL) then 100% code reuse is achievable.      |

### Identifying reuseable code in this scenario

As it happens, there **is** a general tree library transformation that could be used to complete this exercise. Let's package 
this algorithm as a query function and show how it may be used and re-used to summarise hierarchical data.

<details>
<summary><em>Click to for graph theoretic details of this algorithm</em></summary>
To ensure that all leaf data is summarised, we need to map leaf nodes to an [anti-chain](https://en.wikipedia.org/wiki/Antichain)
of nodes across the tree controlled by a flag on nodes that says "if this flag is set, report all lower nodes here".
</details>

If we had a generic query function (say) ```map_leaf_to_summary()``` which returned a list of leaf nodes of a tree and their
corresponding summary nodes then we could create the cost summary as follows.

#### Generate the ```Cost_Summary``` using a Query Function 

```sql
SELECT map.summary_id AS summary_id, sum(p.estimate_to_completion) AS estimate_to_completion
FROM projects p
         JOIN map_leaf_to_summary(...parameter(s)...) map ON p.dept_id = map.leaf_id
GROUP BY map.summary_id;
```

Such a query function ```map_leaf_to_summary()``` may be defined with the following signature:

```sql
CREATE
QUERYFUNCTION map_leaf_to_summary(summary_id, leaf_id)
    -- result is a list of leaf node ids along with their summary node ids.
    GIVEN (tree(id, parent_id, report_flag)) 
    -- tree relation input 
        -- All nodes must have a parent_id. Tree root has id=parent_id. 
        -- When report_flag = True then all descendent leaf nodes will report at this node.
    AS SELECT ...;
```

which may be read as follows:

- given the input relation ```tree(id, parent_id, report_flag)```
- return the relation ```map_leaf_to_summary(summary_id, leaf_id)```

For this particular situation, the required input parameter ```tree``` relation may be returned by this subquery:

```sql
(SELECT dept_id id, parent_dept_id parent_id, cost_report report FROM org)
```

Observe the selection of the columns and their renaming to match the required input relation.

Putting these pieces together allows us to capture the solution for the Cost Summary using a query function.

```sql
SELECT map.summary_id AS summary_id, sum(p.estimate_to_completion) AS estimate_to_completion
FROM projects p
    JOIN 
    map_leaf_to_summary((
        SELECT dept_id id, parent_dept_id parent_id, cost_report report
        FROM org
        )) map 
    ON p.dept_id = map.leaf_id
GROUP BY map.summary_id;
```

The full definition of query function ```map_leaf_to_summary()``` is not central to the current flow of discussion, but if you 
are interested ...
<details>
<summary><em>Click here to see the full definition of the query function map_leaf_to_summary()</em></summary>

```sql
CREATE
QUERYFUNCTION map_leaf_to_summary(summary_id, leaf_id)
    -- result is a list of leaf node ids along with their summary node ids.
    GIVEN (tree(id, parent_id, report_flag)) 
    -- tree relation input 
        -- All nodes must have a parent_id. Tree root has id=parent_id. 
        -- When report_flag = True then all descendent leaf nodes will report at this node.
    AS WITH antichain(summary_id, leaf_id) AS (
                SELECT CASE 
                        WHEN report OR (
                                is_leaf({tree}, (
                                        SELECT tree.id
                                        ))
                                )
                            THEN id
                        END, id
                FROM {tree} tree
                WHERE id = parent_id
                
                UNION
                
                SELECT CASE 
                        WHEN antichain.summary_id IS NOT NULL
                            THEN antichain.summary_id
                        WHEN tree2.report OR (
                                is_leaf({tree}, (
                                        SELECT tree2.id
                                        ))
                                )
                            THEN tree2.id
                        END, tree2.id
                FROM antichain
                JOIN {tree} tree2 ON tree2.parent_id = antichain.leaf_id AND tree2.id <> tree2.parent_id
                )
SELECT summary_id, leaf_id
FROM antichain
WHERE summary_id is not null;
```

Which has been optimised by removing repetitive code that checks for a node's leaf status. Here is the required definition:

```sql
CREATE
QUERYFUNCTION isLeaf (isLeaf) given (tree(id, parent_id), item(id)) AS (
    SELECT CASE 
        WHEN NOT EXISTS (
                SELECT id
                FROM tree tree2
                WHERE tree2.parent_id = tree.id
                ) OR 1 = (
                SELECT count(*)
                FROM tree
                )
            THEN 1
        ELSE 0
        END is_leaf FROM tree tree JOIN item item ON item.id = tree.id
    )
```

</details>

Now lets return to the second scenario of a Risk Summary.

#### Generate the ```Risk_Summary``` using a Query Function 

Given the query function ```map_leaf_to_summary``` defined above, the solution for the Risk Summary requires just two changes to the 
query:

1. we ```SUM``` ```value_at_risk``` and
2. we pass in ```risk_report``` as the report flag.

```sql
SELECT map.summary_id AS summary_id, sum(p.value_at_risk) AS value_at_risk
FROM projects p
         JOIN map_leaf_to_summary((
    SELECT dept_id id, parent_dept_id parent_id, risk_report report
    FROM org
)) map ON p.dept_id = map.leaf_id
GROUP BY map.summary_id;
```

Query Functions have allowed us to abstract out a reusable transformation.  ```map_leaf_to_summary()``` is reusable in any 
hierarchy to map and summarise
information from the leaf level to a data defined reporting level.

Let's look at code reuse statistics comparing the SQL solution and the SQL with query functions solution. 

#### Code Statistics

| Language | Lines of Code for two queries | Duplication across code                                 | Reusable Components                                                                                                                                                                                     |
| ----  |-------------------------------|---------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| SQL | 66                            | 100% all code has at least one duplicate                | 0% nothing reusable                                                                                                                                                                                     |
| SQL with query functions | 64 | 20% as the core ```SELECT``` has at least one duplicate | 80% of the code is packaged for reuse<br/>```isLeaf(isLeaf) GIVEN (tree(id, parent_id), item(id))```, and <br/>```map_leaf_to_summary(summary_id, leaf_id) GIVEN (tree(id, parent_id, report_flag)) ``` |

This is an excellent result of 80% reusability.  In a later worked example when we introduce pipelines, we will increase the 
reusable code further.

Before turning to the simplicity of implementation, lets build on the query functions and introduce query lambdas and pipelines.

#### Query Lambdas
The SQL CTE may be extended with a ```GIVEN``` clause to give an anonymous query function, that is, a query lambda. The CTE-based query function **inherits** the CTE's lambda nature, and can be defined and invoked in place by wrapping the body in
brackets and following it with bracketed parameters, one table or subquery, for each ```local_table_name``` in each of the ```GIVEN``` 
clauses 
in lexical order:
```sql
(WITH ctename(column, ...)
         GIVEN (local_table_name(column, ...), ...) 
         AS (SELECT ...), 
     ...
 SELECT ...)(table_or_subquery, ...)
```

One use case for this lamdba is as a transformation stage in a pipeline, which we will look at next.

#### Query Pipelines
Some data transformations may be modelled as a staged pipeline of changes with each stages consuming the results 
of an earlier stage. Here is a diagram that illustrates a two stage transformation, using query functions: 
- ```qf1()``` which accepts two tables or subqueries as input parameters from the source data 
- ```qf2()``` which also requires two input tables or subqueries, one being the output of qf1(), and an additional one from the source data.

![Pipeline](https://user-images.githubusercontent.com/1667631/147862840-b81a9f17-cd1f-4143-8f53-035d93abc194.png)


<details>
<summary><em>Click to view Graphviz code for this diagram</em></summary>

```
digraph G {
rankdir=LR;
node [ color=gray, fontname=arial, fontsize=8];
edge [  fontname=arial, fontsize=8];
qf1[label=<Transformation 1<br/><br/>qf1()>shape=box]
qf2[label=<Transformation 2<br/><br/>qf2()>shape=box]
"Source Data" -> qf1[label="table_or_subqueryA"]
qf1->qf2
qf2->Result
"Source Data" -> qf1[label="table_or_subqueryB"]
}
```
</details>

Building on the definitions of query functions and lambdas, a ```PIPES TO``` compound operator is proposed to allow functional 
programmers to express computations using a pipeline model.

Here is the ```PIPES TO``` operator used to express the two stage pipeline consisting of ```qf1()``` and ```qf2()```:

```sql
SELECT *
FROM (
    qf1(table_or_subqueryA, table_or_subqueryB)
    
    PIPES TO
        
    qf2(table_or_subqueryC)
    );
```
Notice that the invocation```qf2(table_or_subqueryC)``` has just one parameter, the first parameter is provided by the 
```PIPES TO``` compound operator.

Either, or both, of the query functions could be replaced by a lambda. Here is the example with qf2() rewritten as a 
lambda:

```sql
select *
from (qf1(table_or_subqueryA, table_or_subqueryB) 

      PIPES TO

      (WITH ctename(column, ...)
             GIVEN (local_table_name1(column, ...), local_table_name2(column, ...)) 
            AS (SELECT ...)
      SELECT ...)(table_or_subqueryC)
    )
```

#### Using Query Pipelines to further simplify our scenario.
We are going to use query pipelines to abstract out further reusable code in our reporting scenario.

Here is the summarising part of the solution turned into a query function ```summarise()```: 
```sql
DEFINE QUERYFUNCTION summarise(summary_id, total) 
    GIVEN (mapping(summary_id, id), rawdata(id, amount)) 
    AS SELECT mapping.summary_id, sum(rawdatea.amount)
    FROM rawdata rawdata
             JOIN mapping mapping ON rawdata.id = mapping.id
    GROUP BY map.summary_id;
```
The first parameter is the mapping and the second is the raw data to be summarised. Here is the solution to the ```Cost_Summary``` using ```PIPES TO``` to provide the mapping.

````sql
SELECT * FROM (map_leaf_to_summary((SELECT dept_id id, parent_dept_id parent_id, cost_report report FROM org))
               
               PIPES TO
               
               summarise((SELECT dept_id id, estimate_to_completion amount FROM project))
              );
````

And here is the corresponding solution for the ```Risk_Summary```:
````sql
SELECT * FROM (map_leaf_to_summary((SELECT dept_id id, parent_dept_id parent_id, risk_report report FROM org)))
               
               PIPES TO
               
               summarise((SELECT dept_id id, value_at_risk amount FROM project))
              );
````

### Simplicity of implementation
Query functions, lambdas, and pipelines are zero-cost abstractions.  They may all be trivially reduced back to 
SQL and made available to the query planner / optimiser to execute.  

Here is a high level description of the translation to SQL followed by a fully worked example.

#### Translating Query functions, lambdas, and pipelines to SQL
The following three translations, applied iteratively, until they reach a fixed point, will translate query functions, lambdas, 
and pipelines to SQL.

| Feature         | Translation approach                                                                                                                                                                                 |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Query Function  | Invocations may be reduced by inline macro-like expansion with a copy of the function's body which has first had each table in the GIVEN clause replaced with the parameters which have been passed. |
| Query Lambda    | Invocation of a query lambda may be reduced by removal of the GIVEN Clause after replacing each local_table_name with the table or subquery passed as parameters                                     |
| Query Pipeline  | Each ```PIPES TO``` usage can be reduced by adding the preceding query function/lambda as an explicit parameter to the succeeding query function/lambda.                                             


#### Worked example of the translation of Query functions, lambdas, and pipelines to SQL
For the purposes of this demonstration, we are going to translate the ```Cost_Summary``` to solution to SQL. 

````sql
SELECT * FROM (map_leaf_to_summary((SELECT dept_id id, parent_dept_id parent_id, cost_report report FROM org))
               
               PIPES TO
               
               summarise((SELECT dept_id id, estimate_to_completion amount FROM project))
              );
````


But before we do, 
let's replace the invocation of the ```summarise()``` query function with a lambda.  This way we have examples of all three forms:
- query function,
- lambda, and
- pipeline

to work with.

```sql
SELECT *
FROM (
    map_leaf_to_summary((
            SELECT dept_id id, parent_dept_id parent_id, cost_report report
            FROM org
            )) 
    
    PIPES TO 
    
    (
        WITH summarise(summary_id, total) GIVEN(mapping(summary_id, id), rawdata(id, amount)) AS (
                SELECT mapping.summary_id, sum(rawdatea.amount)
                FROM rawdata rawdata
                JOIN mapping mapping ON rawdata.id = mapping.id
                GROUP BY map.summary_id
                )
        SELECT *
        FROM summarise
        ) (
        (
            SELECT dept_id id, estimate_to_completion amount
            FROM project
            )
        )
    );
    
```

**Note: Just to be clear, this is not a proposal on how query functions, lambdas, and pipelines should be implemented.** The 
purpose of this translation exercise is to show that the proposed features, are well-defined and can be trivially reduced to 
standard SQL-99.

#### Step 1. Reduce the ```PIPES TO``` by moving the preceding query function or lambda to be a parameter to the succeeding query function or lambda.
```sql
SELECT *
FROM (
    (
        WITH summarise(summary_id, total) GIVEN(mapping(summary_id, id), rawdata(id, amount)) AS (
                SELECT mapping.summary_id, sum(rawdatea.amount)
                FROM rawdata rawdata
                JOIN mapping mapping ON rawdata.id = mapping.id
                GROUP BY map.summary_id
                )
        SELECT *
        FROM summarise
        ) (
        map_leaf_to_summary((
                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                FROM org
                )), (
            SELECT dept_id id, estimate_to_completion amount
            FROM project
            )
        )
    );
```
#### Step 2. Reduce the Query Lambda to a SQL CTE
```sql
SELECT *
FROM (
    (
        WITH summarise(summary_id, total) AS (
                SELECT mapping.summary_id, sum(rawdatea.amount)
                FROM map_leaf_to_summary((
                            SELECT dept_id id, parent_dept_id parent_id, cost_report report
                            FROM org
                            )) mapping
                JOIN (
                    SELECT dept_id id, estimate_to_completion amount
                    FROM project
                    ) rawdata ON rawdata.id = mapping.id
                GROUP BY map.summary_id
                )
        SELECT *
        FROM summarise
        )
    );
```

#### Step 3. Reduce the ```map_leaf_to_summary()``` Query Function by macro-like expansion using the body of the function
```sql
SELECT *
FROM (
    (
        WITH summarise(summary_id, total) AS (
                SELECT mapping.summary_id, sum(rawdatea.amount)
                FROM (
                    WITH antichain(summary_id, leaf_id) AS (
                            SELECT CASE 
                                    WHEN report OR (
                                            is_leaf((
                                                    SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                    FROM org
                                                    ), (
                                                    SELECT tree.id
                                                    ))
                                            )
                                        THEN id
                                    END, id
                            FROM (
                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                FROM org
                                ) tree
                            WHERE id = parent_id
                            
                            UNION
                            
                            SELECT CASE 
                                    WHEN antichain.summary_id IS NOT NULL
                                        THEN antichain.summary_id
                                    WHEN tree2.report OR (
                                            is_leaf((
                                                    SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                    FROM org
                                                    ), (
                                                    SELECT tree2.id
                                                    ))
                                            )
                                        THEN tree2.id
                                    END, tree2.id
                            FROM antichain
                            JOIN (
                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                FROM org
                                ) tree2 ON tree2.parent_id = antichain.leaf_id AND tree2.id <> tree2.parent_id
                            )
                    SELECT summary_id, leaf_id
                    FROM antichain
                    WHERE summary_id IS NOT NULL
                    ) mapping
                JOIN (
                    SELECT dept_id id, estimate_to_completion amount
                    FROM project
                    ) rawdata ON rawdata.id = mapping.id
                GROUP BY map.summary_id
                )
        SELECT *
        FROM summarise
        )
    );
```
#### Step 4. Reduce the ```isLeaf()``` Query Function by macro-like expansion using the body of the function
```sql
SELECT *
FROM (
    WITH summarise(summary_id, total) AS (
            SELECT mapping.summary_id, sum(rawdata.amount)
            FROM (
                WITH antichain(summary_id, leaf_id) AS (
                        SELECT CASE 
                                WHEN report OR (
                                        (
                                            SELECT CASE 
                                                    WHEN NOT EXISTS (
                                                            SELECT id
                                                            FROM (
                                                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                                FROM org
                                                                ) tree2
                                                            WHERE tree2.parent_id = tree.id
                                                            ) OR 1 = (
                                                            SELECT count(*)
                                                            FROM (
                                                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                                FROM org
                                                                )
                                                            )
                                                        THEN 1
                                                    ELSE 0
                                                    END is_leaf
                                            FROM (
                                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                FROM org
                                                ) tree
                                            JOIN (
                                                SELECT tree.id
                                                ) item ON item.id = tree.id
                                            )
                                        )
                                    THEN id
                                END, id
                        FROM (
                            SELECT dept_id id, parent_dept_id parent_id, cost_report report
                            FROM org
                            ) tree
                        WHERE id = parent_id
                        
                        UNION
                        
                        SELECT CASE 
                                WHEN antichain.summary_id IS NOT NULL
                                    THEN antichain.summary_id
                                WHEN tree2.report OR (
                                        (
                                            SELECT CASE 
                                                    WHEN NOT EXISTS (
                                                            SELECT id
                                                            FROM (
                                                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                                FROM org
                                                                ) tree2
                                                            WHERE tree2.parent_id = tree.id
                                                            ) OR 1 = (
                                                            SELECT count(*)
                                                            FROM (
                                                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                                FROM org
                                                                )
                                                            )
                                                        THEN 1
                                                    ELSE 0
                                                    END is_leaf
                                            FROM (
                                                SELECT dept_id id, parent_dept_id parent_id, cost_report report
                                                FROM org
                                                ) tree
                                            JOIN (
                                                SELECT tree2.id
                                                ) item ON item.id = tree.id
                                            )
                                        )
                                    THEN tree2.id
                                END, tree2.id
                        FROM antichain
                        JOIN (
                            SELECT dept_id id, parent_dept_id parent_id, cost_report report
                            FROM org
                            ) tree2 ON tree2.parent_id = antichain.leaf_id AND tree2.id <> tree2.parent_id
                        )
                SELECT summary_id, leaf_id id
                FROM antichain
                WHERE summary_id IS NOT NULL
                ) mapping
            JOIN (
                SELECT dept_id id, estimate_to_completion amount
                FROM projects
                ) rawdata ON rawdata.id = mapping.id
            GROUP BY mapping.summary_id
            )
    SELECT *
    FROM summarise
    );
```
And we have now reached a fixed point and no further reductions may be made. The Query function, 
lambda and pipelines have been translated to vanilla SQL-99.

### Conclusion

2. Well defined (transpiles to SQl99)
1. Zero cost abstraction
3. Reuse
4. Functional side effect free
5. Bring the data to the algorithm

### Use cases

#### Standard algorithm with variable inputs from different columns from one table

```
select id, es, ls, ef, lf
from cpm((select id, estimated_remaining_duration 
          from tasks 
          where not complete));
```

```
select id, es, ls, ef, lf
from cpm((select id, original_estimated_duration 
          from tasks));
```

Alternatively

```
select id, estimated_remaining_duration 
    from tasks 
    where not complete PIPE TO CPM() 
PIPE TO 
select id, es, ls, ef, lf
from PIPE;

select id, es, ls, ef, lf
from (select id, estimated_remaining_duration 
    from tasks 
    where not complete PIPE TO CPM())

```

```
select id, es, ls, ef, lf
from cpm((select id, original_estimated_duration 
          from tasks));
```

In this area here are a set of common hierarchy / tree processing algorithms and their transpilation into SQL.

#### Where a single data source is referred to multiple times in particular ways

e.g. via self-join or union of subsets encapsulate this behaviour
[shared attribute](https://www.w3schools.com/sql/sql_join_self.asp)

#### Standardised classifications

```
select classifier1, classifier2, ...
from analyse((select * from customers), (select * from transactions)); 
```

```
select classifier1, classifier2, ...
from analyse((select * from customer_snapshots where snapshot = 34), 
             (select * from transaction_snapshots where snapshot = 34)); 
```

### New ideas

Why not add a construct equivalent to a.b().b(). ...
[Method Chaining](https://en.wikipedia.org/wiki/Method_chaining),
[Fluent Interface known by](https://en.wikipedia.org/wiki/Fluent_interface)

### Chaining / Fluent Interface

#### Common processing pipelines

Instead of:

```
select *
from step_2(step_1(customers)); 
```

write

```
select *
from (customers PIPE TO step_1() PIPE TO step_2()); 
```

or

```
customers PIPE TO step_1() PIPE TO step_2() PIPE TO SELECT * FROM PIPE 
```

And instead of

```
select *
from step_2(step_1(vendors)); 
```

write:

```
select *
from (vendorsor PIPE TO step_1() PIPE TO step_2()); 
```

write:

```
vendors PIPE TO step_1() PIPE TO step_2() PIPE TO SELECT * FROM PIPE; 
```
