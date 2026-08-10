---
layout: post
lang: tr
title: "AI Orkestrasyonu, Bölüm 1: Bindiğim Daldaki Dört CVE"
date: 2026-08-07
permalink: /tr/ai-research/2026/08/07/four-cves-in-my-own-dependency-tree.html
author: ataberk-xyz
categories: [ai-research]
tags: [orchestration, gossipcat, npm, cve, fuzzing, supply-chain]
ledger:
  target: "re2 (node-re2) · stream-json — haftada ~12,1M indirme"
  severity: "MEDIUM 6.2"
  vector: "CWE-835 · CWE-125 · CWE-617 · CWE-407"
  advisory: "CVE-2026-68499 · 67550 · 71430 · 71429"
  impact: "event-loop DoS, process-level crash, bounded heap read"
  status: "PATCHED — re2 1.25.2, stream-json 3.5.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
summary: "Kendi geliştirdiğim multi-agent review sistemi, kendi dependency tree'me — haftada toplam yaklaşık 12,1 milyon npm indirmesi olan re2 ve stream-json paketlerine — yöneltildi. Sonuçta kabul edilen dört advisory yayımlandı; bu yazı, sürecin bir bulguyu az kalsın elden çıkardığı anlar dahil, işin nasıl yürüdüğünün dürüst bir anlatısıdır."
---

> **Gösteri amaçlı örnekler yerine gerçek hedeflere uygulanan AI orkestrasyonu üzerine bir serinin 1. Bölümü.**

### Başlangıç noktası

Bu çalışmada, haftalık toplam indirme hacmi yaklaşık 12,1 milyon olan iki npm paketinde (re2, stream-json) dört güvenlik zafiyeti tespit edilmiştir. Bulguların tamamı maintainer tarafından doğrulanmış, advisory'leri yayımlanmış ve ilgili sürümlerde yamalanmıştır. Dört bulgu da Medium önem derecesinde olup denial-of-service sınıfındadır.

