# RabbitMQ Mesaj Tasarımları
## Mesaj Tasarımlarına Genel Bakış

Mesaj tasarımları, RabbitMQ gibi mesaj broker sistemlerinde **senaryolara göre sergilenecek davranışların** standardize edilmiş biçimidir.
Bir problem senaryosuna uygun olarak mesajın **nasıl iletileceği**, **kim tarafından tüketileceği** ve **hangi modelin kullanılacağı** bu tasarımlarla belirlenir.

> 🔹 “Design Pattern” ile farkı:
> Mesaj tasarımı bir **senaryonun** genel yapısını tanımlar;
> dizayn pattern ise bu senaryo içindeki **pratik uygulama yöntemidir**.

---

## 1. P2P (Point-to-Point) Tasarımı

Bu tasarımda **publisher**, mesajı **tek bir kuyruğa** gönderir ve bu mesaj sadece **bir consumer** tarafından tüketilir.
Genellikle **Direct Exchange** kullanılır.

### Kod Yapısı

**Publisher**

```csharp
var factory = new ConnectionFactory() { Uri = new Uri("amqp://localhost") };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

string queueName = "example-p2p-queue";
channel.QueueDeclare(queue: queueName, durable: false, exclusive: false, autoDelete: false);

byte[] message = Encoding.UTF8.GetBytes("merhaba");
channel.BasicPublish(exchange: string.Empty, routingKey: queueName, body: message);
```

**Consumer**

```csharp
var factory = new ConnectionFactory() { Uri = new Uri("amqp://localhost") };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

string queueName = "example-p2p-queue";
channel.QueueDeclare(queue: queueName, durable: false, exclusive: false, autoDelete: false);

var consumer = new EventingBasicConsumer(channel);
channel.BasicConsume(queue: queueName, autoAck: false, consumer: consumer);

consumer.Received += (sender, e) =>
{
    Console.WriteLine(Encoding.UTF8.GetString(e.Body.Span));
};
```

> “Bir kuyruğa mesaj gönderip sadece o kuyruğu dinleyen consumer'ın işlemesini istiyorsak, bu P2P tasarımıdır.”

---

## 2. Publish/Subscribe (Fanout) Tasarımı

### Özet

Bir **publisher**, mesajı bir **exchange**’e gönderir.
Bu mesaj, o exchange’e **bağlı tüm kuyruklara** yönlendirilir.
Çoklu tüketici senaryolarında kullanılır.
Genellikle **Fanout Exchange** tercih edilir.

### Kod Yapısı

**Publisher**

```csharp
var factory = new ConnectionFactory() { Uri = new Uri("amqp://localhost") };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

string exchangeName = "example-pub-sub-exchange";
channel.ExchangeDeclare(exchange: exchangeName, type: ExchangeType.Fanout);

for (int i = 0; i < 100; i++)
{
    var message = Encoding.UTF8.GetBytes($"merhaba {i}");
    channel.BasicPublish(exchange: exchangeName, routingKey: string.Empty, body: message);
}
```

**Consumer**

```csharp
var factory = new ConnectionFactory() { Uri = new Uri("amqp://localhost") };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

string exchangeName = "example-pub-sub-exchange";
channel.ExchangeDeclare(exchange: exchangeName, type: ExchangeType.Fanout);

string queueName = channel.QueueDeclare().QueueName;
channel.QueueBind(queue: queueName, exchange: exchangeName, routingKey: string.Empty);

var consumer = new EventingBasicConsumer(channel);
channel.BasicConsume(queue: queueName, autoAck: false, consumer: consumer);

consumer.Received += (sender, e) =>
{
    Console.WriteLine(Encoding.UTF8.GetString(e.Body.Span));
};
```

> “Fanout exchange, kuyruğun adı fark etmeksizin tüm bağlı kuyruklara mesaj gönderir.
> Dolayısıyla Publish/Subscribe tasarımı için en ideal exchange tipidir.”

---

## 3. Work Queue (İş Kuyruğu) Tasarımı

Birden fazla **consumer** arasında **iş yükünü adil şekilde** dağıtır.
Her mesaj **yalnızca bir consumer** tarafından işlenir.
Genellikle **Direct Exchange** kullanılır.

