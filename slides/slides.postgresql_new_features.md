## New feautures of Postgresql 9.3 to 9.5

---

# The SQL Language

--

## LATERAL JOIN

select * from opt_item_facs where id = 2;
```
 id |     fac_ids      | item_sources | manufacturer_id | client_vendor_id 
----+------------------+--------------+-----------------+------------------
  2 | {20,17}          | {1}          |         5169    |             3297
```

=>

```
 id |     fac_ids      | item_sources | manufacturer_id | client_vendor_id 
----+------------------+--------------+-----------------+------------------
  2 | {1003,1006}      | {1}          |         5169    |             3297
```

--

```
select t.id
  , array_agg(f.facility_number) as fac_ids
  , t.item_sources
  , t.manufacturer_id
  , t.client_vendor_id 
from (
  select *
    , unnest(fac_ids) as fac_id 
  from opt_item_facs f where id < 10000
) t
inner join facilities f on f.id = t.fac_id
group by t.id, t.fac_ids, t.item_sources, 
  t.manufacturer_id, t.client_vendor_id
ORDER BY t.id
```

--

```
select * 
from opt_item_facs ff
  , lateral (
      select array_agg(f.facility_number) 
      from unnest(ff.fac_ids) fff(fac_id) 
      inner join facilities f on f.id = fff.fac_id
  ) t  
where ff.id < 10000 
ORDER BY id
```

--

Materialized Views - REFRESH MATERIALIZED VIEW CONCURRENTLY
```
CREATE MATERIALIZED VIEW matview_user_groups AS
select email, oug.name AS group_name from org_users ou 
inner join org_user_user_groups ouug on ou.id = ouug.org_user_id
inner join org_user_groups oug on oug.id = ouug.org_user_group_id
where email like 'bridget.rembert@wfhc.org';

SELECT * FROM matview_user_groups;

REFRESH MATERIALIZED VIEW CONCURRENTLY matview_user_groups;
```

--

Recursive View Syntax
```
CREATE RECURSIVE VIEW t(n) AS
  VALUES (1)
UNION ALL
  SELECT n+1 FROM t WHERE n < 100;
```

--

WITH ORDINALITY
```
id |         fac_ids          
----+--------------------------
  3 | {28,18,1,17,21,19,12,20}
  4 | {19,1,12,21,18,17,5,20}
```

=>

```
id | fac_id | ordinality 
----+--------+------------
  3 |     28 |          1
  3 |     18 |          2
  3 |      1 |          3
  3 |     17 |          4
  3 |     21 |          5
  3 |     19 |          6
  3 |     12 |          7
  3 |     20 |          8
  4 |     19 |          1
  4 |      1 |          2
  4 |     12 |          3
  4 |     21 |          4
  4 |     18 |          5
  4 |     17 |          6
  4 |      5 |          7
  4 |     20 |          8
```

--

```
select ff.id
 , fac_id
 , u.ordinality 
from opt_item_facs ff, 
  unnest(fac_ids) with ordinality as u(fac_id) 
where id in (3,4)
```

--

Ordered-set aggregates

```
SELECT mode() WITHIN GROUP (ORDER BY vendor_code) 
FROM master_items;

mode  
--------
V72005
```

```
select vendor_code 
from master_items 
group by vendor_code 
order by count(1) desc limit 1;
```

--

Aggregate FILTER clause
```
select fi.master_item_id, 
 array_agg(facility_id) filter (where f.is_gpo) as gpo_fac_ids , 
 array_agg(facility_id) filter (where not f.is_gpo) as not_gpo_fac_ids
from facility_items fi 
inner join facilities f on fi.facility_id = f.id 
where master_item_id = 54
group by fi.master_item_id
```

=>

```
master_item_id | gpo_fac_ids |                      not_gpo_fac_ids                       
----------------+-------------+------------------------------------------------------------
             54 | {32,28}     | {18,22,8,19,3,23,13,15,6,24,4,21,1,10,7,9,11,2,12,5,17,20}

```

--

