---
layout: post
lang: tr
title: "AI Orchestration, 1. Bölüm: Ayağımın Altındaki Ağaçta Beş CVE"
date: 2026-08-19
permalink: /tr/ai-research/2026/08/19/five-cves-in-my-own-dependency-tree.html
author: ataberk-xyz
categories: [ai-research]
tags: [orchestration, gossipcat, npm, cve, fuzzing, supply-chain]
ledger:
  target: "re2 (node-re2) · stream-json · install-artifact-from-github (toplamda haftada ~13,4M indirme)"
  status: "PATCHED: re2 1.25.2, stream-json 3.5.0, install-artifact-from-github 1.7.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
summary: "Kendi yazdığım multi-agent review sistemini bu sefer kendi dependency tree'me çevirdim: hedefte re2, stream-json, bir de re2'nin binary indirme işini devrettiği installer vardı. Sonuçta beş advisory kabul edildi, biri install-time code execution kadar ciddiydi. Bu yazı bulguları gerçekten bulunma sırasıyla anlatıyor; bir bulguyu neredeyse çöpe atacağımız anlar dahil, işin perde arkasının dürüst bir dökümü."
---

> **Bir serinin 1. Bölümü: AI orkestrasyonunu vitrin örnekleri yerine gerçek hedeflere uyguluyoruz.**

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

Önce şunu belirtelim: haftada toplam yaklaşık 13,4 milyon indirmeye sahip üç npm paketinde toplam beş bulgu çıktı, hepsi maintainer tarafından kabul edilip yayımlandı ve yamalandı. Dördü, asıl denetlemeyi planladığım iki pakette, Medium seviyede, denial-of-service sınıfında. Beşincisi ise kuralı bozuyor: High seviyede, üstelik regex engine'in kendisinde değil, onu getiren tooling'de çıkan bir install-time code execution açığı.

İşin çıkış noktası tek bir soruydu: gossipcat (şimdiye kadar yalnızca kendi yazdığım kodu incelemiş olan multi-agent code-review sistemim) kurulu bağımlılıklara yöneltilirse ne bulur? Zincir, deponun node_modules'ünde gayet net görünüyordu: gossipcat → re2 (^1.25.0) → install-artifact-from-github. re2'yi seçmemin özel bir sebebi vardı: paket, regular-expression denial-of-service riskinden tam olarak kaçınmak için kuruluyor, binding layer'ının hiç incelenmemiş olması da tam olarak bu itibardan kaynaklanıyordu. stream-json ise farklı bir yoldan girdi kapsama: her iki paketin maintainer'ı da aynı kişi, Eugene Lazutkin (uhop). Zincir orada da durmadı, bir adım daha ileri gitti: re2'nin native-binary indirme işini sessizce devrettiği install-artifact-from-github'a kadar uzandı.

| Paket | Haftalık indirme |
|---|---|
| stream-json | ~7,8M |
| install-artifact-from-github | ~2,8M |
| re2 | ~2,8M |
| **Toplam** | **~13,4M** |

---

### 1. stream-json'da quadratic path yeniden hesabı

360KB'lik sıradan bir doküman, bir event loop'u tam on iki saniye kilitledi. İçeriğinde göz alıcı hiçbir şey yoktu: `{"meta":` birkaç bin kez tekrarlanmış, bir `1`, bir de karşılık gelen kapanış parantezleri. Üstelik verildiği filtreyle hiçbir zaman eşleşmedi bile.

Sebep şu: `stream-json`'ın path filtreleri (`pick`, `ignore`, `filter`, `replace`) kontrol ettikleri her token'da `stack.join(separator)`'ı sıfırdan yeniden hesaplıyor. `stack.length` doğrudan nesting depth'e karşılık geldiğinden, D derinliğindeki bir doküman O(D²) maliyete çıkıyor. Hem de bu, kütüphanenin belgelenmiş, birincil API'si olan `pick({filter: 'data'})` üzerinden gerçekleşiyor.

| İç içe geçme derinliği | Süre |
|---|---|
| 5,000 | 160 ms |
| 10,000 | 603 ms |
| 20,000 | 2,511 ms |
| 40,000 | 11,823 ms |

Her katlamada kabaca 4× artış. Beklenen eğri tam olarak bu. Bir-iki megabyte'a gelindiğinde süre dakikalara çıkıyor. Çözümü ise ucuz: birleştirilmiş path'i cache'le, push ve pop'ta güncelle, yeter.

