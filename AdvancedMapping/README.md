# Owned Types, Ham SQL ve Loglama

## Giriş

EF Core'un temel eşlemesi çoğu durumu karşılar, ama bazen daha ileri tekniklere ihtiyaç olur: kimliği olmayan değer nesnelerini saklamak, doğrudan SQL çalıştırmak, veritabanı tarafındaki prosedür ve view'leri kullanmak, ve EF'in ürettiği SQL'i görmek.

Bu yazıda owned types, ham SQL, stored procedure/function/view kullanımı ve loglamayı ele alıyoruz.

---

## Owned Types — Sahiplik İlişkisi

Bazı sınıfların kendi kimliği (birincil anahtarı) yoktur; başka bir entity'nin parçasıdır. Buna **owned type** denir. Örneğin bir `Adres`, bir `Kullanici`'ye aittir ve tek başına anlam taşımaz:

```csharp
modelBuilder.Entity<Kullanici>().OwnsOne(k => k.Adres);
```

Owned type'lar varsayılan olarak sahibinin tablosunda ek sütunlar olarak saklanır; `ToJson()` ile JSON kolon olarak da tutulabilir:

```csharp
modelBuilder.Entity<Kullanici>().OwnsOne(k => k.Adres, b => b.ToJson());
```

Owned types, DDD'deki **value object**'leri saklamanın doğal yoludur — kimliği olmayan, sahibine bağlı kavramlar.

---

## Ham SQL Sorguları

EF her zaman yeterli olmaz; bazen doğrudan SQL gerekir. Parametreleri güvenli biçimde geçirmek için interpolated sürümler kullanılır:

```csharp
var postlar = context.Postlar
    .FromSqlInterpolated($"SELECT * FROM Posts WHERE BlogId = {blogId}")
    .ToList();
```

Sorgu döndürmeyen komutlar için `ExecuteSql` kullanılır:

```csharp
context.Database.ExecuteSqlInterpolated($"EXEC EskiPostlariTemizle {gun}");
```

> Not: Interpolated sürümler parametreleri otomatik olarak SQL parametresine çevirir; string birleştirme yerine bunları kullanmak SQL injection'a karşı korur.

---

## Stored Procedure, Fonksiyon ve View

EF, veritabanı tarafındaki mantığı da kullanabilir:

* **Stored procedure** — `FromSql`/`ExecuteSql` ile çağrılır
* **Veritabanı fonksiyonları** — `HasDbFunction` ile LINQ'e tanıtılır, böylece sorgu içinde kullanılabilir
* **View'ler** — anahtarsız (keyless) entity olarak `ToView(...)` ile eşlenir ve salt-okunur sorgulanır

Böylece veritabanı tarafındaki hazır mantığı, yine EF üzerinden tip güvenli biçimde kullanabilirsin.

---

## Loglama

EF'in ürettiği SQL'i görmek hata ayıklamada çok işe yarar. `LogTo` ile sorgular konsola basılabilir:

```csharp
options.UseSqlServer(connStr)
       .LogTo(Console.WriteLine)
       .EnableSensitiveDataLogging();   // parametre değerlerini de göster (sadece DEBUG)
```

`EnableSensitiveDataLogging`, sorgu parametrelerinin gerçek değerlerini de gösterir; hassas veri içerdiği için yalnızca geliştirme ortamında açılmalıdır.

---

## Sonuç

EF Core, temel eşlemenin ötesine geçen ileri tekniklere de imkan tanır. Owned types, kimliği olmayan değer nesnelerini (value object) sahibinin içinde saklar. Ham SQL, stored procedure, fonksiyon ve view desteği ise veritabanı tarafındaki mantığı EF üzerinden kullanmanı sağlar.

Loglama (`LogTo`) ile de EF'in ürettiği SQL'i görebilir, performans ve doğruluk sorunlarını erken yakalayabilirsin — `EnableSensitiveDataLogging`'i yalnızca geliştirme ortamında açmak kaydıyla.
