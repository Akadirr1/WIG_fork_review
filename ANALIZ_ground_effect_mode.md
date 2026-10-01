# `squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik

**Kaynak:** `github.com/squilter/ardupilot`, dal `squilter/ground_effect_mode`, HEAD `85615706a0` (2021-08-22)
**Ayrıldığı nokta (merge-base):** `27720f2235` (2020-09-26), sürüm **ArduPlane V4.1.0dev** (`ArduPlane/version.h`)
**Karşılaştırılan güncel sürümler:** ArduPilot `master` @ `755258dbb4` (2026-10-01, **V4.8.0-dev**); kararlı sürüm `ArduPlane-stable` = **V4.7.1**
**Kapsam:** 18 commit, 31 dosya, +303 / −14 satır. Kod değiştirilmedi, derleme yapılmadı.

> **Referanslar:** Aksi belirtilmedikçe `dosya:satır` referansları dalın HEAD'ine (`85615706a0`) aittir. Bu dalın değiştirmediği dosyalarda (`Attitude.cpp`, `PID.cpp` vb.) satırlar merge-base ile aynıdır. `master:` önekli referanslar `ArduPilot/ardupilot@755258dbb4` içindir.
>
> **Yöntem:** blob'suz tek dal klonu, `git diff -U8 BASE..HEAD`. graphify grafiği yalnızca `ArduPlane/` ve `libraries/AP_RangeFinder/` için AST modunda kuruldu (3921 düğüm, 5424 kenar, 167 topluluk). Bağımlılıklar (`stabilize`, `set_servos_controlled`, failsafe) bu grafikle bulundu ve yalnızca gereken satır aralıkları okundu. Çakışma tahmini için `git merge-tree` ile derlemesiz bir deneme birleştirmesi yapıldı.

---

## Kısa özet

- Yeni bir uçuş modu ekleniyor: **`GROUND_EFFECT` (no. 25, kısa adı "GDEF")**. Ayrıca AUTO moduna iki standart dışı görev komutu ekleniyor: **50** (yer etkisinde yol noktası) ve **51** (hedef yüksekliği katsayıyla çarpılmış yol noktası).
- Kontrol yalnızca **aşağı bakan mesafe sensörüne** dayanıyor. Aynı yükseklik hatası üç bağımsız **eski tip `PID`** denetleyiciye giriyor: **yunuslama açısı**, **gaz** ve **flap**. TECS kapalı, hava hızı hiç kullanılmıyor, dalga filtresi yok.
- Varsayılan değerler çok küçük bir modele göre seçilmiş: hedef yükseklik **7 cm**, yunuslama ve flap kazançları **0**. IMO Type-B ölçeğinde doğrudan kullanılamaz.
- Kodda gerçek hatalar var: PWM sürücüsü, AUTO'da flap ve gaz kazancı, flap işareti, `int16` taşmaları. Mod kullanılmasa bile sistemin geri kalanını bozan yan etkiler de var: RFND log alanları ve MAVLink `DISTANCE_SENSOR` birimi.
- **Taşıma zorluğu: Orta.** Kod az, ama dokunduğu API'lerin neredeyse tamamı değişmiş. 31 dosyanın 22'si metin düzeyinde çakışıyor. Mantıklı yol, dalı cherry-pick etmek değil, **fikri master'ın mod altyapısıyla yeniden yazmak.**

---

## a) Commit'ler (eskiden yeniye)

| # | Commit | Ne yapıyor (tek satır) |
|---|---|---|
| 1 | `adb6178a01` | `GROUND_EFFECT` mod iskeletini ekliyor: enum, `ModeGroundEffect` sınıfı, mod tablosu, GCS ve failsafe listeleri. |
| 2 | `727aa4571f` | İlk kontrol döngüsü: mesafe sensörüyle yükseklik hatası, gaz ve yunuslama. |
| 3 | `bd4b77e64d` | Hatalı bir yorumu siliyor. |
| 4 | `fcae25a9a7` | `RangeFinder_State`'e `distance_mm` ekliyor (VL53L0X dolduruyor), modu mm'ye geçiriyor, kullanılmayan `AP_Arming` nedeni ekliyor. |
| 5 | `e11e21422a` | Sürücülere `supports_mm_precision()` sorgusu, `find_instance_with_mm_prec()` ve `has_mm_prec_orient()` ekliyor. |
| 6 | `fd4b7e976a` | Mod için ilk 3 parametreyi ve `config.h` varsayılanlarını ekliyor. |
| 7 | `f45f97d543` | Parametreleri `GNDEFCT_ALT_MIN/MAX` ve `GNDEFCT_THR_MIN/MAX` biçimine çeviriyor. |
| 8 | `4a4a075404` | Min/max karışıklığını düzeltiyor. **`DISTANCE_SENSOR` mesajında cm yerine mm gönderilmeye başlanıyor.** |
| 9 | `49932c3636` | Üç ayrı PID ekliyor: `GNDEFCT_THR_`, `GNDEFCT_ELE_`, `GNDEFCT_FLP_`. |
| 10 | `887a4361e2` | Flap kontrolünü `servos.cpp`'ye taşıyor (`desired_flap_percentage`). |
| 11 | `72fa89101b` | PWM ve analog sürücülerin mm döndürmesini sağlamaya çalışıyor. **PWM tarafı hatalı** (bkz. f). |
| 12 | `d76c199714` | Benewake için mm desteği: TFMini-Plus I2C mm çıkış moduna alınıyor, seri sürücüde `cm*10`. |
| 13 | `d44ded893e` | AUTO'da yer etkisi yol noktası tipini (50) ekliyor. |
| 14 | `6954d03735` | Benewake sinyal gücünü okuyor, zayıf okumaları atıyor, gücü RFND loguna yazıyor. |
| 15 | `802f98bab6` | Görev depolamada 50 numaralı komutu, Benewake'i ve RFND logunu düzeltiyor. |
| 16 | `f266743ca8` | AUTO'da 50 numaralı yol noktasının işlenmesini düzeltiyor: L1 yatış, TECS I sıfırlama. |
| 17 | `d7d12fde8d` | Yükseklik çarpanlı (varsayılan iki kat) yol noktası tipini (51) ekliyor. |
| 18 | `85615706a0` | 51 için `GNDEFCT_51_MULT` parametresini ekliyor. |


