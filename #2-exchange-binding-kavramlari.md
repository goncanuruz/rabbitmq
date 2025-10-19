# 🔁 RabbitMQ – Exchange Binding Kavramları ve Exchange Türleri

## 🔁 Exchange Nedir?

- Publisher tarafından gönderilen mesajların nasıl yönlendirileceğini ve hangi route’lara yönlendirileceğini belirlememiz konusunda kontrol sağlayan / karar veren yapıdır.  
- Route ise mesajların exchange üzerinden kuyruklara nasıl yönlendirileceğini tanımlayan mekanizmadır.  
- Bu süreçte exchange’de bulunan routing key değeri kullanılır.  
- Routing key, bir mesajın hangi kuyruklara gönderileceği konusunda exchange’e bilgi sunar.  
- Route ise genel olarak mesajların yolunu ifade eder.

---

## 🔗 Binding Nedir?

- Exchange ve Queue arasındaki ilişkiyi ifade eden yapıdır.  
- Exchange ile kuyruk arasında bağlantı oluşturmanın terminolojik adıdır.  
- Exchange birden fazla queue’ya bind olabiliyorsa, o halde mesajın hangi kuyruğa gideceği exchange türüne göre değişir.  
- Exchange’in bind edildiği kuyruklardan hangisine mesaj göndereceğini anlaması exchange türüne göre değişiklik gösterir.  

> Misal olarak: **Topic** ve **Direct** exchange türleri routing key ile ayırt ederken,  
> **Fanout** ve **Headers** yöntemleri farklı şekilde çalışır.

---

## 🎯 Direct Exchange

- Mesajların **direkt olarak belirli bir kuyruğa** gönderilmesini sağlayan exchange’dir.  
- Mesaj, **routing key’e uygun** olan hedef kuyruğa gönderilir.  
- Bunun için mesaj gönderilecek kuyruğun adını **routing key** olarak belirtmek yeterlidir.

### 🧩 Kullanım Senaryosu
- Genellikle **hata mesajlarının işlendiği** senaryolarda kullanılır.  
- Örneğin sistemde:
  - `dosya yükleme hatası`
  - `veritabanı bağlantı hatası`
  gibi farklı hata türleri olabilir.  
  Her biri için ayrı kuyruklar oluşturularak hataların izlenmesi kolaylaştırılır.

### 🛒 E-Ticaret Örneği
Bir e-ticaret sisteminde sipariş sürecini düşünelim:  
- Sipariş durumları: `Onaylandı`, `İptal Edildi`, `İade Edildi`  
- Her bir durum için farklı routing key kullanılabilir:  
  - `order.confirmed`  
  - `order.canceled`  
  - `order.returned`  

Bu sayede her durum ilgili kuyruğa yönlendirilir ve sistemin takibi kolaylaşır.

---

## 📢 Fanout Exchange

- Mesajların, bu exchange’e **bind olmuş tüm kuyruklara gönderilmesini sağlar.**  
- Publisher mesajların gönderileceği kuyruk isimlerini dikkate almaz, mesajlar **tüm bağlı kuyruklara** gönderilir.

### 💡 Kullanım Senaryosu
- Özellikle **microservice mimarilerinde** kullanılır.  
- Tüm servislere ortak bir bildirim veya event yayınlanmak istendiğinde tercih edilir.  
- Böylece veri paylaşımı merkezi, hızlı ve etkili hale gelir.

🎯 **Kıyaslama:**  
> **Direct Exchange:** Belirli bir kuyruğa gönderim yapar.  
> **Fanout Exchange:** Tüm bağlı kuyruklara yayın yapar (broadcast mantığı).

---

## 🧠 Topic Exchange

- **Routing key**’leri kullanarak mesajların kuyruklara yönlendirilmesini sağlar.  
- Routing key’in bir kısmına / yapısına göre kuyruklara mesaj gönderir.  
- Kuyruklar, routing key desenine göre bu exchange’e abone olabilirler.

### 💡 Kullanım Senaryosu
- Özellikle **log sistemlerinde** kullanılır.  
- Log seviyelerine göre (örneğin `info`, `warning`, `error`) farklı kuyruklara yönlendirme yapılabilir.  
- Böylece sistem yöneticileri sadece kendi ilgilendikleri log seviyelerini dinleyebilir.

**Örnek:**
- Routing key: `usa.news`  
- Routing key: `europe.weather`  
- Binding key: `usa.#` → USA ile başlayan tüm mesajlar  
- Binding key: `*.weather` → yalnızca hava durumu logları

---

## 🧾 Header Exchange

- **Routing key yerine header** bilgilerini kullanarak mesajların kuyruklara yönlendirilmesini sağlar.  
- Her mesaj belirli anahtar–değer çiftleri (**header**) içerir.  
- Kuyruklar, header bilgilerine göre mesaj alabilir.  

**Örnek:**  
- Header: `{ key1: value1, key2: value2 }`  
- Kuyruklar bu header’lara göre eşleşen mesajları tüketir.  
- `x-match = any` → Key’lerden biri eşleşirse mesaj alınır.  
- `x-match = all` → Tüm key–value çiftleri eşleşmelidir.


📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)

