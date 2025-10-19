# 🐇 RabbitMQ'ya Giriş ve Temeller

## 📬 Message Queue Nedir?

- **Message Queue**, yazılım sistemlerinde iletişim için kullanılan bir yapıdır.  
- **Birbirinden bağımsız sistemler** arasında veri alışverişi yapmak için kullanılır.  
- Gönderilen mesajları **kuyrukta saklar** ve sonradan bu mesajların işlenmesini sağlar.  
- Kuyruğa mesaj gönderen: **Producer (Yayıncı / Publisher)**  
- Kuyruktaki mesajları işleyen: **Consumer (Tüketici)**  

**Message (mesaj):**  
İki sistem arasında iletişim için kullanılan veri birimidir.  
Producer’ın Consumer tarafından işlenmesini istediği veridir.

**Örnek:**  
Bir e-ticaret sisteminde siparişe dair mesaj olarak;
- Sipariş numarası  
- Müşteri bilgileri  
- Ürün bilgisi  
- Ödeme bilgileri  
örnek verilebilir.

> Ayrıca, message queue içerisindeki mesajların consumer tarafından **sırayla işlendiğine** dikkat edilmelidir.

---

## 🎯 Message Queue’nun Amacı Nedir?

- Bazı senaryolarda farklı sistemler arasında **senkron haberleşme** kullanıcı deneyimi açısından uygun olmayabilir.  
- Ödeme işlemi tamamlandığında kullanıcıya “başarılı” bilgisi dönülürken, **fatura oluşturma işlemi** arkada asenkron olarak yapılabilir.  
- Bu sayede kullanıcı beklemeden işlemini tamamlamış olur.

**Amaç:**
- Asenkron iletişim modeli ile sistemler arasındaki yoğunluğu azaltmak.  
- İşlemleri arka planda yöneterek uygulamayı **daha verimli hale getirmek.**

> Böylece sistemler arası iletişim daha verimli olur ve gecikme yaşanmaz.

---

## ⚙️ Senkron vs. Asenkron

| Model | Açıklama |
|--------|-----------|
| **Senkron** | Service A, Service B’ye istek atar ve **cevap gelene kadar bekler.** |
| **Asenkron** | Service A, mesajı **Message Broker’a gönderir** ve beklemeden işine devam eder. Service B daha sonra mesajı işler. |

> Mail göndermek, fatura oluşturmak, stok güncellemek gibi **zaman gerektiren işlemler** asenkron iletişim modeliyle işlenmelidir.

---

## 🧩 Message Broker

- **Message Broker**, içerisinde **Message Queue**’leri barındırır.  
- Publisher/Producer ile Consumer arasındaki iletişimi sağlar.  
- Bir Message Broker içinde **birden fazla queue** bulunabilir.

---

## 🐇 RabbitMQ Nedir?

- **Open source** bir message queuing sistemidir.  
- **Erlang diliyle** geliştirilmiştir.  
- **Cross-platform** desteği sayesinde farklı işletim sistemlerinde çalışabilir.  
- **Cloud** hizmeti mevcuttur.  
- **Zengin dokümantasyona** sahiptir.

---

## 💡 RabbitMQ’yu Neden Kullanmalıyız?

- Yazılım uygulamalarında **ölçeklenebilir** bir ortam sağlar.  
- Kullanıcılardan gelen istekler anlık cevaplanamıyorsa veya zaman alan işlemler varsa, bu işlemler **asenkron şekilde** çalıştırılarak sistem yoğunluğu azaltılabilir.  
- Kullanıcı gereksiz yere **uzun response time** beklemez.  
- RabbitMQ, uzun sürebilecek operasyonları uygulamadan bağımsızlaştırarak, başka bir uygulamanın bu işlemleri üstlenmesini sağlar.

**Somut örnek:**  
Bir web uygulaması Word dosyasını PDF’e dönüştürecekse bu işlemi RabbitMQ kuyruğuna atar.  
Farklı bir servis kuyruğu dinleyip dönüştürme işlemini yapar.  
Böylece web uygulaması yoğunluk yaşamaz ve kullanıcı beklemeden devam eder.

---

## 🔄 RabbitMQ’nun İşleyişi Nasıldır?

RabbitMQ, **AMQP (Advanced Message Queuing Protocol)** protokolü üzerine kuruludur.

### 🧱 Temel Bileşenler:
1. **Publisher:** Mesajı oluşturur ve gönderir.  
2. **Exchange:** Mesajı alır, uygun kuyruklara yönlendirir.  
3. **Queue:** Mesajı saklar.  
4. **Consumer:** Mesajı alıp işler.

📤 **Akış:**
> Publisher → Exchange → Queue → Consumer

---

## ⚙️ RabbitMQ’nun İşleyiş Detayları

- RabbitMQ, **mesajların nasıl işleneceğini modelleyen** bir sistem sunar.  
- Publisher/Producer mesajı yayınlar, Consumer ise mesajı tüketir.  
- Yapısal olarak **Exchange** ve **Queue** üzerinden çalışır.  
- Publisher mesajı publish ettikten sonra, mesaj **Exchange** tarafından karşılanır.  
  - Exchange, mesajın hangi **route** üzerinden hangi **queue**’ya gideceğini belirler.  

- Publisher ve Consumer’ın **hangi dil veya platformda geliştirildiği önemli değildir.**  
  RabbitMQ, dilden bağımsız bir iletişim sağlar.

> Tüm bu süreçte RabbitMQ, **AMQP (Advanced Message Queuing Protocol)** protokolünü kullanır.

---

## 🧠 Özet

RabbitMQ;
- Uygulamalar arasında **bağımsız iletişim** sağlar.  
- **Asenkron** süreçleri yönetir.  
- **Yoğunluğu azaltır, performansı artırır.**  
- **Dil ve platform bağımsızdır.**

---

📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=lJ4is5_ShJs&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=2)