- Moving-aggregate support: computes the sum of values starting from the current row and including the next 10 rows after that
```
SUM(x) OVER (ORDER BY y ROWS BETWEEN CURRENT ROW AND 10 FOLLOWING)
```
- Performance of NUMERIC aggregates: The performance of various aggregate functions that use NUMERIC types internally has been improved. This includes the following aggregates:
  - SUM() and AVG() over bigint and numeric values.
STDDEV_POP(), STDDEV_SAMP(), STDDEV(), VAR_POP(), VAR_SAMP() and VARIANCE() over smallint, int, bigint and numeric values.

--

## GROUPING SETS
```
select is_discerned, is_stock, item_source, count(1)
from master_items
group by GROUPING SETS (is_discerned, is_stock, item_source, ());
```

```
 is_discerned | is_stock | item_source |  count  
--------------+----------+-------------+---------
 f            |          |             |  557771
 t            |          |             |  550596
              |          |             | 1108367
              |          |           1 |  276581
              |          |           2 |  675177
              |          |           3 |      33
              |          |           4 |  156576
              | f        |             | 1102525
              | t        |             |    5842
```

--

## CUBE
```
select fi.facility_id, approved_vendor_name, approved_mfr_name, count(1)
from master_items mi 
inner join facility_items fi on mi.id = fi.master_item_id
where fi.facility_id <> 1
group by CUBE (fi.facility_id, approved_vendor_name, approved_mfr_name) 

execution time: 02:13 min
```

```
facility_id  | approved_vendor_name | approved_mfr_name | count 
-------------+----------------------+-------------------+-------
           2 | DAVOL INC.           | DAVOL             |     1
           2 | DAVOL INC.           |                   |     1
           2 |                      |                   |     1
           3 | DAVOL INC.           | DAVOL             |     1
           3 | DAVOL INC.           |                   |     1
           3 |                      |                   |     1
             |                      |                   |     2
           2 |                      | DAVOL             |     1
           3 |                      | DAVOL             |     1
             |                      | DAVOL             |     2
             | DAVOL INC.           | DAVOL             |     2
             | DAVOL INC.           |                   |     2

```

--

## ROLLUP
```
select fi.facility_id, approved_vendor_name, approved_mfr_name, count(1)
from master_items mi 
inner join facility_items fi on mi.id = fi.master_item_id
where fi.facility_id <> 1
group by ROLLUP (fi.facility_id, approved_vendor_name, approved_mfr_name) 
```

```
 facility_id | approved_vendor_name | approved_mfr_name | count 
-------------+----------------------+-------------------+-------
           2 | DAVOL INC.           | DAVOL             |     1
           2 | DAVOL INC.           |                   |     1
           2 |                      |                   |     1
           3 | DAVOL INC.           | DAVOL             |     1
           3 | DAVOL INC.           |                   |     1
           3 |                      |                   |     1
             |                      |                   |     2

```

--

## INSERT ... ON CONFLICT DO NOTHING/UPDATE ("UPSERT")

```
INSERT INTO user_logins (username, logins)
 VALUES ('Naomi',1),('James',1)
 ON CONFLICT (username)
 DO UPDATE SET logins = user_logins.logins + EXCLUDED.logins;
```

--

## JSON / JSONB
```
table booksdata (
  title citext not null,
  isbn isbn not null primary key,
  pubinfo jsonb not null
)
```

```
CREATE INDEX ON booksdata USING GIN (pubinfo);
OR
CREATE INDEX ON booksdata USING GIN (pubinfo json_path_ops);
```

Get the average cost of all books from "It Books":
```
SELECT avg((pubinfo #>> '{"cost"}')::NUMERIC) FROM booksdata
WHERE pubinfo @> '{ "publisher" : "It Books" }';
```

--

jsonb || jsonb (concatenate / overwrite)
```
# SELECT '{"name": "Joe", "age": 30, 
"contact": {"phone": "01234 567890"}}'::jsonb 

|| 

'{"town": "London", "age": 40, 
  "contact": {"fax": "01987 654321"}}'::jsonb;

                    ?column?                   
 ----------------------------------------------
  {"age": 40, "name": "Joe", "town": "London", 
  "contact": {"fax": "01987 654321"}}
```

--

