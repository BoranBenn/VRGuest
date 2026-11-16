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
- [ ] Düşmanların AI Perception (Hearing) ile kırılma sesine yönelmesi + `NOISE_POSITION_ERROR` (noise event zaten yayınlanıyor).
- [ ] Mermilerin yavaş zamanda yavaşladığını VR'da doğrula.

### Faz 7 — Vinç, dalgalar, bıçak, shuriken, swap
- [ ] `BP_Crane` (spline, palet düşürme, raf tepesi yasak bölgeleri).
- [ ] `BP_WaveManager` (spawn telegraph, `SPAWN_MIN_DIST_BEHIND`) — dalga yapısı A10'a bağlı.
- [ ] Bıçak (R13), shuriken (R14 — taşıma A3), düşmanla swap (R16 — A1).
- [ ] Nişancı düşman (R21), kafa duvar içinde karartma (T5).

### Faz 8 — Ortam ve eğitim
- [ ] Greybox depo: raflar, konteynerler, varil yığınları, çatı makasları; ahşap = araç, metal = dekor.
- [ ] Giriş yolu eğitimi (§8).

## Açık karar placeholder'ları

| Karar | Placeholder | Varsayılan (geçici, karar değil) |
|---|---|---|
| A1 swap menzili/sınırı | `SwapRange`, `SwapCooldown` | Faz 7'de eklenecek |
| A2 tahta düşmana çarparsa | `BP_Wood.bA2_WoodHitKillsEnemy` | false (etkisiz) |
| A5 suikast yükseklik sınırı | `BP_XRPawn.A5MinHeightAboveHead` (cm, <0 = sınırsız) | -1 (sınırsız) |
| A6 yavaşlama zayıflaması | `BP_XRPawn.A6SlowmoMaxDuration` (gerçek sn, 0 = sınırsız) | 0 |
| A8 palet ağırlığı | `BP_Wood.ThrowSpeedMultiplier` | 1.0 |
| A9 kilitliyken grip bırakma | `BP_XRPawn.A9KeepInHandWhenLocked` (false → hızsız düşer, state 3 olur) | true |

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
| `KNIFE_HOLD_TIME`, `SPAWN_MIN_DIST_BEHIND`, `SWAP_RANGE`, `SHURIKEN_PER_LEVEL` | — (Faz 7) | |

## unreal-mcp ile çalışırken bilinen tuzaklar
- Editörün düğüm veritabanı başka bir sınıfa **sonradan eklenen** fonksiyon/değişkenleri görmüyor (`Class|BPXRPawn|...` eski hâlini döndürüyor). Sınıflar arası yeni iletişim gerekiyorsa Actor Tag, genel motor fonksiyonları veya mevcut üyeler kullan; editör yeniden başlatılınca muhtemelen düzelir.
- `write_graph_dsl` grafiği silmez, birleştirir; aynı isimli olay/fonksiyonun gövdesini değiştirir. Eski olayları `delete_node` ile sil.
- Enhanced Input olayları DSL ile yazılamıyor → `create_node "Input|EnhancedActionEvents|IA_..."` + `CallFunction|...` + `connect_pins`.
- Kategorili değişkenler `Variables|<Kategori>|Get...`; `b` önekli bool'lar `b`siz görünür (`GetThrownbyPlayer`).
- Fonksiyon adı ile değişken setter'ı çakışmasın (`SetWoodState` fonksiyonu, `WoodState` setter'ıyla karıştı).
- Kendi fonksiyonlarına parametreyi `:Param değer` keyword'üyle ver; yeni parametre ekledikten sonra önce derle. **Pozisyonel literal (`(CallFunction|X true)`) sessizce self pinine gidip kayboluyor** — `get_node_infos` ile doğrula.
- Dizi `Get` çıktıları tipsiz kalıyor; aritmetikte `Math|Float|Max(Float)` gibi tipli bir düğümden geçir.
- `Utilities|IsValid` her zaman makro (exec) sürümünü oluşturur; `(:"Is Valid" ...)` dallarıyla kullan.
- Impure fonksiyon sonucunu doğrudan `return` içinde kullanma; önce `bind` et (aksi hâlde exec bağlanmıyor, değer varsayılan okunuyor).

## İlerleme günlüğü
- 2026-11-14: Plan oluşturuldu; karar: Blueprint, eski sistemler kaldırılacak.
- 2026-11-14: Faz 0–4 uygulandı, R19 eklendi. Tüm BP'ler uyarısız derleniyor; PIE çalışma zamanı hatası yok. **VR başlığıyla oynanarak test edilmedi.**
- 2026-11-14: Geçici klavye/mouse test modu eklendi (VR başlığı yoksa otomatik). IMC_Default değişmedi; ayrıntı ve geri alma: `Docs/InputMappings.md`.
- 2026-11-14: Faz 5 (R11 lazerle çekme) uygulandı; thumbstick teleport kapatıldı; masaüstü modunda çift mouse dönüşü düzeltildi. Derleme + PIE hatasız, oynanarak test edilmedi.
- 2026-11-16: Eski swap/timer asset'leri silindi; tüm iş commit'lendi.
