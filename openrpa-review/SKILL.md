---
name: openrpa-review
description: >
  Use when reviewing an existing OpenRPA workflow for quality, safety, or best
  practices. Triggers on: "workflow doğru mu", "review", "incele", "kontrol et",
  "best practice", "güvenli mi", "iyileştirme", "optimize et", "xaml gönderiyorum",
  "workflow yazıyorum nasıl görünüyor", "eksik var mı", "denetim", "audit",
  "kod kalitesi", "workflow kontrol", "hata var mı".
---

# OpenRPA Workflow Review — İnceleme Rehberi

## Nasıl Kullanılır

Workflow'unu şu biçimde paylaş:
- Adım adım metin açıklaması
- Pseudo-XAML (girintili aktivite listesi)
- Ekran görüntüsü açıklaması

Review yapmadan önce **Hata Sayılmayacaklar (HYS)** bölümünü oku — bazı durumlar
bilinçli tasarım tercihidir ve hata olarak raporlanmamalıdır.

Claude şu kontrol listesini uygular:

---

## Hata Sayılmayacaklar (HYS — False Positive Önleme)

> Bu bölümü denetim yapmadan önce oku. Aşağıdaki durumlar bilinçli tasarım tercihidir.
> Bunları **hata, uyarı veya iyileştirme önerisi olarak kesinlikle raporlama**.

### [HYS-01] — MetricLog ve BusinessLog'da `IfErrorThrow = false` Normaldir

`MetricLog`, `HyperNodeInsertBusinessLog` ve `HyperNodeUpdateBusinessLog` aktivitelerinde
`IfErrorThrow = false` olması bir hata değildir. Bu aktivitelerin hata üretmeden sessizce
geçmesi beklenen ve kabul edilen davranıştır.

### [HYS-02] — `PopWorkitem` Sonrasında `item is Nothing` Kontrolü Zorunlu Değil

`PopWorkitem` aktivitesinin hemen ardından null check yapılmamış olması bir hata değildir.
Bu kontrol isteğe bağlıdır.

### [HYS-03] — `HyperNodeUpdateBusinessLog`'da Boş `Values` Normaldir

`HyperNodeUpdateBusinessLog` aktivitesinin `Values` parametresinin boş bırakılması
bir hata değildir. Güncelleme senaryolarında bazı alanların boş geçilmesi tasarım gereğidir.

### [HYS-04] — `InvokeCode`'da Boş `Arguments` Normaldir

`InvokeCode` aktivitesinin `Arguments` sözlüğünün boş olması bir hata değildir;
bu durumda aktivite yalnızca kendi iç değişkenleriyle çalışıyor demektir.

> **Not:** `InvokeCode` aktivitesinin **varlığı** [KZ-09] kapsamında hata sayılmaya devam
> eder. Bu kural yalnızca `Arguments` alanının boş olmasının ek bir hata oluşturmayacağını belirtir.

---

## Kontrol Listesi

### 1. Hata Yakalama — TryCatch

```
[ ] Ana akışta TryCatch var mı?
[ ] Her InvokeWorkflow çağrısının etrafında TryCatch var mı?
[ ] Catch bloğu:
      - HyperNodeInsertBusinessLog (EntryState: "2") — hata kaydı
      - CreateYouTrackIssue aktivitesi var mı? [KZ-12] ← ZORUNLU
      - Müşteriye gönderilen mesaj ham ex.Message içermiyor mu? [KZ-13]
      - WorkItem varsa UpdateWorkitem → state: "failed"
      - Rethrow veya recovery kararı verilmiş mi?
[ ] Finally bloğu:
      - Excel/COM nesneleri serbest bırakılıyor mu?
      - Uygulama/oturum kapatılıyor mu?
[ ] InvokeCode içinde throw ex yerine throw mi kullanılmış? [stack trace kaybı]
```

**Kırmızı bayrak:** TryCatch olmayan 10+ aktiviteli Sequence veya YouTrack olmayan Catch bloğu [KZ-12].

Detay: `notes/TryCatch.md` | `notes/CreateYouTrackIssue.md`

---

### 2. Selector Kalitesi

```
[ ] automationid veya stabil CSS id kullanılıyor mu?
[ ] Dinamik idx (idx=5, idx=12) veya değişken name kullanılıyor mu? → riskli
[ ] Wildcard doğru yerde mi? (name='Polisoft*' vs name='*')
[ ] Polisoft için cls=TEdit/TButton/TDBGrid kullanılıyor mu?
[ ] Web portal için id > data-testid > css > xpath önceliği var mı?
[ ] Selector URL'lerinde state=, nonce=, hasarIslemId= gibi session parametreleri var mı?
      → Session bazlı parametreler her oturumda değişir, robot kırılır
```

**Kırmızı bayrak:** `idx` bazlı selector veya URL'de hardcoded session parametresi.

