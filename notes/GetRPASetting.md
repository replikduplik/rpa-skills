# GetRPASetting / UpdateRPASetting / GetAndUpdateRPASetting Aktiviteleri

Bu üç aktivite, workflow içindeki sabit değerlerin merkezi konfigürasyondan alınması için
kullanılır. Tüm hardcoded değerler bu aktiviteler ile dışarıdan alınmalıdır **[KZ-01]**.

---

## GetRPASetting

### Ne Yapar
RPA ayar değerini HyperNode API'den getirir.

### Parametreler

| Özellik | Tür | Zorunlu | Varsayılan | Açıklama |
|---------|-----|---------|------------|----------|
| `Key` | `InArgument<string>` | Evet | — | Ayar anahtarı |
| `Path` | `InArgument<string>` | Evet | Config | API yolu |
| `IfErrorThrow` | `InArgument<bool>` | Hayır | `false` | Hata fırlat |

### Çıktılar

| Özellik | Tür | Açıklama |
|---------|-----|----------|
| `RpaSettingValue` | `OutArgument<string>` | Ayar değeri |
| `RpaSettingUpdatedAt` | `OutArgument<DateTime?>` | Son güncelleme tarihi |

### Kullanım

```
GetRPASetting
  Key: "portal_url"
  → RpaSettingValue → portalUrl

GetRPASetting
  Key: "islem_timeout_ms"
  → RpaSettingValue → timeoutMsStr
```

---

## UpdateRPASetting

### Ne Yapar
Mevcut bir RPA ayarının değerini HTTP POST ile günceller.

### Parametreler

| Özellik | Tür | Zorunlu | Varsayılan | Açıklama |
|---------|-----|---------|------------|----------|
| `Key` | `InArgument<string>` | Evet | — | Ayar anahtarı |
| `Value` | `InArgument<string>` | Evet | — | Yeni değer |
| `IfErrorThrow` | `InArgument<bool>` | Hayır | `false` | Hata fırlat |
| `Expiration` | `InArgument<int>` | Hayır | `15000` | Zaman aşımı (ms) |

### Kullanım

```
UpdateRPASetting
  Key: "son_islem_tarihi"
  Value: DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss")
```

---

## GetAndUpdateRPASetting

### Ne Yapar
Mevcut değeri okur ve aynı anda yeni bir değer yazar (atomik GET+POST).
Sayaç veya koordinasyon senaryolarında race condition önler.

### Parametreler

| Özellik | Tür | Zorunlu | Varsayılan | Açıklama |
|---------|-----|---------|------------|----------|
| `Key` | `InArgument<string>` | Evet | — | Ayar anahtarı |
| `Value` | `InArgument<string>` | Evet | — | Güncellenecek yeni değer |
| `IfErrorThrow` | `InArgument<bool>` | Hayır | `false` | Hata fırlat |
| `Expiration` | `InArgument<int>` | Hayır | `15000` | Zaman aşımı (ms) |

### Çıktılar

| Özellik | Tür | Açıklama |
|---------|-----|----------|
| `RpaSettingValue` | `OutArgument<string>` | **Güncelleme öncesi** eski değer |
| `RpaSettingUpdatedAt` | `OutArgument<DateTime?>` | Önceki güncelleme tarihi |

### Kullanım — Sayaç Örneği

```
GetAndUpdateRPASetting
  Key: "islenen_kayit_sayaci"
  Value: (Convert.ToInt32(eskiDeger) + 1).ToString()
  → RpaSettingValue → eskiDeger   ← güncelleme öncesi değer
```

---

## [KZ-01] Hangi Değerler GetRPASetting ile Alınmalı?

Workflow içinde **hardcoded** olan her sabit değer:

```
❌ YANLIŞ
string portalUrl = "https://portal.allianz.com.tr";
int timeoutMs = 15000;
string kullanici = "robot_user";

✅ DOĞRU
GetRPASetting Key: "portal_url"       → portalUrl
GetRPASetting Key: "timeout_ms"       → timeoutMsStr
GetRPASetting Key: "robot_kullanici"  → kullanici
```

Sabit değer kategorileri: URL, dosya yolu, kullanıcı adı, eşik değeri, zaman aşımı süresi,
YouTrack proje/kullanıcı ID, müşteri kodu, süreç kodu.

---

## Fork Notları

- Namespace: `EpochxCreatorStudio.Activities`
- API URL: `Config.local.migrationurl` (varsayılan)
- Bearer token: `Config.local.GetTokenUnprotected()` (otomatik)