Çalışmanın kapsamı dar bir soruyla belirlendi: yalnızca kendi kodum üzerinde kullanılan multi-agent code-review sistemi [gossipcat](https://github.com/gossipcat-ai/gossipcat-ai), kurulu bağımlılıklara yöneltildiğinde ne bulur? İnceleme zinciri deponun node_modules dizininde doğrulanabilir durumdaydı: gossipcat → re2 (^1.25.0) → install-artifact-from-github. re2'nin seçilme gerekçesi önemlidir: regular-expression denial-of-service riskine karşı kurulan bir paketin binding layer'ı, tam da bu itibarı nedeniyle incelenmeye değerdi. stream-json, her iki paketin de maintainer'ı olan Eugene Lazutkin (uhop) üzerinden kapsama alındı.

Atıfla ilgili bir not düşmek gerekir. re2 bulgularında hangi oturumun hangi zafiyeti ortaya çıkardığı kayıtlarımda ayırt edilememektedir; bu ayrımı sonradan yeniden kurmaya çalışmayacağım. Aşağıda iki rol yer almaktadır: işi dispatch eden ve doğrulayan orkestratör ile ben.

---

### Global match döngüsünde ilerlemeyen bir cursor

ASAN altında çalışan bir fuzzer, global flag ile `A*((((((((a)?)?))*)*)?)*)*` üzerinde yirmi bir dakika boyunca meşgul kalmış ve 2.5GB tutmaktaydı. Ne bir sanitizer raporu ne de bir crash vardı; dolayısıyla çalıştırma, yavaş bir testten ayırt edilemiyordu.

Takılan process'in sample'lanması belirsizliği giderdi: stack, sıkı bir döngü içinde `WrappedRE2::Match` → `RE2::Match` → `DFA::Search` gösteriyordu. Nedeni `match.cc`'deki dört satırdır:

```cpp
while (re2->regexp.Match(str, byteIndex, str.size, anchor, &match, 1)) {
    groups.push_back(match);
    byteIndex = match.data() - str.data + match.size();  // += 0 for a zero-width match
}
```

Cursor, az önce bulunan match'in uzunluğu kadar ilerlemektedir. Zero-width match'in uzunluğu olmadığından cursor yerinden kımıldamaz ve aynı match süresiz olarak döndürülürken `groups` bir `std::vector<StringPiece>`'i büyütmeye devam eder.

Patolojik desen tesadüfiydi. Boş string'i match edebilen herhangi bir global RE2 bunu yeniden üretir:

```js
const RE2 = require('re2');
'x'.match(new RE2('a*', 'g'));   // never returns
```

Yayınlanan binary'de bu, yaklaşık dört saniye içinde 736MB'tan 3.7GB'a çıkar ve süreci ancak dışarıdan gelen bir `SIGKILL` sonlandırır. Çağrı senkron ve native olduğu için thread'i tutar; bu nedenle `try`/`catch`, `AbortController` ve JS timer'larının tamamı işlevsiz kalır — timer'lar, tetiklenmek için hiç sıra bulamadıklarından. V8 aynı deseni 0ms'de işler.

Bellek büyümesi, yüklü bir makinenin en kolay uydurabildiği ölçümdür; bu nedenle rakam tek başına kabul edilmedi. Orkestratör vakayı, boş string'i match edemeyen `a+` kontrolüne karşı yeniden çalıştırdı ve her process'i kendi resident set'i için sample'ladı: bug 3860MB'a tırmanırken kontrol 44MB'ta sabit kaldı. Orkestratör ayrıca döngüyü, en son yayımlanan sürümün temiz bir kurulumunda, hem yayınlanan binary hem de kaynaktan derlenmiş bir build üzerinde yeniden doğrulamıştır. Dosyalama ve gönderim bana aitti.

node-re2 bu vakayı başka yerlerde zaten ele almaktadır. Üç yol bir subject boyunca cursor ile ilerler: `split.cc` zero-width match'te bir code point ilerler, `exec` ise `lastIndex` üzerinden ilerler. Guard'ı yalnızca global `Match` yolu atlar. CVE-2026-68499, CWE-835, Medium 6.2, 1.25.2'de düzeltildi.

> **Ders: aynı yapıyı kat eden alternatif yollar tek tek incelenmelidir.** Bir yapı birden fazla yerde kat ediliyorsa, bu yolların advance ve termination logic'leri karşılaştırılmalıdır. Diğer yolların taşıdığı guard'ı taşımayan yol, bug'ın bulunduğu yoldur; sıfır uzunluklu elemanlar da bunun klasik tetikleyicisidir.

---

### Bir character index'ini doğrulayan byte uzunluğu

Fuzzer ikinci bug'ı bulmadı, bulamazdı da. Ona ulaşmak, `re.lastIndex`'in belirli ve yanlış bir değere set edilmesini gerektirir; rastgele girdi üreten bir mekanizmanın o property'ye dokunmak için hiçbir nedeni yoktur. byte-character dönüşüm aritmetiğinin okunması, bug'ı dakikalar içinde yerine oturttu.

`lastIndex` kullanıcı tarafından herhangi bir pozitif tam sayıya set edilebilir. `prepareArgument`, subject'in **UTF-8 byte** sayısını uzunluk olarak saklar; `setIndex` ise kullanıcının **UTF-16** index'ini bu byte sayısına karşı doğrular. ASCII olmayan her subject'te byte sayısı character sayısını aştığından, ikisinin arasına düşen bir index doğrulamadan geçer. Ardından `getUtf16PositionByCounter` o kadar character'ı buffer boyunca hiçbir bounds check yapmadan kat eder.

```js
const RE2 = require('re2');
const re = new RE2('a', 'y');   // 'g' also works
re.lastIndex = 3;               // 3 <= byteLen(4) passes; there are only 2 real chars
re.exec('éé');                  // ASAN: heap-buffer-overflow READ
```

Yayınlanan binary'de, büyük ve ASCII olmayan bir subject counter'ı buffer'ın çok ötesine, eşlenmemiş belleğe kadar sürükler ve process'i, hiçbir JavaScript handler'ının araya giremeyeceği bir SIGSEGV ile sonlandırır. Aynı `lastIndex` ASCII bir subject'e karşı normal şekilde döner; bu da nedeni boyutta değil, birim uyuşmazlığında konumlandırır. Sınırlı bir information leak'e de ulaşılabilir; ancak gösterdiğim over-read'ler sıfır döndürdüğünden, geçerli etki olarak crash'i raporladım ve bunu advisory'de de belirttim.

Bug hayatta kaldı çünkü ASCII güvenlidir: byte uzunluğu character sayısına eşittir, guard tesadüfen doğru çalışır ve sıradan testler geçer. CVE-2026-67550, CWE-125, Medium 5.7; o da 1.25.2'de düzeltildi.

Bu, kümedeki orkestrasyon açısından en zayıf vakadır ve olduğu gibi ifade edilmelidir: bug'ı kaynak okuma buldu, dispatch bulamazdı. Döngünün katkısı sonradan geldi; rapor dosyalanmadan önce döngü, crash'i güncel yayımlanan sürümün temiz bir kurulumunda, hem yayınlanan binary hem de kaynaktan bir ASAN build'i üzerinde iki kez yeniden doğruladı.

> **Ders: bir uzunluğun hangi birimde olduğu sorulmalıdır.** Bir niceliği ölçen bir validator, başka bir niceliği kat eden bir consumer'ı koruyorsa bu bir typo değil, bir bug sınıfıdır. Byte'a karşı character, UTF-8'e karşı UTF-16, count'a karşı size.

---

### Replace'te process seviyesinde bir abort

Bir crash-fuzzer üçüncü bug'a yaklaşık 150 vaka içinde ulaştı; gerçek binary'yi exit 134 için izliyordu. Onu yeniden üretmek iki nedenle epeyce daha uzun sürdü. PRNG `process.pid`'den seed'leniyordu, dolayısıyla hiçbir çalıştırma bir öncekini replay etmedi. Crash log'u da, crash 89KB'lik girdi gerektirirken girdiyi 400 character'da kesiyordu; arıza girdi-*boyutuna* bağlıydı ve boyut tam da atılmıştı. Minimize etme de benzer şekilde yanıltıcıydı: emoji, named group'lar ve sticky flag'in hepsi olmazsa olmaz gibi görünüyordu, hiçbiri değildi.

Minimize edilmiş vaka tek satırdır:

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

`WrappedRE2::Replace`, bir string `String::kMaxLength`'i aştığında V8'in döndürdüğü boş `MaybeLocal`'ı kontrol etmeden `.ToLocalChecked()` çağırır. Çıktıyı büyüten bir template — `$'` ya da `` $` `` — çıktıyı O(input²)'ye şişirir; sınır aşıldığında sonuç `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` ve **SIGABRT** olur — bir JavaScript exception'ı değil, process seviyesinde bir abort; bu nedenle `try`/`catch` onu tutamaz. 30k geçer, 40k crash olur; bu da N²/2'nin 536M'yi geçtiği noktadır.

Bunu belgelenmiş bir sınır değil de bir bug yapan şey karşılaştırmadır: aynı girdide native engine yakalanabilir bir `RangeError` fırlatır, re2 ise abort eder. Orkestratör minimize edilmiş vakayı hem o günkü güncel yayımlanan sürümün temiz bir kurulumunda hem de deponun kendi sürümünde yeniden üretti ve abort'un kendi ağacıma özgü olmadığını doğruladı. C++ kaynak-denetim ajanları bu hedef için sıraya alınmıştı ama hiç dispatch edilmedi, çünkü fuzzing oraya önce ulaştı. CVE-2026-71430, CWE-617, Medium 6.2.

> **Ders: zor yoldan öğrenilen fuzzing hijyeni.** PRNG seed'i sabit olmalı, asla `process.pid` olmamalıdır. Girdi log'ları tam ve kesilmemiş tutulmalıdır. Minimize işlemi deterministik yürütülmelidir, çünkü ilgi çekici görünen kısımlar çoğu zaman yalnızca süstür.

---

### stream-json'da quadratic path yeniden hesabı

360KB'lik bir doküman bir event loop'u on iki saniye boyunca bloke etti. İçeriği sıradandı: birkaç bin kez tekrarlanan `{"meta":`, bir `1` ve bunlara karşılık gelen kapanış parantezleri. Verildiği filtreyle hiçbir zaman match etmedi.

`stream-json`'ın path filtreleri — `pick`, `ignore`, `filter`, `replace` — kontrol edilebilir her token'da `stack.join(separator)`'ı baştan hesaplar. `stack.length` nesting depth olduğundan, D derinliğindeki bir doküman O(D²) maliyet çıkarır. Etkilenen yüzey, belgelenmiş birincil API'dir: `pick({filter: 'data'})`.

| İç içe geçme derinliği | Süre |
|---|---|
| 5,000 | 160 ms |
| 10,000 | 603 ms |
| 20,000 | 2,511 ms |
| 40,000 | 11,823 ms |

Her katlamada yaklaşık 4×; beklenen eğri. Bir-iki megabyte dakikalara ulaşır. Çözüm ucuzdur: birleştirilmiş path cache'lenir ve push ile pop'ta güncellenir.

Bug hayatta kaldı çünkü maliyeti *yapının* bir fonksiyonudur, oysa input guard'ları *byte'a* göre yazılır. 360KB'lik bir payload, endpoint'i koruyan size limiti her neyse onu geçer ve o yol boyunca derinliği sınırlayan hiçbir şey yoktur.

İş bölümü en net burada görünür. Orkestratör filtre kodunu okudu ve taşınabilir bir proof of concept'i temiz bir `stream-json@3.4.0`'a karşı yeniden çalıştırdı; bu, tek bir şüpheli zamanlama değil, yukarıdaki tabloyu üretti. Severity'i belirledim ve raporu 7.5 High olarak gönderdim; maintainer Medium 6.2 olarak yayımladı ve düşük skor savunulabilir, çünkü erişilebilirlik, derecelendirmemin hesaba katmadığı bir biçimde saldırgan kontrolündeki bir dokümana bağlıdır. CVE-2026-71429, CWE-407.

> **Ders: maliyet modelinin hangi boyutta olduğu kontrol edilmelidir.** Pahalı olan nicelik, validator'ların sınırladığı nicelik değilse, o validator'lar süsten ibarettir.

---

### Bir ders daha

**Engine'i değil, marshalling layer'ını denetleyin.** re2, ReDoS'tan *kaçınmak için* kurulur ve tam da bu itibar, onun elle yazılmış C++ N-API layer'ının incelenmemesinin nedenidir. Engine çekirdeği sağlam durdu. Bütün bug'lar binding'deydi; wrapper orada, engine'in önlemek için var olduğu sınıfı yeniden ortaya çıkarıyordu. Bu, runtime'lar arasında saldırgan kontrolündeki string'leri marshal eden her FFI ya da WASM layer'ı için genellenebilir.

### İş bölümü

Orkestratör review'ı dispatch etti, bulguları temiz kurulumlarda yeniden üretti ve iddiaları kontrollere karşı sınadı. Hedef seçimi, severity değerlendirmesi, crash minimizasyonu ve advisory'lerin kaleme alınması ise bana düşen kısımdı. İki bug fuzzing'den, iki tanesi okumaktan geldi; hiçbirini orkestrasyon tek başına üretmedi ve bu yazı da aksini iddia etmiyor.

En çok savunacağım kısım çürütmelerdir. stream-json'da dört aday incelendi ve biri raporlandı: prototype pollution, derin-nesting bir stack overflow ve sınırsız string buffering'in her biri çürütüldü; bir ölçüm artefaktı da — event loop'u aç bırakıp bir leak taklidi yapan senkron bir fuzzing döngüsü — kimseye ulaşmadan yakalandı. Bir maintainer'ın gelen kutusuna gürültü olarak hiç düşmeyen dört öğe; bu da yayına giren dört bulgu kadar değerlidir.

Skor tablosu da buna göre mütevazıdır. Dört bulgunun dördü Medium, dördü denial-of-service sınıfında ve hiçbiri remote code execution değil — write-primitive denetimi node-re2'nin hiçbir yerinde out-of-bounds write bulmadı, dolayısıyla bu sınıfın tavanı, zayıf bir read'le denial of service. Ortaya çıkan sonuç dört CVE tanımlayıcısı ve advisory'lerde kredidir. gossipcat'in kendi dependency pin'i PR #699'da 1.25.0 → 1.26.1'e taşındı; bu da döngüyü kapattı: denetlemeye çıktığım ağaç, yamalanması gereken ağacın ta kendisiydi.

### Egzersizin ürettikleri

Dördü de upstream'de yamalanmış dört advisory; hepsi de bakmadan çok önce zaten ağacımda olan paketlerde. Bu son kısım, dört CVE tanımlayıcısından daha uzun ömürlü bulgudur: `re2`'yi tam da güvenli seçim olduğu için kurmuştum ve düzeltmeye ihtiyaç duyan binding layer'a sahip bileşen o çıktı. Bir dependency, itibarı iyi olduğu için denetlenmez; biri oturup okuduğu için denetlenir.

Yöntem hedeften daha uzağa taşındı. Her iki re2 bug'ı da, codebase'i bilmeden herhangi bir native binding'e sorulabilecek sorulardan çıktı: *bu yapıyı kat eden alternatif yollardan hangisi diğerlerindeki guard'ı taşımamaktadır* ve *bu validator, consumer'ın kat ettiği şeye kıyasla hangi birimi ölçüyor*. stream-json bug'ı da aynı alışkanlığın maliyete uygulanmış hâlinden geldi — pahalı olan nicelik, herhangi bir şeyin sınırladığı nicelik değildi. Bu sorular taşınır; belirli bug'lar taşınmaz.
