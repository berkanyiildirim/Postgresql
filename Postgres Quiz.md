## 1) Write Ahead Log (WAL) nedir?
PostgreSQL'de transactionların kaydedilmesi işlemidir.

## 2) MVCC (Multi Version Concurrency Control) nedir? 
Bir veri tabanında eş zamanlı olarak okuma ve yazma işlemlerinin sorunsuz yapılabilmesi adına geliştirilen bir modeldir.

## 3) Veri tabanı sayısını nasıl listeleyebiliriz?
psql -l 

## 4) Bir veritabanı nasıl oluşturulur?
/usr/pgsq-11/bin/createdb mydatabase 

## 5) -------  log mesajlarını syslog, eventlog, yada log dosyalarına iletir.
* Logging collector
* Row/Index Pointers
* Autovacuum launcher
* Stats collector

## 6) ---------- dirty data bloklarını diske yazar.
* Wal writer
* Autovacuum launcher
* Autovacuum workers
* Background Writer

## 7) ------ yeniden kullanım için boş alanı kurtarır.
* Autovacuum workers
* Disk Write Buffering
* WAL writer
* Disk Read Buffering
* Archiver

## 8)-------- config parametrelerine göre otomatik olarak bir kontrol noktası gerçekleştirir.

* Checkpointer process
* Wal writer
* Autovacuum workers
* Stats collector
 
## 9) ------- kullanım istatistiklerini ilişkiye ve bloğa göre depolar.
* Archiver
* Stats collector 
* Postmaster as Listener
* WAL

## 10) Hangisi exDB isimli bir database oluşturur?
* CREATE TABLE exTB;
* DROP tablename;
* CREATE DATABASE exDB;
* SELECT * FROM weather; 

## 11) Index size'ı öğrenmek için -------- fonksiyonunu kullanabiliriz.
* pg_relation_size
* information_schema.view_table_usage
* pg_catalog.pg_constraint
* pg_dump --version

## 12)Bir veritabanında şema listesi almak için geçerli yollar hangileridir?
* select nspname from pg_namespace;
* \dn
* psql -s
* Yukarıdaki ilk iki seçenek

## 13)Veritabanındaki tabloların bir listesini almanın geçerli yolları nelerdir?
* select tablename from pg_tables;
* \dt *.*
* select relname, relkind from pg_class where relkind = 'r';
* Hepsi

## 14) Şema search path postgresql'de nasıl bulunur?
* show search_path; 
* show search path;
* select * from search_path
* select * from paths where name = 'search'

## 15)