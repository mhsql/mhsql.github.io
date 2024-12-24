---
layout: post
title: "JSON in SQL Server"
---
## How to interact with JSON data

With a [JSON Datatype (Preview)](https://learn.microsoft.com//sql/t-sql/data-types/json-data-type) available in Azure SQL and expected from SQL Server 2025, it's time to have a quick check of the [JSON functions](https://learn.microsoft.com//sql/t-sql/functions/json-functions-transact-sql)

We know the Dev's love JSON and we've had to support JSON in a `NVARCHAR(MAX)` datatype
<br>The new datatype promises better performance, as the internal code will work with a binary JSON object not just a string of text that may not be valid

With a few options available, and I don't use them all, being able to *read* JSON and data cleanse it shouldn't be that scary
<br>With a dedicated datatype, you'll have less chance to say no to the Dev's who want to do more with JSON inside the database

### Creating JSON Documents
#### JSON From a Query

This is relatively easy, with some tricks, and this is where `FOR JSON` and `JSON_QUERY` come in
<br>If we have a nice simple query to get the database objects by schema, we can use

*FOR JSON AUTO* will choose the key names to use
<pre>SELECT s.[name] AS [schema]
    , o.[name] AS [objects.name]
    , o.type_desc AS [objects.type]
  FROM sys.schemas s
    INNER JOIN sys.objects o
        ON o.schema_id = s.schema_id
 FOR JSON AUTO;</pre>

The choice of Key Name for the 1st column is the Column's Alias
<br>The next Key **o**, is the Table Alias and then the Column Aliases are used inside the Array
<pre>
[{"schema": "sys",
    "o": [{"objects.name": "sysrscols", "objects.type": "SYSTEM_TABLE"
        },...,{}]
    },
    {"schema": "dbo",
     "o": [{"objects.name": "spt_values","objects.type": "VIEW"
            },...,{}]
    }
]
</pre>

*FOR JSON PATH* uses the Key name from the Path in the column name/alias;
<pre>SELECT s.[name] AS [schema]
    , o.[name] AS [objects.name]
    , o.type_desc AS [objects.type]
  FROM sys.schemas s
    INNER JOIN sys.objects o
        ON o.schema_id = s.schema_id
 FOR JSON PATH;</pre>

This is not ideal, it processes each row as a list item with *Schema* and *Objects* included in each item
<br>And it didn't group by the *sys.schemas* table like `FOR JSON AUTO`

<pre>[{ "schema": "sys",
    "objects": { "name": "sysrscols", "type": "SYSTEM_TABLE" }
    },
    { "schema": "sys",
    "objects": { "name": "sysrowsets", "type": "SYSTEM_TABLE" }
    },...,
    { "schema": "dbo",
    "objects": { "name": "sp_MScleanupmergepublisher"
                , "type": "SQL_STORED_PROCEDURE" }
    }
]</pre>

What we need is a Sub-Query to use instead of the *INNER JOIN*
<pre>SELECT s.[name] AS [schema]
    , (SELECT o.[name] AS [name], o.type_desc AS [type]
         FROM sys.objects o
        WHERE o.schema_id = s.schema_id
        FOR JSON PATH) AS [objects]
  FROM sys.schemas s
 FOR JSON PATH;</pre>

 You may hit lucky, the Sub-Query can *escape* the JSON text when nesting the *FOR JSON* query
 <br>That is when `JSON_QUERY` comes in to force the string to a JSON Object

<pre>SELECT s.[name] AS [schema]
    , JSON_QUERY((SELECT o.[name] AS [name], o.type_desc AS [type]
         FROM sys.objects o
        WHERE o.schema_id = s.schema_id
        FOR JSON PATH)) AS [objects]
  FROM sys.schemas s
 FOR JSON PATH;</pre>

 I hit lucky and both returned
 <pre>[{ "schema": "dbo",
        "objects": [{"name": "spt_fallback_db", "type": "USER_TABLE" },
            { "name": "spt_fallback_dev", "type": "USER_TABLE" },
            ...,
            { "name": "sp_MScleanupmergepublisher", "type": "SQL_STORED_PROCEDURE" }
        ]
    },
    { "schema": "sys",
        "objects": [{ "name": "sysrscols", "type": "SYSTEM_TABLE" },
            { "name": "sysrowsets", "type": "SYSTEM_TABLE" },
            ...,
            { "name": "polaris_executed_requests_text", "type": "INTERNAL_TABLE" }
        ]
    }
]</pre>

That's just two system tables with a hierarchical relationship
<br>I'm sure you have similar tables
<br>While the relationships may get complex, that's where your skill some in to get the query you need

#### JSON Into a Parameter

For some projects I work on, the database needs to log calls from an Api
<br>It adds logging not available to the calling code

This is a basic sample of how that code works;
<pre>DECLARE @startTime DATETIME2 = SYSUTCDATETIME()
    , @JSONLog NVARCHAR(256) = N'{}';

SELECT COUNT(1) rowsFound INTO #dropMe FROM sys.objects;
SET @JSONLog = JSON_MODIFY(@JSONLog
                    , '$.query'
                    , JSON_QUERY((SELECT DATEDIFF(MICROSECOND
                                            , @startTime
                                            , SYSUTCDATETIME())
                                                / 1000000.0 AS elapsedTime
                                        , @@ROWCOUNT AS rowsFound FOR JSON PATH)));

SELECT ISJSON(@JSONLog) AS checkIsJSON
    , @JSONLog AS JSONLogDocument;

DROP TABLE #dropMe;</pre>

First I create the *@startTime* parameter, to cache the time before running the query
<br>And an Empty JSON object `@JSONLog NVARCHAR(256) = N'{}'`, it needs to be empty or a template with **NULL** value(s) that can be modified
<br>In this case I don't need the output, so I hide it with `INTO #dropMe` (and clean up with `DROP TABLE #dropMe;`)

This is the JSON document it creates
<pre>{
    "query": [
        {
            "elapsedTime": 0.024349000,
            "rowsFound": 1
        }
    ]
}</pre>

It's the [JSON_MODIFY](https://learn.microsoft.com//sql/t-sql/functions/json-modify-transact-sql) function that does the work.
<br>In this case, *@JSONLog* is empty
<br>It adds the key *query* to the root based on the path *'$.query'*
<br>Setting the values from the Sub-Query, forced to a JSON object by *JSON_QUERY*

There is a newer [JSON_OBJECT](https://learn.microsoft.com//sql/t-sql/functions/json-object-transact-sql) function that could replace the sub-query
<br>It allows you to create a JSON (sub)Document based on Key/Value pairings
<pre>JSON_OBJECT('elapsedTime':DATEDIFF(MICROSECOND, @t1, SYSUTCDATETIME()) / 1000000.0
                    , 'rowsFound': @@ROWCOUNT)</pre>

**NB** I have noticed issues with CI/CD and errors due to the semi-colon **:** used as a separator
<br>This may impact automation, though the code works the same way once deployed

The code can be repeated to capture details from each step/query in the code being logged
<br>And each *JSON_MODIFY* will update the document to replace the Key with the new value(s)

**NB** The Update is a complete replace based update and not a merge, if you need to add more context to a Key, then you may need to consider the update process when designing the JSON Document

#### Use Cases

Working with versioned Stored Procedures and a CI/CD pipeline to a K8s Cluster
<br>I have generic logging of the runtime values, the actual Procedure Name and the Host running it
<br>This can be used during a Red/Green deployment to check which Pods in the Cluster are calling the 'old'/'new' version and if the code was deployed as expected.

While I have a valid use case, most JSON documents will be created by Application Code
<br>Any additional Logging you need, may need a parameter to build up the document you need as a more flexible options

Otherwise you may only need to create a JSON Document as part of data cleansing
<br>*JSON_MODIFY* can be used to update the existing JSON Document directly
<br>And remember that with a replace update, you should test and validate your change code if you're not recreating the full Document

### Getting Data Out
#### Extract with OPENJSON()

I don't use [OPENJSON](https://learn.microsoft.com//sql/t-sql/functions/openjson-transact-sql), this demo code may help you understand why
<pre>DECLARE @json NVARCHAR(MAX);

SET @json='{ "name": "John"
            , "surname": "Doe"
            , "age": 45
            , "skills": [ "SQL", "C#", "MVC" ]
        }';

SELECT *
  FROM OPENJSON(@json);</pre>

This is the output

key | value | type
name | John | 1
surname | Doe | 1
age | 45 | 2
skills | [ "SQL", "C#", "MVC" ] | 4

As a data analyst, that can be useful to summarize the JSON Document
<br>As a DBA or DB Developer, it's not ideal for joining to tables and working with
<br>First the output would need to be pivoted
<br>And we may not know the column name or they may not exist in each run and cause issues

#### Extract with ISJSON(), JSON_QUERY() and JSON_VALUE()
 
Hmmm...maybe I'm cheating to include [ISJSON](https://learn.microsoft.com//sql/t-sql/functions/isjson-transact-sql), after all it just let's me know the *NVARCHAR* text is valid *JSON*
<br>And that is useful when the value was malformed and will cause issues in my code, I can check at two levels

1. With a basic `IF ISJSON(...) = 1` when dealing with parameters
1. With a `CASE WHEN ISJSON(...) = 1` to alter extract logic, this can work with Key's extracted from the main JSON Object too
1. While extracting a missing Key, which returns **NULL**, you can try [JSON_PATH_EXISTS](https://learn.microsoft.com//sql/t-sql/functions/json-path-exists-transact-sql) in your logic to know if that was a missing value or explicit **NULL**

That leaves the two key functions I use (and some tricks)

- [JSON_QUERY](https://learn.microsoft.com//sql/t-sql/functions/json-query-transact-sql) is good for Object Type Values, a String or Number Type will return **NULL**
- [JSON_VALUE](https://learn.microsoft.com//sql/t-sql/functions/json-value-transact-sql) is good for String or Number Type Values, an Object Type will return **NULL**

**NB** In both cases the Path is important and is Case Sensitive, meaning a JSON document may contain *name*, *Name* and *NAME* with different values

Code time, the same JSON Document again
<pre>DECLARE @json NVARCHAR(MAX);

SET @json='{ "name": "John"
            , "surname": "Doe"
            , "age": 45
            , "skills": [ "SQL", "C#", "MVC" ]
        }';

SELECT JSON_VALUE(jsn.json_obj, '$.name') AS [Name]
    , JSON_VALUE(jsn.json_obj, '$.surname') AS [Surname]
    , JSON_VALUE(jsn.json_obj, '$.age') AS [Age]
    , JSON_QUERY(jsn.json_obj, '$.age') AS [Age_Q]
    , JSON_VALUE(jsn.json_obj, '$.Age') AS [Age_case]
    , JSON_QUERY(jsn.json_obj, '$.skills') AS [Skills]
    , JSON_VALUE(jsn.json_obj, '$.skills') AS [Skills_value]
  FROM (SELECT JSON_QUERY(@json) json_obj) jsn;</pre>

 To emphasize the impact of Case and Value vs Object on extracting data

 - *Age_Q* - tries to find the Object value from the Key *age*
 - *Age_case* - tries to find the String or Number value from the Key *Age* with the wrong case
 - *Age* - extracts the Number value from the key *age* and then to confuse, aliases with a different case *Age*
 - *Skills_value* - tries to find a String or Number value for a key with an Object value, *skills*

This is the output

Name | Surname | Age | Age_Q | Age_case | Skills | Skills_value
John | Doe | 45 | **NULL** | **NULL** | [ "SQL", "C#", "MVC" ] | **NULL**

Ok, I hear you, the Skills column is an Object and you'd like the individual values
<br>With the help of scripts out of the community, based on [Number Series Generators](https://sqlperformance.com/2021/01/t-sql-queries/number-series-solutions-1), that's a littler easier than it may feel

There are two issues to extract an Item from a List or Array in JSON

1. We don't know how many items exist in the list, so can't rely on *nasty* code with hard coded indexes
1. And the Indexes are Zero based, if you start at 1 instead of 0, you skip the first entry

Back to our JSON document;
<pre>DECLARE @json NVARCHAR(MAX);

SET @json='{ "name": "John"
            , "surname": "Doe"
            , "age": 45
            , "skills": [ "SQL", "C#", "MVC" ]
        }';

/* This Number Series generated to idx via idx0
   The sample has 3 entries
   We have 0..3 (x4) values in the Series */
WITH idx0 (idx) AS (
SELECT 1 AS idx UNION ALL SELECT 1 AS idx
), idx (idx) AS (
/* Start the Series at 0 not 1
   Also, the Path is a string, we cast to VARCHAR */
SELECT CAST(ROW_NUMBER()
                OVER(ORDER BY i1.idx) - 1
            AS VARCHAR(19)) AS idx
  FROM idx0 AS i0
    CROSS JOIN idx0 AS i1
)
/* The same base query */
SELECT JSON_VALUE(jsn.json_obj, '$.name') AS [Name]
    , JSON_VALUE(jsn.json_obj, '$.surname') AS [Surname]
    , JSON_VALUE(jsn.json_obj, '$.age') AS [Age]
    , JSON_QUERY(jsn.json_obj, '$.skills') AS [Skills]
/* We need to convert back to a number and add 1, to get the nth list items number */
    , CAST(idx.idx AS TINYINT) + 1 AS [nth Skill]
/* The Path uses square brackets to identify an item in the list 
   '$.skills[' + idx.idx + ']'
   As the Index needs to be a String, you need the CAST if idx.idx is a number */
    , JSON_VALUE(jsn.json_obj, '$.skills[' + idx.idx + ']') AS [Skill]
  FROM (SELECT JSON_QUERY(@json) json_obj) jsn
    CROSS JOIN idx
/* To avoid returning one row per List/Array index
   The IS NOT NULL drops an Index that has no value
   For an Array, use this at Object level, in case the Key you pick is just NULL  */
 WHERE JSON_VALUE(jsn.json_obj, '$.skills[' + idx.idx + ']') IS NOT NULL;</pre>

Some comments in the code to help
<br>In this sample I converted the Series values to String for the *JSON_VALUE* function
<br>It's possible for the series value to be a number and *CAST* inside the *JSON_VALUE* function

The `WHERE` clause is about removing extra rows from the series with no data to extract
<br>With a List this may drop a row if there is a NULL value in the List
<br>With an Array, using *JSON_QUERY* for the Array Entry can avoid dropping based on the *check* Key inside the Array being NULL

Here is the output

Name | Surname | Age | Skills | nth Skill | Skill
John | Doe | 45 | [ "SQL", "C#", "MVC" ] | 1 | SQL
John | Doe | 45 | [ "SQL", "C#", "MVC" ] | 2 | C#
John | Doe | 45 | [ "SQL", "C#", "MVC" ] | 3 | MVC

If the List/Array order has significance, then `nth Skill` has meaning, otherwise it's a debug value

I hear you, your JSON is a little more complex and has Arrays, with Sub-Lists/Sub-Arrays
<br>And that is where I need to hand over to you, as you know your data structures better than me

Basically, you have what you need and just need to iterate

- You can extract a Value
- You can extract an Object
- You can extract from a List/Array by Index

Just amend the Path you use inside the JSON Functions

- '$.order.customer.address[0].addressline[2]' - the 3rd (index = 2) line of the Customers 1st (index = 0) Address from the Order
- '$.order.items[1].name' - the Name of the 2nd (index = 1) Item on the Order

And yes, this can be messy and why the developer should have the JSON Document model in code and deserialize in code.

While you can re-use the Index Id for different List, you may need to have multiple instances of the Index Series
<br>Then for the 1st Address you can get the 1st to Last Address Line and not just the 1 Address Line
<br>It's the same `CROSS JOIN idx`, just with explicit aliases

### Summary

Once you get passed the JSON is a dev thing and from the No SQL world
<br>The functions are powerful and can extend your abilities

In the case of logging, you can capture the values not available to the calling code
<br>And as JSON, you don't need an overly complex log table that handles everything
<br>That complexity is in the JSON Document Creation and Extract
<br>And in my use case, I Log to a [Temporal Table](https://learn.microsoft.com//sql/relational-databases/tables/temporal-tables) with limited history, for a self clearing log

For your development team, it can help reduce refactoring
<br>For one project, the JSON Document is create how the *target service* needs it
<br>The app code in the middle just passes the Document on *as-is*
<br>Any changes are a refactor in the Database and *target service*
<br>None of the other code is aware of the change

Sadly, the only new Index Support is as an Include Column
<br>So you can't re-create Cosmos DB Indexes from the JSON document
<br>That said, maybe test out indexes on persisted calculated columns