# Entity Framework Core Nedir?

## Giriş

Bir uygulamada veriler nesnelerle (class) tutulur, ama veritabanı bunları tablolar ve satırlar olarak saklar. Bu iki dünya arasında sürekli çeviri yapmak (nesneyi SQL'e, satırı nesneye) hem yorucu hem hata açıktır.

**Entity Framework Core (EF Core)**, bu çeviriyi senin yerine yapan bir **ORM** (Object-Relational Mapper) aracıdır. C# sınıflarınla veritabanı tabloların arasında köprü kurar; sen LINQ ile nesnelerle çalışırsın, EF arkada SQL üretir.

Bu makale, EF Core'un temel kavramlarını — DbContext, migration, ilişkiler, sorgulama, veri manipülasyonu ve performans — tek bir akışta ele alır.

---

## Entity Framework Nedir?

EF Core bir **ORM**'dir: nesne (object) ile ilişkisel veritabanı (relational database) arasındaki eşlemeyi yapar.

* C# sınıfların → veritabanı **tabloları**
* Sınıfın property'leri → tablonun **sütunları**
* Nesne örnekleri → tablonun **satırları**

Sen `context.Postlar.Where(p => p.Yayinda)` yazarsın; EF bunu `SELECT ... FROM Posts WHERE ...` SQL'ine çevirir. Avantajı: SQL'i elle yazmadan, tip güvenli (type-safe) ve okunur kod yazmak.

EF Core genelde **Code First** yaklaşımıyla kullanılır: önce C# sınıflarını yazarsın, veritabanını EF senin modelinden üretir.

---

## Temel Yapı: DbContext ve Entity

Her şeyin merkezinde **DbContext** vardır. Veritabanıyla olan oturumunu temsil eder ve tablolarını `DbSet` olarak açar:

```csharp
public class BlogDbContext : DbContext
{
    public DbSet<Blog> Bloglar { get; set; }
    public DbSet<Post> Postlar { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer("...connection string...");
}
```

Entity'ler ise sıradan C# sınıflarıdır (POCO):

```csharp
public class Blog
{
    public int Id { get; set; }       // birincil anahtar (convention: Id veya BlogId)
    public string Ad { get; set; }
}
```

EF, isimlendirme kurallarıyla (convention) çoğu şeyi otomatik anlar: `Id` adlı property birincil anahtar olur, `DbSet<Blog>` `Blogs` tablosuna eşlenir.

---

## Code First ve Migration'lar

Modelini değiştirdiğinde (yeni sınıf, yeni property), veritabanının da bu değişikliğe uyması gerekir. Bunu **migration**'lar yönetir. Migration, modelin ile veritabanı şeması arasındaki farkı tutan versiyonlanmış bir adımdır.

```bash
# Yeni bir migration üret
dotnet ef migrations add IlkOlusturma

# Migration'ı veritabanına uygula
dotnet ef database update
```

`Add` komutu modeline bakıp gerekli `CREATE TABLE` / `ALTER TABLE` adımlarını üretir; `update` ise bunları veritabanına işler. Böylece şema değişikliklerin kod gibi versiyonlanır ve takip edilebilir.

> Not: `EnsureCreated()` ile `Migrate()` farklıdır. `EnsureCreated` şemayı migration'ları atlayarak doğrudan kurar (hızlı prototip için); `Migrate` ise migration'ları uygular. İkisini bir arada kullanmak önerilmez.

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

Bu veriler bir migration'a dahil edilir ve veritabanı oluşturulurken eklenir.

---

## İlişkiler (Relationships)

Gerçek modellerde tablolar birbirine bağlıdır. EF Core üç temel ilişki türünü destekler. İlişkiler, **navigation property**'ler (bir nesnenin diğerine referansı) ve **foreign key** (yabancı anahtar) ile kurulur.

### Bir-e-Çok (One-to-Many)

En yaygın ilişki. Bir `Blog`'un çok `Post`'u vardır; bir `Post` tek bir `Blog`'a aittir.

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Ad { get; set; }
    public List<Post> Postlar { get; set; } = new();   // "çok" tarafı
}

