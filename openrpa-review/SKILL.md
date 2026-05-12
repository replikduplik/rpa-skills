---
name: openrpa-review
description: >
  Use when reviewing an existing OpenRPA workflow for quality, safety, or best
  practices. Triggers on: "workflow doğru mu", "review", "incele", "kontrol et",
  "best practice", "güvenli mi", "iyileştirme", "optimize et", "xaml gönderiyorum",
  "workflow yazıyorum nasıl görünüyor", "eksik var mı".
---

# OpenRPA Workflow Review — İnceleme Rehberi

## Nasıl Kullanılır

Workflow'unu şu biçimde paylaş:
- Adım adım metin açıklaması
- Pseudo-XAML (girintili aktivite listesi)
- Ekran görüntüsü açıklaması

Claude şu kontrol listesini uygular:

---

## Kontrol Listesi

### 1. Hata Yakalama — TryCatch

```
[ ] Ana akışta TryCatch var mı?
[ ] Her InvokeWorkflow çağrısının etrafında TryCatch var mı?
[ ] Catch bloğu:
      - Log (Error seviye, e.Message + e.StackTrace)
      - WorkItem varsa UpdateWorkitem → state: "failed"
      - Rethrow veya recovery kararı verilmiş mi?
[ ] Finally bloğu:
      - Excel/COM nesneleri serbest bırakılıyor mu?
      - Uygulama/oturum kapatılıyor mu?
```

**Kırmızı bayrak:** TryCatch olmayan 10+ satırlık Sequence.

---

### 2. Selector Kalitesi

```
[ ] automationid veya stabil CSS id kullanılıyor mu?
[ ] Dinamik idx (idx=5, idx=12) veya değişken name kullanılıyor mu? → riskli
[ ] Wildcard doğru yerde mi? (name='Polisoft*' vs name='*')
[ ] Polisoft için cls=TEdit/TButton/TDBGrid kullanılıyor mu?
[ ] Web portal için id > data-testid > css > xpath önceliği var mı?
```

**Kırmızı bayrak:** `idx` bazlı selector — herhangi bir UI değişikliğinde kırılır.

---

### 3. Idempotency

```
[ ] Aynı WorkItem iki kez işlenirse ne olur?
[ ] Veritabanına yazma, kuyruğa ekleme — çift kayıt oluşur mu?
[ ] "Zaten işlendi" kontrolü var mı? (poliçe no, sipariş no)
[ ] Retry senaryosu düşünülmüş mü?
```

**Kontrol şablonu:**
```
GetElement → "Bu kayıt zaten var" mesajı
If (mevcutKayitVar)
    Log → "Atlanıyor: " + policeNo
    UpdateWorkitem → state: "successful" (idempotent kabul)
Else
    [normal işlem]
```

---

### 4. Credential Güvenliği

```
[ ] Kullanıcı adı / şifre workflow içinde hardcode var mı? → KRİTİK HATA
[ ] OpenFlow Credentials kullanılıyor mu?
[ ] Log aktivitesine şifre yazılmış mı? → GİZLİLİK İHLALİ
[ ] JWT token / API key değişken içinde mi, sabit string mi?
```

**Hardcode şifre bulunursa:**
```csharp
// ❌ Asla
string sifre = "Abc123!";

// ✅ Her zaman
var cred = await global::OpenRPA.Interfaces.global.webSocketClient
    .GetCredential("polisoft-prod");
string sifre = cred.password;
```

---

### 5. Modülerlik

```
[ ] Tek workflow 200+ satırı geçiyor mu? → bölünmeli
[ ] Tek sorumluluk: her workflow bir iş yapıyor mu?
[ ] Ortak adımlar (giriş, çıkış, hata bildirimi) tekrar mı yazılmış?
      → InvokeWorkflow ile paylaşımlı alt workflow
[ ] Ana workflow çok derin iç içe mi? (5+ seviye) → düzleştir
```

**Modül önerisi — sigorta:**
```
GirisYap.xaml          ← tek sorumluluk: oturum aç
VeriDogrula.xaml       ← tek sorumluluk: girdileri kontrol et
IslemYap.xaml          ← tek sorumluluk: asıl iş
HataRaporla.xaml       ← tek sorumluluk: bildirim gönder
```

---

### 6. Performans

```
[ ] Gereksiz Delay aktivitesi var mı? (Delay 3000, Delay 5000 sabit)
      → GetElement Timeout veya döngüsel bekleme ile değiştir
[ ] Büyük DataTable loop içinde mi işleniyor?
      → LINQ veya toplu işlem düşün
[ ] Her satır için uygulama yeniden açılıyor mu?
      → Oturumu paylaş, döngü dışında aç
[ ] GetElement Timeout çok düşük mü? (500ms — ağ gecikmesinde başarısız)
      → Minimum 5000ms öner
```

---

### 7. Logging / Audit Trail

```
[ ] WorkItem ID log'a dahil mi? (hangi kaydın işlendiği belli)
[ ] İşlem başı ve sonu loglanıyor mu?
[ ] Hata durumunda yeterli bilgi var mı? (sadece "hata" değil, değerler de)
[ ] Log seviyesi doğru mu? (debug logları production'da kapalı mı?)
```

**Standart log formatı:**
```
[2025-01-15 09:23:11] [WI:wi_abc] [BAŞLADI] PoliceNo: PN-98765
[2025-01-15 09:23:45] [WI:wi_abc] [ADIM] Teklif alındı: T-001
[2025-01-15 09:23:52] [WI:wi_abc] [TAMAM] Polisoft kayıt: PS-12345
[2025-01-15 09:23:53] [WI:wi_abc] [BITTI] Süre: 42 sn
```

---

### 8. Değişken Yönetimi

```
[ ] Değişken isimleri anlamlı mı? (x, temp, var1 → riskli)
[ ] Scope doğru mu? (geniş scope'lu değişken döngüde reset ediliyor mu?)
[ ] String birleştirme döngü içinde mi? → StringBuilder kullan
[ ] DataTable boyutu kontrol ediliyor mu? (100K satır belleği patlatır)
```

---

### 9. Retry Stratejisi

```
[ ] Geçici hatalar retry ile ele alınıyor mu? (ağ, SAP yavaşlığı)
[ ] Retry sayısı sınırlı mı? (sonsuz retry → robot asla ilerlemez)
[ ] Exponential backoff var mı? (hemen retry → aynı hata)
[ ] Kalıcı hata retry'dan muaf mı? (geçersiz veri → direkt failed)
```

**Retry şablonu:**
```csharp
int deneme = 0;
while (deneme < 3)
{
    try { /* işlem */; break; }
    catch (Exception e) when (deneme < 2)
    {
        deneme++;
        System.Threading.Thread.Sleep(2000 * deneme);
    }
}
```

---

## Önem Seviyeleri

| 🔴 Kritik | 🟡 Önemli | 🟢 İyileştirme |
|---|---|---|
| Hardcode credential | Gereksiz Delay | İsimlendirme |
| TryCatch yok | idx selector | Log format |
| Sonsuz döngü riski | Modülerlik eksik | Yorum satırı |
| Null kontrolü yok | Retry yok | Değişken scope |

---

## Review Sonuç Formatı

Review bitince şu yapıda cevap ver:

```
## Kritik Sorunlar (hemen düzelt)
- [varsa listele]

## Önemli Sorunlar (yakın vadede düzelt)
- [varsa listele]

## İyileştirme Önerileri (isteğe bağlı)
- [varsa listele]

## İyi Yapılanlar
- [varsa listele]
```
