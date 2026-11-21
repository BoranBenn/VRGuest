# Uygulama Planı — VR Suikast / Havada Işınlanma

Kaynak: [GDD.md](GDD.md). Bu plan GDD'nin "Önerilen prototip sırası"nı (Bölüm 13) mevcut projeye uyarlar.

## Mevcut durum (2026-11-14)

- UE 5.8, OpenXR, **yalnızca Blueprint** (Source modülü yok). Editör `unreal-mcp` üzerinden düzenleniyor.
- VR Template tabanlı `BP_XRPawn` + eski prototip:
  - `BP_TimeManager`: kafa dönüşüyle zaman akışı → **GDD §12'de reddedildi, kaldırılacak**.
  - `BP_XRPawn` içinde nişan → zaman dondurma → `BP_Swappables`/`I_Swap` ile bakışla nesne swap'ı → **GDD ile çelişiyor, kaldırılacak**.
  - Yeniden kullanılabilir: `BP_RunningEnemy`, `BP_ShootingEnemy`, `BP_EnemyBullet` (görünür mermi), ölüm/yeniden başlatma (`UpdateDeath`, `EventAnyDamage`), kamera karartma, pawn'ı HMD ofsetiyle taşıma kodu.
- Template'in kavrama kodu pawn'dan sökülmüş (Grab input event'leri boş).

## Mimari kararlar (Blueprint uyarlaması)

| GDD önerisi | Bu projede |
|---|---|
| `AVRPlayerPawn` + `UHandComponent` | `BP_XRPawn` içinde el başına fonksiyonlar (`bIsRight` parametreli). |
| `UThrowableWoodComponent` | `BP_Wood` aktörü (fizik simüle eden StaticMesh kökü) + `E_WoodState` enum. |
| `UAirborneTargetSubsystem` | `BP_Wood` sınıf sorgusu + `IsTargetable()` filtresi (prototip için yeterli; performans gerekirse kayıt dizisine geçilir). |
| `UTimeDilationSubsystem` | `BP_XRPawn` içinde `BeginSlowmo/EndSlowmo` (+ pawn `CustomTimeDilation` telafisi). |
| Parametreler (§10) | İlgili BP'de `Tuning` kategorisinde, instance-editable değişkenler. Hiçbir değer graph içine sabit yazılmaz. |
| Açık kararlar (§11) | Kod gerektirenler `OpenDecision` kategorisinde placeholder değişken olarak bırakılır, aşağıdaki tabloda listelenir. |

## Fazlar

### Faz 0 — Temizlik ve temel ✅
- [x] GDD ve plan repoya (`Docs/`).
- [x] `BP_TimeManager` sahneden kaldırıldı ve asset silindi; eski freeze/swap olayları, 10 fonksiyon ve 21 değişken `BP_XRPawn`'dan söküldü (ölüm/restart korundu).
- [x] Sahnedeki 3 `BP_Swappables` ve `BP_Deneme` kaldırıldı; yerlerine + PlayerStart çevresine 6 `BP_Wood` kondu.
- [x] Eski asset'ler silindi: `BP_Swappables`, `I_Swap`, `BP_TimerRing` + `M_TimerRing`, `IA_Swap` (`IMC_VRSpectator`'daki E eşlemesi de kaldırıldı), `BP_Deneme`, `Backup/BP_XRPawn_BeforeTimedSwap`. Gerekirse git geçmişinden (6eba164) geri alınabilir.
- [x] Input Action'lar: `/Game/Input/Actions/IA_Laser_{Left,Right}_{Pressed,Released}` → `IMC_Default`'ta Oculus/Index/Vive/PICO tetiklerine eşlendi (A7: geçici). Grip için mevcut `IA_Grab_*` kullanılıyor.

### Faz 1 — Kavrama ve fırlatma (R7, R10, T6) ✅ (VR'da test edilmedi)
- [x] `BP_Wood` aktörü (`WoodState` int: 0 Resting, 1 Held, 2 AirborneThrown, 3 AirborneDropped, 4 Destroyed — MCP araçları enum asset oluşturamadığı için int).
- [x] Pawn `TryGrab`: grip noktasına `GrabRadiusFromGripPosition` (12 cm) içindeki en yakın tahtayı bağlar (Held/Destroyed hariç).
- [x] El konumu geçmişi (`PushHandSample`, `ThrowSampleFrames`+1 örnek, **gerçek zaman**), `VelocityFromHistory` ortalama hız + `ThrowAssistStrength` ile kamera ileri yönüne hafif düzeltme.