Bug'ın bugüne kadar fark edilmemesinin sebebi basit: maliyet *yapının* bir fonksiyonu, ama input guard'ları hep *byte* üzerinden yazılmış. 360KB'lik bir payload, endpoint'i koruyan boyut limitini rahatça geçiyor, çünkü o yol boyunca derinliği sınırlayan hiçbir mekanizma yok.

İş bölümü burada en net haliyle ortaya çıkıyor. Filtre kodunu orkestratör okudu, taşınabilir bir proof of concept'i temiz bir `stream-json@3.4.0` kurulumuna karşı yeniden çalıştırdı: ortaya tek bir şüpheli zamanlama değil, doğrudan yukarıdaki tablo çıktı. Severity'yi ben belirledim, raporu 7.5 High olarak gönderdim; maintainer ise Medium 6.2 olarak yayımladı. Bu düşük skor da savunulabilir aslında, çünkü erişilebilirlik saldırgan kontrolündeki bir dokümana bağlı, benim derecelendirmemin hesaba katmadığı bir ayrıntı. CVE-2026-71429, CWE-407.

> **Ders: maliyet modelin hangi boyutta işlediğini sorgula.** Pahalı olan şey, validator'ların sınırladığı şey değilse, o validator'lar zaten süsten ibaret.

---

### 2. Replace'te process seviyesinde bir abort

Bir crash-fuzzer, gerçek binary'yi exit 134 için izlerken bu bug'a yaklaşık 150 denemede ulaştı. Ama yeniden üretmek iki sebepten epey uzun sürdü. Birincisi: PRNG `process.pid`'den seed alıyordu, yani hiçbir çalıştırma bir öncekini tekrar etmiyordu. İkincisi daha can sıkıcıydı: crash log'u girdiyi 400 karakterde kesiyordu, oysa crash'in tetiklenmesi için 89KB'lik bir girdi gerekiyordu; arıza tam olarak girdinin *boyutuna* bağlıydı ve log da elindeki tek değerli bilgiyi, o boyutu, atmıştı. Minimize etme aşaması da aynı derecede yanıltıcı çıktı: emoji, named group'lar, sticky flag, hepsi olmazsa olmaz gibi duruyordu, hiçbiri öyle değildi.

Minimize edilince geriye tek satır kaldı:

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

Kaynak basit: `WrappedRE2::Replace`, bir string `String::kMaxLength`'i aştığında V8'in döndürdüğü boş `MaybeLocal`'ı hiç kontrol etmeden `.ToLocalChecked()` çağırıyor. `$'` ya da `` $` `` gibi çıktıyı büyüten bir template kullanıldığında sonuç O(input²)'ye şişiyor; sınır aşılınca da ortaya `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` ve peşinden **SIGABRT** çıkıyor: bir JavaScript exception değil, doğrudan process seviyesinde bir abort, yani `try`/`catch` hiçbir işe yaramıyor. 30k'da sorun yok, 40k'da crash: sınır da tam olarak N²/2'nin 536M'yi geçtiği nokta.

Peki bu neden belgelenmiş bir sınır değil de gerçek bir bug? Karşılaştırma cevap veriyor: aynı girdide native engine yakalanabilir bir `RangeError` fırlatıyor, re2 ise doğrudan abort ediyor. Orkestratör, minimize edilmiş vakayı hem o günkü güncel sürümün temiz kurulumunda hem de deponun kendi sürümünde yeniden üretti. Abort'un sadece benim ağacıma özgü olmadığı böylece doğrulanmış oldu. C++ kaynak-denetim ajanları bu hedef için zaten sıradaydı ama hiç dispatch edilmediler; fuzzing oraya daha önce vardı çünkü. CVE-2026-71430, CWE-617, Medium 6.2.

> **Ders: fuzzing hijyenini zor yoldan öğrendim.** PRNG seed'i sabitle, asla `process.pid` kullanma. Girdi log'larını tam ve kesilmeden tut. Minimize işlemini deterministik yürüt, çünkü ilgi çekici görünen ayrıntılar çoğunlukla süsten ibaret.

---

### 3. Doğrulanmadan diske düşen bir indirme

