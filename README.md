# EntityFrameworkWithCSharp

> Entity Framework Core examples, explanations and real-world implementations using C# and .NET.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![C#](https://img.shields.io/badge/C%23-.NET-512BD4.svg)
![EF Core](https://img.shields.io/badge/EF-Core-512BD4.svg)
![Language](https://img.shields.io/badge/lang-T%C3%BCrk%C3%A7e-orange.svg)

---

## Hakkında

Bu repo, **Entity Framework Core**'u C# ile, sade ve örneğe dayalı bir dille adım adım anlatır. EF Core, C# nesneleri ile ilişkisel veritabanı arasındaki çeviriyi üstlenen bir **ORM** (Object-Relational Mapper)'dır.

Her konu kendi klasöründe, ayrı bir doküman olarak ele alınır ve aynı yapıyı izler. Örnekler tutarlı bir blog modeli (Blog, Post, Etiket) üzerinden ilerler.

---

## Konular

| Konu | Açıklama |
| ---- | -------- |
| [Introduction](./Introduction) | EF nedir, ORM mantığı, DbContext ve Entity'lerle temel yapı. |
| [Migrations](./Migrations) | Code First, migration'larla şema yönetimi ve data seeding. |
| [Relationships](./Relationships) | İlişkiler — one-to-one, one-to-many, many-to-many — ve ilişkisel veri çekme (loading). |
| [Querying](./Querying) | LINQ/IQueryable, Single/First/Find, veri manipülasyonu ve delete behavior. |
| [Advanced Mapping](./AdvancedMapping) | Owned types, ham SQL, stored procedure/function/view ve loglama. |
| [Performance](./Performance) | AsNoTracking, split query, compiled query, DbContext pooling. |

---

## Önerilen Okuma Sırası

1. **Introduction** — temel kavramlar
2. **Migrations** — şema yönetimi
3. **Relationships** — ilişkiler (en kritik konu)
4. **Querying** — sorgulama ve veri manipülasyonu
5. **Advanced Mapping** — owned types, ham SQL
6. **Performance** — ölçeklenme ve iyileştirme

---

## Her Doküman Neler İçeriyor?

Tüm konular tutarlı bir şablonla yazılmıştır:

* **Nedir ve neden gerekir**
* **Örnek kod** — gerçek bir model (Blog/Post) üzerinden
* **Karşılaştırma** — yakın seçeneklerle farkı (uygun olduğunda)
* **Önemli noktalar ve dikkat edilecekler**
* **Sonuç**

---

## Lisans

Bu proje [MIT Lisansı](./LICENSE) ile lisanslanmıştır.