---

## b) Mod: ne ekleniyor, nasıl seçiliyor

**Yeni bir uçuş modu. Ayrıca AUTO'ya bir ek yapılıyor.**

- **Mod numarası ve adı:** `GROUND_EFFECT = 25` (`ArduPlane/mode.h:46`). Sınıf `ModeGroundEffect`, `name()` "GROUND_EFFECT", `name4()` "GDEF" (`ArduPlane/mode.h:559-581`). Mod tablosuna bağlantı: `ArduPlane/control_modes.cpp:84-85`. Mod nesnesi: `ArduPlane/Plane.h:289`.
- **Kumandadan seçim:** Standart mod anahtarıyla, `FLTMODE_CH` kanalındaki `FLTMODEn = 25` ile (`ArduPlane/control_modes.cpp:94,152`). Parametre belgelerine 25 eklenmediği için yer istasyonu (GCS) bunu "bilinmeyen mod" olarak gösterebilir.
- **GCS'den seçim:** MAVLink `SET_MODE` veya `MAV_CMD_DO_SET_MODE` ile custom_mode 25. Mod "stabilize" sınıfında raporlanıyor (`ArduPlane/GCS_Mavlink.cpp:39`, `ArduPlane/GCS_Plane.cpp:67`).
- **Moda giriş şartı:** Aşağı bakan (`ROTATION_PITCH_270`), sürücüsü `supports_mm_precision()==true` diyen bir mesafe sensörü olmalı. Yoksa `_enter()` false döner (`ArduPlane/mode_groundeffect.cpp:23-25`), `set_mode` "Flight mode change failed" mesajı verir ve önceki moda döner (`ArduPlane/system.cpp:244-261`).
- **Görev komutu (AUTO):** Ham MAVLink komut kimlikleri **50** ve **51**. MAVLink'te tanımlı bir komut değiller, yer istasyonu bunları özel komut olarak tanımalı. Navigasyon açısından `MAV_CMD_NAV_WAYPOINT` gibi işleniyorlar: `do_nav_wp` ve `verify_nav_wp` (`ArduPlane/commands_logic.cpp:47-48`, `:221-222`). Konum bilgisiyle saklanıyorlar (`libraries/AP_Mission/AP_Mission.cpp:707-708`), `param1` → `p1` (`libraries/AP_Mission/AP_Mission.cpp:864-866`). 51 numaralı komut hedef yüksekliği `GNDEFCT_51_MULT` ile çarpıyor (`ArduPlane/mode_auto.cpp:88-90`).
- AUTO'daki 50/51 dalı, **mesafe sensörü var mı kontrolü yapmıyor** (`ArduPlane/mode_auto.cpp:79-112`).

---

## c) Kontrol döngüsü

### Sensör ve sürücüler
`find_instance_with_mm_prec()` önce durumu `Good` olan, sonra herhangi bir aşağı bakan ve mm destekli sensörü seçiyor (`libraries/AP_RangeFinder/AP_RangeFinder.cpp:641-664`). "mm destekli" sayılan sürücüler:

| Sürücü | mm verisi nasıl üretiliyor | Gerçek çözünürlük |
|---|---|---|
| VL53L0X (I2C) | `sum_mm/counter` (`libraries/AP_RangeFinder/AP_RangeFinder_VL53L0X.cpp:765-766`) | mm. Menzil yaklaşık 2 m, dış ortam ve su yüzeyi için uygun değil. |
| Benewake TFMini-Plus (I2C) | Sensör mm çıkış moduna alınıyor ve ayar **sensöre kalıcı kaydediliyor** (`...TFMiniPlus.cpp:70-78`). `distance_mm = sum/count` (`:143-144`) | mm |
| Benewake seri (TF02/TF03/TFmini) | `distance_mm = distance_cm*10` (`libraries/AP_RangeFinder/AP_RangeFinder_Backend_Serial.cpp:61-62`) | **cm** (yalnızca biçim mm) |
| Analog | `dist_m*1000` (`libraries/AP_RangeFinder/AP_RangeFinder_analog.cpp:115`) | ADC'ye bağlı |
| PWM (LidarLite) | **Hatalı**, bkz. f) (`libraries/AP_RangeFinder/AP_RangeFinder_PWM.cpp:110-120`) | — |

Commit adlarına bakılırsa yazarın asıl hedefi **Benewake** sensörleri.