Açılıştaki zincirin henüz takip etmediğim bir üçüncü adımı vardı aslında. re2, native binding'ini varsayılan olarak derlemiyor; önceden derlenmiş bir `.node` dosyasını indiriyor, bu işi de `install-from-cache --artifact build/Release/re2.node --host-var RE2_DOWNLOAD_MIRROR ...` üzerinden ayrı bir pakete, `install-artifact-from-github`'a devrediyor. Bu install script'ini önüme getiren şey bir fuzzer değildi. gossipcat'in kendi dependency tree'sini taraması oldu, yani yine bir okuma.

`bin/install-from-cache.js` artifact'ı network'ten indirip doğrudan diske yazıyor. Checksum yok, signature yok, SRI de yok. Paketin tamamında `createHash|sha256|checksum|integrity|digest|signature` için attığım grep sıfır sonuç döndürdü:

```js
// bin/install-from-cache.js
let assetUrl = mirrorHost || process.env[mirrorEnvVar] || 'https://github.com';
// ...
const write = async (name, data) => {
  await fsp.mkdir(path.dirname(name), {recursive: true});
  await fsp.writeFile(name, data);   // data is the raw HTTP response body
};
```

`mirrorEnvVar` tamamen saldırgan kontrolünde ve hiçbir host'a pinlenmemiş durumda: re2 bu değişkeni `--host-var RE2_DOWNLOAD_MIRROR` olarak geçiyor, yani onu set edebilen her şey (bir CI config'i, `.npmrc`'den sızan bir env passthrough, ele geçirilmiş bir CI job'ı, zehirlenmiş bir shell profile) tüm indirmeyi istediği host'a yönlendirebiliyor. Üstüne üstlük transfer plaintext `http://`'yi de kabul ediyor, bir redirect geldiğinde de orijinal scheme değil yeni URL'nin protokolü esas alınıyor. Yani `https://` bir asset `http://`'ye 302 ile yönlendirilirse bu sessizce takip ediliyor: ne bir uyarı var, ne de downgrade'e karşı bir guard.

Evet, bir post-download kontrolü var (`verify-build` ya da `npm test`), ama bu kontrol dosya zaten yazıldıktan sonra devreye giriyor. re2 özelinde bu kontrolün ilk satırı, doğrulaması gereken binary'yi `require()` etmek. Yani native module-init kodu, herhangi bir kontrol çalışmaya başlamadan çoktan tetiklenmiş oluyor. Bu bir gate değil, olsa olsa bir smoke test.

Proof of concept'i bilinçli olarak zararsız tuttum: plaintext HTTP üzerinden sentinel byte'lar servis eden bir "mirror" ayağa kaldırdım, `RE2_DOWNLOAD_MIRROR`'ı ona yönlendirdim, servis edilen byte'ların diske hiç değişmeden düştüğünü gösterdim, o kadar.

```
attacker mirror (plaintext http) on 127.0.0.1:PORT
  [mirror] served 2083 attacker bytes over HTTP for /uhop/node-re2/releases/download/1.24.1/darwin-arm64-137
served   sha256: ec6fdde6...4ebc
written  sha256: ec6fdde6...4ebc
install-from-cache exit: 0
```

Kurulum sorunsuz tamamlanıyor. Gerçek bir `.node` dosyası `require()` edildiği an native init kodunu çalıştırıyor: yani sahtesini oraya koymak, kurulum veya çalışma anında code execution anlamına geliyor. Hem de bu denetimin asıl hedefi olan node-re2'nin kendi kodu henüz hiç çalışmamışken.

CVE-2026-73864, GHSA-88q3-gch3-5396, CWE-494, High 7.5. Advisory 2026-07-07 06:07 UTC'de açıldı: yukarıdaki stream-json ve Replace-abort bulgularından sonra, aşağıda gelecek iki re2 bulgusundan önce.

Ve bu sadece teoride kalan bir tehlike değildi. `install-artifact-from-github`, gerçek dünyadaki ağaçlarda re2'nin altında sık sık karşımıza çıkıyor; kontrol ettiğim sırada pre-fix sürümleri hâlâ birkaç projede pinliydi: `PostHog/posthog`'da (37,7k star) `pnpm-lock.yaml` içinde `re2@1.22.1`/`1.24.1`; `coralproject/talk`'ta (2k star) `re2@1.21.4`; bir Google org'u olan `google-labs-code/stitch-sdk`'da (1,8k star) `re2@1.23.2`. En çarpıcı örnek ise `democratic-csi/democratic-csi` (1,3k star): Dockerfile'ında şu satır var:

