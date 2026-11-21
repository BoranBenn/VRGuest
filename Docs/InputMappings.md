# Girdi Atamaları — VR orijinali ve geçici masaüstü test modu

## 1. Orijinal VR atamaları (DEĞİŞTİRİLMEDİ)

`/Game/XRFramework/Input/IMC_Default` **olduğu gibi duruyor**. Birebir yedeği: `/Game/Backup/IMC_Default_VR_Original`.

| Input Action | Tuşlar |
|---|---|
| IA_Grab_Left_Pressed / Released | ValveIndex_Left_Grip_Axis, OculusTouch_Left_Grip_Axis (DeadZone modifier, yalnız Pressed'de), Vive_Left_Grip_Click, PICO_Left_Grip_Axis |
| IA_Grab_Right_Pressed / Released | ValveIndex_Right_Grip_Axis, OculusTouch_Right_Grip_Axis (DeadZone modifier, yalnız Pressed'de), Vive_Right_Grip_Click, PICO_Right_Grip_Axis |
| IA_Laser_Left_Pressed / Released | OculusTouch_Left_Trigger_Axis, ValveIndex_Left_Trigger_Axis, Vive_Left_Trigger_Click, PICO_Left_Trigger_Axis |
| IA_Laser_Right_Pressed / Released | OculusTouch_Right_Trigger_Axis, ValveIndex_Right_Trigger_Axis, Vive_Right_Trigger_Click, PICO_Right_Trigger_Axis |
| IA_Move | ValveIndex_Right_Thumbstick_Y, OculusTouch_Right_Thumbstick_Y, Vive_Right_Trackpad_Y, PICO_Right_Thumbstick_Y — **pawn tarafı 2026-11-14'te kesildi** (aşağıya bak) |
| IA_Turn | ValveIndex_Left_Thumbstick_X, OculusTouch_Left_Thumbstick_X, Vive_Left_Trackpad_X, PICO_Left_Thumbstick_X → pawn'da `SnapTurn` (hâlâ aktif) |
| IA_Menu_Toggle_Left | ValveIndex_Left_Thumbstick_Click, Vive_Left_Menu_Click, OculusTouch_Left_Menu_Click, OculusTouch_Left_Y_Click, PICO_Left_Menu_Click, PICO_Left_Y_Click |
| IA_Menu_Toggle_Right | ValveIndex_Right_Thumbstick_Click, Vive_Right_Menu_Click, OculusTouch_Right_B_Click, PICO_Right_B_Click |
| IA_Rotate | Mouse2D (Negate Y) — önceki prototipten; pawn'da X → AddControllerYawInput, Y → AddControllerPitchInput (masaüstü mouse bakışı bunu kullanıyor) |

## 1b. Silah girdileri (2026-11-19, A7 için geçici, `IMC_Default`'a eklendi)

| Input Action (`/Game/Input/Actions/`) | Tuşlar |
|---|---|
| IA_Knife_Right_Pressed / Released | OculusTouch_Right_B_Click, ValveIndex_Right_B_Click, PICO_Right_B_Click |
| IA_Knife_Left_Pressed / Released | OculusTouch_Left_Y_Click, ValveIndex_Left_B_Click, PICO_Left_Y_Click |
| IA_Shuriken_Right_Pressed / Released | OculusTouch_Right_A_Click, ValveIndex_Right_A_Click, PICO_Right_A_Click |
| IA_Shuriken_Left_Pressed / Released | OculusTouch_Left_X_Click, ValveIndex_Left_A_Click, PICO_Left_X_Click |

Not: Oculus B/Y şablonun `IA_Menu_Toggle_*` eylemine de bağlı; pawn'da o olaylar boş olduğu için çakışma yok. Vive için silah tuşu atanmadı. `IA_*` asset'leri lazer eylemlerinden kopyalandı (Pressed/Released tetikleri aynı).

## 2. Geçici masaüstü test modu (eklenenler)

Otomatik: VR başlığı aktif değilse (`IsHeadMountedDisplayEnabled == false`) `BP_XRPawn.DesktopMode = true` olur.

- `/Game/Input/IMC_DesktopDebug` (yeni): LMB → Grab_Right, RMB → Laser_Right, Q → Grab_Left, E → Laser_Left, R → Knife_Right (basılı tut), F → Shuriken_Right (basılı tut, mouse'u savurup bırak). Yalnız masaüstü modunda, `DesktopSetup` içinde priority 1 ile eklenir.
- `BP_XRPawn` → kategori **Desktop**: `DesktopMode`, `DesktopEyeHeight` 165, `DesktopGrabRadius` 200, `DesktopThrowSpeed` 1500, `DesktopMoveSpeed` 300, `DesktopMinFlickSpeed` 150, `DesktopHandOffsetRight` (45,22,-22), `DesktopHandOffsetLeft` (45,-22,-22).
- `BP_XRPawn.DesktopSetup` (BeginPlay'den çağrılır): IMC ekler, kamerayı göz yüksekliğine alır, 4 motion controller'ı kameraya bağlar, `GrabRadiusfromGripPosition`'ı DesktopGrabRadius yapar.
- `BP_XRPawn.DesktopTick` (Tick'te, yalnız DesktopMode): WASD → yatay kayma (çarpışmasız, yalnız test). Mouse bakışı ayrıca eklenmedi; mevcut `IA_Rotate` kullanılıyor.
- `BP_XRPawn.GetThrowVelocity`: DesktopMode'da da el geçmişi kullanılır (eller kameraya bağlı, mouse savurması/WASD el hızını oluşturur): el hızı ≥ `DesktopMinFlickSpeed` ise yön = el hareketi, hız = max(el hızı, `DesktopThrowSpeed`); daha yavaşsa kamera ileri × `DesktopThrowSpeed`. VR'da eskisi gibi el geçmişi. (Geri alırken bu dal kaldırılmalı.)

## 3. VR haline geri dönmek için

1. `BP_XRPawn` EventBeginPlay'den `DesktopSetup` çağrısını, EventTick'ten `DesktopMode → DesktopTick` dalını kaldır.
2. `GetThrowVelocity` içindeki `DesktopMode` dalını kaldır (yalnız el geçmişi kalsın).
3. `DesktopSetup`, `DesktopTick` fonksiyonlarını ve **Desktop** kategorisindeki değişkenleri sil.
4. `/Game/Input/IMC_DesktopDebug` asset'ini sil.
5. `IMC_Default` değişmediği için ona dokunma; şüphe olursa `/Game/Backup/IMC_Default_VR_Original` ile karşılaştır.

Not: VR başlığı takılıyken masaüstü kodu zaten devre dışıdır; geri alma yalnızca temizlik içindir.

## 4. Masaüstü modundan bağımsız değişiklik: thumbstick teleport kapatıldı

GDD §12 thumbstick ile hareketi reddediyor. VR template'in `IA_Move` → teleport bağlantısı bu yüzden kesildi. `IMC_Default` ve teleport fonksiyonları yerinde duruyor.
Geri bağlamak için EventGraph'ta `IA_Move` olayında: Triggered → `TeleportTrace`, Started → `StartTeleportTrace`, Completed → `EndTeleportTrace`.
