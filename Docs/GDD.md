# Oyun Tasarım Dokümanı (GDD) — VR Suikast / Havada Işınlanma

> Çalışma adı: belirlenmedi
> Motor: **Unreal Engine 5** (VR, OpenXR)
> Durum: Çekirdek tasarım kesinleşti, prototip aşamasına hazır.

---

## 0. Bu dokümanı okuyan agent için notlar

- **Bölüm 3 (Kurallar)** bağlayıcıdır. Kuralları değiştirme; bir kural uygulanamaz görünüyorsa kullanıcıya sor.
- **Bölüm 11 (Açık Kararlar)** henüz karar verilmemiş konulardır. Bunlar için **kendi başına karar verme**; kod gerektiriyorsa ayarlanabilir bir parametre / placeholder bırak ve kullanıcıya bildir.
- **Bölüm 12 (Reddedilen Fikirler)** bilinçli olarak çıkarılmıştır. Bunları tekrar önerme veya uygulama.
- **Bölüm 9 (UE Uygulama)** öneridir, zorunlu değildir. Mevcut projede farklı bir yapı varsa ona uy.
- Tüm sayısal değerler **Bölüm 10**'da toplanmıştır; koda sabit (hard-coded) yazma, ayarlanabilir yap.
- Kullanıcı Türkçe konuşuyor.

---

## 1. Özet

Terk edilmiş bir endüstriyel depoda geçen, **dalga tabanlı bir VR suikast oyunu**. Oyuncunun silahı belinde taşıdığı bir bıçak ve bölüm başına iki ninja yıldızı. Asıl gücü: **ahşap objeleri fırlatıp, onlar havadayken üzerlerine ışınlanmak**. Fırlattığı tahta havadayken zaman yavaşlar. Herkes (oyuncu dahil) tek vuruşta ölür.

**Çekirdek döngü:**
```
Ahşabı kavra (elle veya lazerle çek)
 → fırlat (zaman yavaşlar)
 → lazerle hedefle, ışınlan (tahta kırılır, zaman normale döner)
 → altındaki ilk yürünebilir yüzeye in (altında düşman varsa = suikast)
 → bıçak / shuriken / düşmanla yer değiştirme ile öldür
 → yeni ahşap çek → tekrar
```

---

## 2. Sözlük

| Terim | Anlamı |
|---|---|
| **Ahşap obje** | Ortamdaki tutulabilir/fırlatılabilir her ahşap: çıta, kasa, koli, palet. |
| **Havada (Airborne)** | Fırlatılmış veya düşmekte olan, henüz bir yüzeyde durgunlaşmamış ahşap obje. Bkz. R3. |
| **Durgun (Resting)** | Yukarı bakan bir yüzeyde durmuş ahşap obje. Işınlanma hedefi olamaz, tutulabilir. |
| **Blink** | Işınlanma. Kamera kayması yok; ekran çok kısa kararıp açılır. |
| **Lazer** | Elden çıkan, hedefe yapışan kavisli işaretçi. Işınlanma, ahşap çekme ve yer değiştirme için kullanılır. |
| **Işınlanma suikastı** | Varış noktasında düşman varsa düşmanın ölmesi. |
| **Yer değiştirme (Swap)** | Oyuncunun bir düşmanla anında konum takası. |
| **Vinç** | Tavandaki raylı vinç; dalga boyunca periyodik palet düşürür. |
| **Raf tepesi** | Metal rafların üstü; yalnızca ışınlanarak ulaşılan yüksek yürünebilir yüzeyler. |

---

## 3. Kurallar (bağlayıcı)

### Hareket
- **R1.** Oyuncu thumbstick ile yürümez. Tek hareket yolları: (a) oda ölçeğinde fiziksel hareket, (b) ahşaba ışınlanma, (c) düşmanla yer değiştirme.
- **R2.** Işınlanma **yalnızca havadaki ahşap objelere** yapılabilir. Hedefler iki türdür:
  - Oyuncunun fırlattığı ahşap,
  - Vinçten düşmekte olan palet.
  Yerde/durgun hiçbir şeye, metal objelere, bıçağa, shuriken'e veya düşman mermisine ışınlanılamaz.
