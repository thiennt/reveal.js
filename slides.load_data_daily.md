## Discuss the solution for the data load issues

---

## Data load issues
- File pending due to other clients
- File status not updated correctly
- FIle not processed the first run, then being ignored for the next runs (find a way to have it retry immediately?)
- File pending due to failure of first file in queue

--

## Expected
- Jane: 45 minutes <!-- .element: class="fragment" data-fragment-index="1" -->
- Phil: 10 minutes <!-- .element: class="fragment" data-fragment-index="2" -->

Note: test

---

## Statistic

--

## Files received per hour per org
```
request_type  | min | max  |         avg         
--------------+-----+------+---------------------
 CONUPLD      |   1 |   17 |  2.2842105263157895
 CONM         |   1 |  318 | 11.1552795031055901
 POH          |   1 |    3 |  1.0064620355411955
 CON          |   1 |    6 |  1.8806584362139918
 ITE          |   1 | 3865 |  8.9206176383252499
```

--

## Line count per file
```
request_type  | min | max     |         avg         
--------------+-----+---------+------------------
 POH          |  19 |135657   |  12206.6412910263
 CON          |   2 |2751575  |  252033.186637109
 ITE          |   0 | 0       |  0
```

---

## Stjose statistics

--

Files received per hour
```
request_type  | min | max  |         avg         
--------------+-----+------+---------------------
 ITE          |   1 |   21 |  5.1486210872
 POH          |   1 |    1 |  1
 CONM         |   1 |  318 |  7.1677944325
```

--

## Line count per file
```
request_type  | min | max     |         avg         
--------------+-----+---------+------------------
 ITE          |   0 | 0       |  0
 POH          |1109 | 135657  |  57745.2622950819
 CON          |   7 | 153493  |  2873.13139695712
```

--

## Process time by sync_statistics
```
request_type  | min               | max                     |   avg         
--------------+-------------------+-------------------------+------------------
 ITE          |  00:02:02.915479  | 22:34:00.767011         |  00:33:55.433255
 POH          |  01:06:35.217862  | 1 day 01:18:36.350299   |  02:53:40.002594
 CON          |  07:48:39.87253   | 1 day 15:04:16.505249   |  12:06:35.27371
```

--

## Process time by worker
```
request_type  | min               | max                     |   avg         
--------------+-------------------+-------------------------+------------------
 SyncWorker   |  02:32.13         | 01:53:26.38             |  0:09:50
 AutoWorker   |  0:32:15          | 12:37:47                |  6:46:26
```

---

## Review log

/home/administrator/Code/msss/Maintenance.ods

- Load IM error:  <!-- .element: class="fragment" data-fragment-index="1" -->
```
[#6362][ORG_ID 27] ERROR:  2016/02/11 15:00:21.254808 supplier_filter.go:184: pq: sorry, too many clients already
[##6353][ORG_ID 52] ERROR:  2016/02/11 15:00:21.470672 Mapping.go:69: pq: sorry, too many clients already
[#6523][ORG_ID 55] ERROR:  2016/02/11 15:00:24.771994 Mapping.go:69: pq: sorry, too many clients already
```

---

## Things to be considered
- Independent Org worker
- Process files sequentially
- When the files are really Completed
- Retry / resume / stop / start mechanism
- Deadlock / load data affect to user usage
- QUEUE or WORKER?
- No notification
- ???

---

### Solution

Note:
- Queue / Worker: http://www.celeryproject.org/, https://www.rabbitmq.com/, http://gearman.org/
=> http://stackshare.io/stackups/amazon-sqs-vs-celery-vs-gearman
- http://redis.io/
- Postgres-XL / citusdata (pg_shard extension): https://www.citusdata.com/blog/86-making-postgresql-scale-hadoop-style?utm_source=cooperpress&utm_medium=email&utm_campaign=nov16postgresweekly
- Monitoring: sensu
- Spark / Golang to build some data
- Use solr for AR / AW
- Use solr for Contract
- ???