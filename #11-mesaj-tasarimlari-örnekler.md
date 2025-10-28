### RabbitMQ Mesaj Tasarımları – Gerçek Hayat Senaryoları ile Özet

### 1. Point-to-Point (Direct) Tasarımı

Tanım:
Mesajlar bir kuyruk aracılığıyla yalnızca bir tüketiciye (consumer) gönderilir. Mesajı ilk alan consumer işler.

Gerçek Hayat Senaryosu:
Bir müşteri hizmetleri uygulamasında, her müşteri destek talebi oluşturduğunda bu talep bir kuyruğa iletilir. Sistemde bulunan destek temsilcilerinden yalnızca biri bu talebi alır ve işlemeye başlar.

Örnek Exchange: direct
Örnek Routing Key: support.low, support.high
Uygun olduğunda birebir yönlendirme yapılır.

### 2. Publish/Subscribe (Fanout) Tasarımı

Tanım:
Bir mesaj birçok consumer'a aynı anda yayınlanır. Routing key dikkate alınmaz.

Gerçek Hayat Senaryosu:
Bir e-ticaret platformunda yeni bir kampanya duyurusu yapıldığında bu bilgi hem mobil push bildirimi, hem e-posta, hem de SMS sistemlerine gönderilir. Her servis kendi kuyruğunda bu mesajı alır ve işler.

Örnek Exchange: fanout
Routing Key: yok
Tüm bağlı kuyruklara aynı mesaj gönderilir.

### 3. Work Queue Tasarımı

Tanım:
Aynı görev türü birden fazla çalışan (consumer) arasında dağıtılır. Her mesaj yalnızca bir çalışan tarafından işlenir.

Gerçek Hayat Senaryosu:
Bir PDF dönüştürme servisi düşünelim. Kullanıcılar sisteme belge yüklüyor. Bu belgeler dönüştürülmek üzere kuyruğa atılıyor. Birden fazla worker (işleyici) paralel olarak çalışıyor ve gelen işleri sırayla alıp dönüştürüyor.

Örnek Exchange: direct
Ekstra Ayar: basic.qos ile yük dengelemesi yapılır.

###  4. Request/Response Tasarımı

Tanım:
Bir sistem bir istek (request) gönderir ve yanıt (response) bekler. Genellikle replyTo ve correlationId kullanılır.

Gerçek Hayat Senaryosu:
Bir mobil uygulama, bir kullanıcının bakiye sorgusu için sunucuya mesaj gönderir. Sunucu bu isteği işler ve kullanıcıya bireysel kuyruğu üzerinden yanıt döner. Bu süreçte correlationId ile eşleşme yapılır.

Örnek Exchange: direct veya topic
Kullanılan Properties: replyTo, correlationId
İstek/yanıt senkronizasyonu sağlanır.

### Özet Tablo
Tasarım Türü	Kullanım Amacı	Örnek Senaryo	Exchange Türü
Point-to-Point	Tek consumer'a yönlendirme	Müşteri destek sistemi	direct
Publish/Subscribe	Aynı mesajı çok noktaya iletmek	Kampanya duyuruları (mobil, e-posta, SMS)	fanout
Work Queue	İş yükünü dağıtmak	PDF dönüştürme/thumbnail oluşturma gibi arka plan işlemleri	direct
Request/Response	İstek ve cevap alışverişi	Kullanıcı bakiye sorgusu gibi birebir işlemler