- **R3. Havada tanımı (hareket tabanlı):** Obje fırlatıldığı / düşmeye başladığı an "havada" olur. Herhangi bir **yukarı bakan yüzeyde** `REST_TIME` boyunca hızı `REST_SPEED` altında kalırsa "durgun" olur. **Yan duvara (dikey yüzeye) değmek havada durumunu bitirmez.** Üst üste istiflenmiş kasalar durgundur, hedef değildir.
- **R4. Varış:** Işınlanma anında objenin konumundan aşağı doğru iz sürülür; oyuncu **altındaki ilk yürünebilir yüzeye** (zemin, raf tepesi, konteyner üstü, varil yığını vb.) ayakta varır.
  - Momentum **sıfırdır**. Oyuncu **hiçbir zaman düşmez**.
  - Varış noktası güvenli varış kontrolünden geçer (Bkz. T3).
- **R5. Işınlanma suikastı:** Varış noktasında düşman varsa düşman ölür. *(Yalnızca baş hizasının üstündeki objelerle mi sınırlı olacağı açık karar — Bkz. A5.)*
- **R6.** Varış anında ahşap obje büyük bir gürültüyle parçalanır. Parçalar **hasar vermez**, **hedeflenemez**, kısa sürede kaybolur.

### Fırlatma ve zaman
- **R7. Fırlatma:** Oyuncu objeyi grip ile tutar, kolunu sallar, grip'i bırakır. Obje bırakma anındaki el hızı ve yönüyle fırlar (Wii tarzı serbest fırlatma). Animasyon gecikmesi yoktur.
- **R8. Tek tahta kuralı:** Oyuncunun fırlattığı bir tahta havadayken (ışınlanılabilir durumdayken) **yeni tahta fırlatılamaz**. Tahta durgunlaşınca veya ışınlanmayla kırılınca kilit kalkar.
- **R9. Zaman yavaşlaması:** Oyuncunun fırlattığı tahta havadayken dünya zamanı `SLOWMO_SCALE` oranında yavaşlar. Tahta durgunlaşınca veya kırılınca zaman normale döner.
  - **Vinç paletleri zaman yavaşlaması tetiklemez.**
  - Oyuncunun kafası ve elleri yavaşlamaz (VR takibi gerçek zamanlıdır).
  - Düşman hareketi ve mermileri yavaşlar.

### Ahşap edinme
- **R10.** Ortamdaki **bütün ahşap objeler** tutulabilir. Duvarlar ve tüm metal öğeler (raflar, variller, çatı makasları) sabit dekordur.
- **R11. Lazerle çekme:** Lazer durgun bir ahşabı işaret ederken grip'e basılırsa obje oyuncunun eline uçar. Oyuncu yerdeki ahşaba yürümek zorunda kalmaz.
- **R12. Raf tepeleri:** Raf tepelerinde ahşap bulunmaz ve vinç oraya palet düşürmez. Raf tepesindeki oyuncu aşağıdan lazerle ahşap çekip fırlatarak ışınlanır.

### Silahlar
- **R13. Bıçak:** Belde durur. İlgili tuşa `KNIFE_HOLD_TIME` kadar basılı tutunca ele gelir. Yakın mesafede tek vuruşta öldürür.
  - Bıçak tutan el ahşap kavrarsa / lazerle çekerse bıçak otomatik **bele döner**.
  - O el tahtayı fırlattığında bıçak otomatik **aynı ele geri gelir**.
- **R14. Ninja yıldızı (shuriken):** Bölüm başına 2 adet. Menzilli, tek vuruşta öldürür. Işınlanma hedefi değildir.
- **R15. Ölüm:** Oyuncu da düşmanlar da tek vuruşta ölür.

