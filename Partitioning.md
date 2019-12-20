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
Partitioning için aşağıdaki sınırlamalar geçerlidir:
- Tüm partitionlarda eşleşen dizinleri otomatik olarak oluşturmak için hiçbir imkan yoktur. Her bölüme ayrı komutlarla indeks eklenmelidir. Bu ayrıca, tüm bölümleri kapsayan bir birincil anahtar, benzersiz kısıtlama veya dışlama kısıtı oluşturmanın bir yolu olmadığı anlamına gelir; Her bir yaprak bölümünü tek tek sınırlamak mümkündür
- Birincil anahtarlar partitioning tablolarda desteklenmediğinden, bölümlenmiş tabloları gösteren yabancı anahtarlar desteklenmez, bölümlenmiş bir tablodan başka bir tabloya yabancı anahtar başvuruları da desteklenmez.
- `ON CONFLICT` ifadesini bölümlenmiş tablolarla kullanmak hataya neden olur, çünkü  unique veya exclusion kısıtlamaları yalnızca bireysel bölümlerde yaratılmıştır. Tüm bölümleme hiyerarşisinde benzersizliği (veya bir exclusion kısıtlamasını) zorlama desteği yoktur.
-  Bir satırın bir partitiondan diğerine hareket etmesine neden olan bir `UPDATE` başarısız olur, çünkü satırın yeni değeri orijinal bölümün örtülü bölüm sınırlamasını yerine getiremez.
-  Row triggers, gerekirse, bölümlenmiş tabloda değil, tek tek bölümlerde tanımlanmalıdır.
- Geçici ve kalıcı ilişkilerin aynı bölüm ağacında karıştırılmasına izin verilmez. Bu nedenle, bölümlenmiş tablo kalıcı ise, bölümler de aynı olmalıdır ve bölümlenmiş tablo geçici ise aynı şekilde olmalıdır. Geçici ilişkiler kullanılırken, bölüm ağacının tüm üyeleri aynı oturumdan olmak zorundadır.

**Kaynaklar**
* https://www.postgresql.org/docs/11/static/ddl-partitioning.html
* https://blog.2ndquadrant.com/partitioning-improvements-pg11/https://pgdash.io/blog/partition-postgres-11.html
 



