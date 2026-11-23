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
- [x] R19: blink sonrası pawn `ReacquireTime` boyunca "Reacquiring" Actor Tag'i taşır; `BP_ShootingEnemy` ateş etmez, `BP_RunningEnemy` vurmaz. 2026-11-23: Reacquiring süresince düşmanlar oyuncuyu **takip da etmez** (R14b arkadan öldürme için): `BP_ShootingEnemy`/`BP_SniperEnemy` dönmez (nişancının nişanı da sıfırlanır), `BP_RunningEnemy` yeniden yol hesaplamaz/saldırı başlatmaz (eski hedefine gider).
- [ ] Düşmanların AI Perception (Hearing) ile kırılma sesine yönelmesi + `NOISE_POSITION_ERROR` (noise event zaten yayınlanıyor). **Not:** düşmanların şu an algı modeli yok; her kare oyuncu kamerasının konumunu biliyorlar. Ses algısının anlamlı olması için önce "oyuncuyu kaybetme" durumu tasarlanmalı.
- [x] A2: havadaki tahta düşmana çarparsa (`BP_Wood` EventHit) `ApplyDamage(DamageCauser = tahta)`; `BP_RunningEnemy.AnyDamage` kaynak `BP_Wood` ise ölmez, `StunDuration` (oyun zamanı, 1 sn) boyunca `bStunned` olur: yürümez/saldırmaz (`TryHitPlayer` dahil), `BP_ShootingEnemy` dönmez/ateş etmez. `bA2_WoodHitKillsEnemy` true → eskisi gibi öldürür.
- [ ] Mermilerin yavaş zamanda yavaşladığını VR'da doğrula.

### Faz 7 — Vinç, dalgalar, bıçak, shuriken, swap
- [x] `BP_Crane` (`/Game/Environment`, sahnede `Crane`: ray üstü z=740, y=380, 18 m ray). Spline yerine düz monoray (gerçek tavan vinci gibi; eğri yol gerekirse spline'a geçilir). Tick: `UpdateTrolley` (vinç arabası ±`TravelExtent` 780 cm arasında `TravelSpeed` 120 cm/s gidip gelir, oyun zamanı → yavaş zamanda yavaşlar) + `UpdatePallet`: elde palet yoksa `ReloadTime` (2.5 sn) sonra ve sahnedeki `BP_Wood` sayısı < `MaxWoodInLevel` (16) ise `SpawnPallet` (BP_Wood doğar, mesh `PalletMesh`=SM_WoodPallet_A, ölçek 1, fizik kapalı, kancadaki `LoadPoint`'e bağlanır, **WoodState 1** → askıdayken çekilemez/tutulamaz/hedeflenemez). `DropInterval` (7 sn) dolunca ve `CanDropHere` true ise `DropPallet` → ayrılır + `StartDropped` (state 3: ışınlanma hedefi, yavaşlama yok). `CanDropHere`: `ForbiddenTag` (`RackTop`) taşıyan aktörlerin çarpışan sınırları + 70 cm pay içindeyse bırakmaz, dışına çıkınca bırakır (R12). `RopeLength` 150 cm → kanca yüksekliği (`ApplyRope`, BeginPlay). Konteyner üstü yasak değil (GDD yalnız raf tepesi diyor).
  - PIE'da doğrulandı: palet kancada state 1, 120×80 cm; 7 sn'de state 3 ile düştü, yere inince state 0 (çekilebilir) oldu; döngü tekrarladı; çalışma zamanı hatası yok. VR'da düşen palete ışınlanma elle test edilmedi.