### Düşmanla yer değiştirme
- **R16.** Oyuncu lazerle işaret ettiği, menzil içindeki bir düşmanla anında konum takası yapar (blink geçişiyle, sıfır momentum). Düşman oyuncunun eski konumuna (örn. raf tepesine) gider. Menzil ve sınırlama mekanizması açık karar (Bkz. A1).

### Düşmanlar
- **R17.** Düşmanlar dalgalar halinde gelir. Ateşli silah veya bıçak kullanırlar.
- **R18.** Düşman mermileri **görünür, kaçılabilir projectile'lardır** (hitscan değil).
- **R19. Yeniden bulma gecikmesi:** Oyuncu ışınlandıktan (veya yer değiştirdikten) sonra düşmanlar onu **`REACQUIRE_TIME` (0.5 sn)** boyunca hedefleyemez/ateş edemez. Bu, "fark edilmeden arkadan çıkıp öldürme" hissini verir.
- **R20.** Kırılma sesi düşmanları çeker, ama sesin yönü düşman için kesin değildir.
- **R21.** Raf tepelerini hedef alan nişancı tipi düşman bulunur (raf tepesinde kamp yapmayı engeller).

---

## 4. Çevre ve Atmosfer

- **Mekân:** Tek katlı, asma katı veya merdiveni olmayan, yüksek tavanlı standart bir endüstriyel depo. Çelik çatı makasları, metal variller, metal raflar, ahşap paletler, nakliye kolileri ve kasalarla dolu.
- **Görsel dil:** Tamamen gerçekçi, analog, yıpranmış, terk edilmiş. Neon, fütüristik, mistik veya tarihi öğe yok. Soluk endüstriyel renkler sayesinde kırmızı düşmanlar ve uçan ahşap her zaman görsel odak noktasıdır.
- **Okunabilirlik kuralı:** Ahşap = araç, metal/duvar = dekor.
- **Dikeylik:** Asma kat olmadığı için yükseklik raf tepeleri, konteyner üstleri ve varil yığınlarından gelir.
- **Palet vinci:** Tavandaki raylı vinç dalgalar boyunca periyodik olarak palet düşürür. Düşen palet havadayken ışınlanma hedefidir; yere inince fırlatılabilir ahşap kaynağı olur. Kırılan tahtaların yerine yenileri gelir, oyun kaynak eksikliğinden kilitlenmez.

## 5. Oyuncu

- Birinci şahıs VR, taktiksel eldivenli eller.
- Ekipman: Belde bıçak, bölüm başına 2 shuriken.
- Hareket: Bkz. R1.

## 6. Düşmanlar

- Detaysız, low-poly, köşeli, tamamen parlak kırmızı insansı figürler.
- Öldüklerinde kırmızı kristal parçalarına ayrılarak dağılırlar.
- Davranış: R17–R21.
- **Spawn:** Uzamsal sesle haber verilerek doğarlar. Oyuncunun kör noktasında, `SPAWN_MIN_DIST_BEHIND` mesafesinden yakına spawn olmazlar.

## 7. Oyun Hissi ve Konfor

- **Blink:** Kamera kayması yok. Ekran `BLINK_FADE_TIME` kadar kararıp açılır (erişilebilirlik ayarıyla değiştirilebilir). Bakış yönü korunur.
- **Sıfır momentum, sıfır düşüş:** Oyuncu hiçbir zaman yapay hareket/düşüş yaşamaz (mide bulantısı önlemi).
- **Lazer hedefleme:** Tetik basılıyken o elden kavisli lazer çıkar. Lazer `AIM_CONE_ANGLE` içindeki en uygun havadaki hedefe **yapışır** ve hedef vurgulanır. Koni içinde hedef yoksa varsayılan hedef **son fırlatılan tahtadır** (hâlâ havadaysa). Tetik bırakılınca ışınlanma gerçekleşir. Lazer yalnızca tuş basılıyken görünür.
- **Fırlatma:** Gecikme yok; obje grip bırakıldığı anda elden çıkar.
- **Ses:** Odun kırılma sesleri Reaper'da tasarlanır. Tekrar yorgunluğunu önlemek için round-robin varyasyonlar, rastgele pitch/volume ve obje boyutuna göre katmanlar (çıta / kasa / palet) kullanılır.

