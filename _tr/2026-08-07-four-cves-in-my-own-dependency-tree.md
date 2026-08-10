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
summary: "Kendi yazdığım multi-agent review sistemini kendi dependency tree'me doğrulttum — re2 ve stream-json, ikisi birlikte haftada ~12,1M npm indirmesi. Dört advisory kabul edildi; ama buradaki asıl mesele, sürecin bir bulguyu az kalsın çöpe attığı anlar dahil, işin nasıl yürüdüğünün dürüst hikâyesi."
---

> **Gösterilerle değil, gerçek hedeflerle çalışan AI orkestrasyonu üzerine bir serinin 1. Bölümü.**

### Soru

Önce sonuç: toplamda haftada 12,1 milyon civarı indirilen iki npm paketinde dört bulgu. Dördünü de maintainer kabul etti; advisory'ler yayımlandı, yamalar çıktı. Hepsi Medium, hepsi denial-of-service sınıfı.

Bütün bunları başlatan soru gayet ufaktı. [gossipcat](https://github.com/gossipcat-ai/gossipcat-ai) diye multi-agent bir code-review sunucusu yazıyordum; o güne kadar sadece kendi kodumu incelemişti. Merak ettim: kendi yazdığım koda değil de `npm install` ettiğim paketlere baksa ne bulur? Zinciri deponun kendi `node_modules`'ında birebir doğrulamak mümkündü: `gossipcat → re2 (^1.25.0) → install-artifact-from-github`. re2'yi zaten tam da regular-expression denial of service'ten kaçınmak için kurarsın; binding layer'ını bakmaya değer kılan da buydu. Denetim buradan sonra ağacı değil maintainer'ı takip etti, çünkü Eugene Lazutkin (`uhop`) aynı zamanda `stream-json`'ı da maintain ediyor.

Atıf konusunda bir not. re2 bulgularında hangi oturumun hangi bug'ı bulduğunu kayıtlarım söylemiyor, ben de iş işten geçtikten sonra bunu geri kurmaya çalışmayacağım. Aşağıda iki rol var: işi dağıtıp doğrulayan orkestratör, bir de ben.

---

### Global match döngüsünde ilerlemeyen bir cursor

ASAN altında koşan bir fuzzer, global flag'li `A*((((((((a)?)?))*)*)?)*)*` üzerinde yirmi bir dakikadır takılıydı ve 2.5GB tutuyordu. Ne sanitizer raporu vardı ne de crash; yani bu çalıştırmayı yavaş bir testten ayırmanın yolu yoktu.

Takılan process'i sample'ladığımda belirsizlik dağıldı: stack, sıkı bir döngü içinde `WrappedRE2::Match` → `RE2::Match` → `DFA::Search` gösteriyordu. Sebep `match.cc`'deki dört satır:

```cpp
while (re2->regexp.Match(str, byteIndex, str.size, anchor, &match, 1)) {
    groups.push_back(match);
    byteIndex = match.data() - str.data + match.size();  // += 0 for a zero-width match
}
```

Cursor, az önce bulduğu eşleşmenin uzunluğu kadar ilerliyor. Zero-width match'in uzunluğu olmadığından cursor kımıldamıyor ve aynı match sonsuza kadar dönüyor; bu arada `groups` bir `std::vector<StringPiece>`'i büyütmeye devam ediyor.

Patolojik desenin özel bir önemi yoktu. Boş string'i eşleyebilen herhangi bir global RE2 aynı şeyi tekrar üretiyor:

```js
const RE2 = require('re2');
'x'.match(new RE2('a*', 'g'));   // never returns
```

Yayınlanan binary'de bu, dört saniye civarında 736MB'tan 3.7GB'a çıkıyor ve işi ancak dışarıdan gelen bir `SIGKILL` bitiriyor. Çağrı senkron ve native olduğundan thread'i tutuyor; o yüzden `try`/`catch`, `AbortController` ve JS timer'ları hep birden işe yaramıyor — timer'lar da tetiklenmek için hiç sıra bulamadıklarından. V8 aynı deseni 0ms'de hallediyor.

Yüklü bir makinenin en kolay uydurduğu ölçüm bellek büyümesidir, o yüzden rakamı tek başına kabul etmedim. Orkestratör vakayı, boş string'i eşleyemeyen `a+` kontrolüne karşı yeniden çalıştırdı ve her process'i kendi resident set'i için sample'ladı: kusur 3860MB'a tırmanırken kontrol 44MB'ta sabit kaldı. Ayrıca döngüyü en son yayımlanan sürümün temiz bir kurulumunda, hem yayınlanan binary'de hem de kaynaktan derlenen bir build'de yeniden doğruladı. Raporu yazıp göndermek bana kaldı.

node-re2 aslında bu vakayı başka yerlerde zaten ele alıyor. Üç yol bir subject'i cursor'la geziyor: `split.cc` zero-width match'te bir code point ilerliyor, `exec` ise `lastIndex` üzerinden ilerliyor. Sadece global `Match` yolunda guard eksik. CVE-2026-68499, CWE-835, Medium 6.2, 1.25.2'de düzeltildi.

> **Ders: aynı yapıyı gezen bütün yolları yan yana koy.** Aynı iş birden fazla yerde yapılıyorsa, her yolun nasıl ilerlediğine ve nerede durduğuna bak — advance ve termination logic'lerini diff'le. Guard'ı eksik olan yol kusurlu olandır; sıfır uzunluklu elemanlar da bu hatanın klasik tetikleyicisi.

---

### Bir character index'ini doğrulayan byte uzunluğu

Fuzzer ikinci kusuru bulmadı, bulamazdı da. Ona ulaşmak için `re.lastIndex`'in belirli ve yanlış bir değere set edilmesi gerekiyor; rastgele girdi üreten bir şeyin o property'ye dokunmak için hiçbir sebebi yok. byte'tan character'a dönüşüm aritmetiğini okumak, kusuru dakikalar içinde yerine oturttu.

`lastIndex`'i kullanıcı istediği pozitif tam sayıya set edebiliyor. `prepareArgument`, subject'in **UTF-8 byte** sayısını uzunluk diye saklıyor; `setIndex` ise kullanıcının **UTF-16** index'ini bu byte sayısına karşı doğruluyor. ASCII olmayan her subject'te byte sayısı character sayısını aştığından, ikisinin arasına düşen bir index doğrulamadan geçiyor. Ardından `getUtf16PositionByCounter` o kadar character'ı buffer boyunca hiçbir bounds check yapmadan geziyor.

```js
const RE2 = require('re2');
const re = new RE2('a', 'y');   // 'g' also works
re.lastIndex = 3;               // 3 <= byteLen(4) passes; there are only 2 real chars
re.exec('éé');                  // ASAN: heap-buffer-overflow READ
```

Yayınlanan binary'de, büyük ve ASCII olmayan bir subject counter'ı buffer'ın çok ötesine, eşlenmemiş belleğe kadar sürüklüyor ve process'i, hiçbir JavaScript handler'ının araya giremeyeceği bir SIGSEGV ile öldürüyor. Aynı `lastIndex` ASCII bir subject'te sorunsuz dönüyor; bu da sebebi boyutta değil, birim uyuşmazlığında sabitliyor. Sınırlı bir information leak'e de ulaşılabiliyor ama gösterdiğim over-read'ler sıfır döndürdüğünden, geçerli etki olarak crash'i raporladım ve bunu advisory'de de açıkça yazdım.

Kusur hayatta kaldı çünkü ASCII güvenli: byte uzunluğu character sayısına eşit, guard tesadüfen doğru çalışıyor ve sıradan testler geçiyor. CVE-2026-67550, CWE-125, Medium 5.7; o da 1.25.2'de düzeltildi.

Bu, kümedeki en zayıf orkestrasyon vakası ve olduğu gibi söylemek lazım: kusuru kaynak okuma buldu, dispatch bulamazdı. Döngünün katkısı sonradan geldi; ben dosyalamadan önce döngü, crash'i güncel yayımlanan sürümün temiz bir kurulumunda, hem yayınlanan binary'de hem de kaynaktan bir ASAN build'inde iki kez yeniden doğruladı.

> **Ders: bir uzunluk hangi birimde, onu sor.** Bir niceliği ölçen bir validator, başka bir nicelikle gezen bir consumer'ı koruyorsa bu bir typo değil, bir bug sınıfıdır. Byte'a karşı character, UTF-8'e karşı UTF-16, count'a karşı size.

---

### Replace'te bir process abort

Bir crash-fuzzer üçüncü kusura yaklaşık 150 vakada ulaştı; gerçek binary'yi exit 134 için izliyordu. Onu yeniden üretmek iki sebepten epey daha uzun sürdü. PRNG `process.pid`'den seed'leniyordu, yani hiçbir çalıştırma bir öncekini replay etmedi. Bir de crash log'u, crash 89KB'lik girdi isterken girdiyi 400 character'da kesiyordu; arıza girdi-*boyutuna* bağlıydı ve tam da boyut atılmıştı. Minimize etmek de aynı şekilde yanıltıyordu: emoji, named group'lar ve sticky flag hepsi olmazsa olmaz gibi duruyordu, hiçbiri değildi.

Minimize edilmiş vaka tek satır:

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

`WrappedRE2::Replace`, bir string `String::kMaxLength`'i aştığında V8'in döndürdüğü boş `MaybeLocal`'ı kontrol etmeden `.ToLocalChecked()` çağırıyor. Çıktıyı büyüten bir template — `$'` ya da `` $` `` — çıktıyı O(input²)'ye şişiriyor; sınırı geçince sonuç `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` ve **SIGABRT** oluyor — bir JavaScript exception'ı değil, process seviyesinde bir abort; dolayısıyla `try`/`catch` onu tutamıyor. 30k geçiyor, 40k crash oluyor; yani N²/2'nin 536M'yi geçtiği nokta.

Bunu belgelenmiş bir sınır değil de kusur yapan şey karşılaştırma: aynı girdide native engine yakalanabilir bir `RangeError` fırlatıyor, re2 ise abort ediyor. Orkestratör minimize edilmiş vakayı hem o günkü güncel yayımlanan sürümün temiz bir kurulumunda hem de deponun kendi sürümünde yeniden üretti ve abort'un benim ağacıma özgü olmadığını doğruladı. C++ kaynak-denetim ajanları bu hedef için sıraya alınmıştı ama hiç dispatch edilmedi, çünkü fuzzing oraya önce vardı. CVE-2026-71430, CWE-617, Medium 6.2.

> **Ders: fuzzing hijyeni, zor yoldan öğrenildi.** Sabit PRNG seed'i, asla `process.pid` değil. Girdi log'ları tam ve kesilmemiş olacak. Deterministik minimize et, çünkü ilgi çekici görünen kısımlar çoğu zaman sadece süs.

---

### stream-json'da quadratic path yeniden hesabı

360KB'lik bir doküman bir event loop'u on iki saniye boyunca bloke etti. İçeriğinde ilginç bir şey yoktu: birkaç bin kez tekrarlanan `{"meta":`, bir `1` ve karşılık gelen kapanış parantezleri. Verildiği filtreyle hiç eşleşmedi bile.

`stream-json`'ın path filtreleri — `pick`, `ignore`, `filter`, `replace` — kontrol edilebilir her token'da `stack.join(separator)`'ı baştan hesaplıyor. `stack.length` nesting depth olduğundan, D derinliğindeki bir doküman O(D²) maliyet çıkarıyor. Etkilenen yüzey de belgelenmiş birincil API: `pick({filter: 'data'})`.

| İç içe geçme derinliği | Süre |
|---|---|
| 5,000 | 160 ms |
| 10,000 | 603 ms |
| 20,000 | 2,511 ms |
| 40,000 | 11,823 ms |

Her katlamada yaklaşık 4×; beklenen eğri. Bir-iki megabyte dakikalara çıkıyor. Çözüm ucuz: birleştirilmiş path'i cache'le, push ve pop'ta güncelle.

Kusur hayatta kaldı çünkü maliyeti *yapının* bir fonksiyonu, oysa input guard'ları *byte'a* göre yazılıyor. 360KB'lik bir payload, endpoint'i koruyan size limiti neyse onu geçiyor ve o yol boyunca derinliği sınırlayan hiçbir şey yok.

İş bölümü en net burada görünüyor. Orkestratör filtre kodunu okudu ve taşınabilir bir proof of concept'i temiz bir `stream-json@3.4.0`'a karşı yeniden çalıştırdı; ortaya tek bir şüpheli zamanlama değil, yukarıdaki tablo çıktı. Severity'i ben belirledim ve raporu 7.5 High olarak gönderdim; maintainer Medium 6.2 olarak yayımladı. Düşük skor da savunulabilir, çünkü erişilebilirlik, benim derecelendirmemin hesaba katmadığı ölçüde saldırgan kontrolündeki bir dokümana bağlı. CVE-2026-71429, CWE-407.

> **Ders: maliyet modelin hangi boyutta, onu kontrol et.** Pahalı olan nicelik, validator'larının sınırladığı nicelik değilse, o validator'lar süsten ibarettir.

---

### Bir ders daha

**Engine'i değil, marshalling layer'ını denetle.** re2, ReDoS'tan *kaçınmak için* kuruluyor ve tam da bu itibar yüzünden onun elle yazılmış C++ N-API layer'ına kimse bakmıyor. Engine çekirdeği sağlam durdu. Bütün kusurlar binding'deydi; wrapper orada, engine'in önlemek için var olduğu sınıfı bir daha ortaya çıkarıyordu. Bu, runtime'lar arasında saldırgan kontrolündeki string'leri marshal eden her FFI ya da WASM layer'ı için geçerli.

### İş bölümü

Orkestratör review'ı dağıttı, bulguları temiz kurulumlarda yeniden üretti ve iddiaları kontrollere karşı sınadı. Hedefi ben seçtim, severity'i ben biçtim, crash'i ben minimize ettim, advisory'leri ben yazdım. İki kusur fuzzing'den, iki tanesi okumaktan geldi; hiçbirini orkestrasyon tek başına üretmedi ve bu yazı da aksini iddia etmiyor.

En çok savunacağım kısım çürütmeler. stream-json'da dört aday incelendi, biri raporlandı: prototype pollution, derin-nesting bir stack overflow ve sınırsız string buffering'in her biri çürütüldü; bir ölçüm artefaktı da — event loop'u aç bırakıp leak taklidi yapan senkron bir fuzzing döngüsü — kimseye ulaşmadan yakalandı. Bir maintainer'ın gelen kutusuna gürültü olarak hiç düşmeyen dört öğe; bu da yayına giren dört bulgu kadar değerli.

Skor tablosu da buna göre mütevazı. Dört bulgunun dördü Medium, dördü denial-of-service sınıfında ve hiçbiri remote code execution değil — write-primitive denetimi node-re2'nin hiçbir yerinde out-of-bounds write bulmadı, dolayısıyla bu sınıfın tavanı, zayıf bir read'le denial of service. Ortaya çıkan şey dört CVE tanımlayıcısı ve advisory'lerde kredi. gossipcat'in kendi dependency pin'i PR #699'da 1.25.0 → 1.26.1'e taşındı; bu da döngüyü kapattı: denetlemeye çıktığım ağaç, yamalanması gereken ağacın ta kendisiydi.

### Egzersiz ne üretti

Dördü de upstream'de yamalanmış dört advisory; hepsi de ben bakmadan çok önce zaten ağacımda olan paketlerde. Asıl kalıcı bulgu bu son kısım, dört CVE tanımlayıcısından da uzun ömürlü: `re2`'yi tam da güvenli seçim olduğu için kurmuştum ve düzeltmeye ihtiyaç duyan binding layer onunki çıktı. Bir dependency, itibarı iyi diye denetlenmez; biri oturup okuduğu için denetlenir.

Yöntem hedeften daha uzağa taşındı. Her iki re2 kusuru da, codebase'i hiç bilmeden herhangi bir native binding'e sorulabilecek sorulardan çıktı: *bu yapıyı gezen yollardan hangisi ötekilerin guard'ını taşımıyor* ve *bu validator, consumer'ın gezdiği şeye kıyasla hangi birimi ölçüyor*. stream-json kusuru da aynı alışkanlığın maliyete uygulanmış hali — pahalı olan nicelik, herhangi bir şeyin sınırladığı nicelik değildi. Bu sorular taşınır; belirli bug'lar taşınmaz.
