# 2. Veri tabanı Yedekleme ve Felaketten Kurtarma Planı

Ağ Tabanlı Paralel Dağıtım Sistemleri dersi için yapılan 2. Veritabanı Yedekleme ve Felaketten Kurtarma Planı projesi

# BLM 4522 PROJE RAPORU 

Zeynep Hacısalihoğlu

22290449

## İçindekiler

- [1. Giriş](#1-giriş)
  - [1.1 Kullanılan Ortam](#11-kullanılan-ortam)
  - [1.2 Veri Tabanı Kurulumu](#12-veri-tabanı-kurulumu)
  - [1.3 Amaç ve Planlama](#13-amaç-ve-planlama)
- [2. Tam Yedekleme (Full Backup)](#2-tam-yedekleme-full-backup)
- [3. Fark Yedekleme (Differential Backup)](#3-fark-yedekleme-differential-backup)
- [4. Transaction Log Yedekleme](#4-transaction-log-yedekleme)

# 1.	Giriş
Bu proje Microsoft SQL Server üzerinde Northwind örnek veri tabanı kullanılarak kapsamlı bir yedekleme ve felaketten kurtarma planının tasarlanması ve uygulanmasını konu almaktadır. Proje boyunca tam yedekleme, fark yedekleme ve transaction log yedekleme stratejileri ele alınmış; bu yedeklerin otomatize edilmesi ve felaket senaryolarında veri kurtarma süreçleri uygulamalı olarak gerçekleştirilmiştir.

## 1.1	Kullanılan Ortam

-  Veritabanı Sistemi: Microsoft SQL Server 2022 Developer Edition, Sürüm 16.0.1000.6 
-  Yönetim Aracı: SQL Server Management Studio (SSMS) 

## 1.2	Veri Tabanı Kurulumu

Proje kapsamında kullanılacak örnek veri tabanı olarak Northwind seçilmiştir. Northwind Microsoft tarafından yayımlanmış, bir ticaret şirketinin sipariş, ürün, müşteri ve çalışan verilerini barındıran klasik bir örnek veritabanıdır. Veritabanı SSMS üzerinden başarıyla yüklenmiş ve aşağıdaki sorgu ile doğrulanmıştır:

```sql
USE Northwind;
SELECT name, database_id, create_date 
FROM sys.databases 
WHERE name = 'Northwind';
```

![Northwind DB Doğrulama](gorseller/gorsel1.png)

## 1.3 Amaç ve Planlama

Amaç: Bir felaket anında (sunucu çökmesi, yanlışlıkla silme, veri bozulması) veriyi geri getirebilmek.

1. Full Backup  => Tüm DB'nin şuanki halini kaydetmek
2. Differential Backup => Full'dan sonraki değişiklikleri yakalamak
3. Transaction Log Backup => Saatlik/dakikalık değişiklikleri yakalamak
4. Otomatik ZamanlamaBunları => SQL Agent ile otomatik çalışmasını sağlamak
5. Test aşaması => DB'yi silmek / veriyi bozmak
6. Geri Yüklemeye çalışmak


## 2. Tam Yedekleme (Full Backup)
Tam yedekleme veri tabanının tüm verilerini ve log dosyasını tek bir .bak dosyasına yazar. Diğer yedekleme türlerinin temeli olduğundan ilk adım olarak uygulanmıştır.

```sql
BACKUP DATABASE Northwind
TO DISK = 'C:\NorthwindBackups\Northwind_Full.bak'
WITH FORMAT,
     MEDIANAME = 'NorthwindBackup',
     NAME = 'Northwind Tam Yedekleme',
     STATS = 10;

```

Sonuç: Yedekleme işlemi başarıyla tamamlanmıştır. Northwind veritabanına ait 976 veri sayfası ve 1 log sayfası olmak üzere toplam 977 sayfa, 0.040 saniyede 190.661 MB/sn hızıyla C:\NorthwindBackups\Northwind_Full.bak dosyasına yazılmıştır. 

![Full Backup](gorseller/gorsel2.png)


## 3. Fark Yedekleme (Differential Backup)

Fark yedekleme, son tam yedeklemeden bu yana değişen sayfaları yazar. Tam yedeğe kıyasla çok daha hızlı ve küçük boyutludur.

```sql
BACKUP DATABASE Northwind
TO DISK = 'C:\NorthwindBackups\Northwind_Diff.bak'
WITH DIFFERENTIAL,
     NAME = 'Northwind Fark Yedekleme',
     STATS = 10;
```

Sonuç: 105 sayfa (104 veri + 1 log) 0.009 saniyede işlenmiştir. Full backup'taki 977 sayfaya kıyasla yalnızca değişen sayfalar yazıldığından boyut ve süre belirgin şekilde düşmüştür. 

![Diff Backup](gorseller/gorsel3.png)

## 4. Transaction Log Yedekleme
Transaction log yedekleme son log yedeklemesinden bu yana gerçekleşen tüm işlemleri kaydeder. Recovery model'in FULL olduğu durumlarda kullanılabilir. Northwind veritabanının recovery model'i aşağıdaki sorguyla kontrol edilmiş ve zaten FULL olduğu doğrulanmıştır:

```sql
SELECT name, recovery_model_desc 
FROM sys.databases 
WHERE name = 'Northwind';
```

![Doğrulama](gorseller/gorsel4.png)

Ardından log yedekleme alınmıştır:

``` sql
BACKUP LOG Northwind
TO DISK = 'C:\NorthwindBackups\Northwind_Log.bak'
WITH NAME = 'Northwind Log Yedekleme';
```

![Log Backup](gorseller/gorsel5.png)

Sonuç: Yalnızca 3 log sayfası 0.002 saniyede işlenmiştir. Bu, log yedeklemenin ne kadar hafif bir işlem olduğunu göstermektedir. 