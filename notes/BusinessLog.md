# Business Log — HyperNodeInsertBusinessLog + HyperNodeUpdateBusinessLog

## Temel Kural: Her Insert'in Bir Update Karşılığı Olmalı

`HyperNodeInsertBusinessLog` ile açılan her kayıt, süreç bittiğinde mutlaka
`HyperNodeUpdateBusinessLog` ile kapatılmalıdır. Aksi hâlde kayıt sonsuza kadar
**"InProcess"** durumda kalır — izleme panellerinde süreç askıda görünür.

```
❌ YANLIŞ — Insert var ama Update yok
HyperNodeInsertBusinessLog (EntryState: "0" = InProcess)
[... işlemler ...]
// Süreç bitiyor ama kayıt hâlâ "InProcess"

✅ DOĞRU — Insert → işlemler → Update
HyperNodeInsertBusinessLog
  Identifier: "ana_islem"
  EntryState: "0"   (InProcess)

[... işlemler ...]

HyperNodeUpdateBusinessLog
  Identifier: "ana_islem"   ← Insert ile AYNI Identifier
  EntryState: "1"            (Succeeded)
```

---

## Zorunlu Şablon — İşlem Başı ve Sonu

Her kritik iş adımı için Insert + Update çifti kullan:

```
// === SÜREÇ BAŞLANGICI ===
HyperNodeInsertBusinessLog
  Identifier: "surec_ana"           ← bu Identifier süreç boyunca benzersiz olmalı
  EntryState: "0"                   ← InProcess
  Role: "0"                         ← Parent (yeni groupId oluşturur)
  GroupId: "police_grubu"
  Subject: GetTextResource("surec_basladi")
  Values:
    workitem_id: item.name
    islem_tarihi: DateTime.Now.ToString("yyyy-MM-dd HH:mm")

TryCatch
├── Try
│   ├── [iş adımları]
│   │
│   └── HyperNodeUpdateBusinessLog
│         Identifier: "surec_ana"   ← Insert ile aynı
│         EntryState: "1"           ← Succeeded
│         Values:
│           sonuc: "Başarılı"
│           bitis_tarihi: DateTime.Now.ToString("yyyy-MM-dd HH:mm")
│
└── Catch (Exception e)
    ├── CreateYouTrackIssue          ← [KZ-12] zorunlu
    └── HyperNodeUpdateBusinessLog
          Identifier: "surec_ana"   ← Insert ile aynı
          EntryState: "2"           ← Failed
          Values:
            hata_mesaji: e.Message
```

---

## Birden Fazla Insert Kullanıyorsan — Her Biri Farklı Identifier

Aynı `Identifier` değeriyle iki farklı Insert çağrılırsa, ikincisi birincinin
ConcurrentDictionary kaydını üzerine yazar. İlk kayıt artık güncellenemez.

```
❌ YANLIŞ — aynı Identifier iki kez
HyperNodeInsertBusinessLog Identifier: "[str_LogID]"  ← 1. kayıt
[...]
HyperNodeInsertBusinessLog Identifier: "[str_LogID]"  ← 2. kayıt, 1.yi ezer!

Sonuç: HyperNodeUpdateBusinessLog Identifier: "[str_LogID]" çağrıldığında
       sadece en son Insert'ın GUID'ine erişilir, ilk kayıt kaybolur.

✅ DOĞRU — her Insert için benzersiz Identifier
Assign: str_LogID_Ana    = Guid.NewGuid().ToString()
Assign: str_LogID_PdfDon = Guid.NewGuid().ToString()

HyperNodeInsertBusinessLog Identifier: "[str_LogID_Ana]"    ← ana süreç
[...]
HyperNodeInsertBusinessLog Identifier: "[str_LogID_PdfDon]" ← PDF döngüsü
[...]
HyperNodeUpdateBusinessLog Identifier: "[str_LogID_PdfDon]" ← PDF kaydını güncelle
[...]
HyperNodeUpdateBusinessLog Identifier: "[str_LogID_Ana]"    ← ana kaydı güncelle
```

---

## Role Parametresi — Parent / Child Hiyerarşisi

| Role | Değer | Ne Yapar |
|---|---|---|
| Parent | `"0"` | Yeni bir `groupId` GUID'i oluşturur ve saklar |
| Child | `"1"` | Mevcut `groupId`'yi ConcurrentDictionary'den arar; bulamazsa exception fırlatır |

```
// Ana süreç kaydı (Parent)
HyperNodeInsertBusinessLog
  Role: "0"
  GroupId: "makbuz_grubu"    ← groupId bu isimle saklanır

// Alt adım kaydı (Child) — aynı grup altında
HyperNodeInsertBusinessLog
  Role: "1"
  GroupId: "makbuz_grubu"    ← aynı grupId'yi arar; bulamazsa hata!
```

**Önemli:** Child Insert çağrılmadan önce aynı GroupId ile Parent Insert çalışmış olmalı.

---

## EntryState Değerleri

| Değer | Anlamı | Ne Zaman |
|---|---|---|
| `"0"` | InProcess | İşlem başladı, devam ediyor |
| `"1"` | Succeeded | İşlem başarıyla tamamlandı |
| `"2"` | Failed | İşlem hata ile bitti |

---

## Key İsimlendirme Kuralları [KZ-02, KZ-03]

```
❌ YANLIŞ — tip prefixi
Values:
  str_musteri_adi: musteriAdi
  int_kayit_sayisi: kayitSayisi
  dt_islem_tarihi: islemTarihi

✅ DOĞRU — tip prefixi yok
Values:
  musteri_adi: musteriAdi
  kayit_sayisi: kayitSayisi
  islem_tarihi: islemTarihi
```

Business Log ID için [KZ-03]: Aktivite kendi GUID'ini otomatik oluşturur.
`Identifier` alanı sadece bir etiket — oraya `DateTime.Now.ToString(...)` **yazma**.

---

## Sık Yapılan Hatalar

| Hata | Sonuç | Çözüm |
|---|---|---|
| Insert var, Update yok | Kayıt sonsuza InProcess kalır | Her Insert için Update yaz |
| Aynı Identifier iki Insert'te | İlk kayıt üzerine yazılır | Benzersiz Identifier kullan |
| Child Insert parent'tan önce | Exception: groupId bulunamadı | Sıra önemli: Parent → Child |
| Update'te yanlış Identifier | Kayıt güncellenemiyor | Insert ile Update Identifier eşleşmeli |

---

## Fork Notları

- `IfErrorThrow = false` varsayılandır — [HYS-01] kapsamında hata değil
- `IsAsync = true` varsayılandır — Insert hemen döner, arka planda gönderilir
- Token: `Config.local.GetTokenUnprotected()` (otomatik)
- URL: `Config.local.migrationurl` (otomatik)
