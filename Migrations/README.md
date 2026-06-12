# Code First ve Migration'lar

## Giriş

Code First yaklaşımında veritabanını elle kurmazsın; modelini C# sınıflarıyla yazarsın ve veritabanını EF senin modelinden üretir. Ama model zamanla değişir — yeni sınıf, yeni property. Veritabanının da bu değişikliklere uyması gerekir.

İşte **migration**'lar bu işi yönetir: modelin ile veritabanı şeması arasındaki farkı versiyonlanmış adımlar hâlinde tutar.

---

## Migration Nedir?

Migration, modelin ile veritabanı şeması arasındaki farkı tutan, versiyonlanmış bir adımdır. Modelini değiştirdiğinde bir migration üretir, sonra bunu veritabanına uygularsın:

```bash
# Yeni bir migration üret
dotnet ef migrations add IlkOlusturma

# Migration'ı veritabanına uygula
dotnet ef database update
```

`add` komutu modeline bakıp gerekli `CREATE TABLE` / `ALTER TABLE` adımlarını üretir; `update` ise bunları veritabanına işler. Böylece şema değişikliklerin tıpkı kod gibi versiyonlanır, takip edilebilir ve geri alınabilir hâle gelir.

---

## EnsureCreated ve Migrate Farkı

İkisi de veritabanını kurar ama farklı çalışır:

* **`EnsureCreated()`** — şemayı, migration'ları atlayarak doğrudan oluşturur. Hızlı prototip veya test için uygundur; migration geçmişi tutmaz.
* **`Migrate()`** — migration'ları uygular. Üretim ortamında ve şema evrildikçe doğru olan budur.

> Önemli: ikisini bir arada kullanmak önerilmez. `EnsureCreated` şemayı migration geçmişi olmadan kurduğu için, ardından gelen `Migrate` çakışabilir. Genelde sadece `Migrate` tercih edilir.

---

## Data Seeding — Başlangıç Verisi

Veritabanı ilk kurulduğunda içinde bulunması gereken sabit veriler (ülkeler, roller, varsayılan kayıtlar) **seeding** ile tanımlanır. `OnModelCreating` içinde `HasData` kullanılır:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>().HasData(
        new Blog { Id = 1, Ad = "Teknoloji" },
        new Blog { Id = 2, Ad = "Seyahat" }
    );
}
```

Bu veriler bir migration'a dahil edilir ve veritabanı oluşturulurken/güncellenirken eklenir. Böylece başlangıç verisi de şemanın bir parçası olarak versiyonlanır.

---

## Sonuç

Migration'lar, Code First yaklaşımında veritabanı şemasını koddan yönetmenin yoludur: modelini değiştirir, migration üretir, uygularsın. Bu sayede şema değişiklikleri versiyonlanır ve takip edilebilir kalır.

`EnsureCreated` hızlı prototip için, `Migrate` ise gerçek şema yönetimi için kullanılır — ikisini karıştırmamak gerekir. Data seeding ile de başlangıç verisini şemanın bir parçası hâline getirirsin; böylece veritabanı her kurulduğunda tutarlı bir başlangıç noktasına sahip olur.
