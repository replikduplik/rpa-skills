# Proje: OpenRPA Geliştirme Asistanı

Bu klasör, şirkete ait **OpenRPA fork'u** için geliştirilmiş bir Claude skill ekosistemidir.

## Bağlam

- **Sektör:** Sigorta (Polisoft, Open sigorta yazılımı, sigorta web portalleri)
- **Platform:** Vanilla OpenRPA'ya yakın yapıda şirket fork'u — farklar `fork-differences.md` dosyasında biriktirilir
- **Teknoloji:** Windows Workflow Foundation (WF), C# InvokeCode, UIAutomation, NativeMessaging, SAP GUI, OpenFlow
- **Ana uygulama:** Polisoft (Delphi/VCL tabanlı) — teklif, poliçe, hasar, zeyilname modülleri

## Skill Ekosistemi

Her OpenRPA sorusunda `openrpa-master/SKILL.md` dosyasını oku — doğru skill'e yönlendiren router burada.

| Skill | Dosya | Ne Zaman |
|---|---|---|
| openrpa-master | `openrpa-master/SKILL.md` | Her soruda ilk oku |
| openrpa-workflow | `openrpa-workflow/SKILL.md` | Workflow tasarımı, aktivite seçimi |
| openrpa-selector | `openrpa-selector/SKILL.md` | Selector yazma, element bulma |
| openrpa-invoke | `openrpa-invoke/SKILL.md` | PowerShell, InvokeWorkflow, API |
| openrpa-openflow | `openrpa-openflow/SKILL.md` | WorkItem queue, orkestrasyon |
| openrpa-debug | `openrpa-debug/SKILL.md` | Hata teşhis, log analizi |
| openrpa-review | `openrpa-review/SKILL.md` | Workflow inceleme, best practice |
| openrpa-csharp | `openrpa-csharp/SKILL.md` | InvokeCode C# kod blokları |
| openrpa-polisoft-deep | `openrpa-polisoft-deep/SKILL.md` | Polisoft tüm modülleri |

## Aktivite Notları

`notes/` klasöründe her OpenRPA aktivitesi için ayrıntılı rehber bulunur:
`GetElement`, `TypeText`, `ClickElement`, `InvokeCode`, `TryCatch`, `OpenApplication`, `InvokeWorkflow`, `Delay`, `ForEach-BreakableLoop`

## Workflow Şablonları

`templates/` klasöründe tekrar eden iş süreçleri için hazır kalıplar toplanır. Yeni şablon formatı için `templates/README.md`'e bak.

## Canlı Belgeler

- `fork-differences.md` — vanilla OpenRPA ile fork arasındaki farkların biriktirildiği belge; çalışırken beklenmedik davranış görünce buraya ekle
- `docs/superpowers/` — tasarım ve planlama dokümanları

## İçerik Ekleme Kuralları

- **Yeni skill:** `openrpa-[konu]/SKILL.md` → `openrpa-master/SKILL.md` tablosunu güncelle
- **Yeni note:** `notes/[AktiviteAdı].md` → `openrpa-master/SKILL.md` tablosunu güncelle
- **Yeni şablon:** `templates/[surecadi].md` → formatı `templates/README.md`'den al
- **Fork farkı:** `fork-differences.md`'e ekle → etkilenen skill/note'u güncelle
