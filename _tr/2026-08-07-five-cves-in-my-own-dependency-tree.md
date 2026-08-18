---
layout: post
lang: tr
title: "AI Orchestration, Episode 1: Five CVEs in the Tree I Was Standing On"
date: 2026-08-07
permalink: /tr/ai-research/2026/08/07/five-cves-in-my-own-dependency-tree.html
author: ataberk-xyz
categories: [ai-research]
tags: [orchestration, gossipcat, npm, cve, fuzzing, supply-chain]
ledger:
  target: "re2 (node-re2) · stream-json · install-artifact-from-github — toplamda haftada ~13,4M indirme"
  status: "PATCHED — re2 1.25.2, stream-json 3.5.0, install-artifact-from-github 1.7.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
summary: "Kendi geliştirdiğim multi-agent review sistemini kendi dependency tree'me yönelttim — re2, stream-json ve re2'nin kendi binary fetch işini devrettiği installer. Beşi de kabul edilen advisory olarak çıktı, biri install-time code execution; bu yazı da, gerçekte bulundukları sırayla, sürecin bir bulguyu az kalsın elden çıkardığı anlar dahil, işin nasıl yürüdüğünün dürüst bir anlatısı."
---

> **Gösteri amaçlı örnekler yerine gerçek hedeflere uygulanan AI orkestrasyonu üzerine bir serinin 1. Bölümü.**

<div class="adv-wrap">
<table class="adv">
  <thead>
    <tr><th>Advisory</th><th>Package</th><th>Severity</th><th>Impact</th></tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x">CVE-2026-71429</a></td>
      <td>stream-json</td>
      <td><span class="sv sv-med">MEDIUM 6.2</span></td>
      <td><span class="cwe">CWE-407</span> event-loop DoS</td>
    </tr>
    <tr>
      <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-8hcv-x26h-mcgp">CVE-2026-71430</a></td>
      <td>node-re2</td>
      <td><span class="sv sv-med">MEDIUM 6.2</span></td>
      <td><span class="cwe">CWE-617</span> process abort</td>
    </tr>
    <tr>
      <td><a href="https://github.com/uhop/install-artifact-from-github/security/advisories/GHSA-88q3-gch3-5396">CVE-2026-73864</a></td>
      <td>install-artifact-from-github</td>
      <td><span class="sv sv-high">HIGH 7.5</span></td>
      <td><span class="cwe">CWE-494</span> install-time RCE</td>
    </tr>
    <tr>
      <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-ff84-5f28-78qj">CVE-2026-67550</a></td>
      <td>node-re2</td>
      <td><span class="sv sv-med">MEDIUM 5.7</span></td>
      <td><span class="cwe">CWE-125</span> OOB read / crash</td>
    </tr>
    <tr>
      <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-6hxr-mr5r-9836">CVE-2026-68499</a></td>
      <td>node-re2</td>
      <td><span class="sv sv-med">MEDIUM 6.2</span></td>
      <td><span class="cwe">CWE-835</span> infinite loop / memory DoS</td>
    </tr>
  </tbody>
</table>
</div>

### Başlangıç noktası

Sonuç önce. Bir dependency chain içindeki üç npm paketinde beş bulgu; hepsi maintainer tarafından kabul edildi, yayımlandı ve yamalandı. Dördü, denetlemeye çıktığım iki pakette, Medium seviyede ve denial-of-service sınıfında. İstisna High — regex engine'de değil, onu getiren tooling'de bulunan bir install-time code execution bug'ı.