```
ENV RE2_DOWNLOAD_MIRROR="https://grpc-uds-binaries.s3-us-west-2.amazonaws.com/re2"
```

Bu, advisory'nin attack surface olarak işaret ettiği değişkenin tam da kendisi, hem de gerçek ve tamamen meşru bir build-speed mirror'ı olarak kullanılıyor. `renovatebot/renovate` (22,3k star) doğrudan bir dependent değil, ama kendi Docker build'inde bu paket için özel yazılmış bir yorum satırı var (`# set npm_config_platform_arch for install-artifact-from-github`), maintainer ekibinin bu paketin install davranışını, bu bug'dan bağımsız olarak, zaten düşünmüş olduğunun kanıtı.

Fix gecikmedi: 1.7.0, advisory açıldıktan yaklaşık 20,5 saat sonra yayımlandı. Üstelik tek satırlık bir yama da değildi. Tüketen paket artık publish anında kendi `package.json`'ına, platform slot başına bir SHA-256 digest damgalıyor; `install-from-cache` da yazmadan önce indirdiğini bu digest'e karşı kontrol ediyor, uyuşmazlık varsa kaynaktan derlemeye geri dönüyor. Aceleye getirilmiş bir yama değil bu: hızlı ama iyi düşünülmüş bir yanıt.

Bir çekincesi de var, açıkça söylemek lazım. Ama bu tasarıma bir eleştiri değil, sadece durumun kendisi: doğrulama yalnızca varsayılan `github.com` kaynağını kapsıyor. Yukarıdaki `RE2_DOWNLOAD_MIRROR` örneğinde olduğu gibi açıkça yapılandırılmış bir mirror kullanılıyorsa, sistem tasarım gereği trust-the-mirror'a geri düşüyor, çünkü tüketen paketin kontrol etmediği bir host için önceden digest damgalamanın bir yolu yok. `democratic-csi`, 1.7.0'a tamamen geçse bile tam da bu yüzden yeni kontrolün kapsamı dışında kalıyor. Keyfi, kullanıcı tanımlı bir kaynağı doğrulamak sabit bir kaynağı doğrulamakla aynı problem değil zaten, fix'i böyle sınırlamak da mantıklı bir tercih, ama sonuçta kapsamı ile gerçek dünyadaki kullanım tam örtüşmüyor.

> **Ders: sadece kodu değil, o kodu sana teslim eden şeyi de denetle.** Bir binding layer uçtan uca memory-safe olsa bile, onu oluşturan byte'ları çalışmadan önce hiçbir şey kontrol etmiyorsa yine de açık kapı bırakır. Install time'da önceden derlenmiş bir artifact indiren her paket (üzerine inşa edenler farkında olsun ya da olmasın) bir supply-chain edge'idir.

---

### 4. Bir character index'ini doğrulayan byte uzunluğu

Bu bug'ı fuzzer bulmadı, zaten bulamazdı da. Çünkü ona ulaşmak için `re.lastIndex`'in çok özel, yanlış bir değere set edilmesi gerekiyor; rastgele girdi üreten bir mekanizmanın o property'ye dokunması için hiçbir sebep yok. Beni oraya götüren, byte-character dönüşüm aritmetiğini satır satır okumak oldu. Bug birkaç dakikada ortaya çıktı.

`lastIndex`, kullanıcı tarafından herhangi bir pozitif tam sayıya set edilebiliyor. `prepareArgument`, subject'in **UTF-8 byte** sayısını uzunluk diye saklıyor; `setIndex` ise kullanıcının verdiği **UTF-16** index'ini bu byte sayısına karşı doğruluyor. Sorun şu: ASCII olmayan her subject'te byte sayısı character sayısından fazla, dolayısıyla ikisinin arasında kalan bir index doğrulamadan rahatça geçiyor. Ardından `getUtf16PositionByCounter`, o kadar character'ı buffer boyunca hiçbir bounds check yapmadan kat ediyor.

```js
const RE2 = require('re2');
const re = new RE2('a', 'y');   // 'g' also works
re.lastIndex = 3;               // 3 <= byteLen(4) passes; there are only 2 real chars
re.exec('éé');                  // ASAN: heap-buffer-overflow READ
```

