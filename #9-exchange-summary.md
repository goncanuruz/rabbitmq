1. Direct Exchange

Ne zaman kullanılır?
Mesajlar belirli bir işlem tipi veya kategoriye göre tam eşleşen bir routing key ile yönlendirilmek isteniyorsa kullanılır.

Gerçek Hayat Senaryosu:
Bir destek sistemi düşünelim. Gelen talepler priority.low, priority.medium, priority.high şeklinde sınıflandırılıyor.

Neden Direct Exchange?
Çünkü her destek seviyesi için farklı bir kuyruk var ve mesajlar tam olarak o seviyeye yönlendirilmek isteniyor.

Routing Key Örneği: priority.high → HighPriorityQueue

2. Fanout Exchange

Ne zaman kullanılır?
Tüm dinleyicilere aynı mesaj yayın yapılmak isteniyorsa, routing key’e bakılmadan yönlendirme yapılır.

Gerçek Hayat Senaryosu:
Yeni bir ürün lansmanı yapıldığında, tüm kullanıcılara hem e-posta, hem SMS, hem mobil bildirim gitmesi gerekiyor.

Neden Fanout Exchange?
Çünkü mesajın içeriği aynıdır ve tüm abonelere aynı anda gönderilmelidir.

Sonuç: Tüm bağlı kuyruklar mesajı alır.

3. Topic Exchange

Ne zaman kullanılır?
Routing key belirli bir desene göre tanımlanıyorsa, örneğin kategori.haber.altkategori gibi durumlarda kullanılır.

Gerçek Hayat Senaryosu:
Bir haber uygulamasında news.sports.football, news.politics.election gibi routing key'ler var.

Neden Topic Exchange?
Çünkü kullanıcılar news.sports.* gibi desenlerle abone olabilir. Bu sayede sadece futbol değil tüm spor haberleri alınabilir.

Joker Karakterler:

* → tek segment

# → bir veya daha fazla segment

Routing Key Örneği: news.sports.football → Queue: SportsNewsConsumer

4. Header Exchange

Ne zaman kullanılır?
Routing key kullanılmak istenmeyip, anahtar-değer çiftleriyle (headers) daha esnek yönlendirme isteniyorsa kullanılır.

Gerçek Hayat Senaryosu:
Bir lojistik sisteminde teslimat türü (express, standard) ve lokasyon gibi bilgiler mesajın header'ında yer alıyor.

Neden Header Exchange?
Çünkü yönlendirme kararları sadece routing key ile verilemeyecek kadar karmaşık. Örneğin delivery: express, region: EU.

Header Örneği: { "delivery": "express", "region": "EU" } → ExpressEuropeQueue

Karşılaştırmalı Özet Tablosu
Exchange Türü	Yönlendirme Kriteri	Örnek Senaryo	Neden Kullanılır?
Direct	Tam routing key eşleşmesi	Destek talebi seviyeleri	Kesin eşleşme gerekli
Fanout	Routing key YOK	Yeni ürün duyurusu	Herkese aynı mesaj
Topic	Routing key deseni	Haber kategorileri (spor, siyaset)	Esnek, çok seviyeli yönlendirme
Header	Header key–value eşleşmesi	Teslimat türü + bölge bazlı sistem	Routing key yeterli değil, detaylı kurallar gerek