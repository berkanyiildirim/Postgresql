# Gruplama Ve Grup Veri Üzerinde Çalışma 
```
postgres=# CREATE TABLE tb_ogrenci (
id int,
isim VARCHAR(50),
dogdugu_sehir VARCHAR(20),
dogum_tarihi date);
```
## GROUP BY: 

* **Group By**, sorgulanan verilerin içerisinde tekrar edenleri gruplayarak tek bir satırda göstermek için kullanılır.
* Group by ifadese, FROM ve WHERE tümcelerinden sonra gelmelidir. Sorguda, WHERE ifadesinden sonraki kriterler işletilden sonra Group By fonksiyonu işlemektedir

* 1987 yılında doğan kişileri doğdukları şehre göre gruplayalım :


```
postgres=# SELECT * from tb_ogrenci ;
 id |         isim          | dogdugu_sehir | dogum_tarihi 
----+-----------------------+---------------+--------------
  1 | berkan yıldırım       | Kırsehir      | 1996-01-01
 13 | Ibrahim Edip Kokdemir | Cankiri       | 1976-02-02
 28 | Dilara Tugce          | Ankara        | 1995-12-28
 57 | Selim Altınsoy        | Konya         | 1995-02-28
 77 | Murat Yıldız          | Ankara        | 1997-02-28
(5 rows)
```

```
postgres=# SELECT dogdugu_sehir FROM tb_ogrenci 
WHERE dogum_tarihi BETWEEN '1987-01-01' AND '1999-01-01'
GROUP BY dogdugu_sehir;
```
  Bu durumda **GROUP BY** ifadesi, sonuç kümesinde yenilenen satırları kaldırarak **DISTINCT** ifadesi gibi davranır.
```
 dogdugu_sehir 
---------------
 Ankara
 Kırsehir
 Konya
(3 rows)
```
*Eğer sonuç listemizin sıralamasını değiştirmek istiyorsak **ORDER BY** komutunu kullanabiliriz.*