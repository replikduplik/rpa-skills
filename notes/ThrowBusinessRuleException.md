# ThrowBusinessRuleException Aktivitesi

## Ne Yapar

İş kuralı ihlali gibi kalıcı hata durumlarında `BusinessRuleException` fırlatır.
Bu aktivite ile fırlatılan istisnalar retry yapılmaz — WorkItem direkt `failed` durumuna geçer.

---

## Parametreler

| Özellik | Tür | Zorunlu | Varsayılan | Açıklama |
|---------|-----|---------|------------|----------|
| `Message` | `InArgument<string>` | Hayır | — | İstisna mesajı |

---

## BusinessException vs SystemException Ayrımı

Bu ayrım retry stratejisi için kritiktir:

| Tür | Örnekler | Retry? | Ne Yapılır |
|---|---|---|---|
| **BusinessException** | Geçersiz TC, boş poliçe no, müşteri bulunamadı, geçersiz IBAN | ❌ Hayır | `ThrowBusinessRuleException` ile fırlat → WorkItem `failed` |
| **SystemException** | Timeout, ağ kopması, SAP yavaşlığı, geçici portal hatası | ✅ Evet | TryCatch içinde tekrar dene (maks 3) |

---

## Kullanım Şablonu

```
TryCatch (WorkItem işleme)
├── Try
│   ├── [veri doğrulama]
│   │   └── If (tcKimlik geçersiz)
│   │       ThrowBusinessRuleException
│   │         Message: GetTextResource("gecersiz_tc_kimlik")
│   │
│   └── [iş adımları]
│
└── Catch (Exception e)
    ├── If (e is BusinessRuleException)
    │   ├── HyperNodeInsertBusinessLog (EntryState: "2")
    │   ├── CreateYouTrackIssue
    │   └── UpdateWorkitem → state: "failed"   ← retry yok
    └── Else (SystemException)
        ├── HyperNodeInsertBusinessLog (EntryState: "2")
        ├── CreateYouTrackIssue
        └── UpdateWorkitem → state: "failed" veya retry
```

**C# ile tip kontrolü:**
```csharp
// BusinessRuleException tespiti
bool isBusinessException = e.GetType().Name.Contains("BusinessRule")
    || e.Message.Contains("geçersiz")
    || e.Message.Contains("bulunamadı");
```

---

## Fork Notları

- Namespace: `EpochxCreatorStudio.WorkItems`
- Aktiviteye erişim: WorkItems kategorisinde
- Exception tipi: `BusinessRuleException` (platform tanımlı)