jsonb - text / int (remove key / array element)
```
SELECT '{"name": "James", "email": "james@localhost"}'::jsonb - 'email';
     ?column?      
-------------------
 {"name": "James"}
```

```
SELECT '["red","green","blue"]'::jsonb - 1;
     ?column?     
 -----------------
  ["red", "blue"]
```

--

jsonb #- text[] / int (remove key / array element in path)

```
SELECT '{"name": "James", 
"contact": {"phone": "01234 567890", "fax": "01987 543210"}}'::jsonb 
  #- '{contact,fax}'::text[];
                       ?column?                         
---------------------------------------------------------
  {"name": "James", "contact": {"phone": "01234 567890"}}

```

```
SELECT '{"name": "James", 
"aliases": ["Jamie","The Jamester","J Man"]}'::jsonb 
  #- '{aliases,1}'::text[];
                      ?column?                     
 --------------------------------------------------
  {"name": "James", "aliases": ["Jamie", "J Man"]}
```

--

### jsonb_set function

```
jsonb_set(
   target jsonb,           # The jsonb value you're amending.
   path text[],            # The path to the value you wish to add to or change, represented as a text array.
   new_value jsonb,        # The new object, key : value pair or array value(s) to add to or change.
   create_missing boolean  # An optional field that, if true (default), creates the value if the key doesn't already exist.
                           #   If false, the path must exist for the update to happen, or the value won't be updated.
 )
```

--

### jsonb_set function

```
SELECT jsonb_set('{"name": "James", "contact": {"phone": "01234 567890"}}'::jsonb,
            '{contact,skype}',
            '"myskypeid"'::jsonb,
            true);
                                               jsonb_set                                               
 ------------------------------------------------------------------------------------------------------
  {"name": "James", "contact": {"fax": "01987 543210", "skype": "myskypeid"}}
```

```
SELECT jsonb_set(
            '{"name": "James", "contact": {"phone": "01234 567890", "fax": "01987 543210"}}'::jsonb,
            '{contact,skype}',
            '"myskypeid"'::jsonb,
            false);                                   jsonb_set                                    
 --------------------------------------------------------------------------------
  {"name": "James", "contact": {"fax": "01987 543210", "phone": "01234 567890"}}
```

```
SELECT jsonb_set('{"name": "James", 
  "skills": ["design","snowboarding","mechnaicalengineering"]}',
  '{skills,2}',
  '"mechanical engineering"'::jsonb,
  true);
                                      jsonb_set                                     
 -----------------------------------------------------------------------------------
  {"name": "James", "skills": ["design", "snowboarding", "mechanical engineering"]}
```

---

## Server Administration

--

- Configuration directive 'include_dir'
- Parallel pg_dump for faster backups
```
pg_dump -U postgres -j4 -Fd -f /tmp/mydb-dump mydb
```
- Parallel VACUUMing: vacuumdb -j4 productiondb
- pg_isready: check the connection status of a PostgreSQL server

---

## Server Programming

- Custom Background Workers
- Event Triggers: Triggers can now be defined on DDL events (CREATE, ALTER, DROP)
- Replication Improvements

--

## Others

BRIN index

```
CREATE INDEX idx_master_items_created_at_brin 
  ON master_items USING BRIN (created_at);

CREATE INDEX idx_master_items_created_at_btree ON master_items (created_at);
```
```
\di+ idx_master_items_created_at_brin
                                         List of relations
 Schema |               Name               | Type  |  Owner   |    Table     |  Size  | Description 
--------+----------------------------------+-------+----------+--------------+--------+-------------
 public | idx_master_items_created_at_brin | index | postgres | master_items | 104 kB | 


\di+ idx_master_items_created_at_btree
                                         List of relations
 Schema |               Name                | Type  |  Owner   |    Table     | Size  | Description 
--------+-----------------------------------+-------+----------+--------------+-------+-------------
 public | idx_master_items_created_at_btree | index | postgres | master_items | 24 MB | 
```

--

- GIN indexes now faster and smaller
- ALTER SYSTEM -- change a server configuration parameter
- Row Security Policies
- Writeable Foreign Tables: postgres_fdw / Inheritance
- pg_prewarm
- pg_pathman
- pg_shard