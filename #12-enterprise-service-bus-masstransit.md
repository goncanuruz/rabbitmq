# RabbitMQ & MassTransit – Worker Service ile Enterprise Service Bus (ESB) Kullanımı

---

## 1. Enterprise Service Bus (ESB) Nedir?

**Enterprise Service Bus (ESB)**, servisler arası entegrasyonu kolaylaştıran, farklı sistemlerin **birbirleriyle iletişim kurmasını** sağlayan bir mimaridir.
Yani sistemler arası mesajlaşma, veri akışı ve orkestrasyon görevlerini üstlenen bir **aracı katman**dır.

**Amaç:**

* Farklı sistemlerin iletişimini kolaylaştırmak
* Teknoloji bağımlılığını azaltmak
* Gelecekte farklı mesaj broker’lara geçişi kolaylaştırmak (örn. RabbitMQ → Kafka)

**Avantaj:**
ESB yapısı sayesinde sistemler **gevşek bağlı** (loosely coupled) hale gelir, değişikliklerden minimum etkilenir.

---

## 2. MassTransit Nedir?

**MassTransit**, .NET için geliştirilmiş, **açık kaynaklı** bir *Enterprise Service Bus (ESB)* kütüphanesidir.

### Özellikleri:

* Ücretsiz ve açık kaynak
* RabbitMQ, Azure Service Bus, ActiveMQ, Amazon SQS gibi çoklu **transport** desteği
* **Asenkron**, **dağıtık** ve **mesaj tabanlı** haberleşme
* **Saga**, **Orchestration**, **Choreography** gibi gelişmiş **design pattern** desteği
* Test edilebilir, gözlemlenebilir, dayanıklı ve ölçeklenebilir yapı

### Ne İşe Yarar?

MassTransit, mikroservisler arası mesaj tabanlı iletişimi kolaylaştırır.
Farklı protokolleri gizleyerek tek bir iletişim yüzeyi sağlar.

---

## 3. RabbitMQ Üzerinde MassTransit Kullanımı

### Temel Kavramlar:

| Rol                       | Açıklama                             |
| ------------------------- | ------------------------------------ |
| **Producer (Publisher)**  | Mesajı üreten servis                 |
| **Consumer (Subscriber)** | Mesajı tüket en servis               |
| **Queue (Kuyruk)**        | Mesajların tutulduğu yapı            |
| **Exchange**              | Mesajların yönlendirildiği merkez    |
| **Binding**               | Exchange ile Queue arasındaki ilişki |

---
## 4. Proje Yapısı

```
RabbitMQ.ESB.MassTransit.Shared.Messages
RabbitMQ.ESB.MassTransit.WorkerService.Publisher
RabbitMQ.ESB.MassTransit.WorkerService.Consumer
```

### Shared

```csharp
public interface IExampleMessage
{
    string Text { get; }
}

public class ExampleMessage : IExampleMessage
{
    public string Text { get; set; }
}
```

---

## 5. Worker Service – Publisher (Mesaj Gönderici)

```csharp
IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddMassTransit(configurator =>
        {
            configurator.UsingRabbitMq((context, _configurator) =>
            {
                _configurator.Host("amqps://...@moose.rmq.cloudamqp.com/befjdvjy");
            });
        });

        services.AddHostedService<PublishMessageService>(provider =>
        {
            using IServiceScope scope = provider.CreateScope();
            IPublishEndpoint publishEndpoint = scope.ServiceProvider.GetService<IPublishEndpoint>();
            return new PublishMessageService(publishEndpoint);
        });
    })
    .Build();

await host.RunAsync();
```

### `PublishMessageService.cs`

```csharp
public class PublishMessageService : BackgroundService
{
    readonly IPublishEndpoint _publishEndpoint;

    public PublishMessageService(IPublishEndpoint publishEndpoint)
    {
        _publishEndpoint = publishEndpoint;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        int i = 0;
        while (!stoppingToken.IsCancellationRequested)
        {
            ExampleMessage message = new() { Text = $"Mesaj {++i}" };
            await _publishEndpoint.Publish(message);
            await Task.Delay(1000);
        }
    }
}
```

---

# 6. Worker Service – Consumer

```csharp
IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddMassTransit(configurator =>
        {
            configurator.AddConsumer<ExampleMessageConsumer>();
            configurator.UsingRabbitMq((context, _configurator) =>
            {
                _configurator.Host("amqps://...@moose.rmq.cloudamqp.com/befjdvjy");
                _configurator.ReceiveEndpoint("example-message-queue", e =>
                {
                    e.ConfigureConsumer<ExampleMessageConsumer>(context);
                });
            });
        });
    })
    .Build();

await host.RunAsync();
```

### `ExampleMessageConsumer.cs`

```csharp
public class ExampleMessageConsumer : IConsumer<ExampleMessage>
{
    public Task Consume(ConsumeContext<ExampleMessage> context)
    {
        Console.WriteLine($"Mesaj alındı: {context.Message.Text}");
        return Task.CompletedTask;
    }
}
```

---

## 7. Publish vs Send Farkı

| Özellik              | **Publish**                                | **Send**                                                                                                                            |
| -------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| **Kapsam**           | Event tabanlı                              | Command tabanlı                                                                                                                     |
| **Yönlendirme**      | Tüm subscriber’lara mesaj gönderir         | Belirtilen kuyruk(lar)a gönderir                                                                                                    |
| **Kullanım Arayüzü** | `IPublishEndpoint`                         | `ISendEndpointProvider`                                                                                                             |
| **Kod Örneği**       | `await _publishEndpoint.Publish(message);` | `var endpoint = await _sendEndpointProvider.GetSendEndpoint(new Uri("queue:example-message-queue")); await endpoint.Send(message);` |
| **Desen (Pattern)**  | Publish/Subscribe                          | Request/Response veya Command                                                                                                       |

---

## 8. Özet

| Konu                              | Açıklama                                                |
| --------------------------------- | ------------------------------------------------------- |
| **ESB Amacı**                     | Servisler arası iletişimi standartlaştırmak             |
| **MassTransit**                   | .NET için açık kaynaklı ESB kütüphanesi                 |
| **Transport Layer**               | RabbitMQ (AMQP protokolü)                               |
| **Worker Service Kullanımı**      | Arka planda sürekli çalışan publish/consume işlemleri   |
| **Send vs Publish**               | Command vs Event tabanlı mesajlaşma farkı               |
| **Serialization**                 | Otomatik olarak MassTransit tarafından yönetilir        |
| **Config & Dependency Injection** | `AddMassTransit()` ve `AddHostedService()` ile sağlanır |

---

## 9. Kaynak Kodda Dikkat Edilecekler

* `IPublishEndpoint` → Event gönderimi için
* `ISendEndpointProvider` → Spesifik kuyruğa mesaj göndermek için
* `IConsumer<T>` → Mesaj tüketici sınıfı
* `ReceiveEndpoint` → Queue tanımı
* `ConfigureConsumer` → Consumer binding işlemi
* `BackgroundService` → Worker Service tabanlı sürekli görev

---

## Özet

MassTransit, RabbitMQ gibi mesaj kuyruklarını **soyutlayarak**, mikroservis mimarisinde mesaj tabanlı iletişimi kolaylaştırır.
**Send** komutu belirli bir alıcıya, **Publish** komutu ise tüm subscriber’lara mesaj iletir.
Bu yapı sayesinde uygulamalar:

* Daha **bağımsız**
* Daha **esnek**
* Daha **ölçeklenebilir** hale gelir.

---

📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=vBv7FbmInqM&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=3)

