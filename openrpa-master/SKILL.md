---
name: openrpa-master
description: >
  Use FIRST for any OpenRPA question, error, workflow design, code review, or
  automation task. Routes to the correct skill or note. Triggers on everything
  RPA-related: "robot", "workflow", "otomasyon", "hata", "element", "selector",
  "Polisoft", "SAP", "InvokeCode", "C#", "queue", "invoke", "sigorta", "fork",
  "aktivite", "script", "review", "debug", "şablon", "not ekle", "skill güncelle".
---

# OpenRPA Master — Ekosistem Rehberi

Her OpenRPA sorusunda **önce bu skill**. Prompt'u analiz et → doğru kaynağa git.

---

## Ekosistem Haritası

### Skill'ler

| Skill | Ne Zaman |
|---|---|
| `openrpa-workflow` | Workflow tasarımı, aktivite seçimi, SAP/Office/mail/Desktop best practice |
| `openrpa-selector` | Selector yazma, debug, UI element bulma |
| `openrpa-invoke` | PowerShell Invoke-OpenRPA, InvokeWorkflow, OpenFlow API |
| `openrpa-openflow` | WorkItem queue, çok robot, zamanlama, orkestrasyon |
| `openrpa-debug` | Hata mesajı teşhis, log analizi, element not found |
| `openrpa-review` | Yazılmış workflow'u incele, best practice kontrol |
| `openrpa-csharp` | InvokeCode için C# kod blokları |
| `openrpa-polisoft-deep` | Polisoft tüm modülleri (teklif, poliçe, hasar, zeyilname) |

### Notes (Aktivite Rehberleri)

| Dosya | Aktivite |
|---|---|
| `notes/GetElement.md` | Element bulma, timeout, MinResults/MaxResults |
| `notes/TypeText.md` | Klavye girdisi, özel tuşlar |
| `notes/ClickElement.md` | Tıklama, VirtualClick vs gerçek |
| `notes/InvokeCode.md` | C# çalıştırma, değişken yönleri |
| `notes/TryCatch.md` | Hata yakalama, Finally, rethrow |
| `notes/OpenApplication.md` | Uygulama açma/odaklama |
| `notes/InvokeWorkflow.md` | Alt workflow çağırma, parametre |
| `notes/Delay.md` | Bekleme — ne zaman kullanılır/kullanılmaz |
| `notes/ForEach-BreakableLoop.md` | Döngü tipleri, Break, bellek |

### Canlı Belgeler

| Dosya | Amaç |
|---|---|
| `fork-differences.md` | Şirket fork'u ile vanilla OpenRPA farkları |
| `templates/README.md` | Workflow şablonları — ekleme rehberi |

---

## Routing Kuralları

```
Prompt içeriyorsa               → Git buraya
─────────────────────────────────────────────────────
"element not found", "timeout"  → openrpa-debug + notes/GetElement.md
"selector", "spy", "highlight"  → openrpa-selector + notes/GetElement.md
"InvokeCode", "C# kodu"         → openrpa-csharp + notes/InvokeCode.md
"Polisoft", "teklif", "hasar"   → openrpa-polisoft-deep + openrpa-workflow
"queue", "workitem", "kuyruk"   → openrpa-openflow
"invoke", "PowerShell", "tetikle" → openrpa-invoke + notes/InvokeWorkflow.md
"workflow doğru mu", "review"   → openrpa-review
"hangi aktivite", "nasıl yapılır" → openrpa-workflow + ilgili notes/
"TypeText", "yazmıyor"          → notes/TypeText.md
"Click", "tıklamıyor"           → notes/ClickElement.md
"TryCatch", "hata yakalama"     → notes/TryCatch.md
"Delay", "bekleme"              → notes/Delay.md
"ForEach", "döngü", "loop"      → notes/ForEach-BreakableLoop.md
"OpenApplication", "pencere"    → notes/OpenApplication.md
"fork farkı keşfettim"          → fork-differences.md güncelle
"yeni aktivite notu"            → notes/ altına yeni .md ekle
"yeni şablon"                   → templates/ altına .md ekle
```

---

## Okunma Önceliği

```
1. openrpa-master          ← her zaman önce (zaten buradasın)
2. İlgili skill             ← routing tablosuna göre
3. İlgili notes/ dosyası   ← aktivite spesifik detay için
4. fork-differences.md     ← fork'a özel notlar için
```

Tek bir soruda birden fazla skill gerekebilir — ikisini de oku.

---

## Yeni İçerik Ekleme Kuralları

### Ne Zaman Yeni Skill?
- Yeni bir **konu alanı** ortaya çıktığında (yeni uygulama, yeni entegrasyon)
- Mevcut skill 150+ satırı geçip farklı bir konuyu kapsıyorsa (böl)
- Sigorta sektörüne yeni bir platform eklendiğinde (Axa portal, SAP modülü vb.)

**Nereye:** Ana dizine `openrpa-[konu]/SKILL.md` olarak ekle.
**Sonra:** Bu dosyadaki Ekosistem Haritası tablosuna ekle.

### Ne Zaman Yeni Note?
- Yeni bir **OpenRPA aktivitesi** keşfedildiğinde
- Mevcut bir aktivite için yeterli detay yoksa

**Nereye:** `notes/[AktiviteAdı].md` olarak ekle.
**Format:** Her note aynı şablonu izler (Ne Yapar / Parametreler / Hatalar / Örnekler / Fork Notları).
**Sonra:** Bu dosyadaki Notes tablosuna ekle.

### Ne Zaman Template?
- Tekrarlayan bir **iş süreci** için hazır akış gerektiğinde
- Birden fazla kez kullanılacak workflow kalıbı netleştiğinde

**Nereye:** `templates/[surecadi].md` olarak ekle.
**Format:** `templates/README.md`'deki şablonu izle.

### Fork Farkı Bulunca
1. `fork-differences.md` dosyasına ekle (format için o dosyaya bak)
2. Etkilenen skill veya note'u güncelle
3. Bu dosyadaki routing kuralları etkileniyorsa güncelle

---

## Hızlı Başvuru — Sigorta Sektörü

| Senaryo | İlk Git |
|---|---|
| Polisoft teklif girişi | `openrpa-polisoft-deep` |
| Web portaldan prim alma | `openrpa-workflow` → Web Portalleri bölümü |
| Toplu poliçe yenileme | `openrpa-openflow` + `openrpa-invoke` |
| Hasar bildirimi kaydı | `openrpa-polisoft-deep` |
| Excel'den kuyruğa yükle | `openrpa-openflow` + `openrpa-csharp` |
| Robot hata verdi | `openrpa-debug` |
| Workflow güvenli mi? | `openrpa-review` |
