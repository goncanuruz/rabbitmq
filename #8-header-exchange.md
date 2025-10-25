## Header Exchange

**Tanım:**

> Routing key yerine **header** (başlık) değerlerini kullanarak mesajların kuyruklara yönlendirilmesini sağlayan exchange türüdür.

**Temel Fikir:**
Header Exchange, mesajların hangi kuyruğa iletileceğine **routing key** yerine **header anahtar-değer (key-value)** eşleştirmelerine göre karar verir. Yani mesajın header kısmındaki bilgiler, kuyruklarla eşleşme kriteri olarak kullanılır.

---

### Header Exchange’in Çalışma Mantığı

* Her mesajın bir veya birden fazla **header (key/value)** çifti olabilir.
* Her kuyruk, kendisine ait **header eşleşme kuralları** ile tanımlanır.
* Bir mesaj geldiğinde, bu kurallar doğrultusunda hangi kuyruğa yönlendirileceğine karar verilir.

**Routing Key kullanılmaz.**
Karar verme işlemi tamamen header verilerine göre yapılır.

---

### `x-match` Özelliği

Header Exchange'de, kuyrukların mesajlarla nasıl eşleşeceğini belirleyen özel bir parametredir.

| x-match Değeri | Açıklama                                                                      |
| -------------- | ----------------------------------------------------------------------------- |
| **any**        | Header'lardan **en az biri** eşleşirse mesaj kuyruğa gönderilir.              |
| **all**        | Header'lardaki **tüm** key/value çiftleri eşleşirse mesaj kuyruğa gönderilir. |

Varsayılan olarak `x-match = any` değerindedir.

---

| Exchange Türü | Eşleşme Kriteri               | Kullanım Amacı                              |
| ------------- | ----------------------------- | ------------------------------------------- |
| **Direct**    | Routing key tam eşleşmesi     | Belirli bir kuyruğa mesaj göndermek         |
| **Topic**     | Routing key pattern eşleşmesi | Desen bazlı mesaj yönlendirme               |
| **Header**    | Header key/value eşleşmesi    | İçerik veya özellik bazlı mesaj yönlendirme |

---

### Publisher

```csharp
using RabbitMQ.Client;
using System.Text;

ConnectionFactory factory = new();
factory.Uri = new("amqps://befjdvjy:X6brcqMd4AMZmJKYHu6RshOAyBD08E0P@moose.rmq.cloudamqp.com/befjdvjy");

using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

channel.ExchangeDeclare(
    exchange: "header-exchange-example",
    type: ExchangeType.Headers);

for (int i = 0; i < 100; i++)
{
    await Task.Delay(200);
    byte[] message = Encoding.UTF8.GetBytes($"Merhaba {i}");

    Console.Write("Lütfen header value'sunu giriniz : ");
    string value = Console.ReadLine();

    IBasicProperties basicProperties = channel.CreateBasicProperties();
    basicProperties.Headers = new Dictionary<string, object>
    {
        ["no"] = value
    };

    channel.BasicPublish(
        exchange: "header-exchange-example",
        routingKey: string.Empty,
        body: message,
        basicProperties: basicProperties
    );
}

Console.Read();
```

**Açıklama:**

* `ExchangeType.Headers` kullanılır.
* `routingKey` boş geçilir çünkü kullanılmaz.
* `basicProperties.Headers` ile mesajın header bilgisi atanır.

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

channel.ExchangeDeclare(
    exchange: "header-exchange-example",
    type: ExchangeType.Headers);

Console.Write("Lütfen header value'sunu giriniz : ");
string value = Console.ReadLine();

string queueName = channel.QueueDeclare().QueueName;

channel.QueueBind(
    queue: queueName,
    exchange: "header-exchange-example",
    routingKey: string.Empty,
    new Dictionary<string, object>
    {
        ["no"] = value
    });

EventingBasicConsumer consumer = new(channel);
channel.BasicConsume(
    queue: queueName,
    autoAck: true,
    consumer: consumer);

consumer.Received += (sender, e) =>
{
    string message = Encoding.UTF8.GetString(e.Body.Span);
    Console.WriteLine(message);
};

Console.Read();
```

**Açıklama:**

* Consumer tarafında, hangi header değerine sahip mesajların alınacağı belirlenir.
* `QueueBind` içinde header eşleşme kuralı (`["no"] = value`) tanımlanır.
* Header değeri eşleşen mesajlar ilgili kuyruğa düşer.

---

### Transcript Özeti

> Header Exchange, routing key kullanmadan header değerleriyle mesaj yönlendirmesi yapar. Bu yönlendirme için header verileri dictionary (key/value) formatında belirlenir.
> `x-match` özelliği sayesinde bir veya birden fazla header koşulunun sağlanmasına göre mesajın hedef kuyruğa gidip gitmeyeceği belirlenir.

**Kısaca:**

* Routing key kullanılmaz.
* Header değerleriyle eşleşme yapılır.
* `x-match` ile tüm eşleşmelerin mi yoksa birinin mi yeterli olacağı belirlenir.

---

### Örnek

| Header               | x-match | Kuyruk Davranışı                         |
| -------------------- | ------- | ---------------------------------------- |
| `{no: "A"}`          | any     | `no = A` olan mesajları alır             |
| `{no: "B"}`          | any     | `no = B` olan mesajları alır             |
| `{no: "A", no: "B"}` | all     | Tüm eşleşmeler sağlandığında mesajı alır |

---

###

Header Exchange, mesaj yönlendirmesinde **routing key** yerine **header bilgilerini** kullanmak isteyen durumlar için uygundur.
Özellikle **filtreleme**, **etiket bazlı mesajlaşma** veya **özellik temelli broadcast** senaryolarında tercih edilir.

---

📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)
