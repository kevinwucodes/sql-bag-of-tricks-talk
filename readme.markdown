These are the notes from the SQL Bag of Tricks talk I gave to Enterprise Systems Integration at UCSB

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

### boolean
```SQL
select 1=1	--doesn't work  :(
```

### "if"
```SQL
select case when 1=1 then 'yes' else 'no' end
```

### huh? null?
```SQL
select null

select @@rowcount --1
--yes null is an expression too!
```
hmm, so select gives us rows, even when they are null!

### how about no rows?
```SQL
select	1
from	(
			select null
      ) t(col1)
where	col1 is not null

select @@rowcount --0

select 1
where	1=0

select @@rowcount --0
```

### lets make some rows
### expressions from values -- this enumerates expressions across your rows
#### first way
```SQL
  --union
  			select 1
	union	select 1		--union removes duplicates

	--union all
				select 1
	union all	select 1	--union all keeps them
```
#### second way
```SQL
  --table value constructor    
  select	1
					, col1
  from	(
    			values ('a'), ('a')
  			) t(col1)  
```
We just define the columns names differently.
Note that union (all) also requires you to define the column names  

### expressions from table
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
```
t is the alias of the "table"
We can define the column names of "t" during the alias creation of "t" like this:
t(col1, col2)

```SQL
-- or alternatively
select		col1

from	( --subquery
				select 1 col1		-- or as aliases in the column
			) t
```
#### 2 levels deep
```SQL
--2 level nested subqueries
select		col1

from		(	--subquery 1
				select	col1

				from	(	--subquery 2
							select 1 col1
						) subq2

				) subq1
```

### but common table expressions (CTE) are so much better
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
--row result:
--2
--3
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
```

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
```

wait, what happened to my NULLs?
group by doesn't care about them
so lets rewrite

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
			, count(1) rows								--1st way
			, count(isnull(stars,0)) rows	--another way
from		s1
group by	name

--determine if it makes sense whether nulls should be counted or not

--more aggregates
--HAVING vs WHERE

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
```

### windowed functions
```SQL
;with stars(starsId, personId, stars, collectedDate) as (
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
```
#### lots of windowed functions: rank, count, sum, ntile, ...

### joins
#### 3 join types that you'll use on a daily basis
##### inner join
the resulting join will show that each record in the two joined tables is matched
```SQL
;with persons(personId, name) as (
			  select 1, 'Kevin'
	union all select 2, 'Sally'
	union all select 3, 'Mike'
)
, stars(starsId, personId, stars, collectedDate) as (
		  select 1, 1, 5,  cast('2016-03-01' as datetime)
union all select 2, 1, 9,  cast('2015-01-01' as datetime)
union all select 3, 3, 10, cast('2014-04-01' as datetime)
)

select		p.personid
			, p.name

			, s.starsId
			, s.personid
			, s.stars
			, s.collectedDate

from		persons	p
inner join	stars	s	on	s.personid = p.personid
-- there is no Sally here because Sally has no stars collected
```

##### left join
the resulting join will contain all records from the "left" table, even if the join condition does not find a matching record from the "right table"
```SQL
;with persons(personId, name) as (
			  select 1, 'Kevin'
	union all select 2, 'Sally'
	union all select 3, 'Mike'
)
, stars(starsId, personId, stars, collectedDate) as (
		  select 1, 1, 5,  cast('2016-03-01' as datetime)
union all select 2, 1, 9,  cast('2015-01-01' as datetime)
union all select 3, 3, 10, cast('2014-04-01' as datetime)
)

select		p.personid
			, p.name

			, s.starsId
			, s.personid
			, s.stars
			, s.collectedDate

from		persons	p
left join 	stars	s	on	s.personid = p.personid
-- Sally is now shown because of left join, even though she has no stars collected
```
##### right join
....

##### a right join can be converted to a left join when you flip the order of the join
so really there are 2 join types that you'll use on a daily basis








joins on table restraints (dont put everything in the where clause)

```SQL

```





  from    table1  t1

  inner join  table2  t2 on t2.id = t1.id
                          and t2.status = 'happy'

  -- the join there is lowering the row count




### putting it all together
before CTE and windowed functions, to get weighted rows, we did something like this
(use travel as example)

....but this is a hack!

... the better way.

CTE, windowed functions


### SQL formatting!



### one last thing....  recursive SQL!

### fun with fibonacci



### done!
