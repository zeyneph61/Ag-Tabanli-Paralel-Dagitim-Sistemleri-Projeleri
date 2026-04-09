# 3. Veritabanı Güvenliği ve Erişim Kontrolü

Ağ Tabanlı Paralel Dağıtım Sistemleri dersi için yapılan 3. Veritabanı Güvenliği ve Erişim Kontrolü projesi.

# BLM 4522 PROJE RAPORU 

Zeynep Hacısalihoğlu

22290449

## İçindekiler


# 1. Giriş

Bu proje Microsoft SQL Server 2022 üzerinde çalışan Northwind veritabanı üzerinde kapsamlı bir güvenlik altyapısının tasarlanması ve uygulanmasını konu almaktadır.

Proje kapsamında dört temel güvenlik konusu ele alınmıştır: kullanıcı erişim yönetimi, veri şifreleme (TDE), SQL Injection saldırılarına karşı koruma ve audit logları ile kullanıcı aktivitelerinin izlenmesi.

## 1.1 Kullanılan Ortam

Veritabanı Sistemi: Microsoft SQL Server 2022 Developer Edition, Sürüm 16.0.1000.6
Yönetim Aracı: SQL Server Management Studio (SSMS)
Örnek Veritabanı: Northwind

## 1.2 Mevcut Durum

Projeye başlamadan önce sistemdeki mevcut login'ler sorgulanmıştır:

```sql
SELECT name, type_desc, is_disabled
FROM sys.server_principals
WHERE type IN ('S', 'U')
ORDER BY name;
```

Sorgu çıktısında 2 SQL Login ve 7 Windows Login'in tanımlı olduğu görülmüştür. 

![Mevcut Login](gorseller/gorsel1.png)

## 1.3 Authentication Modu

SQL Server'ın hangi kimlik doğrulama modunda çalıştığı aşağıdaki sorgu ile kontrol edilmiştir:

```sql
EXEC xp_loginconfig 'login mode';
```

Sonuç: Login mode Mixed olarak yapılandırılmıştır. Bu mod hem SQL Server Authentication hem de Windows Authentication kullanımına izin vermektedir.

![Login Mode](gorseller/gorsel2.png)


## 2. Erişim Yönetimi

### 2.1 Login Oluşturma

Farklı yetki seviyelerinde üç kullanıcı oluşturulmuştur:

``` sql
USE master;
CREATE LOGIN db_admin WITH PASSWORD = 'Admin123!';
CREATE LOGIN db_readonly WITH PASSWORD = 'Readonly123!';
CREATE LOGIN db_dataentry WITH PASSWORD = 'Dataentry123!';
```

### 2.2 Kullanıcı ve Rol Atama

Login'ler Northwind veritabanına kullanıcı olarak eklenmiş ve uygun roller atanmıştır:

``` sql
USE Northwind;
CREATE USER db_admin FOR LOGIN db_admin;
CREATE USER db_readonly FOR LOGIN db_readonly;
CREATE USER db_dataentry FOR LOGIN db_dataentry;

ALTER ROLE db_owner ADD MEMBER db_admin;
ALTER ROLE db_datareader ADD MEMBER db_readonly;
ALTER ROLE db_datareader ADD MEMBER db_dataentry;
ALTER ROLE db_datawriter ADD MEMBER db_dataentry;
```

Kullanıcı-rol atamaları aşağıdaki sorgu ile doğrulanmıştır. 

``` sql
USE Northwind;
SELECT 
    dp.name AS kullanici,
    rol.name AS rol
FROM sys.database_role_members rm
JOIN sys.database_principals dp ON rm.member_principal_id = dp.principal_id
JOIN sys.database_principals rol ON rm.role_principal_id = rol.principal_id
WHERE dp.name IN ('db_admin', 'db_readonly', 'db_dataentry')
ORDER BY dp.name;
``` 

![Rol](gorseller/gorsel3.png)

### 2.3 Yetki Testi - db_readonly

Yetki testleri için SSMS'de yeni bir bağlantı açılmış ve SQL Server Authentication seçilmiştir. 

![db_readonly](gorseller/gorsel4.png)