Detay: `openrpa-selector`

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
[ ] GetRpaSetting veya GetCredentials kullanılıyor mu?
[ ] Log aktivitesine şifre yazılmış mı? → GİZLİLİK İHLALİ
[ ] JWT token / API key değişken içinde mi, sabit string mi?
[ ] CommentOut bloğu içinde hardcoded credential kalıntısı var mı?
      CommentOut = devre dışı değil, sadece gizli — XAML içinde açık metin olarak durur
      Git diff'inde ve kaynak kod taramasında görünür → KRİTİK GÜVENLİK AÇIĞI
```

**Hardcode şifre bulunursa:**
```
// ❌ Asla — Assign içinde veya InvokeCode içinde
item.Value = "desdDEdfCd3NiqFz!"

// ❌ Asla — CommentOut ile "saklanmış" gibi görünse de XAML'da açık metin!
CommentOut_4
  └── InvokeCode_28
        str_DogaPassword = "desdDEdfCd3NiqFz!"   ← HÂLÂ DOSYADA MEVCUT

// ✅ Her zaman — GetRpaSetting veya GetCredentials
GetRpaSetting Key: "portal_kullanici"  → str_KullaniciAdi
GetRpaSetting Key: "portal_sifre"      → str_Sifre
// veya
GetCredentials Name: "portal-prod" → Username, Password
```

**Credential denetim kuralı:** Workflow dosyasında `grep -i "password\|sifre\|parola"` yap —
CommentOut içindekiler dahil tüm eşleşmeler incelenmeli.

Detay: `notes/GetRPASetting.md`

---

### 5. Modülerlik

```
[ ] Tek workflow / sequence içinde 30'dan fazla aktivite var mı? → bölünmeli [KZ-15]
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

**Kırmızı bayrak — tekrarlanan login akışı:**

Aynı portal login sekansı iki farklı sequence içinde birebir kopyalanmışsa
bu hem bakım yükü hem de tutarsızlık riskidir:

```
❌ Kopya-yapıştır login — iki yerde birebir aynı
Sequence_21: OpenURL → GetElement → FocusElement → TypeText → ClickElement
Sequence_44: OpenURL → GetElement → FocusElement → TypeText → ClickElement

✅ InvokeWorkflow ile factoring
PortalGirisYap.xaml  ← tek bir yerde tanımla
  In: str_KullaniciAdi, str_Sifre, str_PortalUrl
  Out: bool_GirisBasarili
Sequence_21: InvokeWorkflow → PortalGirisYap.xaml
Sequence_44: InvokeWorkflow → PortalGirisYap.xaml
```

Detay: `notes/InvokeWorkflow.md`

---

### 6. MetricLog Kontrolü [KZ-07]

```
[ ] Her GetElement, ClickElement, TypeText, GetText, OpenUrl aktivitesi
    MetricLog bloğu içinde mi?
[ ] MetricLog'larda ThrowIfError=false mi? (HYS-01: bu normaldir, hata değil)
[ ] MetricLog Name değerleri anlamlı mı? ([nesne]_[eylem] formatı önerilir)
```

**Kırmızı bayrak:** MetricLog olmayan her UI etkileşim aktivitesi [KZ-07] ihlali.

Detay: `notes/MetricLog.md`

---

### 7. Performans

```
[ ] Gereksiz Delay aktivitesi var mı? (Delay 3000, Delay 5000 sabit) [KZ-06]
      → GetElement Timeout veya döngüsel bekleme ile değiştir
[ ] WriteLine aktivitesi var mı? → [KZ-08] Console.WriteLine ile aynı kapsam, kaldır
[ ] Büyük DataTable loop içinde mi işleniyor?
      → LINQ veya toplu işlem düşün
[ ] Her satır için uygulama yeniden açılıyor mu?
      → Oturumu paylaş, döngü dışında aç
[ ] GetElement Timeout çok düşük mü? (500ms — ağ gecikmesinde başarısız)
      → Minimum 5000ms öner
```

---

### 8. Logging / Audit Trail [KZ-18]

```
[ ] WorkItem ID log'a dahil mi? (hangi kaydın işlendiği belli)
[ ] İşlem başı ve sonu loglanıyor mu? (HyperNodeInsertBusinessLog + HyperNodeUpdateBusinessLog)
[ ] Her Insert için karşılık bir Update var mı?
      → Update olmayan kayıt sonsuza "InProcess" kalır
[ ] Aynı Identifier iki farklı Insert'te kullanılmış mı?
      → İkincisi birincinin üzerine yazar, ilk kayıt kaybolur
[ ] Hata durumunda yeterli bilgi var mı? (sadece "hata" değil, değerler de)
[ ] Business Log key'lerinde str_, int_, dt_ gibi tip prefixi var mı? [KZ-02]
```

**Standart log formatı:**
```
[2025-01-15 09:23:11] [WI:wi_abc] [BAŞLADI] PoliceNo: PN-98765
[2025-01-15 09:23:45] [WI:wi_abc] [ADIM] Teklif alındı: T-001
[2025-01-15 09:23:52] [WI:wi_abc] [TAMAM] Polisoft kayıt: PS-12345
[2025-01-15 09:23:53] [WI:wi_abc] [BITTI] Süre: 42 sn
```

Detay: `notes/BusinessLog.md`

---

### 9. Sabit Metinler [KZ-14]

