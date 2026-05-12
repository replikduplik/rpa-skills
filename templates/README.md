# Workflow Şablonları

Bu klasör, tekrarlayan iş süreçleri için hazır OpenRPA workflow kalıplarını içerir.

---

## Amaç

Bir iş süreci yeterince netleştiğinde ve birden fazla kez kullanılacağı anlaşıldığında, o sürece ait adım adım workflow şablonu buraya eklenir. Şablonlar aktivite listeleri, parametre tanımları ve kritik notlardan oluşur.

---

## Ne Zaman Şablon Eklenir?

- Aynı süreç 2+ farklı iş vakasında tekrar ediyor
- Bir workflow yapısı "standart" haline geliyor
- Yeni takım üyeleri için öğrenme kaynağı olacak

**Henüz netleşmemiş süreçler için şablon yazmak erkendir — gerçek case'ler ortaya çıktıkça ekle.**

---

## Şablon Formatı

Her şablon dosyası şu bölümleri içermelidir:

```markdown
# [Süreç Adı] Şablonu

## Ne Zaman Kullanılır
[Bu şablonun hangi iş vakası için olduğu]

## Ön Koşullar
- Hangi uygulamalar açık olmalı
- Hangi veriler hazır olmalı
- OpenFlow bağlantısı gerekiyor mu?

## Workflow Yapısı
[Aktivite listesi — girintili pseudo-XAML]

## Parametreler
| Parametre | Tip | Açıklama |
|---|---|---|

## Kritik Notlar
[Dikkat edilmesi gereken özel durumlar]

## Bağlantılı Skill/Not
[Hangi skill veya notes/ dosyasına başvurulmalı]
```

---

## Planlanan Şablonlar

Aşağıdaki süreçler için şablon eklenmesi düşünülmektedir. Case'ler netleştikçe güncellenecektir:

| Süreç | Durum |
|---|---|
| Poliçe yenileme batch'i | Beklemede |
| Web portaldan toplu teklif alma | Beklemede |
| Hasar bildirimi kaydı | Beklemede |
| Excel → WorkItem kuyruğu yükleme | Beklemede |
| Günlük rapor oluşturma | Beklemede |

---

## Şablon Ekleme Adımları

1. `templates/[surecadi].md` olarak yeni dosya oluştur
2. Yukarıdaki formatı uygula
3. `openrpa-master/SKILL.md` → Canlı Belgeler tablosuna ekle
4. İlgili skill veya note'u güncelle (referans ver)