db_readonly kullanıcısı ile DESKTOP-DLHAT6J\DB sunucusuna başarıyla bağlanılmıştır. Sol panelde bağlantının "db_readonly" kimliğiyle açıldığı görülmektedir.

![db_readonly1](gorseller/gorsel5.png)

Okuma testi (SELECT): Customers tablosundan veri sorgulanmış ve 5 kayıt başarıyla listelenmiştir. Bu, db_readonly kullanıcısının okuma yetkisine sahip olduğunu doğrulamaktadır. 

``` sql
USE Northwind;
SELECT TOP 5 * FROM Customers;
```

![db_readonly2](gorseller/gorsel6.png)


Yazma testi (INSERT): Customers tablosuna yeni kayıt ekleme denenmiş, ancak "The INSERT permission was denied on the object 'Customers'" hatası alınmıştır. Bu sonuç, db_readonly kullanıcısının yazma yetkisinin olmadığını doğrulamaktadır. 

``` sql
USE Northwind;
INSERT INTO Customers VALUES ('TEST', 'Test Firma', 'Mr', 'Test Co', NULL, NULL, NULL, NULL, NULL, NULL, NULL);
``` 

![db_readonly2](gorseller/gorsel7.png)


### 2.4 Yetki Testi — db_dataentry

db_dataentry kullanıcısı ile SQL Server Authentication kullanılarak bağlanılmıştır.

![db_dataentry1](gorseller/gorsel8.png)

Bağlantının "db_dataentry" kimliğiyle açıldığı sol panelden doğrulanmıştır.

![db_dataentry2](gorseller/gorsel9.png)


Yazma testi (INSERT): Customers tablosuna yeni bir kayıt eklenmiş ve işlem başarıyla tamamlanmıştır. "1 satır etkilendi" mesajı alınmıştır.

``` sql
USE Northwind;
INSERT INTO Customers VALUES ('TEST', 'Test Firma', 'Mr', 'Test Co', NULL, NULL, NULL, NULL, NULL, NULL, NULL);
```

![db_dataentry3](gorseller/gorsel10.png)


Doğrulama (SELECT): Eklenen TEST kaydı sorgulanmış ve tabloda başarıyla görüntülenmiştir.

``` sql
USE Northwind;
SELECT * FROM Customers WHERE CustomerID = 'TEST';
``` 

![db_dataentry4](gorseller/gorsel11.png)


Bu sonuç db_dataentry kullanıcısının hem okuma hem de yazma yetkisine sahip olduğunu doğrulamaktadır. db_readonly kullanıcısının aynı işlemde hata aldığı düşünüldüğünde, rol tabanlı yetkilendirmenin doğru çalıştığı açıkça görülmektedir.


### 2.5 Yetki Testi — db_admin

db_admin kullanıcısı ile bağlanılmıştır. 

![db_admin](gorseller/gorsel12.png)

![db_admin](gorseller/gorsel13.png)


DDL testi (CREATE TABLE & DROP TABLE):


``` sql
USE Northwind;
CREATE TABLE AdminTest (ID INT, Aciklama NVARCHAR(50));
DROP TABLE AdminTest;

``` 

İşlem başarıyla tamamlanmıştır. 

![db_admin](gorseller/gorsel14.png)

db_admin kullanıcısı db_owner rolüne sahip olduğundan tablo oluşturma ve silme gibi DDL işlemlerini gerçekleştirebilmektedir. Bu işlem db_readonly ve db_dataentry kullanıcıları tarafından yapılamamaktadır.

| İşlem | db_readonly | db_dataentry | db_admin |
|-------|-------------|--------------|----------|
| SELECT | İzinli | İzinli | İzinli |
| INSERT | Reddedildi | İzinli | İzinli |
| CREATE TABLE | Reddedildi | Reddedildi | İzinli |


## 3. Veri Şifreleme (TDE - Transparent Data Encryption)

TDE veritabanı dosyalarını disk üzerinde şifreleyerek fiziksel erişim durumunda veri güvenliğini sağlar. Şifreleme uygulama katmanında değil, doğrudan veritabanı motoru tarafından gerçekleştirilir.

Uygulama dört adımda gerçekleştirilmiştir:

Adım 1 - Master Key oluşturuldu:

```sql
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKey123!';
```

Adım 2 - Sertifika oluşturuldu:

