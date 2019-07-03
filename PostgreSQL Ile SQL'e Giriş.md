# POSTGRESQL'e Erişim 
* PostgreSQL kurulduğunda güvenlik ayarları olarak sadece localhost'u dinlemektedir.
* PostgreSQL'iin dinlediği IP listesi **postgresql.conf** dosyası içinde bulunur. Bu alan ön tanımlı olarak localhost olarak belirlenmiştir.
* Bu dosyanın değiştirilmesi işleminden sonra Postgresql servislerinin kapatılıp açılması gerekir.
  ```
  systemctl restart postgresql
  ```

* Postgresql'de erişim hakları **pg_hba.conf** dosyasında tanımlanır.

* **trust** : Kullanıcıların veritabanına parolasız bağlanmasını sağlar.
* **rejet** : Erişimi reddeder.
* **md5**   : md5 formatında şifrelenmiş parola ile giriş gerektirir.
* **crypt** : Bağlantı için crypt formatında şifrelenmiş parola girmesi gerekir.
* **password** : Düz metin parola ile girişe izin verir.
* **pam** : PAM kullanarak yetkilendirme sağlar.

* **pg_hba.conf** dosyasında yapılan değişikliklerin PostgreSQL servisinin etkinleşmesi için "pg_reload_conf();" komutu kullanılır. Bu sayede işletim sistemine gitmeden işlem yapılabilir.

**PostgreSQL'de Rol Kavramı :** PostgreSQL, veritabanı erişim izinlerini rol kavramı ile yönetir. Bir rol, bir veritabanı kullanıcısı veya bu veritabanı kullanıcılarından oluşan bir grup olabilir. 
* Tek kullanıcı bir rol olabilir, çünkü bir kullanıcı aynı zamanda bir roldür.
* Roller, veritabanı nesnelerinin (örn. tablolar) sahibi olabilir.
* Roller, veritabanı kümesinde (database cluster) geçerlidir.

*byildirim adında bir kullanıcı oluşturalım, superuser yetkisi olsun ve şifresi '12345' olsun;*
```
CREATE user byildirim with superuser password '12345';
```

**Yetkiler** : Bir nesne oluşturulduğunda, bir nesne sahibi olarak atanır. Çoğu nesne türü için başlangıç durumu sadece sahibinin nesneyle ilgili herhangi bir şey yapabilmesine olanak sağlar.
* owner : Normalde nesneyi yaratan roldür.
* Yetkileri atamak için **GRANT** komutu kullanılır.
* Bir yetkiyi iptal etmek için **REVOKE** komutu kullanılır.
* Nesne için **ALTER** komutu ile yeni sahibine bir nesne atanabilir.
* Yetki türleri : SELECT, INSERT, UPDATE, DELETE,EXECUTE, USAGE, CREATE, CONNECT, TEMPORARY.

**Rol Üyeliği** (ROLE MEMBERSHIP) : Yetkilerin yönetimini kolaylaştımak için genellikle gruplandırma yolu tercih edilir. Bu şekilde yetkiler bir gruba bütün olarak verilebilir veya gruptan bütün olarak kaldırılabilir.

**Postgresql'de Tablespaces** : Veritabanı yöneticilerinin veritabanı nesnelerini temsil eden dosyaların depolanabileceği dosya sistemindeki konumlarını belirtmelerine olanak sağlar. Veritabanı nesneleri oluşturulurken bir tablespace adına göre yönlendirilebilir. Tablespace kullanıldığında, administrator PostgreSQL kurulumunun disk düzenini kontrol edebilir.
**PostgreSQL'de MVCC** : İki veya daha fazla oturum aynı anda aynı veriye erişmeye çalıştığı durumlarda amaç, tüm oturumlar için etkin erişim ve veri bütünlüğünü sağlamaktır. Dahili olarak veri tutarlılığı, çoklu versiyon eşzamanlama kontrolü (MVCC) kullanılarak korunur. 

# TEMEL SQL KOMUTLARI 

