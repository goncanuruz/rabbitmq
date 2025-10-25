# RabbitMQ Eğitimi #4 - Gelişmiş Kuyruk Mimarisi

---

## 🧩 Gelişmiş Kuyruk Mimarisi Nedir?

RabbitMQ teknolojisinin ana fikri, yoğun kaynak gerektiren işleri/görevleri/operasyonları hemen yapmaya koyularak tamamlanmasını beklemek zorunda kalmaksızın, bu işleri ölçeklendirilebilir bir vaziyette daha sonra yapılacak şekilde planlamaktır.

Tabi bu planlamayı gerçekleştirirken kuyruklardan istifade edilmekte ve görevleri temsil edecek olan mesajlar bu kuyruklara atılmakta ve tüketiciler tarafından bu mesajlar elde edilerek görevlerin asenkron bir şekilde işlenmesi sağlanmaktadır.

Tüm bu süreçte kuyrukların bakımı, yağlanması gerekmekte :) Senaryosuna göre mesajların kalıcılığına dair durumlar vs. konfigüre edilmesi gerekmektedir. Ayrıca birden fazla tüketicinin söz konusu olduğu durumlarda nasıl bir davranışın olacağı vs. durumları da oldukça önem arz etmektedir.

Gelişmiş Kuyruk Mimarisi işte tam da bu noktada devreye girmektedir! Kuyrukların ve mesajların kalıcılığı, mesajların birden fazla tüketiciye karşı dağıtım stratejisi yahut tüketici tarafından işlenmiş bir mesajın kuyruktan silinebilmesi için onay/bildiri sistemi vs. tüm bu detaylar bu başlık altında değerlendirilecektir.

---

## Round Robin Dispatching

### 💬 Tanım

RabbitMQ varsayılan olarak tüm consumer’lara mesajları **sırayla (döngüsel)** gönderir. Bu davranış **Round Robin Dispatching** olarak adlandırılır.

### 🧩 Mekanizma

- Mesajlar kuyruğa gelir.
- RabbitMQ bu mesajları sırayla consumer’lara gönderir.
- Örneğin: 1. mesaj → Consumer A, 2. mesaj → Consumer B, 3. mesaj → Consumer C.
- Bu döngü tüm mesajlar bitene kadar devam eder.

### 🔍 Örnek Senaryo

E-ticaret siparişlerinde her siparişin sıralı olarak farklı servislere dağıtılması sağlanabilir. Ancak bu modelde bazı consumer’ların daha hızlı veya daha yavaş olması durumunda adaletsizlik oluşabilir.

---

## 📩 Message Acknowledgement

RabbitMQ, tüketiciye gönderdiği mesajı başarılı bir şekilde işlensin veya işlenmesin hemen kuyruktan silinmesi üzere işaretler.

Tüketicilerin kuyruktan aldıkları mesajları işlemeleri sürecinde herhangi bir kesinti yahut problem durumu meydana gelirse ilgili mesaj tam olarak işlenemeyeceği için esasında görev tamamlanmamış olacaktır.

Bu tarz durumlara istinaden mesaj başarılı işlendi ise eğer kuyruktan silinmesi için tüketiciden RabbitMQ’nun uyarılması gerekmektedir.

Consumer’dan mesaj işlemenin başarıyla sonuçlandığına dair dönüt alan RabbitMQ mesajı silecektir.

### ⚠️ Sorun

Consumer hata aldığında veya kapanırsa mesaj kuyruktan silinir ve **kaybolur.**

### ✅ Çözüm

**Manual Acknowledgement (Manuel Onaylama)** kullanılır.

```csharp
channel.BasicConsume(queue: "example-queue", autoAck: false, consumer: consumer);

// Mesaj başarıyla işlendiğinde onay gönderme
channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
```

### 🔧 Parametreler

| Parametre          | Açıklama                                                             |
| ------------------ | -------------------------------------------------------------------- |
| **autoAck: false** | RabbitMQ, mesajı onay gelene kadar silmez.                           |
| **BasicAck**       | Consumer, mesajı başarıyla işlediğini bildirir.                      |
| **multiple**       | `true` olursa, belirtilen mesajdan önceki tüm mesajlar da onaylanır. |

---

## ⚠️ Message Acknowledgement Problemleri Nelerdir?

Bir message işlenmeden consumer problem yaşarsa bu mesajın sağlıklı bir şekilde işlenebilmesi için başka bir consumer tarafından tüketilebilir olmalıdır.

Aksi taktirde mesaj kuyruktan tüketici tarafından alındığı an silinirse bu durumda veri kaybı ihtimali söz konusu olacaktır. İşte bu tarz durumlar için Message Acknowledgement özelliği şarttır diyebiliriz.

