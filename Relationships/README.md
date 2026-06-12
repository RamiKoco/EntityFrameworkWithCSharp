# İlişkiler (Relationships) ve Loading

## Giriş

Gerçek modellerde tablolar birbirine bağlıdır. EF Core üç temel ilişki türünü destekler: one-to-one, one-to-many ve many-to-many. İlişkiler, **navigation property**'ler (bir nesnenin diğerine referansı) ve **foreign key** (yabancı anahtar) ile kurulur.

Bu yazıda üç ilişki türünü ve ilişkili veriyi yükleme (loading) yollarını ele alıyoruz. Örnekler bir blog modeli üzerinden ilerler.

---

## Bir-e-Çok (One-to-Many)

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

---

## Bir-e-Bir (One-to-One)

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

---

## Çok-a-Çok (Many-to-Many)

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

---

## İlişki Türleri Özeti

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

## Sonuç

EF Core'da ilişkiler navigation property ve foreign key ile kurulur. One-to-many "çok" tarafında FK taşır; one-to-one FK'nın hangi tarafta olduğunu açıkça belirtmeni ister; many-to-many ise EF Core 5+ ile ara tabloyu otomatik oluşturur.

İlişkili veriyi yüklerken üç seçeneğin var: eager (`Include`), lazy (erişince otomatik) ve explicit (`Load`). Eager loading en öngörülebilir yoldur; lazy loading rahatlık sunar ama N+1 sorgu tuzağına dikkat etmek gerekir.