### Kod Yapısı

**Publisher**

```csharp
channel.QueueDeclare(queue: "example-work-queue", durable: false, exclusive: false, autoDelete: false);

for (int i = 0; i < 100; i++)
{
    var body = Encoding.UTF8.GetBytes($"merhaba {i}");
    channel.BasicPublish(exchange: string.Empty, routingKey: "example-work-queue", body: body);
}
```

**Consumer**

```csharp
channel.QueueDeclare(queue: "example-work-queue", durable: false, exclusive: false, autoDelete: false);
channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);

var consumer = new EventingBasicConsumer(channel);
channel.BasicConsume(queue: "example-work-queue", autoAck: false, consumer: consumer);

consumer.Received += (sender, e) =>
{
    Console.WriteLine(Encoding.UTF8.GetString(e.Body.Span));
    channel.BasicAck(e.DeliveryTag, false);
};
```

> “Her mesaj yalnızca bir consumer tarafından işlenir.
> Böylece tüm consumer’lar aynı iş yükünü ve görev dağılımını paylaşır.”

---

## 4. Request/Response Tasarımı

Bu tasarımda **publisher**, bir mesaj gönderir ve ardından **consumer**’dan gelen cevabı (“response”) bekler.
Her iki taraf da hem **publisher** hem de **consumer** rolünü üstlenebilir.
`CorrelationId` ve `ReplyTo` property’leri kullanılarak mesajlar ilişkilendirilir.

### Kod Yapısı

**Publisher**

```csharp
string requestQueue = "example-request-response-queue";
string replyQueue = channel.QueueDeclare().QueueName;
string correlationId = Guid.NewGuid().ToString();

var props = channel.CreateBasicProperties();
props.CorrelationId = correlationId;
props.ReplyTo = replyQueue;

var body = Encoding.UTF8.GetBytes("Request Message");
channel.BasicPublish(exchange: "", routingKey: requestQueue, basicProperties: props, body: body);

var consumer = new EventingBasicConsumer(channel);
channel.BasicConsume(queue: replyQueue, autoAck: true, consumer: consumer);

consumer.Received += (model, ea) =>
{
    if (ea.BasicProperties.CorrelationId == correlationId)
        Console.WriteLine("Response: " + Encoding.UTF8.GetString(ea.Body.Span));
};
```

**Consumer**

```csharp
string queueName = "example-request-response-queue";
channel.QueueDeclare(queue: queueName, durable: false, exclusive: false, autoDelete: false);

var consumer = new EventingBasicConsumer(channel);
channel.BasicConsume(queue: queueName, autoAck: true, consumer: consumer);

consumer.Received += (sender, e) =>
{
    Console.WriteLine("Request received: " + Encoding.UTF8.GetString(e.Body.Span));
    var response = Encoding.UTF8.GetBytes("İşlem tamamlandı");

    var replyProps = channel.CreateBasicProperties();
    replyProps.CorrelationId = e.BasicProperties.CorrelationId;

    channel.BasicPublish(
        exchange: "",
        routingKey: e.BasicProperties.ReplyTo,
        basicProperties: replyProps,
        body: response
    );
};
``` 

> “Publisher hem mesajı gönderir hem de sonucu bekler;
> Consumer mesajı işler ve sonucu başka bir kuyruğa gönderir.
> İkisi de birer ‘mini publisher/consumer’ gibidir.”

---

## Özet

Bu dört tasarım RabbitMQ dünyasında en sık kullanılan iletişim modelleridir:

| Tasarım           | Amaç                            | Kullanılan Exchange | Mesaj Tüketimi |
| ----------------- | ------------------------------- | ------------------- | -------------- |
| Point-to-Point    | Tek consumer tarafından tüketim | Direct              | 1:1            |
| Publish/Subscribe | Çoklu consumer bildirimi        | Fanout              | 1:N            |
| Work Queue        | İş yükü dağıtımı                | Direct              | Adil paylaşım  |
| Request/Response  | İstek–cevap modeli              | Direct              | İki yönlü      |


📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)

