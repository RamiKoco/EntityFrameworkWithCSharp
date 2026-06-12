# Performans ve Önemli Noktalar

## Giriş

Küçük veriyle her şey hızlı görünür. Asıl fark, veri büyüdüğünde ve trafik arttığında ortaya çıkar. EF Core, doğru kullanıldığında hem hızlı hem ölçeklenebilir olabilir; ama bunun için birkaç tekniği bilmek gerekir.

Bu yazıda AsNoTracking, split query, compiled query ve DbContext pooling gibi performans tekniklerini ele alıyoruz.

---

## AsNoTracking

EF, getirdiği nesneleri **ChangeTracker** ile izler; böylece üzerinde yaptığın değişiklikleri `SaveChanges`'te tespit edebilir. Ama sadece okuma yapıyorsan bu izleme gereksiz bir yüktür.

`AsNoTracking()`, izlemeyi kapatır; salt-okunur sorgularda hem belleği hem süreyi azaltır:

```csharp
var postlar = context.Postlar.AsNoTracking().ToList();
```

Veriyi yalnızca gösterecek, değiştirmeyeceksen `AsNoTracking` neredeyse her zaman doğru tercihtir.

---

## Split Query

Birden çok `Include` içeren sorgular, tek bir JOIN'de **kartezyen çarpıma** (satır patlamasına) yol açabilir: ana kaydın her ilişkili satır için tekrar etmesi, hem ağ trafiğini hem belleği şişirir.

`AsSplitQuery()`, bunu ayrı sorgulara bölerek çözer:

```csharp
context.Bloglar
    .Include(b => b.Postlar)
    .Include(b => b.Editorler)
    .AsSplitQuery()
    .ToList();
```

Her ilişki ayrı bir sorguyla gelir; satır patlaması önlenir. (Tek sorgu mu split query mi daha iyi olduğu duruma bağlıdır; çok sayıda koleksiyon Include'unda split genelde kazandırır.)

---

## Compiled Query

Çok sık çalışan bir sorgu için, EF her seferinde sorguyu yeniden derler. Bu derleme maliyetini ortadan kaldırmak için sorguyu önceden derleyip tekrar kullanabilirsin:

```csharp
private static readonly Func<BlogDbContext, int, Blog> _blogGetir =
    EF.CompileQuery((BlogDbContext ctx, int id) =>
        ctx.Bloglar.First(b => b.Id == id));

// kullanımı
var blog = _blogGetir(context, 5);
```

Compiled query, aynı sorgunun milyonlarca kez çalıştığı sıcak yollarda (hot path) fark yaratır.

---

## DbContext Pooling

Her istekte yeni bir `DbContext` oluşturmak, yüksek trafikte ek bir maliyettir. **DbContext pooling**, hazır context nesnelerini bir havuzdan yeniden kullanır:

```csharp
services.AddDbContextPool<BlogDbContext>(opt => opt.UseSqlServer(connStr));
```

`AddDbContext` yerine `AddDbContextPool` kullanmak, context oluşturma maliyetini düşürür. Özellikle çok sayıda kısa ömürlü isteğin olduğu web uygulamalarında işe yarar.

---

## Common Configurations

Tekrarlayan eşleme kurallarını (örneğin tüm string'lerin maksimum uzunluğu, tüm tarihlerin tipi) her entity'de ayrı ayrı yazmak yerine tek bir yerde toplayabilirsin. Bunun için `ConfigureConventions` veya ortak bir temel konfigürasyon kullanılır. Böylece kurallar tutarlı kalır ve bakım kolaylaşır.

---

## Sonuç

EF Core'da performans, birkaç bilinçli tercihle gelir. `AsNoTracking` salt-okunur sorgularda gereksiz izlemeyi kaldırır. `AsSplitQuery`, çok Include'lu sorgularda satır patlamasını önler. Compiled query, sıcak yollarda sorgu derleme maliyetini ortadan kaldırır. DbContext pooling, yüksek trafikte context oluşturma yükünü azaltır.

Bu teknikler, EF'i küçük projelerde sade tutarken büyük uygulamalarda hızlı ve ölçeklenebilir kılan anahtarlardır. Hepsinin ortak fikri aynıdır: gereksiz işi yapmamak ve tekrar eden maliyeti bir kez ödemek.