```
[ ] Log mesajlarında, mail içeriklerinde veya aktivite parametrelerinde
    inline string kullanılıyor mu?
      "İşlem başarıyla tamamlandı" → KZ-14 ihlali
      GetTextResource(key: "islem_basarili") → doğru
[ ] GetTextResource aktivitesi projede mevcut mu?
    (yoksa eklenmesi gerektiğini bulguda belirt)
```

---

### 10. Döngü Güvenliği

```
[ ] BreakableDoWhile / BreakableWhile koşulunda Or yerine And mi kullanılmış?
      Or = koşulun bir parçası True kalırsa döngü bitmez
      And = her iki koşul da sağlandığında döngü devam eder → güvenli
[ ] GetElement ile bekleme yapılan döngüde Timeout="{x:Null}" var mı?
      → {x:Null} sonsuz bekleme: döngü içindeyse robot kilitlenir
[ ] Döngü içindeki değişken null-safe mi kontrol ediliyor?
      elem.Length = 0 → NullReferenceException (null ise)
      (elem Is Nothing OrElse elem.Length = 0) → güvenli
[ ] Döngüde maksimum iterasyon sınırı var mı?
      sayac < 60 gibi bir üst limit olmadan While/DoWhile sonsuz dönebilir
```

Detay: `notes/ForEach-BreakableLoop.md` | `notes/GetElement.md`

---

### 11. Değişken Yönetimi

```
[ ] Değişken isimleri anlamlı mı? (x, temp, var1 → riskli) [KZ-17]
[ ] Değişken adı içerdiği değeri doğru yansıtıyor mu?
      bool_mKaydiVarMi = true ama aslında "kayıt YOK" anlamına geliyorsa → yanıltıcı
[ ] Koşullarda gereksiz = True / = False var mı?
      If [bool_Odendi = True]  → sadeleştir: If [bool_Odendi]
      If [bool_Odendi = False] → sadeleştir: If [Not bool_Odendi]
[ ] Scope doğru mu? (geniş scope'lu değişken döngüde reset ediliyor mu?)
[ ] String birleştirme döngü içinde mi? → StringBuilder kullan
[ ] DataTable boyutu kontrol ediliyor mu? (100K satır belleği patlatır)
```

---

### 12. Retry Stratejisi

```
[ ] Geçici hatalar retry ile ele alınıyor mu? (ağ, SAP yavaşlığı)
[ ] Retry sayısı sınırlı mı? (sonsuz retry → robot asla ilerlemez)
[ ] Kalıcı hata retry'dan muaf mı? (geçersiz veri → direkt failed)
[ ] BusinessException (iş hatası) ile SystemException (teknik hata) ayrımı var mı?
      BusinessException → retry etme, direkt failed yap
      SystemException   → retry yap, GetElement Timeout ile bekle [KZ-06]
```

**Retry yaklaşımı — [KZ-06] uyumlu:**
```
Retry karar ağacı:
├── BusinessException → ThrowBusinessRuleException ile fırlat, WorkItem "failed"
└── SystemException   → InvokeWorkflow TryCatch içinde tekrar çağır (maks 3)
    Bekleme: Thread.Sleep veya Delay yasak [KZ-06] → GetElement Timeout kullan
```

Detay: `notes/ThrowBusinessRuleException.md`

---

### 13. Süreç Kalitesi (Otomasyon Öncesi)

```
[ ] Otomatize edilecek süreç manuel olarak istikrarlı çalışıyor mu?
      Hayır → önce süreci düzelt, sonra otomatize et
      "Kırık süreci robot daha hızlı kırar, iyileştirmez"
[ ] İstisna oranı kabul edilebilir seviyede mi? (%5 altı önerilir)
[ ] Tüm varyantlar (edge case) belgelenmiş mi?
[ ] Sürecin sık değişmesi bekleniyor mu? → bakım maliyetini hesapla
```

---

## Önem Seviyeleri

| 🔴 Kritik | 🟡 Önemli | 🟢 İyileştirme |
|---|---|---|
| Hardcode credential | Gereksiz Delay / Thread.Sleep [KZ-06] | İsimlendirme [KZ-17] |
| TryCatch yok | idx selector | Log format |
| YouTrack olmayan Catch [KZ-12] | Modülerlik eksik (30+ aktivite) [KZ-15] | Yorum satırı [KZ-05] |
| MetricLog eksik [KZ-07] | Retry yok | Değişken scope |
| WorkItem Finally yok | BusinessException ayrımı yok | GetTextResource eksik [KZ-14] |
| Excel COM cleanup yok | Kırık süreç otomatize | Business Log key'de tip prefixi [KZ-02] |
| InvokeCode production'da [KZ-09] | Insert'siz Update veya Update'siz Insert | Console.WriteLine (Epoch/X) [KZ-08] |
| Ham ex.Message müşteriye [KZ-13] | GetRpaSetting eksik [KZ-01] | WriteLine aktivitesi [KZ-08] |
| `throw ex` kullanımı | CommentOut'ta ölü credential | BreakableDoWhile Or koşulu |
| Session parametresi selector'da | Tekrarlanan login akışı | Timeout="{x:Null}" |

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
