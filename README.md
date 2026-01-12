# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [ ]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [ ]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [ ]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [X]  LRU / CLOCK gibi algoritmaları
- [ ]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [ ]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Kavram      | Bellek (Clock Algoritması) | Disk / DB (WAL Mimarisi)        |
|-------------|----------------------------|---------------------------------|
| Adresleme   | Buffer ID (Pointer)        | LSN (Log Sequence Number)       |
| Hız         | O(1)                       | Sequential Write (Sıralı Yazma) |
| Öncelik     | Usage Count                | Veri Güvenliği                  |
| Veri yapısı | Circular Buffer (Ring)     | Append-Only Log                 |
| Yönetim     | StrategyGetBuffer          | ReserveXLogInsertLocation       |

---

# Video [Linki](https://youtu.be/uB64QI5EHfs)

---

# Açıklama 

Sistem Programlama ve Veri Yapıları Perspektifiyle PostgreSQL Performans Analizi
Modern Veritabanı Yönetim Sistemleri (DBMS), donanım kaynaklarının (CPU, RAM ve Disk) fiziksel sınırlarını aşmak ve kullanıcıya "sonsuz bellek" illüzyonu sunmak üzerine kuruludur. Bu çalışmada, PostgreSQL açık kaynak kodları üzerinde yaptığım incelemelerde; bellek yönetiminde kullanılan Clock Sweep (Saat) algoritması ve disk yönetiminde kullanılan WAL (Write Ahead Log) mimarisi üzerinden performans optimizasyonlarını analiz ettim.

### 1. Bellek Yönetimi ve Clock Algoritması (freelist.c)

Sistem programlama derslerinde öğrendiğimiz en temel darboğaz, disk I/O işlemlerinin RAM erişimine göre binlerce kat yavaş olmasıdır. Bu nedenle veritabanları, diskteki veri bloklarını (Pages) "Buffer Pool" adı verilen bir bellek alanında önbellekler. Ancak RAM sınırlıdır ve dolduğunda hangi sayfanın atılacağına (eviction) karar vermek bir optimizasyon problemidir.

Genel kanının aksine, PostgreSQL saf bir LRU (Least Recently Used) algoritması kullanmaz. Standart LRU, her erişimde bir bağlı listenin (Linked List) güncellenmesini gerektirir. Çok çekirdekli (Multi-threaded) sistemlerde bu durum, liste üzerinde sürekli kilitlenme (lock contention) yaratır ve performansı düşürür.

İncelediğim src/backend/storage/buffer/freelist.c dosyasındaki StrategyGetBuffer fonksiyonu, bunun yerine Clock Sweep algoritmasını uygular. Kodda gördüğümüz ClockSweepTick() fonksiyonu ve onu takip eden sonsuz döngü (for(;;)), dairesel bir veri yapısı üzerinde dönen bir saat ibresini temsil eder. Algoritma şu mantıkla çalışır:

İbre bir sayfaya geldiğinde usage_count (kullanım) değerini atomik olarak okur.

Eğer değer > 0 ise, sayfa yakın zamanda kullanılmış demektir. Algoritma sayfayı atmak yerine skorunu 1 azaltır (local_buf_state -= BUF_USAGECOUNT_ONE) ve ona "İkinci Bir Şans" vererek pas geçer.

Eğer değer 0 ise, bu sayfa "Soğuk Veri" (Cold Data) olarak işaretlenir ve return buf komutu ile kurban seçilerek bellekten atılır.

Bu yöntem, karmaşık liste güncellemeleri yerine basit bit manipülasyonları kullandığı için O(1) karmaşıklığına yakın bir hızda çalışır ve sistem kaynaklarını minimize eder.

### 2. Disk Yönetimi ve WAL Mimarisi (xloginsert.c)

RAM ne kadar hızlıysa, disk o kadar güvenilirdir (Persistent). Ancak veritabanına gelen her yazma isteğini doğrudan ana veri dosyalarına (Heap) yazmak, diskin okuma/yazma kafasını sürekli farklı sektörlere sıçratacağı için (Random I/O) performansı öldürür. PostgreSQL bu sorunu WAL (Write Ahead Log) mekanizması ile çözer.

src/backend/access/transam/xloginsert.c dosyasında incelediğim XLogInsertRecord fonksiyonu, veriyi diske yazmadan önce bir paket haline getirir. Ancak buradaki asıl mühendislik dehası, fonksiyonun içinde çağrılan ReserveXLogInsertLocation bölümündedir.

Kod incelememde tespit ettiğim endbytepos = startbytepos + size; matematiksel işlemi, bu mimarinin "Append-Only" (Sadece Ekleme) çalıştığının en net kanıtıdır. Sistem, veriyi diskin rastgele bir yerine değil, mevcut imlecin (cursor) kaldığı son noktanın tam üzerine ekler. Bu sayede yazma işlemi Sıralı (Sequential Write) hale gelir. Sıralı yazma, mekanik disklerde kafa hareketini sıfıra indirdiği gibi, SSD'lerde de yazma verimliliğini maksimize eder.

Ayrıca kodun START_CRIT_SECTION() bloğu ile başlaması, bu işlemin işletim sistemi seviyesinde kesintiye uğramaması gereken (Atomic) bir süreç olduğunu gösterir. Veri RAM'den diske, CopyXLogRecordToWAL fonksiyonu ile sıralı bir şekilde kopyalanarak veri bütünlüğü garanti altına alınır.

### Sonuç

Özetle PostgreSQL; okuma performansını, kilit (lock) gerektirmeyen ve dairesel veri yapısı kullanan Clock Algoritması ile RAM üzerinde optimize ederken; yazma performansını ve güvenliğini, sıralı erişim mantığına dayanan WAL yapısı ile disk üzerinde sağlamaktadır. Bu iki mekanizma, Sistem Programlama ve Veri Yapıları prensiplerinin gerçek hayattaki en güçlü uygulamalarındandır.
## VT Üzerinde Gösterilen Kaynak Kodları

**Bellek Yönetimi (Clock Algoritması - `StrategyGetBuffer`):** \
[https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/freelist.c](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/freelist.c)

**Disk Yönetimi - WAL (Sıralı Yazma - `XLogInsertRecord`):** \
[https://github.com/postgres/postgres/blob/master/src/backend/access/transam/xloginsert.c](https://github.com/postgres/postgres/blob/master/src/backend/access/transam/xloginsert.c)

