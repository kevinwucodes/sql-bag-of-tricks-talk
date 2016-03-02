These are the notes from the SQL Bag of Tricks talk I gave at UCSB

# SQL (SQL Server edition)

### comments are awesome
```SQL
--this is a comment!

/*
this
is
also
a
comment
*/
```


### Second simplest statement ever?
```SQL
select 1
```

### 1 is an expression.  what about others?
```SQL
select 1, 'hello', 'world', 'hello' + 'world'
```

### basic math works too!
```SQL
select 2-3
```

### aliasing column names
```SQL
select	 'English' language
		, 'English' as language
		, language = 'English'
/*
language language language
-------- -------- --------
English  English  English

(1 row(s) affected)
*/
```

If you dont define column names, you get a "(no column name)" as a placeholder for the name of the column.  To have no column names is generally fine, especially if the query is the last resultset in the chain.  However, if you need to use a resultset that have no column names that end up participating in another resultset downstream from it, SQL Server will give you an error.

### "if"
```SQL
select case when 1=1 then 42 else '?' end meaningOfLife
/*
meaningOfLife
-------------
42

(1 row(s) affected)
*/
```

### huh? null?
```SQL
select null

select @@rowcount --1
--yes null is an expression too!
```
hmm, so select gives us rows, even when they are null!

### how about no rows?
#### first way
```SQL
select	1, null
from	(
			select null col1
		) t
where	col1 is not null

select @@rowcount --0
```
#### second way
```SQL
select	1, null
where	1=0

select @@rowcount --0
```

### lets make some rows
#### first way (unions)
```SQL  
  			select 1
	union	select 1		--union removes duplicates
/*
(no column name)
-----------
1

(1 row(s) affected)
*/
				select 1
	union all	select 1	--union all keeps them
/*
(no column name)
-----------
1
1

(2 row(s) affected)
*/
```

union only works when the number of columns of both tables are the same and the data types are compatible

#### second way (table value constructor)
```SQL
select	1 'one'
		, col1				
from	(
			values
				 ('a')
				,('b')
		) t(col1)
/*
one         col1
----------- ----
1           a
1           b

(2 row(s) affected)
*/  
```
Notice that we define the column names during the alias creation of "t".  We'll talk about that later.

### So by "making" rows, we can enumerate expressions across those rows
```SQL
select	stars
		, howmany = case when stars >= 10 then 'lots' else 'little' end
from	(
			values
				 (1)
				,(5)
				,(10)
				,(7)
				,(11)
		) t(stars)
/*
stars       howmany
----------- -------
1           little
5           little
10          lots
7           little
11          lots

(5 row(s) affected)
*/
```

### expressions from a "real", written-onto-the-disk table
```SQL
  select 'hello', 'world'
  from sys.objects
```

### that's too many, I can't count that high
```SQL
  --just the first 10 please
  select 	top 10
          'hello',
          'world'
  from sys.objects
```

### subqueries
#### 1 level deep
```SQL
select		col1
from	(	--subquery
			select 1
		) t(col1)
/*
col1
-----------
1

(1 row(s) affected)
*/
```
where "t" is defined as the alias of the subquery
We can define the column names of "t" during the alias creation of "t" like this:
t(col1, col2)

We dont have to do it this way.  Remember that we could define the column name as an alias to the expression we want to evaluate.

```SQL
select		col1

from	( --subquery
			select 1 col1		-- or as aliases in the column
		) t
/*
col1
-----------
1

(1 row(s) affected)
*/
```
#### 2 levels deep
```SQL
select		col1

from		(	--subquery 1
				select	col1

				from	(	--subquery 2
							select 1 col1
						) subq2

			) subq1
/*
col1
-----------
1

(1 row(s) affected)
*/
```
This is _nuts_, is there a better way?

### Common table expressions (CTE)
You can think of CTEs as a temporary, in-memory resultset that exist only in the execution context of the query.  Once the query finishes execution in a CTE, the CTE is gone.  This is different from a temporary table because temporary tables (depending on type) are written to disk and therefore can be used beyond the execution context of a query.
#### one table
```SQL
;with
s1 as (
	select 1	col1
	union select 2
	union select 3
)
select	col1
from	s1
where	col1 >= 2
/*
col1
-----------
2
3

(2 row(s) affected)
*/
```
#### two tables
```SQL
;with
s1 as (
	select 1	col1
	union select 2
	union select 3
)
,s2 as (
	select	col1
	from	s1
	where	col1 >= 2
)
select	col1
from		s2
where		col1 = 3
/*
col1
-----------
3

(1 row(s) affected)
*/
```

After this CTE runs, all references to "s1" and "s2" are now gone.  To SQL Server, these never existed.

### group by
```SQL
;with
s1 as (
			select 'kevin' name, 3 stars
	union all select 'kevin', 2
	union all select 'kevin', null
	union all select 'mike', 3
	union all select 'sally', 1
	union all select 'sally', 6
)
select		name
			, count(stars) rows		
from		s1
group by	name
/*
name  rows
----- -----------
kevin 2
mike  1
sally 2
Warning: Null value is eliminated by an aggregate or other SET operation.

(3 row(s) affected)
*/
```

