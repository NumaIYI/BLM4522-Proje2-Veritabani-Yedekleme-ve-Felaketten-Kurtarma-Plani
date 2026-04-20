# Youtube Link: https://youtu.be/HdiZt1p1IPo



# ÖZET
Kritik sektörlerdeki verilerin erişilebilirliğini, bütünlüğünü ve gizliliğini
sağlamak, modern Veritabanı Yöneticiliği (DBA) disiplininin temelidir. Olası
siber saldırılar, donanım arızaları veya insan kaynaklı hatalar kurumlara geri
dönülemez zararlar verebilmektedir. Bu projenin amacı; Microsoft SQL
Server altyapısı üzerinde, uluslararası iş sürekliliği ve siber güvenlik
standartlarına (ISO 27001) uygun, proaktif bir veritabanı yönetim ve koruma
mimarisi tasarlamaktır.
Proje, Kaggle üzerinden temin edilen ve hassas sağlık verileri barındıran bir
veri setinin sisteme entegre edilmesiyle başlamıştır. İlk fazda, iş sürekliliğini
sağlamak adına kapsamlı bir Felaketten Kurtarma planı kurgulanmıştır.
Veritabanı kurtarma modeli 'Full' yapılarak; Tam (Full), Artık (Differential) ve
İşlem Günlüğü (Transaction Log) katmanlarından oluşan üçlü bir yedekleme
hiyerarşisi oluşturulmuştur. Sistemin dayanıklılığını test etmek için kasıtlı bir
felaket senaryosu ile tüm kayıtlar silinmiş, ardından 'Point-in-Time Restore'
teknolojisi ile veriler silindiği anın saniyeler öncesine kayıpsız geri
döndürülmüştür. Bu mimari, SQL Server Agent üzerinden haftalık, günlük ve
saatlik görevler (Jobs) olarak T-SQL scriptleri ile otomatize edilerek RPO ve
RTO hedefleri kurumsal seviyeye çekilmiştir.
İkinci fazda ise siber tehditlere karşı çok katmanlı bir veritabanı güvenlik
mimarisi inşa edilmiştir. Yetkisiz müdahaleleri engellemek için 'En Az
Ayrıcalık Prensibi' uygulanarak kısıtlı okuma yetkisine sahip profil
oluşturulmuş; şüpheli erişimleri izlemek için 'SQL Server Audit'
mekanizması ile kullanıcı aktiviteleri sunucu günlüklerine (log) yazdırılmıştır.
Fiziksel sızıntı ve yedek hırsızlığına karşı veritabanı dosyaları 'At Rest'
konumunda AES-256 algoritmalı TDE (Transparent Data Encryption) ile
şifrelenmiştir. Son olarak, uygulama katmanındaki zafiyetleri gözlemlemek
için bir SQL Injection (SQLi) saldırısı simüle edilmiş ve bu güvenlik açığı,
'Parametrik Sorgular' kullanılarak başarıyla kapatılmıştır.
Sonuç olarak bu proje; felaketten kurtarma, SQL otomasyonu, yetkilendirme
ve kriptografik veri koruma disiplinlerini tek bir yapıda birleştirerek; siber
tehditlere karşı kendi kendini savunan ve sıfır veri kaybı hedefleyen otonom
bir veritabanı ekosisteminin inşasını uygulamalı olarak ortaya koymuştur.

# 1. Giriş

Bu projenin temel amacı, büyük veri setleri barındıran ve 7/24 kesintisiz
hizmet vermesi gereken kritik bilişim sistemlerinde yaşanabilecek olası veri
kayıplarına karşı proaktif mimariler tasarlamaktır. Kurumsal düzeyde bir
veritabanı yönetiminde, donanım arızaları, siber saldırılar (Ransomware
vb.) veya insan kaynaklı hatalar kaçınılmazdır. Bu bağlamda, İş Sürekliliği standartlarını sağlamak amacıyla veritabanını minimum veri kaybıyla (Minimum RPO - Recovery Point Objective) ve en kısa sürede (Minimum RTO - Recovery Time Objective) yeniden ayağa kaldırmak hedeflenmiştir.
Proje kapsamında Microsoft SQL Server (MSSQL) altyapısı üzerinde Tam
(Full), Artık (Differential) ve İşlem Günlüğü (Transaction Log) yedekleme
stratejileri hiyerarşik olarak kurgulanmıştır. Senaryo gereği oluşturulan
felaket anında, 'Point-in-Time Restore' (Belirli Bir Zamana Dönüş) teknolojisi
kullanılarak silinen veriler sıfır kayıpla başarıyla kurtarılmış ve tüm bu
süreçler TSQL scriptleri ile otomatize edilmiştir.
# 2. Veritabanı Ortamının Hazırlanması

Sistem testleri için gerçeğe yakın ve kritiklik derecesi yüksek bir senaryo
kurgulamak adına Kaggle veri bilimi platformu üzerinden detaylı sağlık
verileri içeren (Kalp Hastalıkları / Hasta Kayıtları) CSV formatında bir veri
seti indirilmiştir. Sağlık verileri, doğası gereği geri döndürülemez kayıplara
tahammülü olmayan hassas veriler statüsündedir; bu nedenle felaketten kurtarma senaryosu için en ideal test ortamını sunmaktadır. Elde edilen ham veri seti, SQL Server Management Studio (SSMS) üzerinde yer alan 'Flat File Import Wizard' aracı kullanılarak yapılandırılmış ve KalpHastalik veritabanına dbo.HeartFail tablosu olarak import edilmiştir. Veri tipleri optimize edilerek tablonun bütünlüğü sağlanmıştır.