Eğer ki Message Acknowledgement özelliğini kullanıyorsanız kesinlikle mesaj işleme başarıyla sonlandığı taktirde RabbitMQ’ya mesajın silinmesi için haber göndermeyi unutmayınız. Aksi taktirde mesaj tekrar yayınlanacak ve başka bir tüketici tarafından tekrar işlenecektir. Bu durumda da bir mesajın birden fazla kez işlenmesi durumu söz konusu olabilir.

Tabi ayriyeten mesajlar onaylanarak silinmediği taktirde kuyrukta kalınmasına neden olacak ve bu durum kuyrukların büyümesi ve yavaşlamasıyla sonuçlanıp performans düşüklüğüne neden olacaktır.

---

## 💬 Message Acknowledgement’e Dair Son İstişareler

Anlayacağınız, bu özellik sayesinde bir mesajın kaybolmadığından emin olabilmekteyiz.

Tüketici açısından mesajın alındığını, işlendiğini ve artık kuyruktan silinebilir olduğunu ifade ederek süreci daha güvenilir hale getirmekteyiz.

Tabi her tüketicinin işlem yoğunluğuna göre RabbitMQ’ya karşılık onay bildirim süresi değişkenlik gösterecektir.

RabbitMQ açısından tüketiciden gelecek olan onay bildirimi için bir zaman aşımı süresi söz konusudur. Bu varsayılan olarak 30 dk’dır.

Evet, RabbitMQ tüketicinin işlem neticesini 30 dk boyunca bekler. Bu sürenin nasıl değiştirilebileceğini dersimizin devamında pratik olarak ele alıyor olacağız.

Eğer bu süre dolarda tüketiciden herhangi bir onay bildirimi gelmezse RabbitMQ mesajı tekrar yayınlamaya devam edecektir.

---

## ⚙️ Message Acknowledgement Nasıl Yapılandırılır? (BasicAck)

RabbitMQ’da mesaj onaylama sürecini aktifleştirebilmek için consumer uygulamasında `BasicConsume` metodundaki `autoAck` parametresini görseldeki gibi false değerine getirebilirsiniz.

Böylece RabbitMQ’da varsayılan olarak mesajların kuyruktan silinme davranışı değiştirilecek ve consumer’dan onay beklenecektir.

Consumer, mesajı başarıyla işlediğine dair uyarıyı `channel.BasicAck` metodu ile gerçekleştirir.

Multiple parametresi, birden fazla mesaja dair onay bildirisi gönderir. Eğer true değeri verilirse, DeliveryTag değerine sahip olan bu mesajla birlikte bundan önceki mesajların işlendiğini onaylar. Aksi taktirde false verilirse sadece bu mesaj için onay bildirisinde bulunacaktır.

---

## 🔄 BasicNack ile İşlenmeyen Mesajları Geri Gönderme

Bazen consumer’lar da istemsiz durumların dışında kendi kontrollerimiz neticesinde mesajları işlememek isteyebilir veyahut ilgili mesajın işlenmesini başarıyla sonuçlandıramayacağımızı anlayabiliriz.

Böyle durumlarda `channel.BasicNack` metodunu kullanarak RabbitMQ’ya bilgi verebilir ve mesajı tekrardan işletebiliriz.

Tabi burada requeue parametresi oldukça önem arz etmektedir. Bu parametre, bu consumer tarafından işlenemeyeceği ifade edilen bu mesajın tekrardan kuyruğa eklenip eklenmemesinin kararını vermektedir.

True değeri verildiği taktirde mesaj kuyruğa tekrardan işlenmek üzere eklenecek, false değerinde ise kuyruğa eklenmeyerek silinecektir. Sadece bu mesajın işlenmeyeceğine dair RabbitMQ’ya bilgi verilmiş olunacaktır.

---

## ⛔ BasicCancel ile Bir Kuyruktaki Tüm Mesajların İşlenmesini Reddetme

`BasicCancel` metodu ile verilen consumerTag değerine karşılık gelen queue’daki tüm mesajlar reddedilerek işlenmez.

```csharp
// Tek bir mesajı reddetmek
channel.BasicNack(deliveryTag: ea.DeliveryTag, multiple: false, requeue: true);

// Tüm mesajları reddetmek
channel.BasicCancel(consumerTag);
```

### 🔧 Parametreler

| Parametre          | Açıklama                      |
| ------------------ | ----------------------------- |
| **requeue: true**  | Mesaj kuyruğa tekrar eklenir. |
| **requeue: false** | Mesaj silinir.                |

---