Wait, what happened to my NULLs?  Group by doesn't care about them so lets rewrite this query if *we* care about them

```SQL
;with
s1 as (
			select 'kevin' name, 3 stars
	union all select 'kevin', 2
	union all select 'kevin', null
	union all select 'mike', 3
	union all select 'sally', 1
	union all select 'sally', 6
)
select		name
			, count(1) rows							--1st way
			, count(isnull(stars,0)) rows	--another way
from		s1
group by	name
/*
name  rows        rows
----- ----------- -----------
kevin 3           3
mike  1           1
sally 2           2

(3 row(s) affected)
*/
```
*You need to determine if it makes sense whether nulls should be counted or not*

#### more aggregates, HAVING vs WHERE
```SQL
;with
s1 as (
	select 'kevin' name, 3 stars
	union all select 'kevin', 2
	union all select 'kevin', null
	union all select 'mike', 3
	union all select 'mike', 8
	union all select 'sally', 1
	union all select 'sally', 6
)
select		name
			, count(1) rows		
			, sum(stars) totalStars
			, min(stars) minimumStar			
			, min(case when stars is null then 0 else stars end) minimumStarWithNull
from		s1
where		name <> 'kevin'
group by	name
having		min(stars) >=3
/*
name  rows        totalStars  minimumStar minimumStarWithNull
----- ----------- ----------- ----------- -------------------
mike  2           11          3           3

(1 row(s) affected)
*/
```
The where clause happens _before_ you group by.  The having clause happens _after_ you group by.

### windowed functions
```SQL
;with
stars(starsId, personId, stars, collectedDate) as (
			  select 1, 1, 5,  cast('2016-03-01' as datetime)
	union all select 2, 1, 9,  cast('2015-01-01' as datetime)
	union all select 3, 1, 2,  cast('2012-02-05' as datetime)
	union all select 4, 3, 10, cast('2014-04-01' as datetime)
	union all select 5, 3, 11, cast('2013-02-22' as datetime)
)
select		starsid
			, personid
			, stars
			, collectedDate
			, orderNothing = row_number() over (partition by personid order by (select null))
			, orderStar = row_number() over (partition by personid order by stars)
			, orderStarDesc = row_number() over (partition by personid order by stars desc)
			, orderDate = row_number() over (partition by personid order by collectedDate)
			, orderDateDesc = row_number() over (partition by personid order by collectedDate desc)

from		stars
/*
starsid     personid    stars       collectedDate           orderNothing         orderStar            orderStarDesc        orderDate            orderDateDesc
----------- ----------- ----------- ----------------------- -------------------- -------------------- -------------------- -------------------- --------------------
1           1           5           2016-03-01 00:00:00.000 3                    2                    2                    3                    1
2           1           9           2015-01-01 00:00:00.000 1                    3                    1                    2                    2
3           1           2           2012-02-05 00:00:00.000 2                    1                    3                    1                    3
4           3           10          2014-04-01 00:00:00.000 2                    1                    2                    2                    1
5           3           11          2013-02-22 00:00:00.000 1                    2                    1                    1                    2

(5 row(s) affected)
*/
```
There are lots of windowed functions: ```rank```, ```count```, ```sum```, ```ntile```, ...  Look them up in the SQL documentation and check out their examples

### joins
#### 2 join types that you'll use on a daily basis
##### inner join
the resulting join will show that each record in the two joined tables is matched
```SQL
;with persons(personId, name) as (
			  select 1, 'Kevin'
	union all select 2, 'Sally'
	union all select 3, 'Mike'
)
,stars(starId, personId, stars, collectedDate) as (
		  select 1, 1, 5,  cast('2016-03-01' as datetime)
	union all select 2, 1, 9,  cast('2015-01-01' as datetime)
	union all select 3, 1, 2,  cast('2012-02-05' as datetime)
	union all select 4, 3, 10, cast('2014-04-01' as datetime)
	union all select 5, 3, 11, cast('2013-02-22' as datetime)
)

select		p.personid
			, p.name

			, s.starId
			, s.personid
			, s.stars
			, s.collectedDate

from		persons	p
inner join	stars	s	on	s.personid = p.personid
/*
personid    name  starId      personid    stars       collectedDate
----------- ----- ----------- ----------- ----------- -----------------------
1           Kevin 1           1           5           2016-03-01 00:00:00.000
1           Kevin 2           1           9           2015-01-01 00:00:00.000
1           Kevin 3           1           2           2012-02-05 00:00:00.000
3           Mike  4           3           10          2014-04-01 00:00:00.000
3           Mike  5           3           11          2013-02-22 00:00:00.000

(5 row(s) affected)
*/
```

Notice that there is no Sally because Sally has no stars collected.

