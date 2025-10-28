# 📨 RabbitMQ & MassTransit – Request–Response Pattern & Message Acknowledgement

## 1. Request–Response Pattern Nedir?

**Request–Response (İstek–Yanıt)**, bir servisin başka bir servisten veri veya işlem sonucu beklediği klasik haberleşme desenidir.

* **Request:** Bir isteğin gönderilmesidir. (Publisher tarafı)
* **Response:** İsteğe karşılık verilen cevaptır. (Consumer tarafı)

MassTransit bu deseni otomatik olarak yönetir:

* Kuyruk üzerinden **istek–yanıt akışını** sağlar.
* `IRequestClient<TRequest>` ve `RespondAsync<TResponse>` arayüzleriyle iletişimi soyutlar.

### Akış Diyagramı

```
Publisher (RequestClient)
   ↓  RequestMessage gönderir
Queue (request-queue)
   ↓  Consumer alır, işler
Consumer → RespondAsync(ResponseMessage)
   ↓  ResponseMessage geri döner
Publisher → Cevabı alır ve işler
```
## 2. Proje Yapısı
```
RabbitMQ.ESB.MassTransit.Request.Response.Example
 ┣ Shared (Request & Response message kontratları)
 ┣ Publisher (istek gönderen uygulama)
 ┗ Consumer (isteğe yanıt dönen uygulama)
```
---
## 3. Shared Katmanı – Mesaj Modelleri

```csharp
// Request mesaj modeli
public record RequestMessage
{
    public int MessageNo { get; init; }
    public string Text { get; init; }
}

// Response mesaj modeli
public record ResponseMessage
{
    public string Text { get; init; }
}
```

## 4. Publisher

```csharp
using MassTransit;
using MassTransit.Request.Response.Example.Shared;

string rabbitMqUri = "amqp://localhost";

var bus = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host(rabbitMqUri);
});

await bus.StartAsync();

var requestClient = bus.CreateRequestClient<RequestMessage>(new Uri($"{rabbitMqUri}/request-queue"));

int i = 1;
while (true)
{
    await Task.Delay(200);

    var response = await requestClient.GetResponse<ResponseMessage>(new RequestMessage
    {
        MessageNo = i,
        Text = $"{i++}. request"
    });

    Console.WriteLine($"Response Received: {response.Message.Text}");
}
```

### 💡 Açıklama

* `Bus.Factory.CreateUsingRabbitMq()` → RabbitMQ bağlantısını oluşturur.
* `bus.StartAsync()` → İstek–yanıt iletişimini başlatır.
* `CreateRequestClient<T>()` → RequestClient nesnesi üretir.
* `GetResponse<T>()` → Cevap bekleyen çağrıyı gerçekleştirir.

> **Not:** Request–Response pattern'de Publisher **hem request gönderir hem de response dinler**, bu yüzden `bus.StartAsync()` çağrısı gereklidir.

---

## 5. Consumer

```csharp
using MassTransit;
using MassTransit.Request.Response.Example.Shared;

string rabbitMqUri = "amqp://localhost";

var bus = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host(rabbitMqUri);

    cfg.ReceiveEndpoint("request-queue", endpoint =>
    {
        endpoint.Consumer<RequestMessageConsumer>();
    });
});

await bus.StartAsync();
Console.ReadLine();

// Consumer sınıfı
namespace MassTransit.Request.Response.Example.Consumer.Consumers
{
    public class RequestMessageConsumer : IConsumer<RequestMessage>
    {
        public async Task Consume(ConsumeContext<RequestMessage> context)
        {
            await Console.Out.WriteLineAsync(context.Message.Text);

            await context.RespondAsync(new ResponseMessage
            {
                Text = $"{context.Message.MessageNo}. response to request"
            });
        }
    }
}
```
* `ReceiveEndpoint` → `request-queue` isimli kuyruğu dinler.
* `Consumer<T>` → Gelen `RequestMessage` türündeki mesajları işler.
* `RespondAsync()` → Publisher’a geri cevap gönderir.

### Akış Özeti

1. Publisher mesajı gönderir → `request-queue`
2. Consumer alır → `Consume()`
3. Consumer `RespondAsync()` ile cevap döner
4. Publisher `GetResponse()` üzerinden cevabı alır

---

## 6. MassTransit Message Acknowledgement

### Nedir?

RabbitMQ’da bir mesaj **tüketildiğinde** (Consumed) ama **işlenmeden silinirse**, veri kaybı yaşanabilir.
**Message Acknowledgement (ack)** bu durumu engeller:

* Mesaj işlenene kadar kuyruktan **silinmez**.
* İşlem başarıyla bittiğinde **acknowledge (ACK)** edilir.
* Hata olursa **NACK** (Negative Ack) ile mesaj kuyrukta tutulur ve yeniden işlenir.

### ⚙️ MassTransit’te Ack Yönetimi

MassTransit bu mekanizmayı **otomatik** yönetir.

* `Consume()` metodu başarıyla tamamlanırsa → **ACK** gönderilir.
* `Consume()` içinde hata oluşursa → **NACK** gönderilir ve mesaj **retry queue**’ya alınır.

### Örnek Senaryo

```csharp
public async Task Consume(ConsumeContext<RequestMessage> context)
{
    try
    {
        await ProcessAsync(context.Message);
        await context.RespondAsync(new ResponseMessage { Text = "Success" });
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
        throw; // NACK tetiklenir
    }
}
```

> ⚡ **Sonuç:** `throw` edildiğinde mesaj tekrar işlenir. MassTransit’in `Retry` mekanizması devreye girer.

---

## Özet 

| Konu                  | Açıklama                               |
| --------------------- | -------------------------------------- |
| **Pattern**           | Request–Response                       |
| **Publisher Arayüzü** | `IRequestClient<T>`                    |
| **Consumer Arayüzü**  | `IConsumer<T>` + `RespondAsync()`      |
| **Ack Yönetimi**      | Otomatik (MassTransit tarafından)      |
| **Retry**             | Hata durumunda otomatik yeniden deneme |
| **Queue Adı**         | `request-queue`                        |

---

MassTransit, Request–Response gibi klasik senkron desenleri **asenkron bir mesajlaşma altyapısında** kolaylaştırır.
Message Acknowledgement ise veri güvenliğini sağlar.

Bu yapı sayesinde:

* Servisler **bağımsız** ve **dayanıklı** hale gelir.
* Mesaj kaybı yaşanmaz.
* RabbitMQ + MassTransit altyapısı, **mikroservis iletişimi** için hazır hale gelir.

---
📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)