## 8. Eğitim

Ayrı bir eğitim bölümü yok. Oyuncunun **bölüme girmek için ilerlediği giriş yolu** şu mekanikleri sırayla öğretir:
1. Ahşap kavrama ve fırlatma
2. Fırlatınca zamanın yavaşlaması
3. Havadaki tahtaya lazerle ışınlanma
4. Lazerle ahşap çekme

---

## 9. Unreal Engine Uygulama Notları (öneri)

### Temel
- UE5 + **OpenXR**, başlangıç noktası olarak UE **VR Template** kullanılabilir.
- Girdi: **Enhanced Input** (Input Action / Mapping Context). Kontrol şeması henüz kesin değil (A7); tüm eylemleri ayrı Input Action olarak tanımla ki sonradan kolayca yeniden eşlensin.
- Fizik: **Chaos Physics**.

### Önerilen yapı

| Sınıf / Bileşen | Sorumluluk |
|---|---|
| `AVRPlayerPawn` | HMD, motion controller'lar, el bileşenleri, blink/teleport, ölüm. |
| `UHandComponent` (her el) | Kavrama, fırlatma hızı örnekleme, lazer, bıçak çağırma. |
| `UThrowableWoodComponent` | Ahşap objenin durum makinesi (aşağıda), ağırlık, kırılma. |
| `UAirborneTargetSubsystem` (`UWorldSubsystem`) | Havadaki tüm hedeflerin kaydı; lazer için koni sorgusu. |
| `UTimeDilationSubsystem` | Zaman yavaşlamasını yönetir (R9). |
| `ACraneActor` | Spline üzerinde hareket eden vinç, palet düşürme zamanlaması, yasak bölgeler (raf tepeleri). |
| `AEnemyCharacter` + AI Controller | Behavior Tree / StateTree, AI Perception (Sight + Hearing). |
| `AWaveManager` (veya GameMode içinde) | Dalga spawn'ları, spawn telegraph'ı. |

### Ahşap obje durum makinesi (`UThrowableWoodComponent`)
```
Resting ──(grip / lazerle çek)──▶ Held
Held ──(grip bırak, R8 kilidi yoksa)──▶ AirborneThrown   [zaman yavaşlar]
Vinç ──(düşür)──▶ AirborneDropped                         [zaman yavaşlamaz]
AirborneThrown / AirborneDropped ──(R3 durgunlaşma)──▶ Resting
AirborneThrown / AirborneDropped ──(oyuncu ışınlandı)──▶ Destroyed
```
- Sadece `AirborneThrown` ve `AirborneDropped` durumları `UAirborneTargetSubsystem`'e kayıtlıdır.
- R8 kilidi: `AirborneThrown` durumunda bir obje varken `Held → AirborneThrown` geçişine izin verilmez (grip bırakılınca ne olacağı: A9).