### Filtre
**Özel bir filtre yok.** Var olanlar:
- Sürücü düzeyinde ortalama: Benewake bir okuma turundaki çerçevelerin ortalamasını alıyor (`libraries/AP_RangeFinder/AP_RangeFinder_Benewake.cpp:125-128`), TFMini-Plus biriktirip ortalıyor (`...TFMiniPlus.cpp:143-144`).
- Benewake'te sinyal gücü `<= 20` olan çerçeveler atılıyor (`libraries/AP_RangeFinder/AP_RangeFinder_Benewake.cpp:96-99`). TFMini-Plus'ta güç `< 100` olan okumalar reddediliyor (`...TFMiniPlus.cpp:161`).
- PID'in türev teriminde sabit 20 Hz alçak geçiren filtre var (`libraries/PID/PID.h:123`, `libraries/PID/PID.cpp:62-89`).
- Dosya başındaki "state space" notları (`ArduPlane/mode_groundeffect.cpp:4-14`) yalnızca yorum, uygulanmamış. Barometre, IMU veya EKF düşey hızıyla füzyon, eğim düzeltmesi, aykırı değer eleme ve medyan filtre yok.

### Hedef yükseklik
- `_alt_desired_mm = (GNDEFCT_ALT_MAX + GNDEFCT_ALT_MIN)/2` (`ArduPlane/mode_groundeffect.cpp:31`). AUTO dalında aynı hesap var, 51 için `×GNDEFCT_51_MULT` (`ArduPlane/mode_auto.cpp:85-90`).
- Varsayılanlarla (30+110)/2 = **70 mm = 7 cm** (`ArduPlane/config.h:321-327`). Bu değerler çok küçük bir model için seçilmiş.
- Hata `errorMm = hedef − son_iyi_okuma` (`int16_t`) (`ArduPlane/mode_groundeffect.cpp:57`). Hata pozitifse araç "fazla alçakta" demek.

### Neyi kontrol ediyor
**Yunuslama, gaz ve flap birlikte.** Üçü de aynı hatayı birbirinden bağımsız PID'lerle kullanıyor:

| Çıkış | Kod | Not |
|---|---|---|
| Yunuslama | `nav_pitch_cd = (int16_t) ELE_PID(errorMm)` (`ArduPlane/mode_groundeffect.cpp:62`) | Doğrudan elevatör değil, **yunuslama açısı talebi**. Normal `stabilize_pitch` → `pitchController` zincirinden geçiyor (`ArduPlane/Attitude.cpp:112-127`). Birim: mm başına centi-derece. |
| Gaz | `THR_PID(errorMm) + _thr_ff`, önce `GNDEFCT_THR_MIN..MAX` aralığına, sonra 0..100'e sınırlanıyor (`ArduPlane/mode_groundeffect.cpp:76-80`) | `_thr_ff`, THR_MIN ile THR_MAX'ın ortası (`:28`). Ardından genel `THR_MIN/THR_MAX` sınırı da uygulanıyor (`ArduPlane/servos.cpp:513`). |
| Flap | `desired_flap_percentage = clamp(FLP_PID(errorMm), ±100)` (`ArduPlane/mode_groundeffect.cpp:67`) → `auto_flap_percent = -desired` (`ArduPlane/servos.cpp:632-634`) | İşaret ters çevriliyor ve negatif değerler eziliyor, bkz. f). |
| Yatış | `nav_roll_cd = 0`, kanatlar düz (`ArduPlane/mode_groundeffect.cpp:60`) | `STICK_MIXING=1` (varsayılan) ile pilot FBW biçiminde yatış ekleyebiliyor, `LIM_ROLL_CD` ile sınırlı (`ArduPlane/Attitude.cpp:165-213`, `:411`). |
| Dümen | `steering_control.rudder = pilot` (`ArduPlane/mode_groundeffect.cpp:64`) | Bu atama `calc_nav_yaw_coordinated` tarafından eziliyor (`ArduPlane/Attitude.cpp:497`). Sonuçta pilot dümeni ile yaw damper birlikte çalışıyor. |
| Gaz kesme | Pilotun gaz kolu 0 ise motor 0 (`ArduPlane/mode_groundeffect.cpp:71-74`) | Yalnızca `GROUND_EFFECT` modunda. AUTO 50/51'de böyle bir kesme yok. |

### Döngü hızı
- `update()` fonksiyonu `update_control_mode` görevinden çağrılıyor. Görev 400 Hz için planlanmış (`ArduPlane/ArduPlane.cpp:39`), ama gerçek hız `SCHED_LOOP_RATE` ile sınırlı. **Plane'de varsayılan 50 Hz** (`libraries/AP_Scheduler/AP_Scheduler.cpp:34-38`). QuadPlane'de 300 Hz (`ArduPlane/Parameters.cpp:1418`).
- Koddaki "This method runs at 400Hz" yorumu (`ArduPlane/mode_groundeffect.cpp:56`) varsayılan ayarlarla doğru değil.
- Mesafe sensörü 50 Hz okunuyor (`ArduPlane/ArduPlane.cpp:61`). Türev terimi basamaklı (merdiven biçimli) veri görüyor.

### PID kazançları
- Eski `libraries/PID` sınıfı kullanılıyor. Kurucu ve parametre varsayılanları **0** (`libraries/PID/PID.h:16-25`, `libraries/PID/PID.cpp:17-32`). `ParametersG2` içinde başlangıç değeri verilmemiş (`ArduPlane/Parameters.h:604-606`).
- Bu yüzden **varsayılan olarak `ELE` ve `FLP` kazançları 0**: yunuslama talebi 0°, flap 0. Kutudan çıktığı haliyle yalnızca gaz çalışıyor.
- **Gaz P kazancı her moda girişte eziliyor:** `kP = (THR_MAX−THR_MIN)/(ALT_MAX−ALT_MIN)`, varsayılanlarla 80/80 = **1 %/mm** (`ArduPlane/mode_groundeffect.cpp:33-34`). Gaz, ALT_MIN'de THR_MAX'a, ALT_MAX'ta THR_MIN'e doğrusal gidiyor. Kullanıcının ayarladığı `GNDEFCT_THR_P` bu modda etkisiz. AUTO'da ise bu atama hiç yapılmıyor (bkz. e).
- `get_pid` 1 saniyeden uzun süre çağrılmazsa integratörü sıfırlıyor (`libraries/PID/PID.cpp:44-51`).

