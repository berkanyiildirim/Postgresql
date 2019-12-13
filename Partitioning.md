# PostgreSQL Table Partitioning

## Default partition kurulumu

```
CREATE TABLE measurement (
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);
```

```
CREATE TABLE measurement_y2016 
    PARTITION OF measurement 
    FOR VALUES FROM ('2016-01-01') TO ('2017-01-01');
```

```
CREATE TABLE measurement_y2017 
  PARTITION OF measurement 
  FOR VALUES FROM ('2017-01-01') TO ('2018-01-01');
```

```
INSERT INTO measurement 
  (logdate, peaktemp, unitsales) 
  VALUES ('2016-07-10', 66, 100);
```
* eklenen kayıt measurement_y2016 tablosuna gider.

# Partitionlar Arası Satırların Taşınması

```
SELECT * FROM measurement_y2016;

logdate   |peaktemp|unitsales|
----------|--------|---------|
2016-07-10|      66|      100|
```


```
UPDATE measurement SET logdate='2017-07-10';

logdate|peaktemp|unitsales|
-------|--------|---------|

```

```
SELECT * FROM measurement_y2017;

logdate   |peaktemp|unitsales|
----------|--------|---------|
2017-07-10|      66|      100|
```

# Default Partition Oluşturmak

* Gelen veri hiç bir partition tablosuna uymadığı durumlarda gitmesi için;


```
CREATE TABLE measurement_default PARTITION OF measurement DEFAULT;

INSERT INTO measurement (logdate, peaktemp, unitsales)
    VALUES ('2018-07-10', 66, 100);
```

```
SELECT * FROM measurement_default;

logdate   |peaktemp|unitsales|
----------|--------|---------|
2018-07-10|      66|      100|
```

# Otomatik Index Oluşturmak

```
CREATE INDEX ixsales ON measurement(unitsales);

\d measurement_y2016
           Table "public.measurement_y2016"
  Column   |  Type   | Collation | Nullable | Default 
-----------+---------+-----------+----------+---------
 logdate   | date    |           | not null | 
 peaktemp  | integer |           |          | 
 unitsales | integer |           |          | 
Partition of: measurement FOR VALUES FROM ('2016-01-01') TO ('2017-01-01')
Indexes:
    "measurement_y2016_unitsales_idx" btree (unitsales)
``` 

# Child Tabloda Otomatik Foreign Key Oluşturmak 

```
CREATE TABLE invoices ( invoice_id integer PRIMARY KEY );

CREATE TABLE sale_amounts_1 (
    saledate   date    NOT NULL,
    invoiceid  integer REFERENCES invoices(invoice_id)
)   PARTITION BY RANGE (saledate);
```

# Unique Index Oluşturmak

```
CREATE TABLE sale_amounts_2 (
    saledate   date NOT NULL,
    invoiceid  INTEGER,
    UNIQUE (saledate, invoiceid)
) PARTITION BY RANGE (saledate);
```
* Partition oluşturduğumuz zaman otomatik oluşacak.
```
CREATE TABLE sale_amounts_2_y2016 PARTITION OF sale_amounts_2
    FOR VALUES FROM ('2016-01-01') TO ('2017-01-01');

\d sale_amounts_2_y2016
         Table "public.sale_amounts_2_y2016"
  Column   |  Type   | Collation | Nullable | Default 
-----------+---------+-----------+----------+---------
 saledate  | date    |           | not null | 
 invoiceid | integer |           |          | 
Partition of: sale_amounts_2 FOR VALUES FROM ('2016-01-01') TO ('2017-01-01')
Indexes:
    "sale_amounts_2_y2016_saledate_invoiceid_key" UNIQUE CONSTRAINT, btree (saledate, invoiceid)
```

# Partition-level Aggregation

* Planlayıcı **enable_partitionwise_aggregate** parametresine göre “hash aggregate”i her bir partitiona dağıtır. Varsayılanı kapalıdır. Bunun daha hızlı sorguları beklenmelidir çünkü daha iyi paralellik ve lock kontrolü sağlar.

```
EXPLAIN SELECT logdate, count(*) FROM measurement GROUP BY logdate;
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 HashAggregate  (cost=3.05..3.08 rows=3 width=12)
   Group Key: measurement_y2016.logdate
   ->  Append  (cost=0.00..3.03 rows=3 width=4)
         ->  Seq Scan on measurement_y2016  (cost=0.00..1.00 rows=1 width=4)
         ->  Seq Scan on measurement_y2017  (cost=0.00..1.01 rows=1 width=4)
         ->  Seq Scan on measurement_default  (cost=0.00..1.01 rows=1 width=4)
(6 rows)
```

```
SET enable_partitionwise_aggregate=on;
```
```
EXPLAIN SELECT logdate, count(*) FROM measurement GROUP BY logdate;
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 HashAggregate  (cost=3.05..3.08 rows=3 width=12)
   Group Key: measurement_y2016.logdate
   ->  Append  (cost=0.00..3.03 rows=3 width=4)
         ->  Seq Scan on measurement_y2016  (cost=0.00..1.00 rows=1 width=4)
         ->  Seq Scan on measurement_y2017  (cost=0.00..1.01 rows=1 width=4)
         ->  Seq Scan on measurement_default  (cost=0.00..1.01 rows=1 width=4)
(6 rows)
```

