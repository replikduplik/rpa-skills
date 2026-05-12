# Fork Farkları — Canlı Belge

Bu dosya, vanilla OpenRPA ile şirketin fork'u arasında keşfedilen farkları biriktirir.

**Güncelleme kuralı:** Yeni fark keşfedilince hemen ekle. Etkilenen skill veya note'u da güncelle.

---

## Format

```
## [Tarih] Alan / Aktivite / Özellik

**Vanilla:** [Nasıl çalışıyor / ne döndürüyor / ne gösteriyor]
**Fork:** [Farkı — davranış, parametre, UI, değer]
**Etki:** [Hangi workflow'ları / senaryoları etkiliyor]
**İlgili skill güncellendi mi:** Evet / Hayır / Gerek yok
**Not:** [Varsa ek açıklama]
```

---

## Bilinen Farklar

> Henüz sistematik karşılaştırma yapılmadı.
> İlk fark keşfedildiğinde bu satır silinecek ve kayıt eklenecek.

---

## Keşfedilmesi Gereken Alanlar

Aşağıdaki alanlarda fark olup olmadığı kontrol edilmelidir:

```
[ ] OpenApplication — Selector davranışı
[ ] GetElement — MinResults=0 ile null dönüşü
[ ] TypeText — Password=true log davranışı
[ ] InvokeCode — async/await desteği
[ ] BreakableLoop — Break aktivitesi varlığı
[ ] WorkItem — PopWorkitem ile null dönüşü
[ ] Credential — GetCredential API'si
[ ] Invoke-OpenRPA — array parametre desteği
[ ] Polisoft selector'ları — cls değerleri doğru mu?
[ ] Logging seviyeleri — Output paneli davranışı
```

---

## Güncelleme Protokolü

1. Çalışırken beklenmedik davranış görürsen:
   - Vanilla dokümanlarıyla karşılaştır
   - Farkı bu dosyaya ekle
   - Etkilenen skill/note'ta Fork Notları bölümüne not ekle
   - `openrpa-master/SKILL.md` routing'ini güncelle (gerekirse)

2. Claude sana şunu söylerse ve fork'ta farklı çalışıyorsa:
   - Fork farkı olarak kaydet
   - Claude'u düzelt: "Fork'ta bu farklı çalışıyor: ..."

---

## Örnek Kayıt (Şablon)

```
## 2026-05-13 Örnek: InvokeCode async davranışı

**Vanilla:** async/await tam destekleniyor
**Fork:** async/await destekleniyor — doğrulandı
**Etki:** Yok
**İlgili skill güncellendi mi:** Gerek yok
**Not:** openrpa-csharp skill async/await içeriyor, doğru
```