Yayınlanan binary'de, büyük ve ASCII olmayan bir subject verildiğinde counter buffer'ın çok ötesine, eşlenmemiş belleğe kadar sürükleniyor; process de hiçbir JavaScript handler'ının yetişemeyeceği bir SIGSEGV ile çöküyor. Aynı `lastIndex`, ASCII bir subject'e karşı gayet normal dönüyor, bu da meselenin boyutta değil birim uyuşmazlığında olduğunu net biçimde gösteriyor. Sınırlı bir information leak'e de ulaşmak mümkün aslında, ama gösterdiğim over-read'ler hep sıfır döndürdüğü için crash'i asıl etki olarak raporladım, advisory'de de bunu böyle belirttim.

Bug'ın bugüne kadar fark edilmemesinin sebebi de ASCII'nin güvenli olması: byte uzunluğu character sayısına eşit olduğu için guard tesadüfen doğru çalışıyor, sıradan testler de sorunsuz geçiyor. CVE-2026-67550, CWE-125, Medium 5.7, o da 1.25.2'de düzeltildi.

Bunu olduğu gibi söylemek lazım: bu, kümedeki orkestrasyon açısından en zayıf vaka. Bug'ı kaynak okuma buldu, dispatch hiçbir zaman bulamayacaktı. Orkestratörün katkısı işin sonunda geldi: rapor dosyalanmadan önce crash'i, güncel sürümün temiz bir kurulumunda, hem yayınlanan binary'de hem de kaynaktan derlenmiş bir ASAN build'inde iki kez daha doğruladı.

> **Ders: bir uzunluğun hangi birimde olduğunu her zaman sor.** Bir niceliği ölçen validator, başka bir niceliği kat eden consumer'ı koruyorsa bu bir yazım hatası değil, tam bir bug sınıfıdır. Byte'a karşı character, UTF-8'e karşı UTF-16, count'a karşı size, hep aynı hikaye.

---

### 5. Global match döngüsünde ilerlemeyen bir cursor

ASAN altında çalışan bir fuzzer, global flag ile `A*((((((((a)?)?))*)*)?)*)*` deseni üzerinde tam yirmi bir dakikadır uğraşıyordu ve 2.5GB bellek tutuyordu. Ortada ne bir sanitizer raporu ne de bir crash vardı, yani çalıştırma sadece yavaş bir testten ayırt edilemez haldeydi.

Takılı kalan process'i sample'layınca durum netleşti: stack, sıkı bir döngü içinde `WrappedRE2::Match` → `RE2::Match` → `DFA::Search` sırasını gösteriyordu. Sebebi de `match.cc`'deki şu dört satırdı:

```cpp
while (re2->regexp.Match(str, byteIndex, str.size, anchor, &match, 1)) {
    groups.push_back(match);
    byteIndex = match.data() - str.data + match.size();  // += 0 for a zero-width match
}
```

Cursor, bulunan son match'in uzunluğu kadar ilerliyor. Zero-width bir match'in uzunluğu olmadığı için cursor yerinden kımıldamıyor, aynı match sonsuza dek dönmeye devam ederken `groups` de arkada bir `std::vector<StringPiece>`'i büyütmeyi sürdürüyor.

O karmaşık desen aslında tesadüfiydi: boş string'i match edebilen herhangi bir global RE2, aynı sonucu verir:

```js
const RE2 = require('re2');
'x'.match(new RE2('a*', 'g'));   // never returns
```

Yayınlanan binary'de bu, dört saniye gibi kısa bir sürede 736MB'tan 3.7GB'a fırlıyor ve süreci ancak dışarıdan gelen bir `SIGKILL` durdurabiliyor. Çağrı senkron ve native olduğu için thread'i tamamen kilitliyor: `try`/`catch`, `AbortController`, JS timer'ları, hepsi işlevsiz kalıyor; timer'lar tetiklenmek için sıraya bile giremiyor çünkü. V8 aynı deseni 0ms'de çözüyor, bu arada.