```sql
CREATE CERTIFICATE NorthwindTDECert
WITH SUBJECT = 'Northwind TDE Sertifikası';
```

Adım 3 - Şifreleme anahtarı oluşturuldu ve şifreleme etkinleştirildi:

```sql
USE Northwind;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE NorthwindTDECert;

ALTER DATABASE Northwind
SET ENCRYPTION ON;
```

İşlem sırasında "The certificate used for encrypting the database encryption key has not been backed up" uyarısı alınmıştır. Bu bir hata değil, sertifikanın yedeklenmesi gerektiğini hatırlatan bir uyarıdır.

![TDE1](gorseller/gorsel15.png)

Adım 4 - Şifreleme doğrulandı:

``` sql
SELECT name, is_encrypted 
FROM sys.databases 
WHERE name = 'Northwind';
```

Sorgu çıktısında is_encrypted = 1 değeri, Northwind veritabanının başarıyla şifrelendiğini doğrulamaktadır.

![TDE2](gorseller/gorsel16.png)


## 4. SQL Injection Testleri

SQL Injection, kullanıcı girdisinin SQL sorgusuna doğrudan eklenmesiyle oluşan ve saldırganların yetkisiz veri erişimi sağlamasına olanak tanıyan yaygın bir güvenlik açığıdır.

### 4.1 Savunmasız Senaryo (Unsafe)

Dinamik SQL kullanan savunmasız bir stored procedure oluşturulmuştur. Bu procedure, kullanıcıdan gelen girdiyi doğrudan sorguya yapıştırmaktadır:
```sql
USE Northwind;
GO

CREATE PROCEDURE sp_GetCustomer_Unsafe
    @CustomerID NVARCHAR(50)
AS
BEGIN
    EXEC('SELECT * FROM Customers WHERE CustomerID = ''' + @CustomerID + '''')
END;
```

Aşağıdaki injection saldırısı bu procedure üzerinde test edilmiştir:

```sql
EXEC sp_GetCustomer_Unsafe @CustomerID = 'ALFKI'' OR ''1''=''1';
Girdi sorguya yapıştırıldığında oluşan SQL şu şekildedir:
```
```sql
SELECT * FROM Customers WHERE CustomerID = 'ALFKI' OR '1'='1'
```

'1'='1' ifadesi her zaman TRUE döndürdüğünden WHERE koşulu tüm satırlar için geçerli olmuş ve 92 satır yani tüm Customers tablosu döndürülmüştür. Yalnızca ALFKI kaydının gelmesi gerekirken saldırgan tüm müşteri verisine erişmiştir. 

![SQL1](gorseller/gorsel17.png)



### 4.2 Güvenli Senaryo (Safe)

SQL Injection'a karşı koruma sağlamak için parametreli sorgu kullanan güvenli bir stored procedure oluşturulmuştur:
```sql
USE Northwind;
GO

CREATE PROCEDURE sp_GetCustomer_Safe
    @CustomerID NVARCHAR(50)
AS
BEGIN
    SELECT * FROM Customers 
    WHERE CustomerID = @CustomerID;
END;
```

Aynı injection saldırısı güvenli procedure üzerinde tekrarlanmıştır:

```sql
EXEC sp_GetCustomer_Safe @CustomerID = 'ALFKI'' OR ''1''=''1';
```

Sonuç: Normal kullanımda yalnızca ALFKI kaydı gelmiş (1 satır), injection denemesinde ise 0 satır döndürülmüştür. Parametreli sorgular kullanıcı girdisini SQL kodu olarak değil veri olarak işlediğinden saldırı etkisiz kalmıştır. 

![SQL2](gorseller/gorsel18.png)

### 4.3 Kalıcı Güvenlik Önlemleri

Test sonrasında aşağıdaki kalıcı önlemler uygulanmıştır:
```sql
USE Northwind;

-- Tehlikeli procedure kaldırıldı
DROP PROCEDURE sp_GetCustomer_Unsafe;

-- Kullanıcıların tabloya direkt erişimi kapatıldı
DENY SELECT ON Customers TO db_readonly;
DENY SELECT ON Customers TO db_dataentry;

-- Sadece güvenli procedure üzerinden erişime izin verildi
GRANT EXECUTE ON sp_GetCustomer_Safe TO db_readonly;
GRANT EXECUTE ON sp_GetCustomer_Safe TO db_dataentry;
```

