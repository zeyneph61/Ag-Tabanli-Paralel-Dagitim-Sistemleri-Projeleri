# 3. Veri Tabanı Güvenliği ve Erişim Kontrolü

Ağ Tabanlı Paralel Dağıtım Sistemleri dersi için yapılan Veri Tabanı Güvenliği ve Erişim Kontrolü projesi.

# BLM 4522 PROJE RAPORU

Zeynep Hacısalihoğlu

22290449

## İçindekiler

1. [Giriş](#1-giriş)
   - [1.1 Kullanılan Ortam](#11-kullanılan-ortam)
   - [1.2 Projenin Amacı](#12-projenin-amacı)
   - [1.3 Proje Planı](#13-proje-planı)
   - [1.4 Mevcut Durum](#14-mevcut-durum)
   - [1.5 Authentication Modu](#15-authentication-modu)
2. [Erişim Yönetimi](#2-erişim-yönetimi)
   - [2.1 Login Oluşturma](#21-login-oluşturma)
   - [2.2 Kullanıcı ve Rol Atama](#22-kullanıcı-ve-rol-atama)
   - [2.3 Yetki Testi - db_readonly](#23-yetki-testi---db_readonly)
   - [2.4 Yetki Testi — db_dataentry](#24-yetki-testi--db_dataentry)
   - [2.5 Yetki Testi — db_admin](#25-yetki-testi--db_admin)
3. [Veri Şifreleme (TDE)](#3-veri-şifreleme-tde---transparent-data-encryption)
4. [SQL Injection Testleri](#4-sql-injection-testleri)
   - [4.1 Savunmasız Senaryo (Unsafe)](#41-savunmasız-senaryo-unsafe)
   - [4.2 Güvenli Senaryo (Safe)](#42-güvenli-senaryo-safe)
   - [4.3 Kalıcı Güvenlik Önlemleri](#43-kalıcı-güvenlik-önlemleri)
5. [Audit Logları](#5-audit-logları)
6. [Güvenlik Duvarı Yönetimi](#6-güvenlik-duvarı-yönetimi)
   - [6.1 Neden Güvenlik Duvarı?](#61-neden-güvenlik-duvarı)
   - [6.2 Güvenlik Duvarı Kuralının Oluşturulması](#62-güvenlik-duvarı-kuralının-oluşturulması)
   - [6.3 Yapılandırmanın Doğrulanması](#63-yapılandırmanın-doğrulanması)
7. [Sonuç](#7-sonuç)


# 1. Giriş

Bu proje Microsoft SQL Server 2022 üzerinde çalışan Northwind veritabanı üzerinde kapsamlı bir güvenlik altyapısının tasarlanması ve uygulanmasını konu almaktadır.


Proje kapsamında dört temel güvenlik konusu ele alınmıştır: kullanıcı erişim yönetimi, veri şifreleme (TDE), SQL Injection saldırılarına karşı koruma ve audit logları ile kullanıcı aktivitelerinin izlenmesi.

## 1.1 Kullanılan Ortam

Veritabanı Sistemi: Microsoft SQL Server 2022 Developer Edition, Sürüm 16.0.1000.6
Yönetim Aracı: SQL Server Management Studio (SSMS)
Örnek Veritabanı: Northwind

## 1.2 Projenin Amacı

Bu projenin amacı, Microsoft SQL Server üzerinde gerçek bir veritabanı güvenlik altyapısı tasarlamak ve uygulamalı olarak test etmektir. Proje kapsamında ele alınan güvenlik konuları — erişim yönetimi, veri şifreleme, SQL Injection koruması, audit logları ve güvenlik duvarı yönetimi — modern veritabanı sistemlerinde karşılaşılan başlıca güvenlik tehditlerini yansıtmaktadır. Her konu teorik anlatımla sınırlı kalmayıp SQL sorguları ve sistem yapılandırmaları ile uygulanmış, sonuçlar ekran çıktıları ile doğrulanmıştır.

## 1.3 Proje Planı

Proje aşağıdaki sırayla planlanmış ve uygulanmıştır:

**Erişim Yönetimi:** Farklı yetki seviyelerinde kullanıcılar oluşturulmuş, rol tabanlı erişim kontrolü yapılandırılmış ve yetkiler test edilmiştir.

**Veri Şifreleme (TDE):** Veritabanı dosyaları Transparent Data Encryption ile disk üzerinde şifrelenmiştir.

**SQL Injection Testleri:** Savunmasız ve güvenli stored procedure'lar karşılaştırılmış, kalıcı koruma önlemleri uygulanmıştır.

**Audit Logları:** Kullanıcı aktiviteleri SQL Server Audit ile izlenmiş ve loglar sorgulanmıştır.

**Güvenlik Duvarı Yönetimi:** SQL Server'ın dinlediği TCP 1433 portu güvenlik duvarı kurallarıyla kısıtlanmıştır.

## 1.4 Mevcut Durum

Projeye başlamadan önce sistemdeki mevcut login'ler sorgulanmıştır:

```sql
SELECT name, type_desc, is_disabled
FROM sys.server_principals
WHERE type IN ('S', 'U')
ORDER BY name;
```

Sorgu çıktısında 2 SQL Login ve 7 Windows Login'in tanımlı olduğu görülmüştür.

![Mevcut Login](gorseller/gorsel1.png)

## 1.5 Authentication Modu

SQL Server'ın hangi kimlik doğrulama modunda çalıştığı aşağıdaki sorgu ile kontrol edilmiştir:

```sql
EXEC xp_loginconfig 'login mode';
```

Sonuç: Login mode Mixed olarak yapılandırılmıştır. Bu mod hem SQL Server Authentication hem de Windows Authentication kullanımına izin vermektedir.

![Login Mode](gorseller/gorsel2.png)


# 2. Erişim Yönetimi

## 2.1 Login Oluşturma

Üç farklı yetki seviyesini temsil eden kullanıcı oluşturulmuştur: `db_admin` tam yetkili yönetici, `db_readonly` yalnızca okuma yapabilen kullanıcı, `db_dataentry` ise veri okuma ve ekleme yetkisine sahip kullanıcıdır.

```sql
USE master;
CREATE LOGIN db_admin WITH PASSWORD = 'Admin123!';
CREATE LOGIN db_readonly WITH PASSWORD = 'Readonly123!';
CREATE LOGIN db_dataentry WITH PASSWORD = 'Dataentry123!';
```

## 2.2 Kullanıcı ve Rol Atama

Login'ler Northwind veritabanına kullanıcı olarak eklenmiş ve uygun roller atanmıştır:

```sql
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

```sql
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

Sorgu çıktısı, her kullanıcının yalnızca kendisine atanan rolle eşleştiğini doğrulamaktadır. `db_admin` → `db_owner`, `db_dataentry` → `db_datareader` ve `db_datawriter`, `db_readonly` → `db_datareader`.

![Rol](gorseller/gorsel3.png)

## 2.3 Yetki Testi - db_readonly

Yetki testleri için SSMS'de yeni bir bağlantı açılmış ve SQL Server Authentication seçilmiştir.

![db_readonly](gorseller/gorsel4.png)

db_readonly kullanıcısı ile DESKTOP-DLHAT6J\DB sunucusuna başarıyla bağlanılmıştır. Sol panelde bağlantının "db_readonly" kimliğiyle açıldığı görülmektedir.

![db_readonly1](gorseller/gorsel5.png)

Okuma testi (SELECT): Customers tablosundan veri sorgulanmış ve 5 kayıt başarıyla listelenmiştir. Bu, db_readonly kullanıcısının okuma yetkisine sahip olduğunu doğrulamaktadır.

```sql
USE Northwind;
SELECT TOP 5 * FROM Customers;
```

![db_readonly2](gorseller/gorsel6.png)

Yazma testi (INSERT): Customers tablosuna yeni kayıt ekleme denenmiş, ancak "The INSERT permission was denied on the object 'Customers'" hatası alınmıştır. Bu sonuç, db_readonly kullanıcısının yazma yetkisinin olmadığını doğrulamaktadır.

```sql
USE Northwind;
INSERT INTO Customers VALUES ('TEST', 'Test Firma', 'Mr', 'Test Co', NULL, NULL, NULL, NULL, NULL, NULL, NULL);
```

![db_readonly3](gorseller/gorsel7.png)

## 2.4 Yetki Testi — db_dataentry

db_dataentry kullanıcısı ile SQL Server Authentication kullanılarak bağlanılmıştır.

![db_dataentry1](gorseller/gorsel8.png)

Bağlantının "db_dataentry" kimliğiyle açıldığı sol panelden doğrulanmıştır.

![db_dataentry2](gorseller/gorsel9.png)

Yazma testi (INSERT): Customers tablosuna yeni bir kayıt eklenmiş ve işlem başarıyla tamamlanmıştır. "1 satır etkilendi" mesajı alınmıştır.

```sql
USE Northwind;
INSERT INTO Customers VALUES ('TEST', 'Test Firma', 'Mr', 'Test Co', NULL, NULL, NULL, NULL, NULL, NULL, NULL);
```

![db_dataentry3](gorseller/gorsel10.png)

Doğrulama (SELECT): Eklenen TEST kaydı sorgulanmış ve tabloda başarıyla görüntülenmiştir.

```sql
USE Northwind;
SELECT * FROM Customers WHERE CustomerID = 'TEST';
```

![db_dataentry4](gorseller/gorsel11.png)

Bu sonuç db_dataentry kullanıcısının hem okuma hem de yazma yetkisine sahip olduğunu doğrulamaktadır. db_readonly kullanıcısının aynı işlemde hata aldığı düşünüldüğünde, rol tabanlı yetkilendirmenin doğru çalıştığı açıkça görülmektedir.

## 2.5 Yetki Testi — db_admin

db_admin kullanıcısı ile bağlanılmıştır.

![db_admin1](gorseller/gorsel12.png)

![db_admin2](gorseller/gorsel13.png)

DDL testi (CREATE TABLE & DROP TABLE):

```sql
USE Northwind;
CREATE TABLE AdminTest (ID INT, Aciklama NVARCHAR(50));
DROP TABLE AdminTest;
```

İşlem başarıyla tamamlanmıştır.

![db_admin3](gorseller/gorsel14.png)

db_admin kullanıcısı db_owner rolüne sahip olduğundan tablo oluşturma ve silme gibi DDL işlemlerini gerçekleştirebilmektedir. Bu işlem db_readonly ve db_dataentry kullanıcıları tarafından yapılamamaktadır.

| İşlem | db_readonly | db_dataentry | db_admin |
|-------|-------------|--------------|----------|
| SELECT | İzinli | İzinli | İzinli |
| INSERT | Reddedildi | İzinli | İzinli |
| CREATE TABLE | Reddedildi | Reddedildi | İzinli |


# 3. Veri Şifreleme (TDE - Transparent Data Encryption)

TDE veritabanı dosyalarını disk üzerinde şifreleyerek fiziksel erişim durumunda veri güvenliğini sağlar. Şifreleme uygulama katmanında değil, doğrudan veritabanı motoru tarafından gerçekleştirilir.

TDE etkinleştirildiğinde veritabanı dosyaları (.mdf, .ldf) ve yedek dosyalar (.bak) otomatik olarak şifrelenir. Şifreleme ve çözme işlemi arka planda gerçekleştiği için uygulama tarafında herhangi bir değişiklik gerekmez.

Uygulama dört adımda gerçekleştirilmiştir:

**Adım 1 - Master Key oluşturuldu:**

Master Key, sunucu düzeyinde şifreleme hiyerarşisinin temelidir. Tüm sertifikalar ve anahtarlar bu key ile korunur.

```sql
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKey123!';
```

**Adım 2 - Sertifika oluşturuldu:**

Sertifika, veritabanı şifreleme anahtarını korumak için kullanılır. Sertifika kaybolursa şifreli veritabanına bir daha erişilemez — bu nedenle yedeklenmesi kritik önem taşır.

```sql
CREATE CERTIFICATE NorthwindTDECert
WITH SUBJECT = 'Northwind TDE Sertifikası';
```

**Adım 3 - Şifreleme anahtarı oluşturuldu ve şifreleme etkinleştirildi:**

```sql
USE Northwind;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE NorthwindTDECert;

ALTER DATABASE Northwind
SET ENCRYPTION ON;
```

İşlem sırasında "The certificate used for encrypting the database encryption key has not been backed up" uyarısı alınmıştır. Gerçek bir üretim ortamında sertifika mutlaka yedeklenmelidir. Yedekleme şu komutla yapılır:

```sql
BACKUP CERTIFICATE NorthwindTDECert
TO FILE = 'C:\NorthwindBackups\NorthwindTDECert.cer'
WITH PRIVATE KEY (
    FILE = 'C:\NorthwindBackups\NorthwindTDECert.pvk',
    ENCRYPTION BY PASSWORD = 'CertBackup123!'
);
```

![TDE1](gorseller/gorsel15.png)

**Adım 4 - Şifreleme doğrulandı:**

```sql
SELECT name, is_encrypted
FROM sys.databases
WHERE name = 'Northwind';
```

Sorgu çıktısında is_encrypted = 1 değeri, Northwind veritabanının başarıyla şifrelendiğini doğrulamaktadır.

![TDE2](gorseller/gorsel16.png)


# 4. SQL Injection Testleri

SQL Injection, kullanıcı girdisinin SQL sorgusuna doğrudan eklenmesiyle oluşan ve saldırganların yetkisiz veri erişimi sağlamasına olanak tanıyan yaygın bir güvenlik açığıdır.

Bu bölümde önce saldırıya açık bir senaryo oluşturulmuş ve injection'ın nasıl çalıştığı gözlemlenmiştir. Ardından parametreli sorgu kullanan güvenli bir yapı ile karşılaştırma yapılmış ve kalıcı önlemler uygulanmıştır.

## 4.1 Savunmasız Senaryo (Unsafe)

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
```

Girdi sorguya yapıştırıldığında oluşan SQL şu şekildedir:

```sql
SELECT * FROM Customers WHERE CustomerID = 'ALFKI' OR '1'='1'
```

'1'='1' ifadesi her zaman TRUE döndürdüğünden WHERE koşulu tüm satırlar için geçerli olmuş ve 92 satır yani tüm Customers tablosu döndürülmüştür. Yalnızca ALFKI kaydının gelmesi gerekirken saldırgan tüm müşteri verisine erişmiştir.

Bu sonuç kritik bir güvenlik açığını ortaya koymaktadır: saldırgan yalnızca tek bir müşteriye erişmesi gerekirken tüm müşteri tablosunu elde etmiştir. Gerçek bir sistemde bu tablo; adres, telefon ve iletişim bilgileri gibi kişisel verileri içerebilir.

![SQL1](gorseller/gorsel17.png)

## 4.2 Güvenli Senaryo (Safe)

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

Parametreli sorgularda kullanıcı girdisi SQL motoru tarafından her zaman veri olarak işlenir, kod olarak değil. Bu nedenle `' OR '1'='1` ifadesi bir CustomerID değeri olarak aranmış ve eşleşme bulunamamıştır.

## 4.3 Kalıcı Güvenlik Önlemleri

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

Bu yapılandırma sonucunda kullanıcılar Customers tablosuna artık doğrudan erişememektedir. Tüm sorgular yalnızca `sp_GetCustomer_Safe` üzerinden yapılabilmekte, bu da injection saldırılarına karşı kalıcı bir koruma katmanı oluşturmaktadır. Bir sonraki bölümde bu erişim girişimleri audit loglarıyla izlenecektir.


# 5. Audit Logları

Erişim yönetimi ve SQL Injection koruması tek başına yeterli değildir. Sistemde kimin ne zaman ne yaptığının da kayıt altına alınması gerekir. Audit logları; yetkisiz erişim girişimlerini, başarısız sorguları ve şüpheli aktiviteleri tespit etmek için kullanılır.

Kullanıcı aktivitelerini izlemek için SQL Server Audit kullanılmıştır.

**Adım 1 - Server Audit oluşturuldu:**

```sql
USE master;
GO
CREATE SERVER AUDIT NorthwindAudit
TO FILE (FILEPATH = 'C:\NorthwindBackups\')
WITH (ON_FAILURE = CONTINUE);
GO
ALTER SERVER AUDIT NorthwindAudit WITH (STATE = ON);
GO
```

`ON_FAILURE = CONTINUE` ayarı, log dosyasına yazılamadığı durumlarda veritabanının çalışmaya devam etmesini sağlar. Kritik sistemlerde bu değer `SHUTDOWN` olarak ayarlanabilir; bu durumda loglama başarısız olursa sunucu kendini kapatır.

**Adım 2 - Database Audit Specification oluşturuldu:**

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

Audit specification yalnızca belirli kullanıcıların belirli tablolar üzerindeki işlemlerini izlemektedir. Bu sayede log dosyası gereksiz verilerle şişmez, yalnızca güvenlik açısından kritik olaylar kaydedilir.

**Adım 3 - Test ve Doğrulama:**

4. bölümde uygulanan `DENY SELECT` kısıtlaması nedeniyle `db_readonly` kullanıcısının doğrudan tablo erişimi engellenmiştir. Bu girişim reddedilmiş olsa da audit log tarafından kayıt altına alınmıştır.

![LOG1](gorseller/gorsel19.png)

![LOG2](gorseller/gorsel20.png)

Audit logları aşağıdaki sorgu ile sorgulanmıştır:

```sql
SELECT
    event_time,
    action_id,
    server_principal_name,
    object_name,
    succeeded
FROM sys.fn_get_audit_file('C:\NorthwindBackups\*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;
```

Log çıktısında db_readonly kullanıcısının SELECT girişimi **(SL)** ve audit specification'ın oluşturulması **(AUSC)** kayıt altına alınmıştır. Bu sonuç kullanıcı aktivitelerinin başarılı ve başarısız işlemler dahil olmak üzere eksiksiz loglandığını doğrulamaktadır. 

![LOG3](gorseller/gorsel21.png)

Log çıktısında iki kayıt görülmektedir: db_readonly kullanıcısının SELECT girişimi (SL) ve audit specification'ın oluşturulması (AUSC). SL kullanıcı aktivitesini, AUSC ise yapılandırma değişikliğini temsil etmektedir.

# 6. Güvenlik Duvarı Yönetimi

## 6.1 Neden Güvenlik Duvarı?

Veritabanı güvenliğinin sadece SQL Server içindeki önlemlerle sınırlı kalmaması gerekir. Dışarıdan gelen yetkisiz bağlantı girişimleri de engellenmelidir. SQL Server varsayılan olarak 1433 numaralı TCP portunu kullanır.

Bu port dışarıya açık bırakılırsa saldırganlar doğrudan SQL Server'a bağlanmaya çalışabilir, kaba kuvvet saldırısı yapabilir veya açıkları istismar edebilir.

Bu riski azaltmak için Windows Defender Güvenlik Duvarı üzerinde 1433 portuna yönelik kısıtlı erişim kuralı oluşturulmuştur.

## 6.2 Güvenlik Duvarı Kuralının Oluşturulması

Aşağıdaki yapılandırma adımları uygulanmıştır:

- Kural türü: Bağlantı noktası (Port)
- Protokol: TCP
- Port: 1433
- Eylem: Yalnızca belirtilen koşullarda izin ver
- Profil: Domain ve Private — Public devre dışı
- Uzak IP: Yalnızca 127.0.0.1 (localhost)

Windows Defender Güvenlik Duvarı üzerinde SQL Server için port tabanlı bir inbound kuralı oluşturulmuştur. Bu kural ile belirli bir port üzerinden gelen bağlantılar kontrol altına alınmıştır.

![FIREWALL1](gorseller/gorsel22.png)

SQL Server'ın varsayılan olarak kullandığı TCP 1433 portu için inbound kuralı tanımlanmıştır. Bu sayede yalnızca belirlenen port üzerinden gelen bağlantılar kontrol altına alınmıştır.

![FIREWALL2](gorseller/gorsel23.png)

Kural eylemi 'Bağlantıya izin ver' olarak seçilmiştir. Bu izin yalnızca tanımlanan profil ve IP koşullarıyla birlikte geçerlidir.

![FIREWALL3](gorseller/gorsel24.png)

Güvenlik kuralı yalnızca Domain ve Private ağ profilleri için aktif edilmiştir. Public (ortak) ağ profili devre dışı bırakılarak, kafe, havaalanı gibi güvensiz ortamlardan SQL Server'a erişim engellenmiştir.

![FIREWALL4](gorseller/gorsel25.png)

![FIREWALL5](gorseller/gorsel26.png)

Güvenlik duvarı kuralı kapsamında uzak IP adresi kısıtlaması uygulanmıştır. Yalnızca 127.0.0.1 (localhost) adresinden gelen bağlantılara izin verilmiş, dış ağlardan gelen tüm bağlantılar engellenmiştir. Bu sayede SQL Server yalnızca yerel erişime açık hale getirilmiştir.

![FIREWALL6](gorseller/gorsel27.png)

Oluşturulan "SQL Server 1433 - Kısıtlı Erişim" kuralı Windows Defender Güvenlik Duvarı üzerinde aktif olarak görüntülenmiştir. Bu durum yapılandırmanın başarıyla uygulandığını göstermektedir.

![FIREWALL7](gorseller/gorsel28.png)

SQL Server'ın TCP/IP protokolü aktif hale getirilmiştir. Bu sayede veritabanı sunucusunun TCP üzerinden bağlantı kabul edebilmesi sağlanmıştır.

![FIREWALL8](gorseller/gorsel29.png)

SQL Server yapılandırması incelenmiş ve TCP portunun 1433 olarak ayarlandığı doğrulanmıştır. Ayrıca dinamik port kullanımı devre dışı bırakılarak sabit port üzerinden bağlantı sağlanmıştır.

![FIREWALL9](gorseller/gorsel30.png)

## 6.3 Yapılandırmanın Doğrulanması

SQL Server varsayılan olarak 1433 numaralı TCP portu üzerinden çalışmaktadır. Bu nedenle güvenlik duvarı kuralları bu port üzerinden yapılandırılmıştır.

Yapılandırmanın doğru çalıştığını doğrulamak amacıyla bağlantı kurulurken sunucu adresinde 1433 portu açıkça belirtilmiştir.

![FIREWALL10](gorseller/gorsel31.png)

```sql
SELECT DISTINCT
    local_tcp_port,
    net_transport
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

Sorgu sonucu bağlantının TCP protokolü üzerinden 1433 portundan gerçekleştiğini doğrulamaktadır. Bu, oluşturulan güvenlik duvarı kuralının doğru port üzerinde etkili olduğunu kanıtlamaktadır.

![FIREWALL11](gorseller/gorsel32.png)


# 7. Sonuç

Bu projede Microsoft SQL Server 2022 üzerinde kapsamlı bir veritabanı güvenlik altyapısı tasarlanmış ve uygulamalı olarak test edilmiştir.

Erişim yönetimi kapsamında rol tabanlı yetkilendirme yapılandırılmış, farklı kullanıcıların yalnızca kendilerine tanımlanan işlemleri gerçekleştirebildiği doğrulanmıştır. TDE ile veritabanı dosyaları disk üzerinde şifrelenmiş, fiziksel erişim durumunda veri güvenliği sağlanmıştır. SQL Injection testlerinde dinamik sorguların ciddi güvenlik açıkları oluşturduğu gözlemlenmiş, parametreli sorgularla bu açık giderilmiştir. Audit logları ile kullanıcı aktiviteleri başarılı ve başarısız işlemler dahil eksiksiz kayıt altına alınmıştır. Güvenlik duvarı yapılandırmasıyla SQL Server yalnızca yerel ağdan erişilebilir hale getirilmiş, dış bağlantılar engellenmiştir.

Proje boyunca her güvenlik katmanının birbirini tamamladığı görülmüştür. Yalnızca tek bir önlem almak yerine erişim kontrolü, şifreleme, uygulama güvenliği, izleme ve ağ güvenliğinin bir arada uygulanması sağlam bir güvenlik altyapısı oluşturmaktadır.