# 3. Yedekleme Stratejileri ve Optimizasyon

Zamana dayalı hassas kurtarma işlemlerinin yapılabilmesi için veritabanı mimarisinde öncelikle yapısal bir değişikliğe gidilmiş ve veritabanının Recovery Model ayarı Simple moddan çıkartılarak Full moda alınmıştır. Bu sayede veritabanında yapılan her bir milisaniyelik hareketin log dosyasına yazılması garanti altına alınmıştır.

Ardından, depolama maliyetleri ve işlemci yükü dengelenerek aşağıdaki üç
katmanlı yedekleme adımları sırasıyla uygulanmıştır:

 
- Tam Yedek (Full Backup):
 
Sistemin mevcut stabil halinin tam ve
eksiksiz bir yedeği alınmıştır. Bu yedek, diğer tüm yedekleme
stratejilerinin üzerine inşa edileceği temel referans noktasıdır.

 
- Artık Yedek (Differential Backup):
 
Depolama alanından (Storage)
ve yedekleme zamanından büyük ölçüde tasarruf sağlamak amacıyla
kullanılmıştır. Sisteme yeni test kayıtları (INSERT) eklenmiş ve
sadece son Tam Yedek'ten itibaren değişen verilerin (Data Extents)
yedeği alınmıştır


- İşlem Günlüğü (Transaction Log) Yedeği:
 
Felaket anında
veritabanını "saniyesi saniyesine" eski haline döndürebilmek için
hayati öneme sahip olan log yedeği işlemi gerçekleştirilmiştir.

# 4. Felaket Senaryosu (Disaster Simulation)

Kurgulanan mimarinin sağlamlığını test etmek, kötü niyetli bir siber saldırıyı
veya yetkili bir sistem yöneticisinin yapabileceği geri dönülemez bir
dikkatsizliği simüle etmek amacıyla, veritabanındaki tüm hasta kayıtları
kasıtlı olarak yok edilmiştir.

Bunun için SSMS üzerinden
 
DELETE FROM dbo.heartFail; komutu
çalıştırılmıştır. İşlem sonucun
da
 
veritabanı tamamen boşaltılarak "Kritik Veri
Kaybı"
 
durumu başlatılmıştır.
# 5. Point-in-
Time Restore İşlemi (Kurtarma)

Kritik veri kaybı yaşandıktan hemen sonra önceden planlanan acil durum
protokolü devreye alınmıştır. Kurtarma işleminin MSSQL tarafından
engellenmemesi
 
için öncelikle arka plandaki tüm aktif kullanıcı bağlantıları
ve açık sorgular (
Close existing connections parametresi ile) zorla
sonlandırılmıştır.

Ardından
 
Restore Database > Timeline
 
mimarisi kullanılmıştır. Sistemde
alınan zincirleme yedekler (Full + Diff + Log) sayesinde, verilerin silindiği an
tespit edilmiş ve silinme komutunun çalıştırılmasından saniyeler
öncesindeki spesifik saat dilimi manuel olarak seçilerek hedef veritabanı
başarıyla o ana geri döndürülmüştür.
# 6. Sürdürülebilirlik İçin Çok Katmanlı Otoma
syon

Sistemin güvenliğini anlık testlerin ötesine taşımak ve insan inisiyatifini
ortadan kaldırmak amacıyla tüm yedekleme süreçleri otomatize edilmiştir.

T-
SQL scriptleri kullanılarak dinamik dosya adlandırma (o anki tarih ve saati
GETDATE()
 
ile dosya ismine ekleme) yapısı kurulmuştur. Hazırlanan bu
dinamik scriptler, msdb.dbo.sp_add_job
 
prosedürleri ile
 
SQL Server Agent
servisine "Görev" (Job) olarak tanımlanmıştır:

•
 
Haftalık Görev:
 
Her Pazar gece yarısı "Tam Yedek" alınması
zamanlanmıştır.
•
 
Günlük Görev:
 
Her gün 23:00'da gün içindeki değişimleri kapsayan
"Fark Yedeği" zamanlanmıştır.

•
 
Saatlik Görev:
 
Olası bir çökmede maksimum veri kaybını 1 saat ile
sınırlandırmak için düzenli "Log Yedeği" görevleri aktifleştirilmiştir.
# 7. Sonuç

Yapılan kapsamlı Point
-in-
Time Restore işlemi sonrasında hedef tablo
tekrar sorgulanmış (
SELECT TOP 1000 ROWS
) ve silinen binlerce sağlık
kaydının eksiksiz, veri bozulması
 
yaşanmadan sisteme geri döndüğü
doğrulanmıştır.

Ek olarak kurulan SQL Server Agent otomasyonları ile sistem, gelecekteki
felaketlere karşı kendi kendini yedekleyen otonom bir yapıya
kavuşturulmuştur. Bu proje çalışması ile MSSQL üzerinde sürdürülebilir,
güvenilir ve kurumsal standartlara
 
uygun bir felaketten kurtarma senaryosu
başarıyla uçtan uca test edilmiş ve uygulanmıştır.

Bu proje ile MSSQL üzerinde sürdürülebilir bir felaketten kurtarma
senaryosu başarıyla test edilmiştir.