# Partition by Hash

* Bir hash değerine göre gelen satırı partitionlara dağıtır.
```
CREATE TABLE hp ( foo text ) PARTITION BY HASH (foo);

CREATE TABLE hp_0 PARTITION OF hp FOR VALUES WITH (MODULUS 3, REMAINDER 0);

CREATE TABLE hp_1 PARTITION OF hp FOR VALUES WITH (MODULUS 3, REMAINDER 1);

CREATE TABLE hp_2 PARTITION OF hp FOR VALUES WITH (MODULUS 3, REMAINDER 2);

INSERT INTO hp SELECT md5(v::text) FROM generate_series(0,10000) v;
```

```
SELECT count(*) FROM hp_0;
 count 
-------
  3402
(1 row)

SELECT count(*) FROM hp_1;
 count 
-------
  3335
(1 row)

SELECT count(*) FROM hp_2;
 count 
-------
  3264
(1 row)
```

# Partition Bakımı 

* Artık gerekmeyen partitionu kaldırmak için;
```
DROP TABLE measurement_y2016;
```
* Genellikle tercih edilen başka bir seçenek, bölümü bölümlenmiş tablodan çıkarmak, ancak bu tabloya erişimi kendi başına bir tablo olarak tutmaktır:
```
ALTER TABLE measurement DETACH PARTITION measurement_y2016;
```
```
CREATE TABLE measurement_y2016 PARTITION OF measurement
    FOR VALUES FROM ('2008-02-01') TO ('2008-03-01')
    TABLESPACE fasttablespace
```

# Limitations 
* Partitioning, desteklenmeyen çeşitli özelliklere izin veren tablo Inheritance kullanılarak gerçekleştirilebilir:
* logs isimli bir parent tablo oluşturarak başlayalım
```
CREATE TABLE logs (
  created_at timestamp without time zone default now(),
  content text);
```
* Tabloyu tarihe göre yılın dört çeyreğine ayıracağız.
```
CREATE TABLE logs_q1(
  CHECK (created_at >= date '2014-01-01' and created_at <= date '2014-03-31')
) inherits (logs);

CREATE TABLE logs_q2(
  CHECK (created_at >= date '2014-04-01' and created_at <= date '2014-06-30')
) inherits (logs);

CREATE TABLE logs_q3(
  CHECK (created_at >= date '2014-07-01' and created_at <= date '2014-09-30')
) inherits (logs);

CREATE TABLE logs_q4(
  CHECK (created_at >= date '2014-10-01' and created_at <= date '2014-12-30')
) inherits (logs);
```
* Bir sonraki adım, her alt tablonun ana sütununda indeksler oluşturmaktır.

```
create index logs_q1_created_at on logs_q1 using btree (created_at);
create index logs_q2_created_at on logs_q2 using btree (created_at);
create index logs_q3_created_at on logs_q3 using btree (created_at);
create index logs_q4_created_at on logs_q4 using btree (created_at);
```
* Ardından, verileri alt tablolar arasında göndermek için bir tetikleyici işlevi oluşturalım.
```
create or replace function on_logs_insert() returns trigger as $$
begin
    if ( new.created_at >= date '2014-01-01' and new.created_at <= date '2014-03-31') then
        insert into logs_q1 values (new.*);
    elsif ( new.created_at >= date '2014-04-01' and new.created_at <= date '2014-06-30') then
        insert into logs_q2 values (new.*);
    elsif ( new.created_at >= date '2014-07-01' and new.created_at <= date '2014-09-30') then
        insert into logs_q3 values (new.*);
    elsif ( new.created_at >= date '2014-10-01' and new.created_at <= date '2014-12-31') then
        insert into logs_q4 values (new.*);
    else
        raise exception 'created_at date out of range';
    end if;

    return null;
end;
$$ language plpgsql;

CREATE FUNCTION
```

* tanımlanan tetikleyici işlevini logs tablosuna ekleyelim. 

```
create trigger logs_insert
  before insert on logs
  for each row execute procedure on_logs_insert();

CREATE TRIGGER
```
* Son olarak, partitionları görmek için bazı verileri logs tablosuna ekleyelim.
```
insert into logs (created_at, content) 
  values(date '2014-02-03', 'Content 1'),
        (date '2014-03-11', 'Content 2'),
        (date '2014-04-13', 'Content 3'),
        (date '2014-07-08', 'Content 4'),
        (date '2014-10-23', 'Content 5');
```
```
select * from logs_q1;
     created_at      |  content
---------------------+-----------
 2014-02-03 00:00:00 | Content 1
 2014-03-11 00:00:00 | Content 2

select * from logs_q2;
     created_at      |  content
---------------------+-----------
 2014-04-13 00:00:00 | Content 3

select * from logs_q3;
     created_at      |  content
---------------------+-----------
 2014-07-08 00:00:00 | Content 4

select * from logs_q4;
     created_at      |  content
---------------------+-----------
 2014-10-23 00:00:00 | Content 5

```

**Kaynaklar**
* https://www.postgresql.org/docs/11/static/ddl-partitioning.html
* https://zaiste.net/table_inheritance_and_partitioning_with_postgresql/ 
* https://blog.2ndquadrant.com/partitioning-improvements-pg11/https://pgdash.io/blog/partition-postgres-11.html
 