---

## d) Yeni parametreler

Hiçbirinde `@Param`, `@Range` veya `@Units` metadata'sı yok. Bu yüzden GCS'de açıklama ve aralık görünmez, sınır kontrolü de yapılmaz.

| Parametre | Tip | Varsayılan | Aralık (kodda) | Birim | Açıklama ve kaynak |
|---|---|---|---|---|---|
| `GNDEFCT_THR_MIN` | AP_Int16 | 10 | yok (sonra 0..100'e sınırlanıyor) | % | Gaz alt sınırı. İleri besleme ve gaz P kazancı hesabına da giriyor (`ArduPlane/Parameters.cpp:1102`, `ArduPlane/config.h:313-315`) |
| `GNDEFCT_THR_MAX` | AP_Int16 | 90 | yok | % | Gaz üst sınırı (`ArduPlane/Parameters.cpp:1103`, `ArduPlane/config.h:317-319`) |
| `GNDEFCT_ALT_MIN` | AP_Int16 | 30 | yok, en fazla 32767 | **mm** | Hedef yüksekliğin alt ucu (`ArduPlane/Parameters.cpp:1104`, `ArduPlane/config.h:321-323`) |
| `GNDEFCT_ALT_MAX` | AP_Int16 | 110 | yok, en fazla 32767 | **mm** | Hedefin üst ucu. **Sensör kaybında kullanılan sahte okuma** da bu (`ArduPlane/Parameters.cpp:1105`, `ArduPlane/config.h:325-327`) |
| `GNDEFCT_51_MULT` | AP_Float | 2.0 | yok | × | 51 numaralı yol noktasında hedefi ve sahte okumayı çarpıyor (`ArduPlane/Parameters.cpp:1107`) |
| `GNDEFCT_THR_P/I/D/IMAX` | PID | 0/0/0/0 | yok | %/mm | Gaz PID'i. P, GE moduna girişte eziliyor (`ArduPlane/Parameters.cpp:1305`) |
| `GNDEFCT_ELE_P/I/D/IMAX` | PID | 0/0/0/0 | yok | cd/mm | Yunuslama açısı PID'i (`ArduPlane/Parameters.cpp:1306`) |
| `GNDEFCT_FLP_P/I/D/IMAX` | PID | 0/0/0/0 | yok | %/mm | Flap PID'i (`ArduPlane/Parameters.cpp:1307`) |

**Parametre anahtarlarıyla ilgili sorunlar:**
- `k_param_gndefct_51_multiplier = 246`, eski `k_param_pidNavPitchAltitude` ile aynı enum değerini alıyor. `k_param_gndEffect_alt_max = 247`, eski `pidWheelSteer` anahtarının yerine geçiyor. Yazarın kendi notu: "not sure if that's kosher" (`ArduPlane/Parameters.h:340-350`).
- Upstream kuralı eski anahtarların yeniden kullanılmasını yasaklıyor, çünkü eski EEPROM verisi yeni parametre olarak okunabilir.
- G2 indeksleri 29, 30 ve 31, master'da başka parametrelere ait (bkz. g).

---

## e) Mevcut sistemlerle ilişki

### TECS
- `GROUND_EFFECT` modunda kapalı (`plane.auto_throttle_mode = false`, `ArduPlane/mode_groundeffect.cpp:19`). Hava hızı ya da enerji yönetimi yok, **stall koruması da yok.** Hava hızı hiçbir yerde kullanılmıyor.
- AUTO 50/51'de TECS arka planda çalışmaya devam ediyor (`ArduPlane/ArduPlane.cpp:38`), ama yunuslama ve gaz çıktısı kullanılmıyor. Yalnızca her döngüde yunuslama integratörü sıfırlanıyor (`ArduPlane/mode_auto.cpp:80`). Yol noktasının irtifa alanı düşey kontrolde yok sayılıyor. GE bacağından normal bacağa geçişte TECS'in ani bir düzeltme yapması beklenmeli.

### Yol noktası navigasyonu (AUTO)
- **Evet, AUTO'da yükseklik tutma yol noktası uçarken çalışıyor, ama yalnızca 50/51 komutlarında.** Yanal yönlendirme normal L1 ile yapılıyor (`plane.calc_nav_roll()`, `ArduPlane/mode_auto.cpp:107`). Düşey eksende GE PID'leri devrede (`:104-112`). Yol noktasına varış kontrolü normal `verify_nav_wp` ile (`ArduPlane/commands_logic.cpp:221-223`).
- `GROUND_EFFECT` modunun kendi navigasyonu yok: kanatlar düz, yön pilotta.
- **AUTO dalındaki hatalar:**
  1. Gaz P kazancı yalnızca `ModeGroundEffect::_enter()` içinde ayarlanıyor (`ArduPlane/mode_groundeffect.cpp:33-34`). Açılıştan beri GE moduna hiç girilmediyse ve `GNDEFCT_THR_P` elle ayarlanmadıysa, AUTO'da gaz **sabit ileri besleme değerinde kalıyor.**
  2. Flap PID'i AUTO'da hiç hesaplanmıyor, ama flap çıkışı `mode_groundeffect.desired_flap_percentage` değerini kullanıyor (`ArduPlane/servos.cpp:632-633`). Bu, en son GE modunda kalan **eski bir değer**.
  3. Durum `static` değişkenlerde tutuluyor ve her 50/51 bacağında paylaşılıyor (`ArduPlane/mode_auto.cpp:83-84`).
  4. Gaz kesme ve sensör varlık kontrolü yok.

### Dönüşler
- **Dönüşe özel bir kod yok.** GE modunda kanatlar düz. Dönüş ancak pilotun stick mixing ile verdiği yatışla (`LIM_ROLL_CD` sınırlı) ya da dümenle yapılabiliyor.
- AUTO'da L1, `LIM_ROLL_CD`'ye kadar (varsayılan 45°) yatış veriyor (`ArduPlane/Attitude.cpp:582-661`). WIG'e özel bir yatış sınırı ya da kanat ucu–su mesafesi hesabı yok.
- **Kritik nokta:** Aşağı bakan sensör yatışta **eğik mesafeyi** ölçüyor ve `cos(roll)·cos(pitch)` düzeltmesi yapılmıyor. Ölçülen mesafe gerçekten uzun olduğu için denetleyici aracı yüksekte sanıp alçaltıyor. Tam o sırada alçaktaki kanat ucu suya yaklaşıyor. Dönüşte bu, **kanat ucunun suya çarpması** riski demek.

### Kalkış ve iniş
- **Özel kod yok.** GE modunda gaz kolu 0'dan büyük olduğu anda denetleyici devreye giriyor. Su üstünde okuma hedefin altında kaldığı için gaz THR_MAX'a doğru gidiyor ve (ELE_P > 0 ise) burun kalkıyor. Bu fiilen bir kalkış ivmesi sağlıyor, ama planing/hump evresini yöneten bir mantık yok.
- İnişte pilot gazı kesiyor (motor 0). Yunuslama PID'i ise yüksekliği korumaya çalışmaya devam ediyor: alçaldıkça burun kalkıyor. Hız da düştüğü için **stall riski** oluşuyor.
- AUTO'da `NAV_TAKEOFF` ve `NAV_LAND` dalları 50/51'den önce işlendiği için değişmeden kalıyor (`ArduPlane/mode_auto.cpp:61-78`).

---

## f) Güvenlik

### Mesafe sensörü verisi kaybolursa
- Durum `Good` değilse son iyi okuma **1 saniye boyunca tutuluyor** (`ArduPlane/mode_groundeffect.cpp:46-49`). Sonra **"yüksekteyim" varsayılıp `GNDEFCT_ALT_MAX` sahte okuma olarak kullanılıyor** (`:52-54`). Hata negatif oluyor, burun aşağı iniyor ve gaz THR_MIN'e doğru gidiyor. Kısacası araç bilerek alçalıyor.
- Su üstünde bu "yavaşça suya otur" anlamına gelebilir. Ancak hızlıyken ya da dönüşteyken burun-aşağı komutu tehlikeli.
- GCS uyarısı, log kaydı, mod değişimi ve arming öncesi (pre-arm) kontrol yok. `AP_Arming::Method::GROUND_EFFECT_MODE_BAD_RANGEFINDER` tanımlanmış ama **hiç kullanılmıyor** (`libraries/AP_Arming/AP_Arming.h:75`).

### Menzil dışına çıkarsa
- Durum yalnızca `RNGFNDx_MIN_CM` ile `RNGFNDx_MAX_CM` arasında `Good` oluyor (`libraries/AP_RangeFinder/AP_RangeFinder_Backend.cpp:56-66`).
- **Varsayılan `MIN_CM = 20 cm`** (`libraries/AP_RangeFinder/AP_RangeFinder_Params.cpp:49`), varsayılan GE hedefi ise 7 cm. Varsayılan ayarlarla okuma **hiçbir zaman `Good` olmuyor** ve denetleyici sürekli sahte değerle çalışıyor. `MIN_CM` mutlaka düşürülmeli.
- `OutOfRangeLow` (suya değme) ile `OutOfRangeHigh` aynı şekilde "veri kaybı" sayılıyor. 1 saniye sonra ikisinde de "yüksekteyim, alçal" komutu veriliyor. Alçaktayken yön yanlış, ama araç su üstündeyse zararı sınırlı.

### Dalga ve gürültü
- **Dalga için hiçbir önlem yok.** Ham hata doğrudan PID'e giriyor. D kazancı dalga frekansındaki değişimi büyütüyor ve yunuslama ile gaz dalgayı takip ediyor.
- Tek koruma, Benewake'in zayıf sinyal eşiği (`<=20`, sabit kodlanmış). Bu eşik su yüzeyinden gelen zayıf yansımaları attığında sensör `NoData` durumuna düşüyor (seri sürücü zaman aşımı 200 ms) ve yukarıdaki kayıp davranışı başlıyor.
- IMO Type-B'de dalga yüksekliği uçuş yüksekliğinin kayda değer bir kısmı olabilir. Ortalama su seviyesi tahmini, düşük geçiren filtre ve EKF düşey hızıyla tamamlayıcı (complementary) filtre şart.

### Sayısal hatalar
- `(int16_t)` dönüşümü: PID çıktısı ±32767'yi aşarsa C++'ta tanımsız davranış oluşuyor (`ArduPlane/mode_groundeffect.cpp:62`, `:76`). Büyük kazanç ve büyük hata kombinasyonunda gerçekçi bir risk.
- Seri sürücüde `distance_cm*10` hesabı `uint16` olarak 65.5 m üstünde taşıyor (`libraries/AP_RangeFinder/AP_RangeFinder_Backend_Serial.cpp:62`). Bu yalnızca `MAX_CM > 6553` ayarlanırsa önem taşıyor.
- **PWM sürücüsü hatalı:** `distance_mm` yalnızca okuma başarısız olduğunda ve cm değeriyle atanıyor (`libraries/AP_RangeFinder/AP_RangeFinder_PWM.cpp:110-111`). Başarılı her döngüde ise üstüne ofset ekleniyor (`:119-120`), değer sürekli kayıyor. **PWM sensörü kullanılmamalı.**
- **Flap mantığı:** `auto_flap_percent = -desired` atanıyor (`ArduPlane/servos.cpp:633`). Hemen ardından `abs(manual) > auto` koşulu geliyor (`:637-638`). Manuel flap 0 iken her negatif değer 0 ile eziliyor. Sonuç: flap yalnızca tek yönde çalışıyor. Hangi yönde olduğu kazancın işaretine bağlı.

### Mod kullanılmasa bile etkileyen yan etkiler
- **RFND logu:** `Status` ve `Orient` alanlarına sinyal gücünün yüksek ve düşük baytı yazılıyor (`libraries/AP_RangeFinder/AP_RangeFinder.cpp:793-794`). Log analiz araçları bu alanları yanlış yorumlar.
- **MAVLink `DISTANCE_SENSOR`:** `current_distance` alanı cm olarak tanımlı, ama **mm gönderiliyor** (`libraries/GCS_MAVLink/GCS_Common.cpp:295`). GCS ve companion bilgisayar 10 kat yanlış mesafe görür.
- **TFMini-Plus:** Açılışta sensör mm moduna alınıp ayar **kalıcı olarak kaydediliyor** (`...TFMiniPlus.cpp:70-78`). Aynı sensör başka bir firmware'e takılırsa yanlış birimle çalışır.

### Failsafe ve `FS_LONG_ACTN`
Değer tanımları: `ArduPlane/defines.h:35-45`.

| Olay | `GROUND_EFFECT` modundayken | AUTO'da 50/51 bacağındayken |
|---|---|---|
| Kısa FS (`FS_SHORT_ACTN`) | 0 veya 1 → **CIRCLE**, 2 → **FBWA**. Moddan hemen çıkılıyor (`ArduPlane/events.cpp:9-27`). CIRCLE, TECS ve yatışla tur atıyor; su üstünde alçakta tehlikeli. | 0 → değişiklik yok (görev sürüyor). 1 → CIRCLE, 2 → FBWA (`ArduPlane/events.cpp:43-56`) |
| Uzun FS, `FS_LONG_ACTN=0` (Continue) | **RTL** (else dalı, `ArduPlane/events.cpp:76-97`) | Görev sürüyor, GE kontrolü RC olmadan devam ediyor (`ArduPlane/events.cpp:112-126`) |
| Uzun FS, `FS_LONG_ACTN=1` (RTL) | RTL | RTL |
| Uzun FS, `FS_LONG_ACTN=2` (Glide) | **FBWA, gaz 0**: kanatlar düz süzülüp suya oturma | FBWA, gaz 0 |
| Uzun FS, `FS_LONG_ACTN=3` | Paraşüt | Paraşüt |

- RTL, `ALT_HOLD_RTL` irtifasına tırmanıyor (varsayılan 100 m). Bu, yer etkisinden çıkmak demek. Type-B yer etkisinden geçici çıkışa izin verse de bu davranış istenmez.
- **WIG için en güvenli seçenek büyük olasılıkla `FS_SHORT_ACTN=2` ve `FS_LONG_ACTN=2` (FBWA, kanatlar düz, süzülerek suya oturma).** Önce SITL'de doğrulanmalı.

---

## g) Güncel sürüme taşıma

**Zorluk: Orta.**

**Gerekçe:**
- Kod küçük (yaklaşık 300 satır) ve mantığı basit.
- Ama dokunduğu API'lerin neredeyse hepsi değişmiş. Deneme birleştirmesinde (`git merge-tree`, merge-base `27720f2235`, `upstream/master` @ `755258dbb4`) **31 dosyanın 22'si metin düzeyinde çakışıyor.**
- Metin düzeyinde çakışmayan dosyalar da anlam düzeyinde bozuk. Örneğin `Parameters.cpp` çakışmıyor, ama G2 indeksleri 29-31 başka parametrelerle çakışıyor.
- Öte yandan diff'in yarısından fazlası (mm altyapısı) master'da **gereksiz**, çünkü mesafe artık `float` metre olarak tutuluyor. Yeniden yazılacak kısım mod sınıfı ve parametrelerle sınırlı.
- Kararlı Plane'e (4.7.1) taşımak master'a taşımakla aynı zorlukta: API'ler 4.2-4.5 döneminde değişti.

### Değişen API'ler

| Konu | Dal (4.1.0dev, 2020) | master (4.8.0-dev) |
|---|---|---|
| Mod numarası | 25 (`ArduPlane/mode.h:46`) | 25 = `LOITER_ALT_QLAND`, 26 = `AUTOLAND` (`master:ArduPlane/mode.h:68,71`). Boş bir numara seçilmeli, en az 27. |
| Mod bayrakları | `plane.auto_throttle_mode`, `auto_navigation_mode`, `throttle_allows_nudging` alanları (`ArduPlane/mode_groundeffect.cpp:18-20`) | Sanal metotlar: `does_auto_throttle()`, `does_auto_navigation()`, `allows_throttle_nudging()` (`master:ArduPlane/mode.h:130-138`) |
| Tutum döngüsü | Global `stabilize()` | `Mode::run()` (`master:ArduPlane/mode.h:87`) |
| Mesafe sensörü | `uint16 distance_cm`, dalın eklediği `distance_mm` | `float distance_m`, `signal_quality_pct`, `distance_orient()` (`master:libraries/AP_RangeFinder/AP_RangeFinder.h:227-228,297,314`). mm altyapısına gerek kalmıyor. |
| Benewake seri | `get_reading(uint16 &cm)` | `get_reading(float &reading_m)` (`master:libraries/AP_RangeFinder/AP_RangeFinder_Benewake.cpp:66`). Model bazında güç eşikleri yalnızca yorumlarda (`:50-62`). Zayıf sinyal filtresi gerekiyorsa `signal_quality_pct` ile eklenmeli. |
| TFMini-Plus | Dal mm moduna alıyor | Master hâlâ CM modunda (`master:...TFMiniPlus.cpp:47,52`), çözünürlük 1 cm. Güç `<100` filtresi zaten var (`:145`). |
| G2 parametreleri | 29, 30, 31 indeksleri | Master'da 41'e kadar dolu (`master:ArduPlane/Parameters.cpp:1298`). Çakışma kesin. Kendi `AP_Param` grubu kullanılmalı. |
| Eski `k_param` anahtarları | 246, 247 yeniden kullanılmış | Master'da bu anahtarlar hâlâ "unused" olarak duruyor (`master:ArduPlane/Parameters.h:340-350`). Yeniden kullanılmamalı. |
| PID sınıfı | `libraries/PID` | Master'da hâlâ var, ama upstream'de tercih edilen `AC_PID` (filtre, slew, PIDx logu). |
| Alternatif yol | — | Lua: `NAV_SCRIPT_TIME` ve `set_target_throttle_rate_rpy` (`master:ArduPlane/Plane.h:1252-1254`) ile firmware'i forklamadan GE bacakları yazılabilir. |

### Çakışan dosyalar (merge-tree çıktısı, 22 dosya)
- **ArduPlane:** `GCS_Mavlink.cpp`, `Parameters.h`, `Plane.h`, `config.h`, `control_modes.cpp`, `events.cpp`, `mode.h`, `mode_auto.cpp`
- **AP_Arming, AP_Mission, MAVLink:** `AP_Arming/AP_Arming.h`, `AP_Mission/AP_Mission.cpp`, `GCS_MAVLink/GCS_Common.cpp`
- **AP_RangeFinder:** `AP_RangeFinder.cpp/.h`, `_Backend.h`, `_Backend_Serial.cpp/.h`, `_Benewake.cpp/.h`, `_Benewake_TFMiniPlus.cpp`, `_PWM.cpp`, `_VL53L0X.cpp`, `_analog.cpp`

Metin olarak birleşen ama elle gözden geçirilmesi gereken dosyalar: `Parameters.cpp`, `servos.cpp`, `commands_logic.cpp`, `GCS_Plane.cpp`.

### Önerilen taşıma biçimi
1. RangeFinder değişikliklerinin hiçbirini taşımayın. Master'ın `float` metre ve `signal_quality_pct` alanlarını kullanın.
2. Kendi `AP_Param` grubu olan yeni bir `ModeGroundEffect` sınıfı yazın (örneğin `GE_` öneki). `run()` içinde `AC_PID` kullanın. Yunuslama çıkışını `LIM_PITCH` aralığına sınırlayın ve hava hızı tabanı ekleyin.
3. Görev için 50/51 gibi ham numaralar yerine tanımlı bir komut kullanın: `NAV_SCRIPT_TIME`, ya da bir `DO_` komutuyla "GE bacağı" bayrağı.
4. Kesin bir mod numarası almak ya da ileride çakışmamak için upstream'e PR açmak bakım yükünü en aza indirir.

---

## h) SITL testi

### Mesafe sensörünü simüle etme
- **Dalın kodunda:** SITL'in kendi sensör tipi (`RNGFND1_TYPE=100`) `supports_mm_precision()` metodunu ezmediği için GE moduna **girilemez.** Bunun yerine analog tip kullanılmalı. Analog sürücü mm destekli sayılıyor (`libraries/AP_RangeFinder/AP_RangeFinder_analog.h`). Autotest'teki hazır ayarlar (`Tools/autotest/common.py:2686-2691`):
  `RNGFND1_TYPE=1`, `RNGFND1_PIN=0`, `RNGFND1_SCALING=12.12`, `RNGFND1_MIN_CM=0`, `RNGFND1_MAX_CM=4000`. `RNGFND1_ORIENT=25` (aşağı) varsayılan değer.
- SITL, yatış ve yunuslamada eğik mesafeyi zaten simüle ediyor (`libraries/SITL/SIM_Aircraft.cpp:452-480`). Bu, dönüş testi için yararlı.
- **Gürültü ve kesinti:** `SIM_SONAR_RND` (gürültü, m) ve `SIM_SONAR_GLITCH` (kesinti) (`libraries/SITL/SITL.cpp:50-55`). Master'da da var (`master:libraries/SITL/SITL.cpp:1554-1565`).
- **Dalga:** `SIM_WAVE_*` parametreleri yalnızca `SIM_Sailboat` modelinde etkili, Plane modelinde değil. Dalgayı Plane'de taklit etmek için `SIM_SONAR_RND` kullanılabilir. Daha gerçekçi bir test için `rangefinder_range()`'e sinüs dalga eklenmesi gerekir (yalnızca test için SITL değişikliği).
- **Master'a taşıdıktan sonra:** `RNGFND1_TYPE=100` (SIM) kullanılabilir (`master:libraries/AP_RangeFinder/AP_RangeFinder.h:202`).

### Önerilen test adımları
1. Dalı derleyin: `./waf configure --board sitl && ./waf plane`. 2020 kodu eski bir derleyici ve Python gerektirebilir, Ubuntu 20.04 konteyneri önerilir. (Bu analizde derleme yapılmadı.)
2. `sim_vehicle.py -v ArduPlane -f plane --console --map` ile başlatın, yukarıdaki RNGFND ayarlarını yazıp yeniden başlatın.
3. SITL uçağının ölçeğine göre parametre girin: örneğin `GNDEFCT_ALT_MIN=2000`, `GNDEFCT_ALT_MAX=4000` (2–4 m), `GNDEFCT_ELE_P` küçük bir başlangıç değeri (örneğin 0.5 cd/mm), `GNDEFCT_THR_MIN/MAX` seyir gazının ±20 puan çevresi, `FLTMODE6=25`.
4. FBWA'da kalkın, yaklaşık 3 m'ye inin, mod 25'e geçin. `RFND`, `ATT.DesPitch` ve `CTUN.ThO` loglarını izleyin. RFND logunda `Status/Orient` alanlarının artık sinyal gücü olduğunu unutmayın.
5. **Sensör kaybı:** uçuş sırasında `SIM_SONAR_GLITCH=1` verin. 1 saniye tutma ve ardından alçalma beklenir.
6. **Gürültü:** `SIM_SONAR_RND` değerini 0.05, 0.2 ve 0.5 m yapın. Yunuslama ve gaz salınımına bakın, D kazancını ayarlayın.
7. **Failsafe:** `SIM_RC_FAIL=1` ile `FS_SHORT_ACTN` ve `FS_LONG_ACTN` kombinasyonlarını (0/1/2) deneyin. Yukarıdaki tabloyu doğrulayın.
8. **AUTO:** QGC WPL görev dosyasında komut kimliği 50 veya 51 olan satırlar kullanın, ör. `1 0 3 50 0 0 0 0 <lat> <lon> 0 1`. 45° dönüşte eğik mesafe etkisini ve gazın sabit kalmasını (THR_P hatası) gözlemleyin.

---

## i) Lisans (GPLv3, kısa hatırlatma, hukuki görüş değildir)
- ArduPilot GPLv3 lisanslı. Bu daldaki değişiklikler ve bunlara dayanan kendi kodumuz **türev çalışma** sayılır.
- Firmware'i **üçüncü kişilere dağıtırsak**, ilgili kaynak kodu GPLv3 ile vermek ya da yazılı teklif sunmak zorundayız. Dağıtım, aracın teslim edilmesini ya da satılmasını da kapsar. Yalnızca kurum içinde kullanım dağıtım sayılmaz.
- Telif başlıkları korunmalı. Dalın eklediği `ArduPlane/mode_groundeffect.cpp` dosyasında lisans başlığı yok; taşırken GPLv3 başlığı eklenmeli.
- Kapalı ve tescilli bir eklenti istenirse firmware'e değil, ayrı bir companion bilgisayar sürecine konmalı (MAVLink üzerinden). Lua betikleri firmware'le birlikte dağıtılırsa GPL kapsamında değerlendirilmeli.

---

## Bizim araç için sonraki 5 adım

1. **Dalı cherry-pick etmeyin; master (4.8-dev) ya da 4.7.1 üzerinde yeniden yazın.** Yeni bir `ModeGroundEffect` sınıfı (numara ≥27, kendi `GE_` parametre grubu, `run()`, `AC_PID`). RangeFinder, log ve MAVLink değişikliklerinin hiçbirini almayın.
2. **Yükseklik tahmincisini tasarlayın:** mesafe sensörü (`float` m, `signal_quality_pct` eşiği), `cos(roll)·cos(pitch)` eğim düzeltmesi, aykırı değer ve medyan eleme, EKF düşey hızıyla tamamlayıcı filtre ve dalga için ortalama yüzey tahmini. Sensör seçimi: su yüzeyinde güvenilir ölçüm yapan bir model (Benewake TF02-Pro/TF03 ya da radar altimetre), mümkünse iki sensörle yedeklilik.
3. **Güvenlik zarfını ekleyin:** yunuslama çıkışını `LIM_PITCH` aralığına sınırlayın, hava hızı tabanı (stall koruması), yüksekliğe bağlı yatış sınırı (kanat ucu mesafesi), sensör kaybında GCS uyarısı ve log kaydı, pre-arm kontrolü, ve WIG için `FS_SHORT_ACTN=2` ve `FS_LONG_ACTN=2` (FBWA ile süzülerek suya oturma).
4. **Önce SITL'de doğrulayın:** `RNGFND1_TYPE=100`, `SIM_SONAR_RND` ve `SIM_SONAR_GLITCH` ile; dalga için `rangefinder_range()`'e sinüs eklenmiş bir test yaması. h) bölümündeki 8 adımı otomatik bir autotest senaryosuna çevirin.
5. **Görev entegrasyonu ve bakım planı:** GE bacaklarını ham 50/51 yerine `NAV_SCRIPT_TIME` (Lua) ya da tanımlı bir `DO_` komutuyla işaretleyin. Fork bakım yükünü azaltmak için modu upstream'e PR olarak önerin. GPLv3 yükümlülükleri için kaynak yayımlama sürecini şimdiden belirleyin.