public class Post
{
    public int Id { get; set; }
    public string Baslik { get; set; }

    public int BlogId { get; set; }     // yabancı anahtar (foreign key)
    public Blog Blog { get; set; }      // "tek" tarafı
}
```

EF bunu `BlogId` adından otomatik anlar. İstersen Fluent API ile açıkça da tanımlayabilirsin:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Postlar)
    .WithOne(p => p.Blog)
    .HasForeignKey(p => p.BlogId);
```

### Bir-e-Bir (One-to-One)

Bir `Post`'un tam olarak bir `PostDetay` kaydı vardır.

```csharp
public class Post
{
    public int Id { get; set; }
    public PostDetay Detay { get; set; }
}

public class PostDetay
{
    public int Id { get; set; }
    public string Ozet { get; set; }

    public int PostId { get; set; }     // FK, aynı zamanda benzersiz (unique)
    public Post Post { get; set; }
}
```

Bir-e-bir'de yabancı anahtarın hangi tarafta olduğunu EF'e söylemek gerekir:

```csharp
modelBuilder.Entity<Post>()
    .HasOne(p => p.Detay)
    .WithOne(d => d.Post)
    .HasForeignKey<PostDetay>(d => d.PostId);
```

### Çok-a-Çok (Many-to-Many)

Bir `Post`'un çok `Etiket`'i, bir `Etiket`'in de çok `Post`'u olabilir.

```csharp
public class Post
{
    public int Id { get; set; }
    public List<Etiket> Etiketler { get; set; } = new();
}

public class Etiket
{
    public int Id { get; set; }
    public string Ad { get; set; }
    public List<Post> Postlar { get; set; } = new();
}
```

EF Core 5 ve sonrasında, bu iki tarafı bağlayan **ara tabloyu (join table)** EF otomatik oluşturur — sen ayrı bir `PostEtiket` sınıfı yazmak zorunda kalmazsın. İstersen yine Fluent API ile özelleştirebilirsin:

```csharp
modelBuilder.Entity<Post>()
    .HasMany(p => p.Etiketler)
    .WithMany(e => e.Postlar);
```

Özet:

| İlişki        | Örnek              | Yabancı anahtar nerede     |
| ------------- | ------------------ | -------------------------- |
| One-to-Many   | Blog → Post        | "Çok" tarafında (Post)     |
| One-to-One    | Post ↔ PostDetay   | Bağımlı tarafta (PostDetay)|
| Many-to-Many  | Post ↔ Etiket      | Otomatik ara tabloda       |

---

## İlişkisel Veri Çekme: Loading

İlişkili veriyi yüklemenin üç yolu vardır.

**Eager Loading** — ilişkili veriyi aynı sorguda `Include` ile getir:

```csharp
var bloglar = context.Bloglar
    .Include(b => b.Postlar)
        .ThenInclude(p => p.Etiketler)   // iç içe ilişki
    .ToList();
```

**Lazy Loading** — navigation property'ye erişildiği an EF arka planda otomatik sorgu atar (proxy gerektirir, `virtual` navigation):

```csharp
var blog = context.Bloglar.First();
var postlar = blog.Postlar;   // bu satırda EF yeni sorgu atar
```

**Explicit Loading** — ilişkili veriyi sonradan, elle yükle:

```csharp
context.Entry(blog).Collection(b => b.Postlar).Load();
```

Eager loading genelde en öngörülebilir olanıdır; lazy loading rahattır ama farkında olmadan çok sayıda sorgu (N+1 problemi) doğurabilir.

---

## Sorgulama: LINQ ve IQueryable

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

## Owned Types — Sahiplik İlişkisi