### Faz 2 — Ahşap durum makinesi + havada tespiti (R3, T1) ✅
- [x] `OnGrabbed / Throw / StartDropped / Shatter / ChangeWoodState`.
- [x] `UpdateRestDetection`: havadayken aşağı kısa trace; normal.z ≥ `WalkableNormalZ` ve hız < `RestSpeed` (veya fizik uyudu) `RestTime` boyunca → Resting. Dikey duvar teması sayılmaz.
- [x] `Shatter`: `BP_WoodFragment` parçaları (Pawn/Visibility/Camera'yı yok sayar, `Lifetime` sonra yok olur), `BreakSound` (boş — Reaper sesleri bekleniyor), `ReportNoiseEvent`.

### Faz 3 — Lazer + ışınlanma (R2, R4, R5, R6, T2, T3, T4, T9, T10) ✅ (VR'da test edilmedi)
- [x] Tetik basılıyken `UpdateLaser` → `FindLaserTarget`: `AimConeAngle` içinde açı + mesafe×`AimDistanceWeight` skoru; yoksa hâlâ havadaki son fırlatılan tahta. Görsel: düz silindir ışın + hedef küresi (kavisli ışın daha sonra).
- [x] Bırakınca `StartBlink`: `BlinkFadeTime` karartma (yavaş zamanda süre telafili) → `TraceWalkableBelow` (ahşap ve pawn'ları atlayarak ilk yürünebilir yüzey) → `KillEnemiesAt` (`AssassinateRadius`) → `FindSafeArrival` (kapsül testi, `SafeArrivalRings`×8 aday) → HMD yatay ofsetiyle pawn taşıma → tahta parçalanır → `BP_SwapPulse` efekti.

### Faz 4 — Zaman yavaşlaması + tek tahta kuralı (R8, R9, T8) ✅
- [x] `BeginSlowmo/EndSlowmo/UpdateSlowmo`: global dilation = `SlowmoScale`, pawn `CustomTimeDilation` = 1/scale. Tahta durgunlaşınca/kırılınca/tutulunca biter. Vinç paletleri (state 3) tetiklemez.
- [x] R8 kilidi: `ReleaseGrab` içinde; `A9KeepInHandWhenLocked` placeholder.

### Faz 5 — Lazerle çekme (R11) ✅ (VR'da test edilmedi)
- [x] `UpdateLaser`: koni içinde havada hedef yoksa `FindPullTarget` (state 0 = Resting tahta, `PullConeAngle` / `PullMaxDistance`, skor açı + mesafe×`AimDistanceWeight`) → lazer ona yapışır, `LaserPullTarget{Left,Right}`.
- [x] Grip girdileri artık `GrabPressed` → el boşsa ve çekme hedefi varsa `OnGrabbed` + `SetHeldWood`, ardından `UpdatePull` tahtayı `PullDuration` (gerçek zaman, ease-out) içinde ele uçurur. Çekme hedefi yoksa eskisi gibi `TryGrab`.
- [x] Uçuş sırasında grip bırakılırsa normal fırlatma/bırakma akışı geçerli; tahta zaten Held sayılır (hedeflenemez).
- Not: Çekme hedefine bakarken tetik bırakılırsa ışınlanma olmaz (çekilebilir tahta durgun, hedeflenebilir değil); varsayılan "son fırlatılan tahta" yalnızca havadayken geçerli.
- Template'in thumbstick teleport'u (`IA_Move`) GDD §12 gereği bağlantısı kesilerek kapatıldı; `SnapTurn` (`IA_Turn`) karar için kullanıcıya soruldu.

### Faz 6 — Düşman entegrasyonu (R15, R17–R20, T11, T12, T15) — kısmen
- [x] Işınlanma suikastı (`KillEnemiesAt` → `ApplyDamage`, düşmanlar tek vuruşta ölür).
- [x] R19: blink sonrası pawn `ReacquireTime` boyunca "Reacquiring" Actor Tag'i taşır; `BP_ShootingEnemy` ateş etmez, `BP_RunningEnemy` vurmaz.
- [ ] Düşmanların AI Perception (Hearing) ile kırılma sesine yönelmesi + `NOISE_POSITION_ERROR` (noise event zaten yayınlanıyor). **Not:** düşmanların şu an algı modeli yok; her kare oyuncu kamerasının konumunu biliyorlar. Ses algısının anlamlı olması için önce "oyuncuyu kaybetme" durumu tasarlanmalı.
- [x] A2: havadaki tahta düşmana çarparsa (`BP_Wood` EventHit) `ApplyDamage(DamageCauser = tahta)`; `BP_RunningEnemy.AnyDamage` kaynak `BP_Wood` ise ölmez, `StunDuration` (oyun zamanı, 1 sn) boyunca `bStunned` olur: yürümez/saldırmaz (`TryHitPlayer` dahil), `BP_ShootingEnemy` dönmez/ateş etmez. `bA2_WoodHitKillsEnemy` true → eskisi gibi öldürür.
- [ ] Mermilerin yavaş zamanda yavaşladığını VR'da doğrula.

### Faz 7 — Vinç, dalgalar, bıçak, shuriken, swap
- [ ] `BP_Crane` (spline, palet düşürme, raf tepesi yasak bölgeleri).
- [ ] `BP_WaveManager` (spawn telegraph, `SPAWN_MIN_DIST_BEHIND`) — dalga yapısı A10'a bağlı.
- [x] Düşmanla yer değiştirme (R16, A1): lazer önceliği havadaki tahta → düşman (`FindSwapTarget`: `SwapRange` 1500, `SwapConeAngle` 10°, ölü değil) → durgun tahta (çekme). Tetik bırakılınca `StartBlink` karartması → `DoArrive` → `DoSwapArrive`: düşman oyuncunun eski yerine (HMD altı, kendi zemin yüksekliğiyle), oyuncu düşmanın zeminine `FindSafeArrival` ile; sıfır momentum, R19 reacquire, iki noktada `BP_SwapPulse`. Sınırlama `OpenDecision.SwapCooldown` (0 = yok). Görüş hattı kontrolü yok (çekmede de yok).
- [x] Bıçak (R13, A4): `/Game/Weapons/BP_Knife` (BeginPlay'de `SetupWeapons` ile spawn, gizli = belde). Tuşa `KnifeHoldTime` (0.15 sn, gerçek zaman) basılı tutunca o ele gelir (`UpdateKnife`); aynı elde tekrar basılı tutmak bele döndürür. Elde ve görünürken düşmana değerse öldürür (OverlapAllDynamic). Bıçaklı el tahta tutar/lazerle çekerse bıçak bele döner (`SetHeldWood` → `StowKnife`, `KnifeReturn{Right,Left}`); o el tahtayı bırakınca/fırlatınca aynı ele geri gelir. Fırlatılamaz. Belde görsel yok (yalnız gizleniyor).
- [x] Shuriken (R14, A3): `/Game/Weapons/BP_Shuriken` (fizikli disk, CCD). Bölüm başına `ShurikensPerLevel` = 2 (`ShurikenCount`). Tuşa basınca ele alınır (el boş, bıçak o elde değilse), basılı tutup savurup bırakınca `GetThrowVelocity` ile fırlar; hız < `ShurikenMinThrowSpeed` ise cebe geri döner. Düşmana çarparsa öldürür (sersemletmez), duvar/zemine çarpınca fiziği durur ve yapışır, tahtaya çarparsa seker. Yapışmış shuriken'e grip ile (`TryRetrieveShuriken`, yarıçap max(GrabRadius, 25 cm)) dokununca cebe geri gelir. Lazerle çekilemez, ışınlanma hedefi değildir.
- Girdiler (A7, geçici): `IA_Knife_{Left,Right}_{Pressed,Released}` → Oculus/PICO B ve Y, Index B; `IA_Shuriken_*` → Oculus/PICO A ve X, Index A. Masaüstü: R = sağ bıçak, F = sağ shuriken. Ayrıntı: `Docs/InputMappings.md`.
- [ ] Nişancı düşman (R21), kafa duvar içinde karartma (T5).

### Faz 8 — Ortam ve eğitim
- [ ] Greybox depo: raflar, konteynerler, varil yığınları, çatı makasları; ahşap = araç, metal = dekor.
- [ ] Giriş yolu eğitimi (§8).

## Açık karar placeholder'ları

| Karar | Placeholder | Varsayılan (geçici, karar değil) |
|---|---|---|
| A1 swap menzili/sınırı | **KARAR (2026-11-19, kullanıcı): menzil şimdilik makul bir değere ayarlansın.** `BP_XRPawn.SwapRange`; sınırlama mekanizması (cooldown/ritim) hâlâ açık → `SwapCooldown` placeholder (0 = yok). | 1500 cm |
| A2 tahta düşmana çarparsa | **KARAR (2026-11-19, kullanıcı): düşman kısa süre durur, sonra devam eder.** Süre `StunDuration` (ayarlanabilir). | 1.0 sn |
| A3 shuriken taşıma | **KARAR (2026-11-19, kullanıcı): shuriken ele alınır, atılınca çarptığı yere yapışır, oradan geri alınır.** | — |
| A4 bıçak fırlatma | **KARAR (2026-11-19, kullanıcı): bıçak fırlatılamaz.** | — |
| A5 suikast yükseklik sınırı | `BP_XRPawn.A5MinHeightAboveHead` (cm, <0 = sınırsız) | -1 (sınırsız) |
| A6 yavaşlama zayıflaması | `BP_XRPawn.A6SlowmoMaxDuration` (gerçek sn, 0 = sınırsız) | 0 |
| A8 palet ağırlığı | `BP_Wood.ThrowSpeedMultiplier` | 1.0 |
| A9 kilitliyken grip bırakma | **KARAR (2026-11-17, kullanıcı): kilit kalkınca otomatik fırlatılır.** Bırakma anındaki hız `PendingThrowVel{Right,Left}`'e kaydedilir; Tick'teki `UpdatePendingThrow` önceki tahta durgunlaşınca/kırılınca `FirePendingThrow` ile atar. Tekrar grip'e basmak bekleyen atışı iptal eder (`ClearPendingThrow`). `A9KeepInHandWhenLocked` true kalmalı (false → eski "hızsız düşer" davranışı). | true |

## Parametre eşlemesi (§10 → Blueprint)

| GDD | Blueprint değişkeni | Değer |
|---|---|---|
| `REST_TIME` / `REST_SPEED` / `WALKABLE_NORMAL_Z` | `BP_Wood.RestTime / RestSpeed / WalkableNormalZ` (+ `BP_XRPawn.WalkableNormalZ` varış için) | 0.2 / 10 / 0.7 |
| `SLOWMO_SCALE` | `BP_XRPawn.SlowmoScale` | 0.3 |
| `REACQUIRE_TIME` | `BP_XRPawn.ReacquireTime` | 0.5 |
| `BLINK_FADE_TIME` | `BP_XRPawn.BlinkFadeTime` | 0.065 |
| `AIM_CONE_ANGLE` | `BP_XRPawn.AimConeAngle` (+ `AimMaxDistance`, `AimDistanceWeight`) | 12° |
| (GDD'de yok) çekme konisi / menzil / süre | `BP_XRPawn.PullConeAngle` 12°, `PullMaxDistance` 3000, `PullDuration` 0.15 sn | başlangıç önerisi |
| `ASSASSINATE_RADIUS` | `BP_XRPawn.AssassinateRadius` | 60 |
| `THROW_SAMPLE_FRAMES` / `THROW_ASSIST_STRENGTH` | `BP_XRPawn.ThrowSampleFrames / ThrowAssistStrength` | 5 / 0.1 |
| `NOISE_POSITION_ERROR` | — (Faz 6) | |
| `KNIFE_HOLD_TIME` | `BP_XRPawn.KnifeHoldTime` | 0.15 |
| `SHURIKEN_PER_LEVEL` | `BP_XRPawn.ShurikensPerLevel` (+ `ShurikenMinThrowSpeed` 150, `ShurikenSpinDeg` 1080) | 2 |
| `SWAP_RANGE` | `BP_XRPawn.SwapRange` (+ `SwapConeAngle` 10°) | 1500 |
| `SPAWN_MIN_DIST_BEHIND` | — (Faz 7, dalgalar) | |
| (A2) sersemletme süresi | `BP_RunningEnemy.StunDuration` | 1.0 |

## unreal-mcp ile çalışırken bilinen tuzaklar
- Editörün düğüm veritabanı başka bir sınıfa **sonradan eklenen** fonksiyon/değişkenleri görmüyor (`Class|BPXRPawn|...` eski hâlini döndürüyor). Sınıflar arası yeni iletişim gerekiyorsa Actor Tag, genel motor fonksiyonları veya mevcut üyeler kullan; editör yeniden başlatılınca muhtemelen düzelir.
- `write_graph_dsl` grafiği silmez, birleştirir; aynı isimli olay/fonksiyonun gövdesini değiştirir. Eski olayları `delete_node` ile sil.
- Enhanced Input olayları DSL ile yazılamıyor → `create_node "Input|EnhancedActionEvents|IA_..."` + `CallFunction|...` + `connect_pins`.
- Kategorili değişkenler `Variables|<Kategori>|Get...`; `b` önekli bool'lar `b`siz görünür (`GetThrownbyPlayer`).
- Fonksiyon adı ile değişken setter'ı çakışmasın (`SetWoodState` fonksiyonu, `WoodState` setter'ıyla karıştı).
- Kendi fonksiyonlarına parametreyi `:Param değer` keyword'üyle ver; yeni parametre ekledikten sonra önce derle. **Pozisyonel literal (`(CallFunction|X true)`) sessizce self pinine gidip kayboluyor** — `get_node_infos` ile doğrula.
- Dizi `Get` çıktıları tipsiz kalıyor; aritmetikte `Math|Float|Max(Float)` gibi tipli bir düğümden geçir.
- `Utilities|IsValid` her zaman makro (exec) sürümünü oluşturur; `(:"Is Valid" ...)` dallarıyla kullan.
- `BP_XRPawn`/`BP_Wood` grafiklerinde eski DSL yazımlarından kalmış **bağlantısız kopya zincirler** var (ör. EventGraph'ta 3 `UpdateSlowmo`). `find_nodes` hepsini döndürür; düğüm eklerken olaydan/fonksiyon girişinden gerçekten ulaşılan zinciri seç, sonra `read_graph_dsl` ile doğrula.
- `read_graph_dsl` çıktısı her zaman geri yazılamıyor (`|SetbX`, `Math|Vector|vector*vector` "does not exist"). Var olan fonksiyona ekleme yaparken `create_node` + `break_pins`/`connect_pins` ile araya sok.
- `read_graph_dsl` bazen sonradan `create_node`/`add_node_pin` ile eklenen girişleri göstermiyor (ör. `Or` düğümüne eklenen 3. pin); doğrulamayı `get_node_infos` ile yap.
- Başka bir aktörün `Tags` dizisine DSL'den erişilemiyor (`Variables|Actor|GetTags` yalnızca self). Sınıflar arası sinyal için `ApplyDamage` + `DamageCauser` sınıfı gibi motor yolları kullan.
- `ActorTools.set_actor_transform`'a yalnız `scale` vermek konumu/rotasyonu sıfırlıyor — her zaman tam transform ver.
- Impure fonksiyon sonucunu doğrudan `return` içinde kullanma; önce `bind` et (aksi hâlde exec bağlanmıyor, değer varsayılan okunuyor).

## İlerleme günlüğü
- 2026-11-14: Plan oluşturuldu; karar: Blueprint, eski sistemler kaldırılacak.
- 2026-11-14: Faz 0–4 uygulandı, R19 eklendi. Tüm BP'ler uyarısız derleniyor; PIE çalışma zamanı hatası yok. **VR başlığıyla oynanarak test edilmedi.**
- 2026-11-14: Geçici klavye/mouse test modu eklendi (VR başlığı yoksa otomatik). IMC_Default değişmedi; ayrıntı ve geri alma: `Docs/InputMappings.md`.
- 2026-11-14: Faz 5 (R11 lazerle çekme) uygulandı; thumbstick teleport kapatıldı; masaüstü modunda çift mouse dönüşü düzeltildi. Derleme + PIE hatasız, oynanarak test edilmedi.
- 2026-11-16: Eski swap/timer asset'leri silindi; tüm iş commit'lendi.
- 2026-11-17: Masaüstü testinde bildirilen hatalar: (1) tutulan tahta kinematik ve çarpışmalı olduğu için diğer kutuları savuruyordu → `BP_Wood.OnGrabbed` çarpışmayı `QueryOnly` yapıyor, `Throw` `QueryAndPhysics`'e geri alıyor. (2) Kutular 1 m / 100 kg küptü ve masaüstünde 2 m uzaktan tutulup kamerayla savruluyordu → `BP_Wood` ölçeği 0.5 (50 cm), sahnedeki 6 tahta yere oturacak şekilde yeniden konumlandı. (Masaüstünde tahtayı ele ışınlama denendi, kullanıcı istemedi ve geri alındı: tahta tutulduğu yerde kalıp elle birlikte hareket eder.) (3) R8 kilidi devredeyken bırakma sessizce yok sayılıyordu → `ReleaseGrab` ekrana "R8 kilidi" mesajı basıyor. Derleme + PIE hatasız; tutup fırlatma elle test edilmedi (MCP girdi gönderemiyor).
- 2026-11-17: Kullanıcı isteği: tutma Physics Handle ile. `BP_XRPawn`'a `PhysicsHandleRight/Left` eklendi; `TryGrab` → `StartPhysicsGrab` (tahta fizikli kalır, eldeki göreli konumu `PhysGrabRel{Right,Left}`'e kaydedilir), Tick'te `UpdatePhysicsGrab` hedefi elin pozisyonuna taşır, `DoThrow`/`FirePendingThrow` atmadan önce handle'ı bırakır, `SetHeldWood(None)` handle'ı bırakır. Aç/kapa: `Tuning.UsePhysicsHandleGrab` (true; false → eski ele bağlama). Yay ayarları bileşenlerde (LinearStiffness 750, LinearDamping 200, AngularStiffness 1500, InterpolationSpeed 50). Lazerle çekme (R11) o sırada hâlâ ele bağlıyordu (aşağıya bkz.). PIE'da geçici otomatik testle doğrulandı: tutulan tahta pawn'ı takip etti, bırakınca fırladı, diğer kutular yerinde kaldı.
- 2026-11-17: A9 kararı (kullanıcı): R8 kilidinde bırakılan tahta, kilit kalkınca bırakma anındaki hızla otomatik fırlatılıyor. Derleme + PIE hatasız; elle test edilmedi.
- 2026-11-19: Kullanıcı isteği: lazerle çekme de fizik tabanlı. `GrabPressed` çekme dalında `SetHeldWood`'dan sonra `StartPhysicsGrab` çağrılıyor (tahta fizikli kalır, Physics Handle ile tutulur). `UpdatePull` artık tahtayı `SetActorLocation` ile taşımıyor; `PhysGrabRel{Right,Left}`'in konumunu `PullStartRel × (1−α)²` ile `PullDuration` boyunca sıfıra indiriyor → handle hedefi tahtadan ele doğru kayıyor, tahta yay/sönümle fiziksel olarak uçuyor (yoldaki engellere çarpabilir), varınca normal fiziksel tutmaya dönüşüyor. `UsePhysicsHandleGrab` false ise eski kinematik çekme çalışır. Eski bağlantısız `UpdatePull` zinciri silindi. Derleme uyarısız; PIE/VR'da elle test edilmedi.
- 2026-11-19: Masaüstü testinde: fırlatılan tahta el hareketinden bağımsız hep kamera ileri yönüne gidiyordu (`GetThrowVelocity` masaüstünde sabit `kamera ileri × DesktopThrowSpeed` döndürüyordu). Artık masaüstünde de el geçmişinin yönü kullanılıyor (ayrıntı: `Docs/InputMappings.md`). Yeni `Desktop.DesktopMinFlickSpeed` = 150. Derleme uyarısız; elle test edilmedi.
- 2026-11-19: A1–A4 kararları (kullanıcı) kaydedildi. A2 sersemletme ve R16 düşmanla yer değiştirme uygulandı. `BP_ShootingEnemy`'nin `BP_RunningEnemy`'den türediği görüldü (ortak mantık üst sınıfta). Derleme uyarısız, PIE açılışında hata yok; sersemletme ve yer değiştirme elle test edilmedi.
- 2026-11-19: Bıçak (R13) ve shuriken (R14) uygulandı; geçici tuşlar kullanıcı onaylı (masaüstünde Q/E dolu olduğu için R/F). PIE'da geçici otomatik testle doğrulandı: shuriken ele geliyor (ölçek doğru, sayaç 2→1), bırakınca 1500 cm/s ile fizikli fırlıyor; bıçak 0.15 sn basılı tutunca sağ ele geliyor. Test düğümleri silindi. Düşmana saplanma, yapışma, geri alma ve bıçakla öldürme elle test edilmedi.
- 2026-11-19: Kullanıcı isteği: (1) shuriken daha hızlı: uçuş hızı = max(el hızı × `ShurikenSpeedMultiplier` (2), `ShurikenMinFlightSpeed` (3000 cm/s)). (2) Bıçak ve shuriken düşmanı **yalnızca arkadan** vurulunca öldürür: `BP_RunningEnemy.AnyDamage`, hasar kaynağı `BP_Knife`/`BP_Shuriken` ise kaynağın yatay yönü ile düşmanın ileri yönünün dot'u < `BackstabMaxDot` (-0.2, ~arka 156° yay) olmalı; önden vuruş etkisiz. Işınlanma suikastı ve `bA2_WoodHitKillsEnemy` bundan etkilenmez; tahta isabeti hâlâ sersemletir. Derleme uyarısız, PIE hatasız; elle test edilmedi.