##### left join
the resulting join will contain all records from the "left" table, even if the join condition does not find a matching record from the "right" table
```SQL
;with
persons(personId, name) as (
			  select 1, 'Kevin'
	union all select 2, 'Sally'
	union all select 3, 'Mike'
)
,stars(starsId, personId, stars, collectedDate) as (
	select 1, 1, 5,  cast('2016-03-01' as datetime)
union all select 2, 1, 9,  cast('2015-01-01' as datetime)
union all select 3, 1, 2,  cast('2012-02-05' as datetime)
union all select 4, 3, 10, cast('2014-04-01' as datetime)
union all select 5, 3, 11, cast('2013-02-22' as datetime)
)

select		p.personid
			, p.name

			, s.starsId
			, s.personid
			, s.stars
			, s.collectedDate

from		persons	p
left join 	stars	s	on	s.personid = p.personid
/*
personid    name  starsId     personid    stars       collectedDate
----------- ----- ----------- ----------- ----------- -----------------------
1           Kevin 1           1           5           2016-03-01 00:00:00.000
1           Kevin 2           1           9           2015-01-01 00:00:00.000
1           Kevin 3           1           2           2012-02-05 00:00:00.000
2           Sally NULL        NULL        NULL        NULL
3           Mike  4           3           10          2014-04-01 00:00:00.000
3           Mike  5           3           11          2013-02-22 00:00:00.000

(6 row(s) affected)
*/
```

Sally is now shown because of left join, even though she has no stars collected

##### right join
A right join is similar to left join.  A right join is such that the resulting join will contain all records from the "right" table, even if the join condition does not find a matching record from the "left" table

This means that **a right join can be converted to a left join when you flip the order of the join**

The join resultset from this:
```SQL
from		stars	s
left join	persons p on p.personid = s.personid
```
is the same as this:
```SQL
from		persons p
right join	stars	s	on	s.personid = p.personid
```
So really, only 2 types you use **most** of the time.


### putting it all together
#### Question: Can we get a list of persons with their very first star count?

Before we had CTEs and windowed functions, we did something like this to get weighted rows:
```SQL
...
--pseudocode
from 				persons p
inner join	stars s	ON	s.starid =	(	-- this gets the first star record per personid
												 SELECT  TOP 1 starid
												 FROM    stars
												 WHERE   personid = p.personid
												 order by collectedDate
												)
```

...but this is a hack!

Instead, with CTEs and windowed functions, it looks like this:

```SQL
;with
persons(personId, name) as (
			  select 1, 'Kevin'
	union all select 2, 'Sally'
	union all select 3, 'Mike'
)
,stars(starsId, personId, stars, collectedDate) as (
	select 1, 1, 5,  cast('2016-03-01' as datetime)
union all select 2, 1, 9,  cast('2015-01-01' as datetime)
union all select 3, 1, 2,  cast('2012-02-05' as datetime)
union all select 4, 3, 10, cast('2014-04-01' as datetime)
union all select 5, 3, 11, cast('2013-02-22' as datetime)
)
,rankedStars as (
	select
				starsId, personid, stars, collectedDate
				, ranked = row_number() over (partition by personid order by collectedDate)

	from		stars  s
)
select		p.name
			,rs.stars
			,rs.collectedDate

from		persons		p

inner join	rankedStars	rs	on	rs.personid = p.personid
							and	rs.ranked = 1
/*
name  stars       collectedDate
----- ----------- -----------------------
Kevin 2           2012-02-05 00:00:00.000
Mike  11          2013-02-22 00:00:00.000

(2 row(s) affected)
*/
```
Yes, this might be longer, but it's easier to follow what's going on as each "table" is pipelined into the next table

### Final topic - recursive SQL

#### n + n of numbers
```SQL
;with tableN as (
	select 1 n
)
,b as (
	--anchor member (where you define a starting point)
	select	n
	from	tableN

	union all		--creating the "next" "rows" of data

	--recursive member
	select n + n
	from b --notice that the b here is defined in our CTE!  This is how SQL "loops" through recursively
	where	n + n < 1000	--terminating condition
)
--show final results
select	n
from	b
/*
n
-----------
1
2
4
8
16
32
64
128
256
512

(10 row(s) affected)
*/
```

#### fibonacci
```SQL
;with tableN as (
	select  start = 1
			,next = 1
)
,b as (
	--anchor member (where you define a starting point)
	select	start
			, next
	from	tableN

	union all		--creating the "next" "rows" of data

	----recursive member
	select next, start + next
	from b 		--notice that the b here is defined in our CTE!  This is how SQL "loops" through recursively
	where start < 100	--terminating condition
)
--show final results
select
		n = row_number() over (partition by (select null) order by (select null))
		,start 'fibonacci(n)'
from b
/*
n                    fibonacci(n)
-------------------- ------------
1                    1
2                    1
3                    2
4                    3
5                    5
6                    8
7                    13
8                    21
9                    34
10                   55
11                   89
12                   144

(12 row(s) affected)
*/
```

### What's the point?  Why recursion?
What's the real world problem you're trying to solve?

Hierarchy
* employee -> manager
* genealogy charts
* geographic region -> geographic region

Social
* friends -> friends

You'll likely use recursive SQL where you know your resultset is either tree-like or nested but you dont know the depth of the tree/nest

[Recursive Queries Using Common Table Expressions](https://technet.microsoft.com/en-us/library/ms186243(v=sql.105).aspx)

### Done!  Questions?