## 🚫 BasicReject ile Tek Bir Mesajın İşlenmesini Reddetme

RabbitMQ’da kuyrukta bulunan mesajlardan belirli olanların consumer tarafından işlenmesini istemediğimiz durumlarda `BasicReject` metodunu kullanabiliriz.

---

## 🧱 Message Durability

Consumer’ların sıkıntı yaşama durumunda mesajların kaybolmayacağının garantisinin nasıl sağlanacağını öğrenmiş olduk.

Ancak RabbitMQ sunucusunun bir zevale uğraması durumunda ne olacağını da konuşmamız gerekmektedir.

Evet, RabbitMQ sunucusunda bir problem meydana geldiğinde yahut kapandığında ne olacak sizce?

RabbitMQ’da normal şartlarda bir kapanma durumu söz konusu olursa tüm kuyruklar ve mesajlar silinecektir!

Böyle bir durumda mesajların kaybolmaması, yani kalıcı olabilmesi için ekstradan çalışma gerçekleştirmemiz gerekmektedir.

Bu çalışma; kuyruk ve mesaj açısından kalıcı olarak işaretleme yapmamızı gerektirmektedir.

```csharp
// Kuyruğu kalıcı yapmak
channel.QueueDeclare(queue: "orders", durable: true, exclusive: false, autoDelete: false);

// Mesajı kalıcı yapmak
var properties = channel.CreateBasicProperties();
properties.Persistent = true;
channel.BasicPublish(exchange: "", routingKey: "orders", basicProperties: properties, body: body);
```

### ⚠️ Uyarı

Kalıcı olarak işaretlenmiş mesajlar **%100 garanti** vermez; fiziksel disk hatalarına karşı ek olarak **Outbox / Inbox pattern** önerilir.

---

---

## ⚖️ Fair Dispatch

RabbitMQ’da tüm consumer’lara eşit şekilde mesajları iletebilirsiniz.

Bu da kuyrukta bulunan mesajların, mümkün olan en adil şekilde dağıtımını sağlamak için kullanılan bir özelliktir.

Consumer’lara eşit şekilde mesajların iletilmesi sistemdeki performansı düzenli bir hale getirecektir.

Böylece bir consumer’ın diğer consumer’lardan daha fazla yük alması ve sistemdeki diğer consumer’ların kısmi aç kalması engellenmiş olur.

---

## ⚙️ Mesaj İşleme Konfigürasyonu

RabbitMQ’da `BasicQos` metodu ile mesajların işleme hızını ve teslimat sırasını belirleyebiliriz. Böylece `Fair Dispatch` özelliği konfigüre edilebilmektedir.

```csharp
channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);
```

Bir consumer tarafından alınabilecek en büyük mesaj boyutunu byte cinsinden belirler. `0` sınırsız demektir.

Bir consumer tarafından aynı anda işlenebilecek mesaj sayısını belirler.

Bu konfigürasyonun tüm consumerlar için mi yoksa sadece çağrı yapılan consumer için mi geçerli olacağını belirler.

```csharp
channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);
```

### 📘 Parametreler

| Parametre         | Açıklama                                                        |
| ----------------- | --------------------------------------------------------------- |
| **prefetchSize**  | Alınabilecek maksimum mesaj boyutu (0 = sınırsız).              |
| **prefetchCount** | Bir consumer’ın aynı anda işleyebileceği mesaj sayısı.          |
| **global**        | Ayar tüm consumer’lar için mi, sadece biri için mi uygulanacak. |

### 🧠 Mantık

Bir consumer, elindeki mesajı işlemeden yeni mesaj almaz. Böylece tüm consumer’lar dengeli çalışır.

## 🧩 Özet

| Konu                        | Amaç                                               |
| --------------------------- | -------------------------------------------------- |
| **Round Robin**             | Varsayılan sıralı mesaj dağıtımı.                  |
| **Message Acknowledgement** | Mesaj işlemenin garantisini sağlar.                |
| **Rejection**               | Mesajı bilinçli reddetme / yeniden kuyruğa ekleme. |
| **Durability**              | Kuyruk ve mesaj kalıcılığı.                        |
| **Fair Dispatch**           | Consumer’lar arasında adil yük dağıtımı.           |

---

📘 **Kaynak:**  
Bu notlar, **Gençay Yıldız** tarafından hazırlanmış aşağıdaki eğitim videosu temel alınarak derlenmiştir:  
🎥 [RabbitMQ Eğitimi] (https://www.youtube.com/watch?v=P-6O2w7m8Tw&list=PLQVXoXFVVtp2aVwD6GX2KCjcD3hSe6vWM&index=5)