Bazı sınıfların kendi kimliği (birincil anahtarı) yoktur; başka bir entity'nin parçasıdır. Buna **owned type** denir. Örneğin bir `Adres`, bir `Kullanici`'ye aittir ve tek başına anlam taşımaz:

```csharp
modelBuilder.Entity<Kullanici>().OwnsOne(k => k.Adres);
```

Owned type'lar varsayılan olarak sahibinin tablosunda ek sütunlar olarak saklanır; `ToJson()` ile JSON kolon olarak da tutulabilir.

> Bu kavram tanıdık gelebilir: senin `SqlServerCache` projende `CacheItem`, `OwnsOne(...).ToJson()` ile tam olarak böyle bir owned type olarak saklanıyor.

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

## Ham SQL, Stored Procedure, Fonksiyon ve View

EF her zaman yeterli olmaz; bazen doğrudan SQL gerekir.

**Ham SQL sorgusu:**

```csharp
var postlar = context.Postlar
    .FromSqlInterpolated($"SELECT * FROM Posts WHERE BlogId = {blogId}")
    .ToList();
```

**Stored procedure / komut çalıştırma:**

```csharp
context.Database.ExecuteSqlInterpolated($"EXEC EskiPostlariTemizle {gun}");
```

**Veritabanı fonksiyonları** `HasDbFunction` ile LINQ'e tanıtılabilir; **view'ler** ise anahtarsız (keyless) entity olarak `ToView(...)` ile eşlenip salt-okunur sorgulanabilir. Böylece veritabanı tarafındaki mantığı da EF üzerinden kullanabilirsin.

---

## Performans ve Önemli Noktalar

EF Core'da büyük veriyle çalışırken birkaç teknik fark yaratır:

* **AsNoTracking** — salt-okunur sorgularda ChangeTracker'ı kapatır, belleği ve süreyi azaltır.

* **Split Query** — birden çok `Include` içeren sorgular, tek bir JOIN'de kartezyen çarpıma (satır patlamasına) yol açabilir. `AsSplitQuery()` bunu ayrı sorgulara böler:

  ```csharp
  context.Bloglar.Include(b => b.Postlar).AsSplitQuery().ToList();
  ```

* **Compiled Query** — çok sık çalışan bir sorguyu önceden derleyip tekrar tekrar kullanmak (`EF.CompileQuery`), sorgu oluşturma maliyetini ortadan kaldırır.

* **DbContext Pooling** — her istekte yeni `DbContext` oluşturmak yerine, hazır context'leri bir havuzdan yeniden kullanmak. Yüksek trafikte nesne oluşturma maliyetini düşürür:

  ```csharp
  services.AddDbContextPool<BlogDbContext>(opt => opt.UseSqlServer(connStr));
  ```

* **Common Configurations** — tekrarlayan eşleme kurallarını (örneğin tüm string'lerin maksimum uzunluğu) tek yerde toplamak; `ConfigureConventions` veya ortak bir base konfigürasyon ile.

---

## Sonuç

Entity Framework Core, C# nesneleri ile ilişkisel veritabanı arasındaki çeviriyi üstlenen bir ORM'dir. Code First ve migration'larla şemanı koddan yönetir; DbContext ve ChangeTracker ile veriyi izler; LINQ ve `IQueryable` ile tip güvenli, ertelenmiş sorgular yazmanı sağlar.

İlişkiler (one-to-one, one-to-many, many-to-many) navigation property ve foreign key ile kurulur; veriyi `Include` ile eager, lazy ya da explicit olarak yükleyebilirsin. Owned types, ham SQL, stored procedure ve view desteğiyle veritabanı tarafına da uzanır.

Büyük uygulamalarda ise asıl fark performans tarafında çıkar: `AsNoTracking`, split query, compiled query ve DbContext pooling, EF'i hızlı ve ölçeklenebilir tutmanın anahtarlarıdır. EF Core'u güçlü kılan, bütün bunları tek bir tutarlı modelin arkasında sunmasıdır.
