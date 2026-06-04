# CreateYouTrackIssue Aktivitesi

## Ne Yapar

YouTrack üzerinde hata kaydı (issue) oluşturur. Her `Catch` bloğunda çağrılması **zorunludur** [KZ-12].
Varsayılan olarak asenkron çalışır; hata oluştursa bile sessizce geçer (istisna fırlatmaz).

---

## Parametreler

| Özellik | Tür | Zorunlu | Varsayılan | Açıklama |
|---------|-----|---------|------------|----------|
| `Project` | `string` | Evet | — | YouTrack proje kimliği. Boş bırakılırsa "Epoch - Platform" adıyla aranır |
| `Summary` | `string` | Evet | — | Kayıt özeti. Otomatik "By CreatorStudio " ön eki eklenir |
| `Description` | `string` | Evet | — | Açıklama. Proje/workflow adı ve örnek kimliği otomatik ön ek olarak eklenir |
| `Analyst` | `string` | Evet | — | Sorumlu analist (YouTrack kullanıcı ID) |
| `Customer` | `string` | Evet | — | Müşteri enum değeri |
| `ProcessCode` | `string` | Evet | — | Süreç kodu enum değeri |
| `NeedCustomerAcceptance` | `string` | Evet | `"No"` | Müşteri onayı gerekli mi: `"Yes"` / `"No"` |
| `Type` | `string` | Evet | `"Bug"` | Kayıt türü |
| `Expiration` | `int` | Hayır | `15000` | HTTP zaman aşımı (ms) |

**Config değerleri:**
- `Config.local.youtrack_bearer_token` — Bearer token
- `Config.local.youtrack_url` — YouTrack base URL (boşsa: `https://xenius.youtrack.cloud`)

---

## Kullanım — Her Catch Bloğunda [KZ-12]

```
TryCatch
├── Try
│   └── [iş akışı adımları]
└── Catch (Exception e)
    ├── HyperNodeInsertBusinessLog
    │     EntryState: "2" (Failed)
    │     Values: { "hata_mesaji": ex.Message }
    ├── CreateYouTrackIssue          ← [KZ-12] zorunlu
    │     Project: GetRpaSetting("youtrack_project_id")
    │     Summary: workflowAdi + " - " + ex.Message
    │     Description: ex.Message + "\n" + ex.StackTrace
    │     Analyst: GetRpaSetting("analyst_id")
    │     Customer: GetRpaSetting("customer_code")
    │     ProcessCode: GetRpaSetting("process_code")
    │     NeedCustomerAcceptance: "No"
    │     Type: "Bug"
    └── [WorkItem state → "failed" veya rethrow]
```

---

## İhlal Durumu

```
❌ YANLIŞ — YouTrack olmadan Catch bloğu [KZ-12]
Catch (Exception e)
└── Log → ex.Message

✅ DOĞRU — YouTrack ile Catch bloğu
Catch (Exception e)
├── HyperNodeInsertBusinessLog (EntryState: "2")
├── CreateYouTrackIssue (...)
└── UpdateWorkitem → state: "failed"
```

---

## Önemli Notlar

- `Summary` alanına otomatik olarak `"By CreatorStudio "` ön eki eklenir.
- `Description` alanına proje adı, workflow adı ve workflow örnek kimliği otomatik eklenir.
- Aktivite hata oluştursa bile exception fırlatmaz — Catch bloğunu bozma riski yok.
- `Project` kimliği boş bırakılırsa YouTrack'ta "Epoch - Platform" projesi aranır.
- Tüm sabit değerler (`analyst_id`, `customer_code`, `process_code`) `GetRpaSetting` ile alınmalı [KZ-01].

---

## Fork Notları

- Namespace: `EpochxCreatorStudio.Activities`
- Temel sınıf: `AsyncTaskCodeActivity` — varsayılan asenkron çalışır
- Token kaynağı: `Config.local.youtrack_bearer_token`