Çalışmanın kapsamı dar bir soruyla belirlendi: yalnızca kendi kodum üzerinde kullanılan multi-agent code-review sistemi [gossipcat](https://github.com/gossipcat-ai/gossipcat-ai), kurulu bağımlılıklara yöneltildiğinde ne bulur? İnceleme zinciri deponun node_modules dizininde doğrulanabilir durumdaydı: gossipcat → re2 (^1.25.0) → install-artifact-from-github. re2'nin seçilme gerekçesi önemlidir: regular-expression denial-of-service riskine karşı kurulan bir paketin binding layer'ı, tam da bu itibarı nedeniyle incelenmeye değerdi. stream-json, her iki paketin de maintainer'ı olan Eugene Lazutkin (uhop) üzerinden kapsama girdi — zincir bir adım daha ileri gitti, re2'nin kendi native-binary fetch işini sessizce devrettiği install-artifact-from-github'a kadar.

Atıfla ilgili bir not düşmek gerekir. re2 bulgularında hangi oturumun hangi zafiyeti ortaya çıkardığı kayıtlarımda ayırt edilememektedir; bu ayrımı sonradan yeniden kurmaya çalışmayacağım. Aşağıda iki rol yer almaktadır: işi dispatch eden ve doğrulayan orkestratör ile ben. Aşağıdaki sıralama yukarıdaki zincire göre değil, her advisory'nin gerçekte açıldığı zamana göre.

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

Bu bug, maliyeti *yapının* bir fonksiyonu olduğu, input guard'ları ise *byte'a* göre yazıldığı için ayakta kaldı. 360KB'lik bir payload, endpoint'i koruyan size limiti her neyse onu geçer ve o yol boyunca derinliği sınırlayan hiçbir şey yoktur.

Roller burada en belirgin biçimde ayrışır. Orkestratör filtre kodunu okudu ve taşınabilir bir proof of concept'i temiz bir `stream-json@3.4.0`'a karşı yeniden çalıştırdı; bu, tek bir şüpheli zamanlama değil, yukarıdaki tabloyu üretti. Severity'i belirledim ve raporu 7.5 High olarak gönderdim; maintainer Medium 6.2 olarak yayımladı ve düşük skor savunulabilir, çünkü erişilebilirlik, derecelendirmemin hesaba katmadığı bir biçimde saldırgan kontrolündeki bir dokümana bağlıdır. CVE-2026-71429, CWE-407.

> **Ders: maliyet modelinin hangi boyutta olduğu kontrol edilmelidir.** Pahalı olan nicelik, validator'ların sınırladığı nicelik değilse, o validator'lar süsten ibarettir.

---

### Replace'te process seviyesinde bir abort

Bir crash-fuzzer bu bug'a yaklaşık 150 vaka içinde ulaştı; gerçek binary'yi exit 134 için izliyordu. Onu yeniden üretmek iki nedenle epeyce daha uzun sürdü. PRNG `process.pid`'den seed'leniyordu, dolayısıyla hiçbir çalıştırma bir öncekini replay etmedi. Crash log'u da, crash 89KB'lik girdi gerektirirken girdiyi 400 character'da kesiyordu; arıza girdinin *boyutuna* bağlıydı ve atılan şey tam da boyuttu. Minimize etme de benzer şekilde yanıltıcıydı: emoji, named group'lar ve sticky flag'in hepsi olmazsa olmaz gibi görünüyordu, hiçbiri değildi.

Minimize edilmiş vaka tek satırdır:

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

`WrappedRE2::Replace`, bir string `String::kMaxLength`'i aştığında V8'in döndürdüğü boş `MaybeLocal`'ı kontrol etmeden `.ToLocalChecked()` çağırır. Çıktıyı büyüten bir template — `$'` ya da `` $` `` — onu O(input²)'ye şişirir; sınır aşıldığında sonuç `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` ve **SIGABRT** olur — bir JavaScript exception'ı değil, process seviyesinde bir abort; bu nedenle `try`/`catch` onu tutamaz. 30k geçer, 40k crash olur; bu da N²/2'nin 536M'yi geçtiği noktadır.

Bunu belgelenmiş bir sınır değil de bir bug yapan şey karşılaştırmadır: aynı girdide native engine yakalanabilir bir `RangeError` fırlatır, re2 ise abort eder. Orkestratör minimize edilmiş vakayı hem o günkü güncel yayımlanan sürümün temiz bir kurulumunda hem de deponun kendi sürümünde yeniden üretti ve abort'un kendi ağacıma özgü olmadığını doğruladı. C++ kaynak-denetim ajanları bu hedef için sıraya alınmıştı ama hiç dispatch edilmedi, çünkü fuzzing oraya önce ulaştı. CVE-2026-71430, CWE-617, Medium 6.2.

> **Ders: zor yoldan öğrenilen fuzzing hijyeni.** PRNG seed'i sabit olmalı, asla `process.pid` olmamalıdır. Girdi log'ları tam ve kesilmemiş tutulmalıdır. Minimize işlemi deterministik yürütülmelidir, çünkü ilgi çekici görünen kısımlar çoğu zaman yalnızca süstür.

---

### Hiçbir şeyin doğrulamadığı bir indirilen artifact

Açılış paragrafındaki zincirin, henüz takip etmediğim üçüncü bir adımı var. re2 varsayılan olarak native binding'ini derlemez — önceden derlenmiş bir `.node` dosyasını indirir; bu indirme işi de ayrı bir pakete, `install-artifact-from-github`'a, `install-from-cache --artifact build/Release/re2.node --host-var RE2_DOWNLOAD_MIRROR ...` üzerinden devredilir. Bu install script'i önüme koyan şey gossipcat'in kendi dependency tree'sini taramaktı — bir fuzzer değil, bir okuma daha.

`bin/install-from-cache.js` artifact'ı network üzerinden indirir ve doğrudan diske yazar. Hiçbir checksum, signature ya da SRI yok — paketin tamamında `createHash|sha256|checksum|integrity|digest|signature` için yapılan bir grep hiçbir şey döndürmüyor:

```js
// bin/install-from-cache.js
let assetUrl = mirrorHost || process.env[mirrorEnvVar] || 'https://github.com';
// ...
const write = async (name, data) => {
  await fsp.mkdir(path.dirname(name), {recursive: true});
  await fsp.writeFile(name, data);   // data is the raw HTTP response body
};
```

`mirrorEnvVar` tamamen saldırgan tarafından yönlendirilebilir ve hiçbir host'a pinlenmemiş: re2, `--host-var RE2_DOWNLOAD_MIRROR` geçiyor; dolayısıyla bu değişkeni set edebilen herhangi bir şey — CI config, bir `.npmrc` env passthrough'u, ele geçirilmiş bir CI job'ı, zehirlenmiş bir shell profile'ı — tüm indirmeyi keyfi bir host'a yönlendirir. Transfer ayrıca plaintext `http://`'yi de kabul ediyor ve bir redirect, orijinal scheme'i pinlemek yerine protokolünü yeni URL'den yeniden türetiyor; yani `https://` bir asset'in `http://`'ye 302 ile yönlenmesi sessizce takip ediliyor — hiçbir uyarı, hiçbir downgrade guard'ı yok.

Paket bir post-download kontrolü de çalıştırıyor — `verify-build` ya da `npm test` — ama bu, dosya zaten yazıldıktan sonra oluyor; re2 için bu kontrolün ilk satırı da doğrulaması gereken binary'yi `require()` etmek. Native module-init kodu, herhangi bir kontrol çalışana kadar zaten çalışmış oluyor. Bu bir smoke test, bir gate değil.

Proof of concept kasıtlı olarak zararsız: plaintext HTTP üzerinde sentinel byte'lar servis eden bir "mirror" ayağa kaldırıp `RE2_DOWNLOAD_MIRROR`'ı ona yönlendirmek ve servis edilen byte'ların diske değişmeden düştüğünü göstermek yeterli.

```
attacker mirror (plaintext http) on 127.0.0.1:PORT
  [mirror] served 2083 attacker bytes over HTTP for /uhop/node-re2/releases/download/1.24.1/darwin-arm64-137
served   sha256: ec6fdde6...4ebc
written  sha256: ec6fdde6...4ebc
install-from-cache exit: 0
```

Kurulum başarıyla tamamlanıyor. Gerçek bir `.node` dosyası, `require()` edildiği anda native init kodunu çalıştırır; dolayısıyla birini yerine koymak, kurulum ya da çalışma sürecinde code execution demektir — bu denetimin asıl konusu olan node-re2'nin kendi kodu daha çalışmadan önce.

CVE-2026-73864, GHSA-88q3-gch3-5396, CWE-494, High 7.5. Advisory, 2026-07-07 06:07 UTC'de açıldı — yukarıdaki stream-json ve Replace-abort bulgularından sonra, aşağıdaki iki re2 bulgusundan önce.

Teorik de değildi. `install-artifact-from-github`, gerçek ağaçlarda re2'nin altında sıkça yer alıyor ve bu kontrol sırasında pre-fix sürümleri hâlâ birkaç yerde pinliydi: `PostHog/posthog` (37,7k star — `pnpm-lock.yaml`'da `re2@1.22.1`/`1.24.1`), `coralproject/talk` (2k star — `re2@1.21.4`) ve `google-labs-code/stitch-sdk` (1,8k star, bir Google org'u — `re2@1.23.2`). `democratic-csi/democratic-csi` (1,3k star) daha keskin bir örnek: Dockerfile'ı şunu set ediyor:

```
ENV RE2_DOWNLOAD_MIRROR="https://grpc-uds-binaries.s3-us-west-2.amazonaws.com/re2"
```

— advisory'nin attack surface olarak adlandırdığı değişkenin ta kendisi, gerçek ve tamamen meşru bir build-speed mirror'ı olarak kullanımda. `renovatebot/renovate` (22,3k star) bir dependent değil, ama kendi Docker build'i bu paket için özel yazılmış bir comment taşıyor (`# set npm_config_platform_arch for install-artifact-from-github`) — maintainer'ların bu paketin install davranışı üzerine, bu bug'la ilgisiz şekilde, zaten düşündüğünün kanıtı.

Fix hızlı geldi: 1.7.0, advisory açıldıktan yaklaşık 20,5 saat sonra yayımlandı ve tek satırlık bir patch değil. Tüketen paket artık publish anında kendi `package.json`'ına platform slot başına bir SHA-256 digest damgalıyor; `install-from-cache`, yazmadan önce indirmeyi bu digest'e karşı kontrol ediyor ve uyuşmazlıkta kaynaktan derlemeye düşüyor. Bu, aceleye getirilmiş değil, hızlı ve iyi kapsamlanmış bir yanıt.

Açıkça belirtilmesi gereken bir çekince var, çünkü bu tasarıma bir eleştiri değil, spesifik bir durum: doğrulama yalnızca varsayılan `github.com` kaynağını kapsıyor. Yukarıdaki `RE2_DOWNLOAD_MIRROR` örneğindeki gibi açıkça yapılandırılmış bir mirror, tasarım gereği trust-the-mirror olarak kalıyor, çünkü tüketen paketin kontrol etmediği bir host için önceden bir digest damgalamanın bir yolu yok. `democratic-csi`, 1.7.0'a tamamen güncellense bile, tam da bu yüzden yeni kontrolün kapsamı dışında kalıyor. Keyfi, kullanıcı tarafından yapılandırılmış bir kaynağı doğrulamak, sabit bir kaynağı doğrulamakla aynı problem değil ve fix'i bu şekilde kapsamlamak makul bir karar — ama kapsamı ile gerçek dünyadaki kullanımı tam örtüşmüyor.

> **Ders: kodu değil, kodu teslim eden şeyi de denetleyin.** Bir binding layer uçtan uca memory-safe olabilir ve yine de açık olabilir, eğer çalışmadan önce onu oluşturan byte'ları hiçbir şey kontrol etmiyorsa. Install time'da önceden derlenmiş bir artifact indiren her paket, üzerine inşa edenler öyle düşünsün ya da düşünmesin, bir supply-chain edge'idir.

---

### Bir character index'ini doğrulayan byte uzunluğu

Fuzzer bu bug'ı bulmadı, bulamazdı da. Ona ulaşmak, `re.lastIndex`'in belirli ve yanlış bir değere set edilmesini gerektirir; rastgele girdi üreten bir mekanizmanın o property'ye dokunmak için hiçbir nedeni yoktur. byte-character dönüşüm aritmetiğini okumak bug'ı dakikalar içinde ortaya çıkardı.

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

### Global match döngüsünde ilerlemeyen bir cursor

ASAN altında çalışan bir fuzzer, global flag ile `A*((((((((a)?)?))*)*)?)*)*` üzerinde yirmi bir dakika boyunca meşguldü ve 2.5GB tutuyordu. Ne bir sanitizer raporu ne de bir crash vardı; dolayısıyla çalıştırma, yavaş bir testten ayırt edilemiyordu.

Takılan process'in sample'lanması belirsizliği giderdi: stack, sıkı bir döngü içinde `WrappedRE2::Match` → `RE2::Match` → `DFA::Search` gösteriyordu. Nedeni `match.cc`'deki şu dört satır:

```cpp
while (re2->regexp.Match(str, byteIndex, str.size, anchor, &match, 1)) {
    groups.push_back(match);
    byteIndex = match.data() - str.data + match.size();  // += 0 for a zero-width match
}
```

Cursor, az önce bulunan match'in uzunluğu kadar ilerler. Zero-width match'in uzunluğu olmadığından cursor yerinden kımıldamaz ve aynı match süresiz olarak döndürülürken `groups` bir `std::vector<StringPiece>`'i büyütmeye devam eder.

Patolojik desen tesadüfiydi. Boş string'i match edebilen herhangi bir global RE2 bunu yeniden üretir:

```js
const RE2 = require('re2');
'x'.match(new RE2('a*', 'g'));   // never returns
```

Yayınlanan binary'de bu, yaklaşık dört saniye içinde 736MB'tan 3.7GB'a çıkar ve süreci ancak dışarıdan gelen bir `SIGKILL` sonlandırır. Çağrı senkron ve native olduğu için thread'i tutar; bu nedenle `try`/`catch`, `AbortController` ve JS timer'larının tamamı işlevsiz kalır — timer'lar, tetiklenmek için hiç sıra bulamadıklarından. V8 aynı deseni 0ms'de işler.

Bellek büyümesi, yüklü bir makinenin en kolay uydurabildiği ölçümdür; bu nedenle rakam tek başına kabul edilmedi. Orkestratör vakayı, boş string'i match edemeyen `a+` kontrolüne karşı yeniden çalıştırdı ve her process'in resident set'ini ayrı ayrı sample'ladı: bug 3860MB'a tırmanırken kontrol 44MB'ta sabit kaldı. Ayrıca döngüyü, en son yayımlanan sürümün temiz bir kurulumunda, hem yayınlanan binary hem de kaynaktan derlenmiş bir build üzerinde yeniden doğruladı. Dosyalama ve gönderim bana aitti.

node-re2 bu vakayı başka yerlerde zaten ele alır. Üç yol bir subject'i cursor ile kat eder: `split.cc` zero-width match'te bir code point ilerler, `exec` ise `lastIndex` üzerinden ilerler. Guard'ı yalnızca global `Match` yolu atlar. CVE-2026-68499, CWE-835, Medium 6.2, 1.25.2'de düzeltildi.

> **Ders: aynı yapıyı kat eden alternatif yollar tek tek incelenmelidir.** Bir yapı birden fazla yerde kat ediliyorsa, bu yolların advance ve termination logic'leri karşılaştırılmalıdır. Diğer yolların taşıdığı guard'ı taşımayan yol, bug'ın bulunduğu yoldur; sıfır uzunluklu elemanlar da bunun klasik tetikleyicisidir.

---

### Bir ders daha

**Engine'i değil, marshalling layer'ını denetleyin.** re2, ReDoS'tan *kaçınmak için* kurulur; elle yazılmış C++ N-API layer'ının hiç incelenmemesi de işte bu itibardan ileri gelir. Engine çekirdeği sağlam durdu. node-re2'nin kendi içindeki bütün bug'lar binding'deydi; wrapper orada, engine'in önlemek için var olduğu sınıfı yeniden ortaya çıkarıyordu. Bu, runtime'lar arasında saldırgan kontrolündeki string'leri marshal eden her FFI ya da WASM layer'ı için genellenebilir.

### İş bölümü

Orkestratör review'ı dispatch etti, bulguları temiz kurulumlarda yeniden üretti ve iddiaları kontrollere karşı sınadı. Hedef seçimi, severity değerlendirmesi, crash minimizasyonu ve advisory'lerin kaleme alınması ise bana düşen kısımdı. İki bug fuzzing'den, üç tanesi okumaktan geldi — install-artifact-from-github bulgusu, bir fuzzing çalıştırması değil, orkestratörün dependency chain üzerindeki kendi review pass'i sırasında ortaya çıktı; bir code-review aracı için gayet yerinde bir bulma şekli. Hiçbirini orkestrasyon tek başına üretmedi ve bu yazı da aksini iddia etmiyor.

En çok savunacağım kısım çürütmelerdir. stream-json'da dört aday incelendi ve biri raporlandı: prototype pollution, derin-nesting bir stack overflow ve sınırsız string buffering'in her biri çürütüldü; bir ölçüm artefaktı da — event loop'u aç bırakıp bir leak taklidi yapan senkron bir fuzzing döngüsü — kimseye ulaşmadan yakalandı. Bir maintainer'ın gelen kutusuna gürültü olarak hiç düşmeyen dört öğe; bu da yayına giren beş bulgu kadar değerlidir.

Skor tablosu da, bir istisna dışında, buna göre mütevazı. Beş bulgunun dördü, node-re2 ve stream-json'ın kendi runtime kodunda, Medium seviyede ve denial-of-service sınıfında — write-primitive denetimi node-re2'nin hiçbir yerinde out-of-bounds write bulmadı, dolayısıyla özelde bu engine için tavan, zayıf bir read'le denial of service. İstisna bu tavanı kırıyor: High seviyede, install-time code execution — ama engine'de değil, binary'yi teslim eden tooling'de; diğer dördün temsil ettiği memory-safety işinde bir delik değil, farklı bir pakette farklı bir bug sınıfı. Sonuç, beş CVE tanımlayıcısı ve advisory'lerde kredi. gossipcat'in kendi dependency chain'i her zafiyetli pin'den kurtuldu — re2, PR #699'da 1.25.0 → 1.26.1'e taşındı, install-artifact-from-github da onunla birlikte 1.7.0'a çekildi — bu da döngüyü kapattı: denetlemeye çıktığım ağaç, yamalanması gereken ağacın ta kendisiydi.

### Egzersizin ürettikleri

Beşi de upstream'de yamalanmış beş advisory; hepsi de bakmadan çok önce zaten ağacımda olan paketlerde. Asıl kalıcı bulgu, beş CVE tanımlayıcısından daha uzun ömürlüdür ve tek cümleye sığar: re2'yi tam olarak güvenli seçim olduğu için kurmuştum ve binding layer'ı düzeltilmesi gereken bileşen oydu — kendi teslimatını sessizce devrettiği araç ise daha da fazlasına ihtiyaç duyuyordu. Bir dependency, itibarı iyi olduğu için denetlenmez; biri oturup, install script'ine kadar, okuduğu için denetlenir.

Yöntem hedeften daha uzağa taşındı. İki re2 memory-safety bug'ı da, codebase'i bilmeden herhangi bir native binding'e sorulabilecek sorulardan çıktı: *bu yapıyı kat eden alternatif yollardan hangisi diğerlerindeki guard'ı taşımıyor* ve *bu validator, consumer'ın kat ettiği şeye kıyasla hangi birimi ölçüyor*. stream-json bug'ı da aynı alışkanlığın maliyete uygulanmış hâlinden geldi — pahalı olan nicelik, herhangi bir şeyin sınırladığı nicelik değildi. install-artifact-from-github bug'ı ise aynı alışkanlığın trust'a uygulanmasından geldi: *diske düşen byte'ları, çalışmadan önce ne doğruluyor?* Hiçbir şey doğrulamıyordu — var olan tek kontrol, artifact zaten yüklendikten sonra çalışıyordu. Bu soru, ekosistem ya da dil fark etmeksizin, install time'da bir şey indiren her pakete uygulanabilir. Bu sorular taşınır; belirli bug'lar taşınmaz.

---

### Kaynaklar

- **[CVE-2026-71429](https://www.cve.org/CVERecord?id=CVE-2026-71429)** · [GHSA-528h-pc64-c93x](https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x) — stream-json: `pick`/`ignore`/`filter`/`replace` filters are O(depth²) on nested input — small crafted JSON blocks the event loop for seconds→minutes (DoS). CWE-407, Medium 6.2. 3.5.0'de düzeltildi.
- **[CVE-2026-71430](https://www.cve.org/CVERecord?id=CVE-2026-71430)** · [GHSA-8hcv-x26h-mcgp](https://github.com/advisories/GHSA-8hcv-x26h-mcgp) — node-re2: `String.prototype.replace(re2, template)` aborts the Node process (uncatchable `ToLocalChecked` on an empty `MaybeLocal`) when the result exceeds V8's max string length. CWE-617, Medium 6.2. 1.25.2'de düzeltildi.
- **[CVE-2026-73864](https://www.cve.org/CVERecord?id=CVE-2026-73864)** · [GHSA-88q3-gch3-5396](https://github.com/advisories/GHSA-88q3-gch3-5396) — install-artifact-from-github: `install-from-cache` writes and loads a network-downloaded native `.node` with no integrity check (env-controlled mirror + plaintext-HTTP redirect downgrade) → arbitrary code execution at install. CWE-494, High 7.5. 1.7.0'de düzeltildi.
- **[CVE-2026-67550](https://www.cve.org/CVERecord?id=CVE-2026-67550)** · [GHSA-ff84-5f28-78qj](https://github.com/advisories/GHSA-ff84-5f28-78qj) — re2: out-of-bounds heap read in `exec`/`test`/`match` via attacker-influenced `lastIndex` on a non-ASCII subject → uncatchable process crash (DoS). CWE-125, Medium 5.7. 1.25.2'de düzeltildi.
- **[CVE-2026-68499](https://www.cve.org/CVERecord?id=CVE-2026-68499)** · [GHSA-6hxr-mr5r-9836](https://github.com/advisories/GHSA-6hxr-mr5r-9836) — re2: global `String.prototype.match` with an empty-matchable pattern never advances → infinite loop with unbounded native memory growth (DoS). CWE-835, Medium 6.2. 1.25.2'de düzeltildi.
