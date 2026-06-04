# MetricLog Aktivitesi

## Ne Yapar

Alt aktivitenin çalışma süresini milisaniye cinsinden ölçer ve HyperNode API'ye metrik olarak gönderir.
Her UI etkileşim aktivitesi (`GetElement`, `ClickElement`, `TypeText`, `GetText`, `SendKeys`, `OpenUrl` vb.)
bir `MetricLog` bloğu içine alınmalıdır **[KZ-07]**.

---

## Parametreler

| Özellik | Tür | Zorunlu | Varsayılan | Açıklama |
|---------|-----|---------|------------|----------|
| `Name` | `InArgument<string>` | Evet | — | Metrik adı (anlamlı, iş odaklı) |
| `Description` | `InArgument<string>` | Hayır | — | Ek açıklama |
| `IsAsync` | `InArgument<bool>` | Hayır | `true` | Asenkron gönder |
| `ThrowIfError` | `InArgument<bool>` | Hayır | `false` | Hata fırlat |
| `Expiration` | `InArgument<int>` | Hayır | `15000` | API zaman aşımı (ms) |
| `Body` | `Activity` | Hayır | — | Ölçülecek aktivite(ler) |

> **[HYS-01]** `ThrowIfError = false` normaldir — MetricLog aktivitesinin hata üretmeden
> sessizce geçmesi beklenen davranıştır. Bunu hata olarak raporlama.

---

## Kullanım Şablonu

```
MetricLog
  Name: "policeNo_GetElement"     ← anlamlı isim: nesne_eylem formatı
  Description: "Poliçe no alanı bulunuyor"
  Body:
    └── GetElement
          Selector: [poliçe no alanı]
          Timeout: 10000ms
```

**Her UI etkileşimini MetricLog ile sar:**

```
MetricLog Name: "girisFormu_GetElement"
└── GetElement → giriş formu

MetricLog Name: "kullanici_TypeText"
└── TypeText → kullanici adı

MetricLog Name: "sifre_TypeText"
└── TypeText → şifre + "{Enter}"

MetricLog Name: "dashboard_GetElement"
└── GetElement → dashboard yüklendi (Timeout: 20000ms)
```

---

## Hangi Aktiviteler MetricLog Gerektirir

KZ-07 kapsamında MetricLog zorunlu olan aktiviteler:

- `GetElement` (tüm provider'lar: Windows, NM, SAP, IE, Image, Java)
- `ClickElement` / `ClickCoordinates`
- `TypeText`
- `GetText` (Image OCR dahil)
- `SendKeys`
- `OpenUrl` / `OpenApplication`
- `FocusElement`
- `MoveMouse`

---

## İhlal Durumu

```
❌ YANLIŞ — MetricLog olmadan kullanım
GetElement → kullanıcı alanı

✅ DOĞRU — MetricLog ile sarılmış
MetricLog Name: "kullaniciAlani_GetElement"
└── GetElement → kullanıcı alanı
```

---

## Fork Notları

- `ThrowIfError` varsayılanı `false` → [HYS-01] kapsamında normal
- Metrik adı formatı: `[nesne]_[eylem]` (örn. `policeNo_GetElement`, `loginBtn_Click`)
- API URL: `Config.local.migrationurl` üzerinden otomatik alınır
