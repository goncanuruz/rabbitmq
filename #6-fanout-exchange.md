# RabbitMQ – Fanout Exchange

## Tanım

> **Fanout Exchange**, mesajların bu exchange’e bind edilmiş olan **tüm kuyruklara gönderilmesini sağlar**.
> Publisher, mesajların gönderildiği kuyruğun adını dikkate almaz ve mesajları **her kuyrukla paylaşır**.

---

## Davranış Özeti

* **Producer** bir mesajı Fanout Exchange’e gönderir.
* **Exchange**, mesajı bağlı olan **tüm kuyruklara broadcast eder.**
* **Routing key kullanılmaz.** (Yani mesaj yönlendirmesi yapılmaz, hepsi alır.)

**Kullanım Alanları:**

* Sistem genelinde bildirim, loglama veya duyuru yayını.
* Örneğin bir kullanıcı kayıt olduğunda hem e-posta servisine hem log sistemine hem de analiz servisine bilgi gitmesi.

> “Mesajların bu exchange’e bind olmuş olan tüm kuyruklara gönderilmesini sağlar. Publisher mesajların gönderildiği kuyruğu dikkate almaz ve mesajları tüm kuyruklara gönderir.”

---

### Publisher

```csharp
using RabbitMQ.Client;
using System.Text;
using System.Threading.Tasks;

ConnectionFactory factory = new()
{
    Uri = new("amqps://username:password@host/")
};

using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// 1️⃣ Fanout Exchange tanımla
channel.ExchangeDeclare(
    exchange: "fanout-exchange-example",
    type: ExchangeType.Fanout);

// 2️⃣ Mesaj gönder (örnek: 100 adet)
for (int i = 0; i < 100; i++)
{
    await Task.Delay(200);
    byte[] message = Encoding.UTF8.GetBytes($"Merhaba {i}");

    channel.BasicPublish(
        exchange: "fanout-exchange-example",
        routingKey: string.Empty, // Fanout'ta routing key önemsiz
        body: message
    );
}

Console.Read();
```

* `ExchangeType.Fanout` kullanılır.
* `routingKey` değeri **boş (string.Empty)** olmalıdır.
* Mesaj tüm kuyruklara aynı anda gider.

---

### Consumer

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

ConnectionFactory factory = new()
{
    Uri = new("amqps://username:password@host/")
};

using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// 1️⃣ Aynı Exchange tanımlanır
channel.ExchangeDeclare(
    exchange: "fanout-exchange-example",
    type: ExchangeType.Fanout);

// 2️⃣ Kullanıcıdan kuyruk adı alınır
Console.Write("Kuyruk adını giriniz: ");
string queueName = Console.ReadLine();

// 3️⃣ Kuyruk oluşturulur
channel.QueueDeclare(
    queue: queueName,
    exclusive: false);

// 4️⃣ Kuyruk Exchange'e bind edilir
channel.QueueBind(
    queue: queueName,
    exchange: "fanout-exchange-example",
    routingKey: string.Empty);

// 5️⃣ Mesajlar dinlenir
EventingBasicConsumer consumer = new(channel);
channel.BasicConsume(
    queue: queueName,
    autoAck: true,
    consumer: consumer);

consumer.Received += (sender, e) =>
{
    string message = Encoding.UTF8.GetString(e.Body.Span);
    Console.WriteLine($"[{queueName}] => {message}");
};

Console.Read();
```

---

1️⃣ Publisher’da Fanout türünde bir exchange oluşturulur.
2️⃣ Consumer, kullanıcıdan kuyruk adını ister ve bu isimle yeni bir kuyruk oluşturur.
3️⃣ Bu kuyruk `QueueBind` metodu ile exchange’e bağlanır.
4️⃣ Publisher mesaj gönderdiğinde exchange’e bağlı **tüm kuyruklara** mesaj ulaşır.
5️⃣ Her consumer, kendi kuyruğunu dinleyerek aynı mesajı alır.

---
## Özet

* Fanout Exchange, **tüm bağlı kuyruklara aynı mesajı yayınlar.**
* Routing key **önemsizdir**.
* Publisher döngüyle mesaj gönderir.
* Consumer tarafında kullanıcı kuyruk adını girer → kuyruk oluşturulur → exchange’e bind edilir.
* Her kuyruk aynı mesajı alır.
* Farklı exchange’lere bağlı kuyruklar bu mesajı almaz.

**Sonuç:**
Fanout Exchange, mesajların **broadcast (yayın)** mantığıyla **birden fazla kuyruğa aynı anda** iletilmesini sağlar.
Bu yapı genellikle **bildirim**, **loglama**, **event broadcast** gibi senaryolarda tercih edilir.


📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)