* Keyword'ler ve alıntı içermeyen tanımlayıcılar (unquoted identifiers) case sensitive değildir.
* Keyword'lerin büyük harfler ile, Identifier'ların küçük harflerle yazılması tavsiye edilir.

**SELECT** komutunun kullanımı;
* **WITH** ile kullanım: With listesindeki tüm sorgular hesaplanır. Bunlar **FROM** listesinde referans gösterilebilecek geçici tablolar olarak etkili olur.
* **FROM** ile kullanım: FROM listesindeki tüm öğeler hesaplanır.
* **WHERE** ile kullanım : WHERE tümcesinde yer alan koşullara göre, bu koşulları sağlamayan tüm satırlar çıktıda yer almaz;
* **GROUP BY** ile kullanım : Çıktı bir veya daha fazla değerle eşleşen satırların gruplarına birleştirilir. **HAVING** tümcesi mevcutsa, verilen koşulu sağlamayan gruplar çıktıda yer almaz.
* **DISTINCT** ile kullanım : Sonucu tekrar eden satırlar tek olarak çıktıda yer alır.
* **ORDER BY** ile kullanım : Sonuçta yer alan satırlar belirtilen sırada sıralanır.
* **LIMIT veya OFFSET** ile kullanım : Bu durumda SELECT tümcesi, yalnızca sonuç satırlarının bir alt kümesini döndürür.
* **JOIN**'ler ile kullanım : LEFT, RIGHT, FULL OUTER,CROSS, INNER olarak kullanılabilir.
* **BETWEEN, IN ve LIKE** ile kullanım : BETWEEN ile belirli aralık içerisinde yer alan sonuç değerlerini döndürür. IN tümcesi ile içinde verilen sunuçların sorgu çıktısında yer alan değerleri döndürür.

*en sade haliyle **SELECT** kullanımı:*
```
SELECT * FROM tablo_ismi;
```

* id değeri 1 ile 10 arasında olan öğrencileri bulalım:
  ```
  SELECT * FROM tb_ogrenci o WHERE o.id BETWEEN 1 AND 10;
  ```
* id değeri 1 ile 10 arasında olmayan öğrencileri bulalım:
    ```
    SELECT * FROM tb_ogrenci o WHERE o.id NOT BETWEEN 1 AND 10;
    ```
* id değeri 1,2,3,8,12 ve 50 olan öğrencileri bulalım :
  ```
  SELECT * FROM tb_ogrenci o WHERE o.id IN (1,2,3,8,12,50);
  ```
* Adı Ahmet ile başlayan tüm öğrencileri bulalım:
  ```
  SELECT * FROM tb_ogrenci o WHERE o.ad ILIKE 'ahmet%'
  ```
* Adı içince Ahmet geçen tüm öğrencileri bulalım:
  ```
  SELECT * FROM tb_ogrenci o WHERE o.ad ILIKE '%ahmet%'
  ```
*ILIKE ile yapılan sorgularda büyük küçük harf ayrımı yapılmaz*

* Soyadı boş (NULL) olmayan tüm öğrencileri bulalım:
  ```
  SELECT * FROM tb_ogrenci o WHERE o.soyad  IS NOT NULL;
  ```
**DISTINCT İle Tekil Veri Getirme :** 
* Sonuç listesinde tekrarlı veriler istenmiyorsa **DISTINCT** ifadesi kullanılır.
  ```
  SELECT DISTINCT dogdugu_sehir FROM tb_ogrenci;
  ``` 
**ALIAS İle Sütun Ve Tablolara Takma Isim Vermek :** 
Bir tabloyu veya sütunu geçici olarak yeniden adlandırmak için ALIAS kullanılır. Bu işlem geçicidir, veritabanında gerçek tablo veya sütun adları değişmez.
  ```
    SELECT ogr.ogrenci_adi, ogr.ogrenci_soyadi,ogr.ders_adi,ogr.ders_adi, ogr.vize_notu FROM tb_ogrenci AS ogr WHERE ogr.vize_notu >=70; 
  ```