Bu sayede kullanıcılar Customers tablosuna doğrudan erişemez, yalnızca güvenli procedure üzerinden veri sorgulayabilir. 

## 5. Audit Logları

Kullanıcı aktivitelerini izlemek için SQL Server Audit kullanılmıştır.

Adım 1 - Server Audit oluşturuldu:

``` sql
USE master;
GO
CREATE SERVER AUDIT NorthwindAudit
TO FILE (FILEPATH = 'C:\NorthwindBackups\')
WITH (ON_FAILURE = CONTINUE);
GO
ALTER SERVER AUDIT NorthwindAudit WITH (STATE = ON);
GO
```

Adım 2 — Database Audit Specification oluşturuldu:

```sql
USE Northwind;
GO
CREATE DATABASE AUDIT SPECIFICATION NorthwindAuditSpec
FOR SERVER AUDIT NorthwindAudit
ADD (SELECT ON dbo.Customers BY db_readonly),
ADD (INSERT ON dbo.Customers BY db_dataentry),
ADD (DELETE ON dbo.Customers BY db_dataentry)
WITH (STATE = ON);
GO
```

Adım 3 — Test ve Doğrulama:

db_readonly kullanıcısıyla Customers tablosuna SELECT yapılmaya çalışılmış ancak daha önce uygulanan DENY kısıtlaması nedeniyle reddedilmiştir. 

![LOG1](gorseller/gorsel19.png)

![LOG2](gorseller/gorsel20.png)


Audit logları aşağıdaki sorgu ile sorgulanmıştır:

``` sql
SELECT 
    event_time,
    action_id,
    server_principal_name,
    object_name,
    succeeded
FROM sys.fn_get_audit_file('C:\NorthwindBackups\*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;
```

Log çıktısında db_readonly kullanıcısının SELECT girişimi (SL) ve audit specification'ın oluşturulması (AUSC) kayıt altına alınmıştır. Bu sonuç, kullanıcı aktivitelerinin başarılı ve başarısız işlemler dahil olmak üzere eksiksiz loglandığını doğrulamaktadır. 

![LOG3](gorseller/gorsel21.png)

## 6. Güvenlik Duvarı Yönetimi

Veritabanı güvenliğinin sadece SQL Server içindeki önlemlerle sınırlı kalmaması gerekir. Dışarıdan gelen yetkisiz bağlantı girişimleri de engellenmelidir. SQL Server varsayılan olarak 1433 numaralı TCP portunu kullanır. Bu port dışarıya açık bırakılırsa saldırganlar doğrudan SQL Server'a bağlanmaya çalışabilir, kaba kuvvet saldırısı yapabilir veya açıkları istismar edebilir.

Bu riski azaltmak için Windows Defender Güvenlik Duvarı üzerinde 1433 portuna yönelik kısıtlı erişim kuralı oluşturulmuştur.
Yapılan yapılandırma:

Kural türü: Bağlantı noktası (Port) - SQL Server'ın dinlediği porta özel kural
Protokol: TCP - SQL Server TCP üzerinden iletişim kurar
Port: 1433
Eylem: Bağlantıya izin ver — ama yalnızca belirtilen koşullarda
Profil: Etki alanı ve Özel seçildi, Ortak kaldırıldı — kafeterya, havalimanı gibi ortak ağlardan SQL Server'a erişim tamamen engellendi
Uzak IP kısıtlaması: Yalnızca 127.0.0.1 (localhost) eklendi — sadece aynı bilgisayardan bağlantıya izin verildi, dışarıdan gelen tüm bağlantılar reddedildi.

Bu sayede SQL Server yalnızca yerel ortamdan erişilebilir hale getirilmiş, dış ağlardan gelen yetkisiz bağlantı girişimleri güvenlik duvarı seviyesinde engellenmiştir.


Windows Defender Güvenlik Duvarı üzerinde SQL Server için port tabanlı bir inbound kuralı oluşturulmuştur. Bu kural ile belirli bir port üzerinden gelen bağlantılar kontrol altına alınmıştır.

