## Basit Düzeyde Publisher ve Consumer Uygulamaları Oluşturma Detayları  

## ⚙️ Gerekli Kurulumlar

- .NET projelerinde RabbitMQ kullanmak için gerekli kütüphane:  
  ```bash
  Install-Package RabbitMQ.Client
  ```
- Hem **Publisher** hem **Consumer** uygulamasına yüklenmelidir.  
- Kütüphane, **NuGet** üzerinden eklenir.  
- Hem **.NET Console**, hem **ASP.NET Core**, hem de **Microservice** mimarilerinde kullanılabilir.

> 💡 İleri seviye mikroservislerde, `RabbitMQ.Client` doğrudan değil,  
> **MassTransit** veya **Enterprise Service Bus** gibi kütüphaneler aracılığıyla dolaylı kullanılır.

---

## 🧩 Publisher Uygulaması  
### 🧭 İşlem Sırası

1. **Bağlantı Oluşturma**  
   RabbitMQ sunucusuna bağlantı oluşturulur.

2. **Bağlantıyı Aktifleştirme ve Kanal Açma**  
   Bağlantı üzerinden işlemler yapılabilmesi için bir kanal açılır.

3. **Queue Oluşturma**  
   Mesajların gönderileceği kuyruk oluşturulur.

4. **Queue’ya Mesaj Gönderme**  
   Kuyruğa mesaj gönderilir.

---

### 💻 Publisher Kod Örneği

```csharp
using RabbitMQ.Client;
using System.Text;

// 🧩 Bağlantı Oluşturma
ConnectionFactory factory = new();
factory.Uri = new("amqps://<username>:<password>@<host>/<vhost>");

// 🔗 Bağlantıyı Aktifleştirme ve Kanal Açma
using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// 📦 Queue Oluşturma
channel.QueueDeclare(queue: "example-queue", exclusive: false);

// ✉️ Queue’ya Mesaj Gönderme
byte[] message = Encoding.UTF8.GetBytes("Merhaba");
channel.BasicPublish(exchange: "", routingKey: "example-queue", body: message);

Console.Read();
```

---

### 📘 Açıklamalar

| Aşama | Açıklama |
|--------|-----------|
| `ConnectionFactory` | Bağlantı için gerekli nesne. |
| `factory.Uri` | CloudAMQP veya lokal bağlantı bilgileri. |
| `CreateConnection()` | RabbitMQ sunucusuna bağlantı oluşturur. |
| `CreateModel()` | Bağlantı üzerinden bir kanal açar. |
| `QueueDeclare()` | Kuyruğu tanımlar, `exclusive: false` diğer bağlantıların da erişimine izin verir. |
| `BasicPublish()` | Mesajı kuyruğa gönderir. |
| `exchange: ""` | Varsayılan olarak **Direct Exchange** kullanılır. |
| `routingKey` | Mesajın hedef kuyruğunun adıdır. |

---

### 🧾 Default Exchange Davranışı

- Exchange parametresi boş (`""`) bırakıldığında **Default (Direct) Exchange** devreye girer.  
- Bu durumda **Routing Key = Queue Adı** olan kuyruk hedef alınır.

---

## 📥 Consumer Uygulaması  
### 🧭 İşlem Sırası

1. **Bağlantı Oluşturma**  
   RabbitMQ sunucusuna bağlanılır.  

2. **Bağlantıyı Aktifleştirme ve Kanal Açma**  
   İşlemleri gerçekleştirecek kanal açılır.  

3. **Queue Oluşturma**  
   Publisher’daki kuyrukla **aynı yapılandırmada** kuyruk tanımlanır.  

4. **Queue’dan Mesaj Okuma**  
   Kuyruktaki mesajlar alınır ve işlenir.  

---

### 💻 Consumer Kod Örneği

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

// Bağlantı oluşturma
ConnectionFactory factory = new();
factory.Uri = new("amqps://<username>:<password>@<host>/<vhost>");

// Kanal açma
using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();

// Kuyruğu tanımlama (Publisher ile aynı olmalı)
channel.QueueDeclare(queue: "example-queue", exclusive: false);

// Mesajları dinleme
var consumer = new EventingBasicConsumer(channel);
channel.BasicConsume(queue: "example-queue", autoAck: true, consumer: consumer);

consumer.Received += (model, ea) =>
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($"[x] Received: {message}");
};

Console.Read();
```

---

### 📘 Açıklamalar

| Aşama | Açıklama |
|--------|-----------|
| `EventingBasicConsumer` | Kuyruktaki mesajları dinler. |
| `BasicConsume()` | Kuyruğa gelen mesajları okur. |
| `autoAck` | Mesaj okunduğunda otomatik olarak silinip silinmeyeceğini belirler. |
| `Received` | Mesaj geldiğinde tetiklenir. |
| `ea.Body.ToArray()` | Mesaj içeriğini byte dizisi olarak alır. |
| `Encoding.UTF8.GetString()` | Byte dizisini string’e dönüştürür. |

---

## ⚡ Çalışma Mantığı

- Publisher, kuyruğa mesaj gönderir.  
- Consumer, aynı kuyruğu dinler ve gelen mesajı yakalar.  
- Mesajlar **asenkron** olarak işlenir.  
- Kuyrukta biriken mesajlar, RabbitMQ arayüzünden izlenebilir.  
- **Default Exchange** üzerinden yönlendirme yapılır.

---

## 🧪 Test Senaryosu

Publisher tarafında mesaj gönderimini döngüyle test edebilirsin:

```csharp
for (int i = 0; i < 100; i++)
{
    byte[] message = Encoding.UTF8.GetBytes($"Merhaba {i}");
    channel.BasicPublish("", "example-queue", body: message);
}
```

Consumer tarafında bu mesajlar sırayla ekrana yazdırılır.

> Gözlemlemek için gönderim arasına kısa gecikme eklenebilir:
> ```csharp
> Thread.Sleep(200);
> ```

---
📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)
