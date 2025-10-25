## 🧱 Direct Exchange

**Tanım:**

> Mesajların direkt olarak belirli bir kuyruğa gönderilmesini sağlayan exchange’dir.

**Yapı:**

* **Producer**, mesajı **Direct Exchange**’e yollar.
* Mesaj, **Routing Key** değerine göre ilgili kuyruğa yönlendirilir.
* Örneğin:
  * Routing key = "green" → **green** kuyruğuna gider
  * Routing key = "red" → **red** kuyruğuna gider
  * Routing key = "orange" → **orange** kuyruğuna gider

**Kullanım Senaryosu:**

* Her mesajın **tek ve belirli** bir kuyruğa gitmesi isteniyorsa kullanılır.
* Örneğin: sipariş durum mesajlarının "approved", "rejected", "pending" gibi farklı kuyruklara ayrılması.

💬 Görsel not:

> “Mesajların direkt olarak belirli bir kuyruğa gönderilmesini sağlayan exchange’dir.”

---

## ⚙️ Direct Exchange Davranışı – Pratik İnceleme

### 🎓 Teorik Hatırlatma

* **Direct Exchange**, birden fazla kuyruğun bulunduğu senaryolarda, mesajın belirli bir kuyruğa yönlendirilmesini sağlar.
* **Publisher**, mesajı gönderirken `routing key` değeriyle hangi kuyruğa gideceğini belirtir.
* **Consumer**, bu kuyruğa bağlanarak sadece o anahtar ile gönderilen mesajları tüketir.

📌 **Özet:**

* Publisher → Exchange → Queue (routing key ile eşleşen)
* Hedef kuyruğa nokta atışı mesaj yönlendirme yapılır.

---

### 🧩 Publisher (Sol taraf)

```csharp
using RabbitMQ.Client;
using System.Text;

ConnectionFactory factory = new()
{
    Uri = new("amqps://USERNAME:PASSWORD@HOST/")
};

using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// 1️⃣ Exchange tanımlanır
channel.ExchangeDeclare(exchange: "direct-exchange-example", type: ExchangeType.Direct);

while (true)
{
    Console.Write("Mesaj : ");
    string message = Console.ReadLine();
    byte[] byteMessage = Encoding.UTF8.GetBytes(message);

    // 2️⃣ Mesaj gönderimi
    channel.BasicPublish(
        exchange: "direct-exchange-example",
        routingKey: "direct-queue-example",
        body: byteMessage
    );
}

Console.Read();
```

---

### 🧩 Consumer (Sağ taraf)

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

using IModel channel = connection.CreateModel();

// 1️⃣ Publisher ile aynı exchange tanımlanır
channel.ExchangeDeclare(exchange: "direct-exchange-example", type: ExchangeType.Direct);

// 2️⃣ Queue oluşturulur ve exchange'e bağlanır
string queueName = channel.QueueDeclare().QueueName;
channel.QueueBind(
    queue: queueName,
    exchange: "direct-exchange-example",
    routingKey: "direct-queue-example"
);

// 3️⃣ Mesajlar tüketilir
EventingBasicConsumer consumer = new(channel);
channel.BasicConsume(queue: queueName, autoAck: true, consumer: consumer);

consumer.Received += (sender, e) =>
{
    string message = Encoding.UTF8.GetString(e.Body.Span);
    Console.WriteLine(message);
};

Console.Read();
```

---

### 🧠 Açıklama Adımları

1️⃣ **Publisher’da**, consumer tarafında da kullanılacak olan isim ve type’a sahip bir exchange tanımlanmalıdır.
2️⃣ **Publisher** tarafından `routing key`’de bulunan değerdeki kuyruğa gönderilen mesajlar, consumer tarafından oluşturulan aynı isimli kuyrukla tüketilmelidir.
Bunun için öncelikle bir kuyruk oluşturulmalıdır.
3️⃣ **Binding:** Consumer tarafında `QueueBind` metodu ile queue, exchange ve routing key arasında bağlantı kurulmalıdır.
4️⃣ **Consume:** `BasicConsume` ile consumer kuyruğu dinleyip gelen mesajları ekrana yazdırır.

---

### 🧾 Transcript Özeti

* **Exchange türleri** teorik olarak daha önce anlatılmıştı, bu derste pratik olarak uygulanıyor.
* İlk örnek olarak **Direct Exchange** seçildi.
* Publisher, mesaj gönderirken hedef kuyruğu `routing key` ile belirtiyor.
* Consumer tarafında aynı isimde bir exchange tanımlanıp queue oluşturuluyor.
* Bu queue, routing key’e göre gelen mesajları alacak şekilde `QueueBind` ile bağlanıyor.
* Consumer, `BasicConsume` metodu ile gelen mesajları işliyor.
* Böylece “nokta atışı” mesaj yönlendirmesi sağlanıyor.

🎯 **Sonuç:**
Direct Exchange modeli, birden fazla kuyruğun bulunduğu durumlarda belirli kuyruğa mesaj göndermeyi sağlar.
Bu yapı sayesinde sistemler hedef odaklı, kontrollü mesaj akışı elde eder.

---
📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)
