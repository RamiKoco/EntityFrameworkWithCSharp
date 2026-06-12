# Sorgulama ve Veri Manipülasyonu

## Giriş

EF Core'da veriyi okumak ve değiştirmek LINQ ile yapılır. Ama bu basit görünen işin altında önemli ayrıntılar yatar: sorgunun ne zaman çalıştığı, tek kayıt almanın farklı yolları, değişikliklerin nasıl izlendiği ve silme davranışları.

Bu yazıda LINQ/IQueryable, Single/First/Find farkı, veri manipülasyonu ve delete behavior'ı ele alıyoruz.

---

## LINQ ve IQueryable

EF sorguları LINQ ile yazılır ama asıl güç **`IQueryable`**'dadır. `IQueryable`, sorguyu hemen çalıştırmaz; **ertelenmiş (deferred)** tutar. Sen filtre eklemeye devam ettikçe sorgu büyür, ancak `ToList()`, `First()`, `Count()` gibi bir metot çağrılınca SQL'e çevrilip çalışır.

Bu sayede **dinamik filtreler** kurmak mümkündür:

```csharp
IQueryable<Post> sorgu = context.Postlar;

if (sadeceYayinda)
    sorgu = sorgu.Where(p => p.Yayinda);

if (!string.IsNullOrEmpty(arama))
    sorgu = sorgu.Where(p => p.Baslik.Contains(arama));

var sonuc = sorgu.ToList();   // SQL ancak BURADA çalışır
```

Buradaki kritik fark: `IQueryable` filtrelerini veritabanında uygular; `IEnumerable`'a düşersen (`.ToList()` sonrası) filtreler artık bellekte çalışır. Bu yüzden filtrelemeyi `ToList()`'ten **önce** yapmak performans açısından önemlidir.

---

## Single, First ve Find Arasındaki Farklar

Tek bir kayıt almanın üç yolu vardır ve davranışları farklıdır:

| Metot       | Davranış                                        | Bulamazsa            |
| ----------- | ----------------------------------------------- | -------------------- |
| `Single`    | Tam olarak 1 kayıt bekler; 0 veya >1 → hata     | `SingleOrDefault` → null |
| `First`     | Eşleşenlerin ilkini alır (genelde `OrderBy` ile)| `FirstOrDefault` → null  |
| `Find`      | Birincil anahtarla arar; önce belleğe (ChangeTracker) bakar | null |

`Find`'ın özel yanı: aradığın kayıt zaten belleğe yüklenmişse, veritabanına hiç gitmeden onu döndürür. `Single` ise "bu koşula uyan tek kayıt olmalı" garantisi istediğinde kullanılır.

---

## Veri Manipülasyonu: Insert, Update, Delete

EF, değişiklikleri **ChangeTracker** ile izler. Sen nesneleri değiştirirsin, EF neyin eklendiğini/değiştiğini/silindiğini takip eder ve `SaveChanges()` çağrısında hepsini tek seferde SQL'e çevirir.

```csharp
// Insert
context.Postlar.Add(new Post { Baslik = "Merhaba" });

// Update
var post = context.Postlar.First();
post.Baslik = "Güncellendi";   // ChangeTracker bu değişikliği fark eder

// Delete
context.Postlar.Remove(post);

context.SaveChanges();   // hepsi tek transaction'da işlenir
```

Sadece okuma yapıyorsan ve değişiklik izlemeye ihtiyacın yoksa, `AsNoTracking()` ile ChangeTracker'ı devre dışı bırakmak sorguyu hızlandırır:

```csharp
var postlar = context.Postlar.AsNoTracking().ToList();
```

---

## Delete Behavior — Hiyerarşik Silme

Bir kayıt silindiğinde, ona bağlı kayıtlara ne olacağını **delete behavior** belirler:

* **Cascade** — ana kayıt silinince bağlı kayıtlar da silinir (Blog silinince Post'ları da silinir)
* **Restrict / NoAction** — bağlı kayıt varsa silmeyi engeller
* **SetNull** — bağlı kayıtların yabancı anahtarı `null` yapılır

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Postlar)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.Restrict);   // Post'u olan Blog silinemez
```

Doğru davranışı seçmek önemlidir; yanlış bir `Cascade`, beklenmedik toplu silmelere yol açabilir.

---

## Sonuç

EF Core'da sorgulama LINQ ile yazılır ama gücü `IQueryable`'ın ertelenmiş yapısındadır: filtreleri veritabanında uygular ve dinamik sorgu kurmaya izin verir — yeter ki `ToList()`'ten önce filtrelensin.

Tek kayıt alırken `Single` (tek olmalı garantisi), `First` (ilkini al) ve `Find` (PK ile, önce bellekten) farklı amaçlara hizmet eder. Veri değişiklikleri ChangeTracker ile izlenir ve `SaveChanges()` ile tek transaction'da işlenir. Delete behavior ise bağlı kayıtların silme sırasındaki kaderini belirler — `Cascade`'i bilinçli seçmek gerekir.