### Teknik detaylar
- **T1. Havada tespiti (R3):** Hız + temas normaline göre. Temas normali Z bileşeni `WALKABLE_NORMAL_Z` üstündeyse yüzey "yukarı bakan" sayılır; dikey duvar temasları sayılmaz. `REST_SPEED` altında `REST_TIME` kalınca `Resting`.
- **T2. Varış yüzeyi (R4):** Objenin konumundan aşağı line/sphere trace. Normali yürünebilir olan ilk yüzey varış noktasıdır. VR'da pawn'ı taşırken **HMD'nin yatay ofseti** hesaba katılmalı (oyuncunun kafası hedef noktanın üstüne gelecek şekilde pawn kökü kaydırılır).
- **T3. Güvenli varış noktası:** Varıştan önce oyuncu kapsülüyle sweep/overlap kontrolü. Hedef tavan makasına, duvara veya rafa çok yakınsa en yakın geçerli boş noktaya kaydırılır. Oyuncu asla geometrinin içine varmaz.
- **T4. Işınlanma suikastı (R5):** Varış noktasında `ASSASSINATE_RADIUS` içinde düşman kapsülü varsa düşman ölür, oyuncu o noktaya varır.
- **T5. Kafa duvar içinde:** Oyuncu fiziksel olarak kafasını geometrinin içine sokarsa (kamera konumunda küçük sphere overlap) ekran kararır.
- **T6. Fırlatma hızı:** Elin son `THROW_SAMPLE_FRAMES` karedeki konumları bir ring buffer'da tutulur; hız ortalaması alınır, bırakma anındaki titreme filtrelenir. Hafif yön düzeltmesi (`THROW_ASSIST_STRENGTH`) uygulanır. Hız hesaplamasında **dilate edilmemiş (gerçek) delta time** kullanılmalı, çünkü el takibi gerçek zamanlıdır.
- **T7. Palet ağırlığı:** Ağır objeler (palet) iki elle tutulur; ağırlık atış hızını azaltır. *(Öneri, Bkz. A8.)*
- **T8. Zaman yavaşlaması:** `UGameplayStatics::SetGlobalTimeDilation` kullanılabilir. Dikkat: oyuncunun pawn'ı, el bileşenleri ve UI dilation'dan etkilenmemeli (gerekirse oyuncu aktörlerinde `CustomTimeDilation` ile telafi). Chaos substepping ayarları yavaş zamanda test edilmeli.
- **T9. Lazer:** Kavisli spline/beam. `UAirborneTargetSubsystem`'den koni sorgusu; hedef skoru açı + mesafe ile hesaplanır. Durgun ahşap ve düşmanlar için ayrı sorgular (çekme ve yer değiştirme).
- **T10. Blink geçişi:** `PlayerCameraManager->StartCameraFade` ile kısa karartma; ışınlanma karartmanın tepe noktasında yapılır.
- **T11. Yeniden bulma gecikmesi (R19):** Blink/swap sonrası tüm düşmanların hedefi `REACQUIRE_TIME` boyunca geçersiz kılınır (örn. Blackboard'da bir flag + zamanlayıcı).
- **T12. Gürültü (R20):** Kırılma anında `ReportNoiseEvent` (AI Perception Hearing). Düşman sesin konumunu `NOISE_POSITION_ERROR` kadar sapmalı algılar.
- **T13. Kırılma:** Ahşap ve düşman kırılması için Chaos Geometry Collection veya önceden parçalanmış mesh + kısa ömürlü parçalar. Parçalarda collision hedeflenebilirliği kapalı.
- **T14. Ses:** Reaper'dan export edilen sesler **MetaSounds** ile round-robin, rastgele pitch/volume ve boyut katmanlarıyla çalınır. Düşman spawn telegraph'ı uzamsal ses (attenuation + spatialization) ile.
- **T15. Mermiler:** `UProjectileMovementComponent`, görünür mesh/trail, yavaş zamanda yavaşlar.

---

## 10. Parametreler (tümü ayarlanabilir, değerler başlangıç önerisi)

| Parametre | Başlangıç değeri | Not |
|---|---|---|
| `REST_TIME` | 0.2 sn | R3 |
| `REST_SPEED` | ~10 cm/sn | R3 |
| `WALKABLE_NORMAL_Z` | ~0.7 | T1, T2 |
| `SLOWMO_SCALE` | ~0.3 | R9 — test edilecek |
| `REACQUIRE_TIME` | **0.5 sn** | R19 — kullanıcı kararı |
| `KNIFE_HOLD_TIME` | ~0.15 sn | R13 |
| `BLINK_FADE_TIME` | 50–80 ms | Bölüm 7 |
| `AIM_CONE_ANGLE` | ~10–15° | Lazer yapışma konisi |
| `ASSASSINATE_RADIUS` | düşman kapsül yarıçapı + küçük pay | T4 |
| `THROW_SAMPLE_FRAMES` | 3–5 kare | T6 |
| `THROW_ASSIST_STRENGTH` | düşük | T6 |
| `NOISE_POSITION_ERROR` | birkaç metre | T12 |
| `SPAWN_MIN_DIST_BEHIND` | belirlenecek | Bölüm 6 |
| `SWAP_RANGE` | **belirlenecek** | A1 |
| `SHURIKEN_PER_LEVEL` | 2 | R14 |

---

## 11. Açık Kararlar (agent bunlara kendi başına karar vermemeli)

- **A1.** Düşmanla yer değiştirme: menzil ve sınırlama (cooldown mu, ritim barı mı?).
- **A2.** Tahta düşmana çarparsa ne olur? (Öldürür / sersemletir / etkisiz.)
- **A3.** Shuriken nerede taşınır, nasıl çekilir, geri alınabilir mi?
- **A4.** Bıçak fırlatılabilir mi?
- **A5.** Işınlanma suikastı her yükseklikten mi geçerli, yoksa yalnızca baş hizasının üstündeki objelerden mi?
- **A6.** Zaman yavaşlaması zamanla zayıflasın mı? (Dik yukarı atılan tahta en uzun yavaşlamayı verir; sınır gerekebilir.)
- **A7.** Kontrol şeması (en son çözülecek).
- **A8.** Palet ağırlığı / iki elle tutma.
- **A9.** R8 kilidi aktifken oyuncu grip'i bırakırsa obje elde mi kalır, düşer mi?
- **A10.** Bölüm ve dalga yapısı: bir bölüm kaç dalga, dalga arası ne olur, puan/ilerleme (en son belirlenecek).
- **A11.** Hedef platform (Quest standalone / PCVR) ve performans bütçesi (şimdilik ertelendi).

## 12. Reddedilen Fikirler (tekrar önerme)

- Işınlanma sonrası havada asılı kalma / süzülme / "coyote time".
- Momentumu koruma veya objenin hızını devralma (sıfır momentum seçildi).
- Oyuncunun havadan düşmesi, düşüş suikastı (yerine: ilk yürünebilir yüzeye varış + ışınlanma suikastı).
- Thumbstick ile yürüme / yapay lokomosyon.
- Bakışla (gaze) hedefleme (yerine: yapışan lazer).
- "Oyuncu hareket ettikçe zaman akar" mekaniği (yerine: tahta havadayken zaman yavaşlar).
- Kıymıkların hasar vermesi veya hedeflenebilir olması.
- Durgun/istiflenmiş objelere ışınlanma.
- Raf tepelerinden otomatik aşağı blink (yerine: lazerle ahşap çekip ışınlanma).

## 13. Yapılacaklar

- [ ] Düşman spawn telegraph'ı (uzamsal ses) ve kör noktaya yakın spawn engeli.
- [ ] Görünür, kaçılabilir düşman mermileri.
- [ ] Fırlatma hızı örnekleme + yön düzeltmesi (T6).
- [ ] Güvenli varış noktası (T3) ve varış yüzeyi (T2).
- [ ] Kafa duvar içinde karartması (T5).
- [ ] Giriş yolunda eğitim akışı (Bölüm 8).
- [ ] Kontrol şemasını kesinleştir (A7).
- [ ] Bölüm/dalga yapısını tanımla (A10).

### Önerilen prototip sırası
1. VR pawn + el kavrama/fırlatma (T6).
2. `UThrowableWoodComponent` durum makinesi + havada tespiti (T1).
3. Lazer + ışınlanma + varış yüzeyi + güvenli varış (T2, T3, T9, T10).
4. Zaman yavaşlaması (R8, R9, T8).
5. Lazerle çekme (R11).
6. Tek tip düşman + mermi + ışınlanma suikastı + yeniden bulma gecikmesi.
7. Vinç, dalgalar, bıçak, shuriken, yer değiştirme.