Bellek büyümesi, yüklü bir makinede en kolay uydurulabilecek ölçümlerden biri, bu yüzden rakamı tek başına kabul etmedik. Orkestratör, vakayı boş string'i hiç match edemeyen `a+` kontrolüne karşı yeniden çalıştırıp her process'in resident set'ini ayrı ayrı sample'ladı: bug 3860MB'a tırmanırken kontrol grubu 44MB'ta sabit kaldı. Üstüne, döngüyü en güncel sürümün temiz bir kurulumunda, hem yayınlanan binary'de hem de kaynaktan derlenmiş bir build'de tekrar doğruladı. Dosyalama ve gönderim kısmı bana kaldı.

İşin ilginç yanı, node-re2 bu durumu başka yerlerde zaten çözmüş: bir subject'i cursor'la kat eden üç yol var, `split.cc` zero-width match'te bir code point ilerliyor, `exec` de `lastIndex` üzerinden ilerliyor. Guard'ı atlayan tek yol, global `Match`. CVE-2026-68499, CWE-835, Medium 6.2, 1.25.2'de düzeltildi.

> **Ders: aynı yapıyı kat eden bütün yolları tek tek incele.** Bir yapı birden fazla yerden kat ediliyorsa, o yolların advance ve termination logic'lerini karşılaştır. Diğerlerinin taşıdığı guard'ı taşımayan yol, bug'ın saklandığı yoldur, sıfır uzunluklu elemanlar da bunun klasik tetikleyicisi.

---

### Bir ders daha

**Engine'i değil, marshalling layer'ını denetle.** re2, tam olarak ReDoS'tan kaçınmak için kuruluyor. Elle yazılmış C++ N-API layer'ının hiç incelenmemesinin sebebi de bu itibarın ta kendisi. Oysa engine çekirdeği gayet sağlam çıktı; node-re2'de bulunan bütün bug'lar binding'de saklıydı. Wrapper, engine'in önlemeye çalıştığı sınıfı kendi elleriyle yeniden yaratmıştı. Bu ders taşınabilir: iki runtime arasında saldırgan kontrolündeki string'leri marshal eden her FFI ya da WASM layer'ı için geçerli.

### İş bölümü

İşin dağılımı şöyleydi: orkestratör review'ı dispatch etti, bulguları temiz kurulumlarda yeniden üretti, iddiaları kontrollere karşı sınadı. Hedef seçimi, severity değerlendirmesi, crash minimizasyonu, advisory yazımı: bunların hepsi bana kaldı. İki bug fuzzing'den çıktı, üçü okumaktan. install-artifact-from-github bulgusu ise ayrı bir yerde duruyor: bir fuzzing çalıştırmasından değil, orkestratörün dependency chain üzerindeki kendi review pass'inden geldi, bir code-review aracı için tam yerinde bir bulma şekli, açıkçası. Hiçbir bulguyu orkestrasyon tek başına üretmedi; bu yazı da tersini iddia etmiyor zaten.

En arkasında durduğum kısım, aslında çürütmeler. stream-json'da dört aday incelendi, sadece biri raporlandı: prototype pollution, derin-nesting bir stack overflow, sınırsız string buffering: üçü de tek tek çürütüldü. Bir ölçüm artefaktı da kimseye ulaşmadan yakalandı (event loop'u açık bırakıp bir leak taklidi yapan senkron bir fuzzing döngüsüydü sadece). Bir maintainer'ın gelen kutusuna hiç düşmeyen bu dört öğe, yayına giren beş bulgu kadar değerli bence.

Skor tablosu da, bir istisna hariç, oldukça mütevazı görünüyor. Beş bulgunun dördü node-re2 ve stream-json'ın kendi runtime kodunda, Medium seviyede, denial-of-service sınıfında: write-primitive denetimi node-re2'de hiçbir yerde out-of-bounds write bulmadı, yani bu engine özelinde tavan zaten zayıf bir read'le sınırlı denial of service. İstisna ise bu tavanı yerle bir ediyor: High seviyede, install-time code execution. Ama dikkat, engine'de değil: binary'yi teslim eden tooling'de, farklı bir pakette, tamamen farklı bir bug sınıfında; diğer dördün temsil ettiği memory-safety çalışmasında bir delik değil bu. Sonuçta ortaya beş CVE ve advisory'lerde kredi çıktı. gossipcat'in kendi dependency chain'i de her zafiyetli pin'den kurtuldu: re2, PR #699'da 1.25.0'dan 1.26.1'e taşındı, install-artifact-from-github da onunla birlikte 1.7.0'a çekildi. Döngü böylece kapandı: denetlemeye çıktığım ağaç, sonunda yamalanması gereken ağaç oldu.