- [ ] `BP_WaveManager` (spawn telegraph, `SPAWN_MIN_DIST_BEHIND`) — dalga yapısı A10'a bağlı.
- [x] Düşmanla yer değiştirme (R16, A1): lazer önceliği havadaki tahta → düşman (`FindSwapTarget`: `SwapRange` 1500, `SwapConeAngle` 10°, ölü değil) → durgun tahta (çekme). Tetik bırakılınca `StartBlink` karartması → `DoArrive` → `DoSwapArrive`: düşman oyuncunun eski yerine (HMD altı, kendi zemin yüksekliğiyle), oyuncu düşmanın zeminine `FindSafeArrival` ile; sıfır momentum, R19 reacquire, iki noktada `BP_SwapPulse`. Sınırlama `OpenDecision.SwapCooldown` (0 = yok). Görüş hattı kontrolü yok (çekmede de yok).
- [x] Bıçak (R13, A4): `/Game/Weapons/BP_Knife` (BeginPlay'de `SetupWeapons` ile spawn, gizli = belde). Tuşa `KnifeHoldTime` (0.15 sn, gerçek zaman) basılı tutunca o ele gelir (`UpdateKnife`); aynı elde tekrar basılı tutmak bele döndürür. Elde ve görünürken düşmana değerse öldürür (OverlapAllDynamic). Bıçaklı el tahta tutar/lazerle çekerse bıçak bele döner (`SetHeldWood` → `StowKnife`, `KnifeReturn{Right,Left}`); o el tahtayı bırakınca/fırlatınca aynı ele geri gelir. Fırlatılamaz. Belde görsel yok (yalnız gizleniyor).
- [x] Shuriken (R14, A3): `/Game/Weapons/BP_Shuriken` (fizikli disk, CCD). Bölüm başına `ShurikensPerLevel` = 2 (`ShurikenCount`). Tuşa basınca ele alınır (el boş, bıçak o elde değilse), basılı tutup savurup bırakınca `GetThrowVelocity` ile fırlar; hız < `ShurikenMinThrowSpeed` ise cebe geri döner. Düşmana çarparsa öldürür (sersemletmez), duvar/zemine çarpınca fiziği durur ve yapışır, tahtaya çarparsa seker. Yapışmış shuriken'e grip ile (`TryRetrieveShuriken`, yarıçap max(GrabRadius, 25 cm)) dokununca cebe geri gelir. Lazerle çekilemez, ışınlanma hedefi değildir.
- Girdiler (A7, geçici): `IA_Knife_{Left,Right}_{Pressed,Released}` → Oculus/PICO B ve Y, Index B; `IA_Shuriken_*` → Oculus/PICO A ve X, Index A. Masaüstü: R = sağ bıçak, F = sağ shuriken. Ayrıntı: `Docs/InputMappings.md`.
- [x] Nişancı düşman (R21): `/Game/Enemies/BP_SniperEnemy` (`BP_ShootingEnemy` alt sınıfı; `Fire`, mermi, ölüm, sersemletme miras). Tick override → `SniperTick`: oyuncuya döner; oyuncu kökü z > `HighGroundMinZ` (150 cm → raf 250 / konteyner 260 üstü, zemin değil) ve "Reacquiring" değil ve sersem değilse `CampGraceTime` (2 sn) sonra nişan alır: kırmızı `AimBeam` (silahtan oyuncu göğsüne, kalınlık `BeamThicknessStart`→`End` 0.4→1.5 cm, `MI_SniperBeam`) `AimTime` (1.25 sn) görünür → `Fire` (normal görünür mermi, 1500 cm/s) → `SniperCooldown` (3 sn) sonra tekrar. Yerdeki oyuncuya ateş etmez. Yükseklikten inince / blink sonrası (R19) nişan iptal. Sahnede `Sniper_1` (620,150) — test için; dalgalar gelince spawn sistemine taşınacak.
  - PIE'da doğrulandı: raf üstünde oyuncu ~3.3 sn'de vuruldu (seviye yeniden yüklendi); zeminde 4 sn boyunca nişan/atış yok; ışın 14 m, doğru konum ve kalınlıkta. VR'da ışının uzaktan okunabilirliği test edilmedi.
- [x] Kafa duvar içinde karartma (T5): `BP_XRPawn.UpdateHeadInWall` (Tick, ölü değilken). Kamera konumunda `HeadInWallRadius` (10 cm) küre çakışması, yalnız **WorldStatic** (duvar/zemin/metal dekor; tahta, düşman, bıçak, shuriken tetiklemez). Durum `HeadInWall` değişince `StartCameraFade` siyaha (`HeadInWallFadeTime` 0.1 sn × global dilation, tutulur) / geri. Aç/kapa `Tuning.HeadInWallEnabled`.

### Faz 8 — Ortam ve eğitim
- [~] Greybox depo: ahşap = araç, metal = dekor.
  - [x] Tavan 8 m: her duvar karosunun üstüne z=400'de `*_Up` kopyası (1 cm dışarı kaydırılmış, örtüşen 50 cm'de z-fighting yok); tavan karoları z=800; ışıklar z=770, 80 cd / 1400 yarıçap.
  - [x] `/Game/Environment/BP_Rack` (tag `RackTop`) ×4 → artık `SM_PalletRack_A` modeli (406×115 cm, üst güverte 250, orta 130 cm; mesh complex-as-simple + iki güvertede ince `BlockAll` kutu). Kaynak: `Desktop/VR_3D_Art/Scripts/SM_PalletRack_A` (Blender'sız Python, SPEC.md): `Rack_A` (−800,600), `Rack_B` (−800,150), `Rack_C` (−50,550, 90°), `Rack_D` (−750,−300).
  - [x] `/Game/Environment/BP_Container` (600×240×260; tag `ContainerTop`) ×1: `Container_NE` (400,640).
  - [x] Vinç modelleri: `SM_CraneRail_A` (IPE300, askılar), `SM_CraneHoist_A`, `SM_CraneRope_A` (1 m, Z ölçeklenir), `SM_CraneHook_A`; `SM_WoodPallet_A` (EUR palet, 1-hull convex çarpışma). Kaynak: `Desktop/VR_3D_Art/Scripts/SM_Crane_A`, `SM_WoodPallet_A` (Blender'sız Python).
  - [ ] Varil yığınlarını yükseklik kaynağı olarak düzenle, çatı makasları (tavan karosunda kiriş var).
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
- Dizi tipli enum pinleri (`SphereOverlapActors.ObjectTypes`) DSL'de `MakeArray` ile bağlanamıyor; DSL'i pinsiz yaz, sonra `create_node Utilities|Array|MakeArray` → `connect_pins` (önce bağla, tip çözülür) → `set_pin_value "ObjectTypeQuery1"` (= WorldStatic).
- `execute_tool_script` içinde `time.sleep` oyunu ilerletmez (oyun thread'ini bloklar); PIE'da kare beklemek için ayrı araç çağrıları yap.
- `SM_WarehouseWall_A` çarpışması: iç yüz yerel y=0 (panel 0..20 cm dışarı), kolonlar y −20..0 içeri. Görsel sınır −26'ya uzanıyor ama bu kolonlar; duvar iç yüzü = aktör konum çizgisi.
- `StaticMeshTools.import_file` OBJ'yi Y-up → Z-up **dönüştürmüyor** (yalnız Y'yi aynalıyor). Prosedürel OBJ'leri Z-up (cm) yaz.
- BP'ye `ActorTools.add_component` ile bileşen ekleyip özelliğini sonra ayarlamak sahnedeki mevcut örnekleri güncellemiyor (örnekte `StaticMesh` None kalıyor). Örnekleri silip aynı transformla yeniden yerleştir.
- `BoxComponent` için `BodyInstance.collisionProfileName` tek başına uygulanmıyor; `objectType` ve `collisionEnabled`'ı da aynı struct içinde ver.
- Fonksiyondan literal döndürmek (`(return false)`) çıkış pini oluşturmuyor. Önce `add_function_param` (output) + derle, sonra `(return (Math|Boolean|NOTBoolean true))` gibi bir ifade döndür. `remove_function_graph` sonrası aynı isimle hemen yeniden `add_function_graph` grafiği kaybedebiliyor → yeni isim kullan.
- `ActorTools.set_actor_transform`'a yalnız `scale` vermek konumu/rotasyonu sıfırlıyor — her zaman tam transform ver.
- `RInterpToConstant`/`RInterpTo` hız ≤ 0 iken **anında hedefe atlar** (dönmemek için 0 değil, `Reacquiring` gibi bir dal kullan).
- `ObjectTools.set_properties` PIE'da yalnız instance-editable değişkenleri yazabiliyor; test bayrağı eklerken `set_variable_instance_editable` yap. DSL getter adları alt çizgiyi düşürür (`TMP_X` → `GetTMPX`).
- PIE `startTransform` pawn kökünü verir; normal oyunda kök z=0 (göz 165). z=100 ile test edersen eller düşman başlarının üstünde kalır.
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
- 2026-11-22: L_Warehouse 16×12 m → 20×16 m (batıya 1 sütun, kuzeye 1 sıra; doğu kapısı/güney duvarı sabit; duvara yaslı proplar duvarla birlikte taşındı; ışıklar 3×3; NavMeshBounds genişletildi). Tavan hâlâ 4 m.
- 2026-11-22: T5 kafa duvar içinde karartma uygulandı. Derleme uyarısız; PIE'da pawn'ı duvar paneline taşıyarak doğrulandı (`HeadInWall` true → geri çekince false). VR'da elle test edilmedi.
- 2026-11-22: Kullanıcı kararı: tavan yükseltilsin + raflar. Tavan 4 → 8 m; 4 raf + 1 konteyner greybox. Yukarıdan aşağı trace ile raf üstleri 250, konteyner 260 cm, zemin 0 doğrulandı. Raf üstüne ışınlanma PIE/VR'da elle test edilmedi.
- 2026-11-22: Kullanıcı isteği: raflar için gerçek model. Blender kurulu değil → `SM_WoodCrate_A` gibi saf Python prosedürel `SM_PalletRack_A` (paletli raf: mavi delikli dikmeler, çaprazlar, turuncu kirişler, galvaniz paneller, kolon koruyucular; 960 üçgen, 2048 BC/N/ORM, 812 px/m paylaşımlı UV). `BP_Rack` küpleri kaldırılıp bu mesh'e geçirildi, 4 raf yeniden yerleştirildi. Trace: üst 250 (panel aralarında da), orta 130, koridor 0.
- 2026-11-22: Vinç (Faz 7): prosedürel vinç + EUR palet modelleri, `BP_Crane` mantığı, sahneye yerleştirildi; PIE'da asma → düşme (state 3) → durgun (state 0) döngüsü doğrulandı. A8 (palet ağırlığı/iki elle tutma) hâlâ açık: palet şu an normal `BP_Wood` gibi tek elle tutulur, `ThrowSpeedMultiplier` 1.
- 2026-11-22: R21 nişancı düşman uygulandı ve PIE'da doğrulandı (raf üstünde vurulma, zeminde güvenli). Not: oyuncu yüksekliği pawn kökünden ölçülüyor; VR'da kök = zemin (blink varışı kökü yüzeye koyar), PlayerStart'ta kök z=100.
- 2026-11-23: Kullanıcı isteği: arkadan öldürme. R14b zaten uygulanmıştı (`BP_RunningEnemy.AnyDamage`, tüm alt sınıflar miras alır); PIE'da geçici test aktörüyle doğrulandı: arkadan shuriken hasarı nişancıyı öldürdü, önden etkisiz. Eksik olan: atıcı/nişancı Reacquiring'de de oyuncuya dönüyordu → arkalarına geçmek imkânsızdı; düzeltildi, PIE'da nişancının Reacquiring boyunca sabit kaldığı, sonra döndüğü doğrulandı. Koşucu değişikliği derlendi ama sahnede koşucu olmadığı için PIE'da denenmedi. Geçici test aktörü/asset'i silindi.
- 2026-11-23: Kullanıcı isteği: daha fazla tahta. L_Warehouse'a 12 `BP_Wood` eklendi (toplam 25; 7 kasa + 5 palet, klasör Warehouse/Wood, batı yarısı + raf orta katları: Rack_B kasa, Rack_D palet). `BP_Crane.MaxWoodInLevel` 16 → 28 (vinç yine 3 boşlukla çalışsın). Hata bulundu/düzeltildi: `SM_WoodCrate_A` ve `SM_WoodPallet_A` otomatik convex hull'ları çarpıktı (kasa üstü düz değil, palet kama şeklinde) → istifler oyun başında devriliyor, palet zeminden düşüyordu; ikisi de tek kutu çarpışmasıyla değiştirildi (kasa 50×45.6×50, palet 120×80×14.4). PIE'da 25 tahtanın hepsi düz oturuyor.
- 2026-11-23: Kullanıcı isteği: düşmanlar oyuncuya bu kadar hızlı dönmesin. `BP_ShootingEnemy`/`BP_SniperEnemy` dönüşü `RInterpTo` (üstel, `TurnSpeed` 4 → 180°'de ilk anda ~720°/sn) yerine `RInterpToConstant`; `TurnSpeed` artık **derece/sn** = 90 (Sniper_1 örneğine de yazıldı). Nişancı ancak yüzü `AimTolerance` (10°) içindeyken nişan sayacını başlatır. `BP_RunningEnemy` CharMoveComp `RotationRate` yaw 270 → 120. PIE: nişancı arkadaki oyuncuya sabit ~90°/sn döndü. Kullanıcı ayrıca 'zaman yalnız biz hareket edince aksın' mekaniğinin çalışmadığını söyledi → GDD §12'de reddedilmiş, kaldırılmıştı; geri getirilip getirilmeyeceği kullanıcıya soruldu.
- 2026-11-23: Kullanıcı isteği (masaüstü testi): düşmanların içinden yürünmesin + tuşla saldırı. `DesktopTick` WASD hareketine düşman engeli (`DesktopEnemyBlockRadius` 60) ve V tuşuyla bıçak darbesi (`DesktopStab`/`UpdateDesktopStab`) eklendi (ayrıntı: InputMappings.md). PIE (geçici istek bayrağıyla, sonra silindi): 10 cm'ye konan oyuncu 60 cm'ye itildi; V arkadan nişancıyı öldürdü, önden (bıçak kapsüle girdiği halde) öldürmedi. VR'da düşman içinden fiziksel yürümeyi engelleyen bir şey yok (yapay itme GDD'ye aykırı); karar kullanıcıda. Not: sahnede kullanıcının koyduğu bir `BP_Knife` (290,210,120) var; yerden alma mekaniği yok. Sniper_1 kullanıcı tarafından (100,90)'a taşınmış.
