# Entity Framework Core'a Giriş

## Giriş

Bir uygulamada veriler nesnelerle (class) tutulur, ama veritabanı bunları tablolar ve satırlar olarak saklar. Bu iki dünya arasında sürekli çeviri yapmak (nesneyi SQL'e, satırı nesneye) hem yorucu hem hata açıktır.

**Entity Framework Core (EF Core)**, bu çeviriyi senin yerine yapan bir **ORM** (Object-Relational Mapper) aracıdır. C# sınıflarınla veritabanı tabloların arasında köprü kurar; sen LINQ ile nesnelerle çalışırsın, EF arkada SQL üretir.

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

## Sonuç

Entity Framework Core, C# nesneleri ile ilişkisel veritabanı arasındaki çeviriyi üstlenen bir ORM'dir. Code First yaklaşımıyla, önce modelini C# sınıflarıyla yazar; veritabanını EF senin yerine üretir.

Temel yapı taşları **DbContext** (veritabanı oturumu ve tablolara `DbSet` erişimi) ile **entity**'lerdir (tabloya eşlenen POCO sınıfları). EF'in convention'ları sayesinde çoğu eşleme otomatik olur — sonraki bölümlerde bu temelin üzerine migration, ilişkiler, sorgulama ve performansı ekleyeceğiz.