### Egzersizin ürettikleri

Sonuçta beş advisory, hepsi upstream'de yamalandı. Hepsi de daha ben hiç bakmadan zaten ağacımdaydı. Asıl kalıcı olan bulgu ise beş CVE'den çok daha uzun ömürlü ve tek cümleye sığıyor: re2'yi tam olarak güvenli seçim olduğu için kurmuştum, ama düzeltilmesi gereken tam da onun binding layer'ıydı. Kendi teslimatını sessizce devrettiği araç ise ondan bile fazlasına ihtiyaç duyuyordu. Bir dependency, itibarı iyi diye denetlenmez; biri oturup (install script'ine kadar) okuduğu için denetlenir.

Yöntem, hedeften çok daha öteye taşındı aslında. İki re2 memory-safety bug'ı da, codebase'i hiç bilmeden herhangi bir native binding'e sorulabilecek sorulardan çıktı: *bu yapıyı kat eden alternatif yollardan hangisi diğerlerinin taşıdığı guard'ı taşımıyor*, ve *bu validator, consumer'ın kat ettiğine kıyasla hangi birimi ölçüyor*. stream-json bug'ı aynı alışkanlığın maliyete uygulanmasından geldi: pahalı olan nicelik, hiçbir şeyin sınırladığı nicelik değildi çünkü. install-artifact-from-github bug'ı ise aynı sorgulama alışkanlığının trust'a uygulanmış hali: *diske düşen byte'ları, çalışmadan önce ne doğruluyor?* Cevap hiçbir şeydi. Var olan tek kontrol, artifact zaten yüklenip çalıştıktan sonra devreye giriyordu. Bu soru, dil ya da ekosistem fark etmeksizin, install time'da bir şey indiren her pakete sorulabilir. Sorular taşınır, belirli bug'lar taşınmaz, asıl mesele bu.

---

### Kaynaklar

- **[CVE-2026-71429](https://www.cve.org/CVERecord?id=CVE-2026-71429)** · [GHSA-528h-pc64-c93x](https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x) — stream-json: `pick`/`ignore`/`filter`/`replace` filters are O(depth²) on nested input; small crafted JSON blocks the event loop for seconds→minutes (DoS). CWE-407, Medium 6.2. 3.5.0'de düzeltildi.
- **[CVE-2026-71430](https://www.cve.org/CVERecord?id=CVE-2026-71430)** · [GHSA-8hcv-x26h-mcgp](https://github.com/advisories/GHSA-8hcv-x26h-mcgp) — node-re2: `String.prototype.replace(re2, template)` aborts the Node process (uncatchable `ToLocalChecked` on an empty `MaybeLocal`) when the result exceeds V8's max string length. CWE-617, Medium 6.2. 1.25.2'de düzeltildi.
- **[CVE-2026-73864](https://www.cve.org/CVERecord?id=CVE-2026-73864)** · [GHSA-88q3-gch3-5396](https://github.com/advisories/GHSA-88q3-gch3-5396) — install-artifact-from-github: `install-from-cache` writes and loads a network-downloaded native `.node` with no integrity check (env-controlled mirror + plaintext-HTTP redirect downgrade) → arbitrary code execution at install. CWE-494, High 7.5. 1.7.0'de düzeltildi.
- **[CVE-2026-67550](https://www.cve.org/CVERecord?id=CVE-2026-67550)** · [GHSA-ff84-5f28-78qj](https://github.com/advisories/GHSA-ff84-5f28-78qj) — re2: out-of-bounds heap read in `exec`/`test`/`match` via attacker-influenced `lastIndex` on a non-ASCII subject → uncatchable process crash (DoS). CWE-125, Medium 5.7. 1.25.2'de düzeltildi.
- **[CVE-2026-68499](https://www.cve.org/CVERecord?id=CVE-2026-68499)** · [GHSA-6hxr-mr5r-9836](https://github.com/advisories/GHSA-6hxr-mr5r-9836) — re2: global `String.prototype.match` with an empty-matchable pattern never advances → infinite loop with unbounded native memory growth (DoS). CWE-835, Medium 6.2. 1.25.2'de düzeltildi.