![FIREWALL1](gorseller/gorsel22.png)

SQL Server’ın varsayılan olarak kullandığı TCP 1433 portu için inbound kuralı tanımlanmıştır. Bu sayede yalnızca belirlenen port üzerinden gelen bağlantılar kontrol altına alınmıştır.

![FIREWALL2](gorseller/gorsel23.png)

Oluşturulan kural kapsamında belirtilen port üzerinden gelen bağlantılara izin verilmiştir. Bu izin, yalnızca tanımlanan diğer güvenlik koşulları (profil ve IP kısıtlaması) ile birlikte geçerli olacak şekilde yapılandırılmıştır.

![FIREWALL3](gorseller/gorsel24.png)

Güvenlik kuralı yalnızca Domain ve Private ağ profilleri için aktif edilmiştir. Public (ortak) ağ profili devre dışı bırakılarak, kafe, havaalanı gibi güvensiz ortamlardan SQL Server’a erişim engellenmiştir.

![FIREWALL4](gorseller/gorsel25.png)

![FIREWALL5](gorseller/gorsel26.png)

Güvenlik duvarı kuralı kapsamında uzak IP adresi kısıtlaması uygulanmıştır. Yalnızca 127.0.0.1 (localhost) adresinden gelen bağlantılara izin verilmiş, dış ağlardan gelen tüm bağlantılar engellenmiştir. Bu sayede SQL Server yalnızca yerel erişime açık hale getirilmiştir.

![FIREWALL6](gorseller/gorsel27.png)

Oluşturulan “SQL Server 1433 - Kısıtlı Erişim” kuralı Windows Defender Güvenlik Duvarı üzerinde aktif olarak görüntülenmiştir. Bu durum yapılandırmanın başarıyla uygulandığını göstermektedir.

![FIREWALL7](gorseller/gorsel28.png)

SQL Server’ın TCP/IP protokolü aktif hale getirilmiştir. Bu sayede veritabanı sunucusunun TCP üzerinden bağlantı kabul edebilmesi sağlanmıştır.

![FIREWALL8](gorseller/gorsel29.png)

SQL Server yapılandırması incelenmiş ve TCP portunun 1433 olarak ayarlandığı doğrulanmıştır. Ayrıca dinamik port kullanımı devre dışı bırakılarak sabit port üzerinden bağlantı sağlanmıştır.

![FIREWALL9](gorseller/gorsel30.png)

SQL Server varsayılan olarak 1433 numaralı TCP portu üzerinden çalışmaktadır. Bu nedenle güvenlik duvarı kuralları bu port üzerinden yapılandırılmıştır.

Yapılandırmanın doğru çalıştığını doğrulamak amacıyla bağlantı kurulurken sunucu adresinde 1433 portu açıkça belirtilmiştir. Bu sayede istemci ile sunucu arasındaki iletişimin tanımlanan port üzerinden gerçekleştiği doğrulanmış ve firewall kuralının etkili olduğu gösterilmiştir.

![FIREWALL10](gorseller/gorsel31.png)


SQL Server bağlantısının hangi port ve protokol üzerinden gerçekleştiğini doğrulamak amacıyla aşağıdaki sorgu çalıştırılmıştır:

``` sql
SELECT DISTINCT 
    local_tcp_port,
    net_transport
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;

```

Sorgu sonucunda bağlantının 1433 portu üzerinden ve TCP protokolü kullanılarak gerçekleştirildiği görülmüştür.

Bu sonuç, SQL Server’ın varsayılan portu olan 1433 üzerinden çalıştığını ve oluşturulan firewall kuralının doğru port üzerinde etkili olduğunu doğrulamaktadır. Böylece istemci ile sunucu arasındaki iletişimin güvenlik duvarı tarafından kontrol edilen port üzerinden gerçekleştiği kanıtlanmıştır.

![FIREWALL11](gorseller/gorsel32.png)

``` sql

BACKUP CERTIFICATE NorthwindTDECert
TO FILE = 'C:\NorthwindBackups\NorthwindTDECert.cer'
WITH PRIVATE KEY (
    FILE = 'C:\NorthwindBackups\NorthwindTDECert.pvk',
    ENCRYPTION BY PASSWORD = 'CertBackup123!'
);
``` sql
