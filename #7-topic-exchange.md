## RabbitMQ – Topic Exchange

**Tanım:**

> Routing key’leri kullanarak mesajları kuyruklara yönlendirmek için kullanılan bir exchange türüdür.
> Bu exchange ile routing key’in bir kısmına veya formatına göre kuyruklara mesaj gönderilir.

---

### Davranış Özeti

* **Producer**, mesajı `routing key` bilgisiyle birlikte **Topic Exchange**’e gönderir.
* **Exchange**, routing key’in **pattern’ine (şablonuna)** göre uygun kuyruklara mesajı iletir.
* **Consumer’lar**, kendi kuyruklarını belirli bir routing key formatına göre bind eder.
* Böylece yalnızca **ilgili pattern’e (eşleşmeye)** uygun mesajları alırlar.

 **Routing Key Formatı:**

* `.` (nokta) karakteri ile segmentlere ayrılır.
  Örnek: `asker.subay.yuzbasi`
* `*` → Tek bir kelime yerine geçer.
  Örnek: `asker.*.yuzbasi`
* `#` → Sıfır veya daha fazla kelime yerine geçer.
  Örnek: `asker.#`

> Routing key’leri kullanarak mesajları kuyruklara yönlendirmek için kullanılan bir exchange’dir.
> Bu exchange ile routing key’in bir kısmına/formatına göre kuyruklara mesaj gönderilir.
> Kuyruklar da routing key’e göre bu exchange’e abone olabilir ve sadece ilgili routing key’e göre gönderilen mesajları alabilir.

---

## Topic Exchange – Pratik İnceleme

### Publisher

```csharp
using RabbitMQ.Client;
using System.Text;

ConnectionFactory factory = new();
factory.Uri = new("amqps://befjdvjy:X6brcqMd4AMZmJKYHu6RshOAyBD08E0P@moose.rmq.cloudamqp.com/befjdvjy");

using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// 1️⃣ Topic Exchange tanımlanır
channel.ExchangeDeclare(
    exchange: "topic-exchange-example",
    type: ExchangeType.Topic
);

for (int i = 0; i < 100; i++)
{
    await Task.Delay(200);
    byte[] message = Encoding.UTF8.GetBytes($"Merhaba {i}");

    Console.Write("Mesajın gönderileceği topic formatını belirtiniz : ");
    string topic = Console.ReadLine();

    channel.BasicPublish(
        exchange: "topic-exchange-example",
        routingKey: topic,
        body: message
    );
}

Console.Read();
```

* `ExchangeType.Topic` tipi kullanılarak routing key bazlı mesaj yönlendirmesi yapılır.
* Kullanıcıdan alınan `topic` değeri, mesajın yönlendirilmesini belirler.
* Örnek routing key: `asker.subay.yuzbasi`
* Bu anahtar, consumer tarafındaki `binding key` ile eşleşirse mesaj kuyruğa düşer.

---

### Consumer 

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

ConnectionFactory factory = new();
factory.Uri = new("amqps://befjdvjy:X6brcqMd4AMZmJKYHu6RshOAyBD08E0P@moose.rmq.cloudamqp.com/befjdvjy");

using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// 1️⃣ Topic Exchange tanımlanır
channel.ExchangeDeclare(
    exchange: "topic-exchange-example",
    type: ExchangeType.Topic
);

// 2️⃣ Kullanıcıdan dinlenecek topic formatı alınır
Console.Write("Dinlenecek topic formatını belirtiniz : ");
string topic = Console.ReadLine();

// 3️⃣ Kuyruk oluşturulur
string queueName = channel.QueueDeclare().QueueName;

// 4️⃣ Kuyruk exchange’e bind edilir
channel.QueueBind(
    queue: queueName,
    exchange: "topic-exchange-example",
    routingKey: topic
);

// 5️⃣ Mesajlar tüketilir
EventingBasicConsumer consumer = new(channel);
channel.BasicConsume(
    queue: queueName,
    autoAck: true,
    consumer
);

consumer.Received += (sender, e) =>
{
    string message = Encoding.UTF8.GetString(e.Body.Span);
    Console.WriteLine(message);
};

Console.Read();
```

---

### Çalışma Adımları

1️⃣ **Exchange Tanımı:**
Publisher ve Consumer tarafında aynı isimle (`topic-exchange-example`) bir exchange oluşturulur.

2️⃣ **Publisher:**
Kullanıcıdan `routing key` formatı alınır.
Örnekler:

* `asker.subay.yuzbasi`
* `asker.#`
* `*.subay.*`

3️⃣ **Consumer:**
Kullanıcıdan `binding key` formatı alınır.
Örnek:

* `asker.#` → asker ile başlayan tüm mesajlar
* `#.subay.*` → ortasında “subay” geçen tüm mesajlar
* `*.subay.yuzbasi` → tam 3 segmentli ve ortası “subay” olan mesajlar

4️⃣ **Routing ve Binding Eşleşmesi:**
Publisher mesajı gönderdiğinde, sadece `routing key` ile `binding key` formatı eşleşen kuyruklar mesajı alır.

5️⃣ **Mesaj Dağıtımı:**

* Her consumer kendi pattern’ine göre mesajı alır.
* Uygun olmayan consumer’lar o mesajı görmez.

---


| Routing Key           | Binding Key | Eşleşme Durumu | Açıklama                                 |
| --------------------- | ----------- | -------------- | ---------------------------------------- |
| `asker.subay.yuzbasi` | `asker.#`   | ✅              | asker ile başlayan tüm mesajlar          |
| `asker.subay.yuzbasi` | `*.subay.*` | ✅              | ortasında subay olan 3 segmentli pattern |
| `asker.subay.yuzbasi` | `#.teğmen`  | ❌              | sonu teğmen olmadığı için eşleşmez       |
| `asker.subay.yuzbasi` | `#`         | ✅              | tüm mesajları alır                       |

---
* Topic Exchange, **pattern tabanlı mesaj yönlendirme** sağlar.
* `*` ve `#` karakterleri routing key içinde esnek eşleşme kurallarını temsil eder.
* Publisher, mesajları belirli bir pattern’e göre gönderir.
* Consumer’lar, ilgilendikleri pattern’leri dinleyerek sadece gerekli mesajları alır.
* RabbitMQ yalnızca ilgili exchange’e bağlı kuyruklara mesaj gönderir.

---

### Özet

Topic Exchange, **Direct** ve **Fanout**’un bir karışımıdır.

* Direct gibi routing key kullanır.
* Fanout gibi birden fazla kuyrukla çalışabilir.
  Ama farkı: routing key pattern’i üzerinden **filtreleme yapabilmesidir.**

🔸
* `asker.subay.yuzbasi` → tam eşleşme
* `asker.*.yuzbasi` → ortası fark etmeyen eşleşme
* `asker.#` → asker ile başlayan tüm routing key’ler



📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)
