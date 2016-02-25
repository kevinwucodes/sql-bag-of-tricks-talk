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

### huh? null?
```SQL
select null

select @@rowcount --1
--yes null is an expression too!
```
#### select gives us rows, even when they are null!

### how about no rows?
```SQL
select 1
from (
	select null
) t(col1)
where col1 is not null

select @@rowcount --0
```

### lets make some rows
### expressions from values -- this enumerates expressions across your rows
```SQL
  --first way, union (all)
  select 1 union select 1

  select 1 union all select 1

  --second way, table value constructor    
  select 1, col1
  from (
    values ('a'), ('a')
  ) t(col1)
  --but we just define the columns names differently
  --union also requires you to define the column names  
```

### expressions from table
```SQL
  select 'hello', 'world'
  from sys.objects
```

### that's too many, I can't count that high
```SQL
  --just the first 10 please
  select top 10
          'hello',
          'world'
  from sys.objects
```

### joins
joins on table restraints (dont put everything in the where clause)

  from    table1  t1

  inner join  table2  t2 on t2.id = t1.id
                          and t2.status = 'happy'

  -- the join there is lowering the row count

### group by



### subqueries



### but common table expression are so much better
```SQL
  ;with single as (
    select 1
  )
  select * from single
```

### windowed functions

before CTE and windowed functions, to get weighted rows, we did something like this
(use travel as example)

....but this is a hack!

... the better way.


### SQL formatting!


### one last thing....  recursive SQL!



### done!
