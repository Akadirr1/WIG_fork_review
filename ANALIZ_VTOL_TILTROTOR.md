# ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite

**Kaynak:** ArduPilot `ArduPlane-stable` etiketi, commit `dbe792162d` (2026-09-02). `ArduPlane/version.h` → `"ArduPlane V4.7.1"`.
**Kapsam:** QuadPlane, tiltrotor ve geçiş kodu. Ek olarak assist, VTOL kalkış/iniş, mod altyapısı ve GCS mesajları.
**Yöntem:** Kodu sığ klonla okudum. SITL'i derledim ve `quadplane-tilttri` modeliyle iki kısa uçuş yaptım (§8). ArduPilot kodunda değişiklik yapmadım.

> **Referanslar:** Bütün `dosya:satır` referansları ArduPilot kök dizinine göre ve `dbe792162d` commit'ine aittir. **Çıkarım:** koddan çıkarıldı ama çalıştırarak doğrulanmadı. **Doğrulanmadı:** emin değilim. Parametre varsayılanları, `AP_GROUPINFO` satırlarındaki değerlerdir. SITL parametre dosyaları bu değerleri değiştiriyor (§8).

---

## Kısa özet

- **Kalkış yüksekliği yalnızca EKF'den geliyor** (varsayılan kaynak baro). Mesafe sensörü kalkışta hiç kullanılmıyor. Alt sınır yok: 2–5 m'lik bir kalkış kodun izin verdiği bir şey.
- **Tiltrotor ileri geçişi iki aşamalı.** `AIRSPEED_WAIT` aşamasından `TIMER` aşamasına geçmek için hava hızının `AIRSPEED_MIN`'in üstüne çıkması ve assist'in kapalı olması gerekiyor. `TIMER`'dan `DONE`'a geçiş, motorlar tam ileri dönünce oluyor. `AIRSPEED_WAIT` boyunca tilt açısı `Q_TILT_MAX` ile sınırlı.
- **Geçiş kodunda minimum yükseklik kontrolü yok.** 5–10 m'de geçiş kodun izin verdiği bir şey. Ancak `TIMER` aşamasında VTOL motorlar açık döngüde kısılıyor. O aşamada yüksekliği yalnızca TECS tutuyor (pitch en fazla ±8°).
- **Geçiş başarısızlık zaman aşımı varsayılan olarak kapalı** (`Q_TRANS_FAIL=0`). Açılırsa varsayılan eylem **QLAND**, yani suya dikey iniş.
- **SITL'de hazır `quadplane-tilttri` ayarlarıyla geçiş hiç tamamlanmadı.** Araç 72 saniye geçiş durumunda kaldı ve hiçbir uyarı mesajı gelmedi. Neden: `Q_ASSIST_SPEED=18` m/s, aracın 45° tilt ile ulaşabildiği yaklaşık 15 m/s'nin üstünde. Yalnızca `Q_ASSIST_SPEED=12` yapınca geçiş 5,3 s'de tamamlandı.
- **Assist, yüzey etkisinde güvenlik ağı olarak zayıf.**
  - `Q_ASSIST_ALT` tam metre cinsinden. Seyir yüksekliğine eşit ya da yüksek ayarlanırsa assist sürekli açık kalır.
  - Hız assist'i anında tetikleniyor. Ama motorların devir alması, tilt'in geri kalkması ve tırmanma talebinin 2 s'lik rampası araya giriyor.
- **İnişte temas algılama, su için uygun değil.** Koşullar: gaz yaklaşık 5 s alt sınırda olacak ve EKF yüksekliği 4 s boyunca ±0,2 m içinde kalacak. Dalgada inip çıkan bir gövde bu koşulu sağlamayabilir (çıkarım). Bu durum için zaman aşımı da yok.
- **Yeni mod için doğal giriş noktası `Mode` alt sınıfı.** Ancak QuadPlane'in assist ve geçiş durum makinesi, MANUAL/ACRO/TRAINING dışındaki **her** sabit kanat modunda çalışıyor. Asıl çakışma burada.
- **GCS mesajları:**
  - Var: geçiş başladı/bitti (INFO), geçiş başarısız (CRITICAL), irtifa ve açı assist'i (WARNING), olası itki kaybı (EMERGENCY).
  - Yok: hız assist'ine özel mesaj, assist bitişi mesajı, ESC/motor arızası mesajı.

---

## 1. VTOL kalkışı

### 1.1 Yükseklik hangi kaynaktan ölçülüyor?

| Konu | Koddaki davranış | Referans |
|---|---|---|
| Aracın yüksekliği | `current_loc`, `ahrs.get_location()` ile EKF'den geliyor. `relative_altitude`, EKF'nin home'a göre yüksekliği. | `ArduPlane/Plane.cpp:1061`, `:1064-1065` |
| EKF dikey kaynağı | `EK3_SRC1_POSZ` varsayılanı **BARO**. GPS ya da mesafe sensörü ancak bu parametre değiştirilirse kullanılıyor. | `libraries/AP_NavEKF/AP_NavEKF_Source.cpp:46` |
| AUTO `NAV_VTOL_TAKEOFF` hedefi (varsayılan) | Hedef = o anki EKF AMSL yüksekliği + komuttaki irtifa. Komutun frame'i (relative/terrain) **yok sayılıyor**. | `ArduPlane/quadplane.cpp:3396-3397` |
| `Q_OPTIONS` bit 3 (RESPECT_TAKEOFF_FRAME) | Hedef ABSOLUTE frame'e çevriliyor. Araç zaten hedefin üstündeyse kalkış adımı atlanıyor. | `ArduPlane/quadplane.cpp:3388-3394` |
| Silahsızken | Hedef her döngüde yeniden hesaplanıyor. Referans, arming anındaki EKF yüksekliği oluyor. | `ArduPlane/quadplane.cpp:3480-3484` |
| Tırmanma | AUTO'da hedef irtifasız, sabit tırmanma hızı: `Q_WP_SPD_UP` (2,5 m/s). | `ArduPlane/quadplane.cpp:3243`, `:3258-3260`; `libraries/AC_WPNav/AC_WPNav.cpp:12`, `:74` |
| GUIDED kalkış | Konum hedefi kullanılıyor: EKF origin'e göre hedef + 5 cm. | `ArduPlane/quadplane.cpp:3244-3257` |
| Mesafe sensörü / terrain | Kalkış hedefinde de bitiş kontrolünde de **kullanılmıyor**. `do_vtol_takeoff` ve `verify_vtol_takeoff` içinde çağrı yok. | `ArduPlane/quadplane.cpp:3375-3530` |
| `Q_WP_RFND_USE` | Varsayılanı 1, ama Plane'de etkisiz. ArduPlane, `set_rangefinder_terrain_U_*` fonksiyonunu hiç çağırmıyor. | `libraries/AC_WPNav/AC_WPNav.cpp:29`; `AC_WPNav.h:26`, `:30` |
| EKF yer etkisi telafisi | Kalkışın ilk 3 s'sinde `set_takeoff_expected(true)` çağrılıyor (`Q_OPTIONS` bit 13 bunu kapatır). Baro ölü bandı `EK3_GND_EFF_DZ` = 4 m. | `ArduPlane/quadplane.cpp:3486-3489`; `libraries/AP_NavEKF3/AP_NavEKF3.cpp:713` |

### 1.2 Kalkış ne zaman bitmiş sayılıyor?

| Koşul | Sonuç | Mesaj | Referans |
|---|---|---|---|
| `current_loc.alt >= next_WP_loc.alt` (marj yok, kontrol 10 Hz) | Kalkış tamam. Ardından `transition->restart()`, TECS pitch sınırı ±`Q_TRAN_PIT_MAX` ve `set_alt_target_current()` geliyor. | **Yok** (yalnızca "Mission: N ..." çıkıyor) | `ArduPlane/quadplane.cpp:3505-3511`; `ArduPlane/Plane.cpp:75` |
| `Q_TKOFF_FAIL_SCL > 0` ve süre aşıldı (limit = max(tahmini süre × ölçek, 5 s)) | QLAND (`VTOL_FAILED_TAKEOFF`) | CRITICAL "Failed to complete takeoff within time limit" | `ArduPlane/quadplane.cpp:3432`, `:3492-3496`; varsayılan 0 (kapalı) `:391` |
| `Q_TKOFF_ARSP_LIM > 0` ve hava hızı bu değerin üstünde | QLAND | CRITICAL "Failed to complete takeoff, excessive wind" | `ArduPlane/quadplane.cpp:3499-3503`; varsayılan 0, `:400` |

### 1.3 Su üstünde birkaç metrelik kalkış mümkün mü?

**Evet, kod buna izin veriyor.** Kalkış irtifası için alt sınır yok. İrtifa 0 verilirse kalkış ilk kontrolde tamamlanıyor (`ArduPlane/quadplane.cpp:3396-3397`, `:3505`). SITL'de 10 m'lik hedef 10,1 m'de, 5,25 s'de tamamlandı (§8).

Alçak kalkıştan sonra devreye girebilecek mekanizmalar:

| Mekanizma | Varsayılan | Etki | Referans |
|---|---|---|---|
| `Q_NAVALT_MIN` | 0 (`quadplane.cpp:475`) | Bu yüksekliğe kadar roll ve pitch sıfırda tutuluyor, yatay konum tutulmuyor. | `ArduPlane/quadplane.cpp:3212-3227` |
| `Q_ASSIST_ALT` | 0 (`quadplane.cpp:409`) | Kalkış yüksekliğinden büyükse geçiş hiç bitmiyor (§2.4, §4). | `ArduPlane/VTOL_Assist.cpp:104-113` |
| `Q_LAND_FINAL_ALT` | 6 m (`quadplane.cpp:132`) | 6 m'nin altında başlayan her VTOL inişi doğrudan final hızıyla (0,5 m/s) iniyor. | `ArduPlane/quadplane.cpp:3604-3615` |
| QRTL tırmanması | `Q_RTL_ALT_MIN` 10 (`:508`), `Q_RTL_ALT` 15 (`:180`) | QRTL önce en az `constrain(10, 6, 15)` = 10 m'ye tırmanıyor. Geçiş başarısızlığında `Q_TRANS_FAIL_ACT=1` seçilmişse bu da geçerli. | `ArduPlane/mode_qrtl.cpp:26-49` |
| Gaz bastırma (throttle suppression) | — | VTOL motorlar ≥2 s kapalıysa, stick 0 ise, \|vz\| < 1 m/s ise ve yükseklik ≤ 5 m ise motorlar GROUND_IDLE'a alınıyor. AUTO VTOL kalkışı bundan muaf. | `ArduPlane/quadplane.cpp:1856-1917` (5 m: `:1904`; muafiyet: `:1909-1911`) |

**Su için dikkat (çıkarım):**
- Kalkış yüksekliği tamamen baroya bağlı. Dalgalı yüzeyde "suya göre yükseklik" bilinmiyor; referans, arming anındaki EKF yüksekliği.
- `Q_TKOFF_FAIL_SCL=0` iken, suya yapışıp kalkamayan bir araç için zaman aşımı yok.

---

## 2. Tiltrotor ileri geçişi (hover → ileri uçuş)

### 2.1 Geçiş nerede çalışıyor?

- `QuadPlane::update()`, `ArduPlane/servos.cpp:882`'den her döngüde çağrılıyor. Sabit kanat modlarında davranış şöyle (`ArduPlane/quadplane.cpp:1768-1783`):
  - MANUAL, ACRO, TRAINING → VTOL motorlar kapanıyor ve `force_transition_complete()` çağrılıyor.
  - **Diğer bütün sabit kanat modları** → `transition->update()`.
- `Tiltrotor_Transition`, `update()` fonksiyonunu ezmiyor (`ArduPlane/tiltrotor.h:138`). Yani tiltrotorda da `SLT_Transition::update()` çalışıyor (`ArduPlane/quadplane.cpp:1478`).
- Q modundayken `VTOL_update()` durumu `AIRSPEED_WAIT` olarak kuruyor (`ArduPlane/quadplane.cpp:1696-1716`). Sonraki sabit kanat moduna geçilince ileri geçiş kendiliğinden başlıyor.
- AUTO'da VTOL kalkışı bitince `transition->restart()` çağrılıyor (`ArduPlane/quadplane.cpp:3508`).

### 2.2 Durum makinesi

Durumlar: `AIRSPEED_WAIT=0`, `TIMER=1`, `DONE=2` (`ArduPlane/transition.h:111-115`). Log'da `QTUN.Trn` alanında görünüyor.

| Durum | Ne oluyor | Çıkış koşulu | Referans |
|---|---|---|---|
| `AIRSPEED_WAIT` | VTOL motorlar `hold_hover()` ile kapalı döngü Z kontrolü yapıyor. Tilt en fazla `Q_TILT_MAX`'a kadar ileri dönüyor; ileri gaz ≥ %50 ise tam `Q_TILT_MAX`'ta. | `have_airspeed && aspeed > AIRSPEED_MIN && !assisted_flight` → `TIMER` | `ArduPlane/quadplane.cpp:1542-1624` (koşul `:1584`); `ArduPlane/tiltrotor.cpp:326-331` |
| `TIMER` | VTOL gazı `Q_TRANSITION_MS` boyunca doğrusal olarak azalıyor (`hold_stabilize`, Z kontrolü **yok**). Tilt tam ileriye dönüyor. | (a) `fully_fwd()` → "Transition FW done", **ya da** (b) `süre > Q_TRANSITION_MS && tilt_angle_achieved()` → "Transition done" | (a) `:1515-1532`; (b) `:1626-1648`, `:1673` |
| `DONE` | VTOL motorlar `SHUT_DOWN` | — | `ArduPlane/quadplane.cpp:1683-1688` |

Tilt hareketi:
- Sürekli tilt'te açı `slew()` ile hız sınırlı değişiyor (`ArduPlane/tiltrotor.cpp:187-196`).
- Hız `Q_TILT_RATE_UP` (40°/s, `tiltrotor.cpp:29`). `Q_TILT_RATE_DN` 0 ise aşağı dönüşte de bu hız kullanılıyor (`tiltrotor.cpp:53`, `:162`).
- `TIMER` boyunca `assisted_flight=true` kalıyor (`quadplane.cpp:1672`). Bu yüzden 90°/s'lik hızlı tilt yolu kullanılmıyor (`tiltrotor.cpp:172-179`).
- `fully_fwd()`: `current_tilt >= get_fully_forward_tilt()` (`tiltrotor.cpp:528-534`).
- `tilt_angle_achieved()`: `ArduPlane/tiltrotor.h:71`.

### 2.3 Geçiş hangi koşullarda tamamlanmış sayılıyor?

| Kriter | Gerekli mi? | Açıklama | Referans |
|---|---|---|---|
| Hava hızı > `AIRSPEED_MIN` | **Evet** | Varsayılan 9 m/s. SITL dosyası bunu 13 yapıyor. | `ArduPlane/Parameters.cpp:295`; `ArduPlane/config.h:124` |
| Assist kapalı | **Evet** | Hız `Q_ASSIST_SPEED`'in altındaysa ya da irtifa/açı/zorla assist aktifse `TIMER`'a geçilmiyor. | `ArduPlane/quadplane.cpp:1584` |
| Motor açısı tam ileri | **Evet** | `TIMER`'da tilt tam ileri olunca hemen `DONE`. | `ArduPlane/quadplane.cpp:1515` |
| Süre `Q_TRANSITION_MS` | Pratikte hayır | Varsayılan 5000 ms, 500–30000 aralığına sınırlanıyor. Sürekli tilt'te (a) yolu süre dolmadan tetikleniyor. Süre bu durumda yalnızca VTOL gaz rampasını belirliyor; motorlar rampa bitmeden kesiliyor. SITL'de "airspeed reached" ile "FW done" arası **1,25 s** oldu, 5 s değil. | `ArduPlane/quadplane.cpp:31`, `:1631`; §8 |
| Hava hızı sensörü | Hayır | Sensör yoksa AHRS sentetik hava hızı (EKF rüzgar tahmini) veriyor ve geçiş buna göre tamamlanıyor. Sentetik hızın uçuştaki doğruluğu **doğrulanmadı**. | `libraries/AP_AHRS/AP_AHRS.cpp:1099-1120`; `ArduPlane/system.cpp:441` |

> **SITL'den çıkan kritik ders:** `AIRSPEED_WAIT`'te tilt `Q_TILT_MAX` ile sınırlı. Araç, bu kısıtlı tilt'le hem `AIRSPEED_MIN`'i hem de `Q_ASSIST_SPEED`'i aşabilmeli. Aşamazsa geçiş hiç bitmiyor ve `Q_TRANS_FAIL=0` iken bunu bildiren bir mesaj da gelmiyor (§8, koşu 1).

### 2.4 Geçiş başarısız olursa

| Parametre | Varsayılan | Anlamı | Referans |
|---|---|---|---|
| `Q_TRANS_FAIL` | 0 s (kapalı) | `AIRSPEED_WAIT`'te geçebilecek en uzun süre | `ArduPlane/quadplane.cpp:346` |
| `Q_TRANS_FAIL_ACT` | 0 | −1 yalnızca uyarı, 0 QLAND, 1 QRTL | `ArduPlane/quadplane.cpp:449-454`; enum `quadplane.h:319-327` |
| `Q_OPTIONS` bit 19 (TRANS_FAIL_TO_FW) | kapalı | Yalnızca tiltrotorda ve yer hızı > ½·`AIRSPEED_MIN` ise geçişi zorla `TIMER`'a alıyor, yani tamamlatıyor | `ArduPlane/quadplane.cpp:1560-1563` |

Akış (`ArduPlane/quadplane.cpp:1550-1581`):
- Kontrol **yalnızca `AIRSPEED_WAIT`'te** yapılıyor; `TIMER`'da yapılmıyor.
- Süre aşılınca CRITICAL "Transition failed, exceeded time limit" mesajı **bir kez** gönderiliyor (`:1555`). Ardından QLAND (`:1567`) ya da QRTL (`:1571-1572`) geliyor.
- Mod değişikliğinin nedeni log'a `ModeReason::VTOL_FAILED_TRANSITION` olarak yazılıyor.
- Sayaç assist başladığında da çalışmaya başlıyor ve yalnızca `DONE`'da sıfırlanıyor (`:1503-1505`, `:1530`, `:1636`). Bu yüzden **ileri uçuştaki uzun bir assist de QLAND'a yol açabilir.**
- `Q_TRANS_FAIL=0` iken araç süresiz olarak `AIRSPEED_WAIT`'te kalabiliyor. SITL koşu 1'de 72 s böyle kaldı.

Geçiş sırasında pitch sınırı (`ArduPlane/quadplane.cpp:4668-4703`). Bu sınır **yalnızca `does_auto_throttle()` modlarında** uygulanıyor (`:4680-4683`); FBWA'da sınır yok.
- `AIRSPEED_WAIT`: ±`Q_TRAN_PIT_MAX` (3°, `:143`). Yer hızı < 3 m/s iken 0°.
- `TIMER`: ±(3+1)·2 = **±8°**.

### 2.5 Geri geçiş (ileri uçuş → hover), kısaca

- Ayrı bir durum makinesi yok. Q moduna girilince `in_vtol_mode()` true oluyor ve tilt `Q_TILT_RATE_UP` hızıyla kalkıyor (`ArduPlane/tiltrotor.cpp:162-163`).
- QRTL ve AUTO `NAV_VTOL_LAND`, `APPROACH → AIRBRAKE → POSITION1 → POSITION2` sırasını izliyor:
  - Durma mesafesi `Q_TRANS_DECEL` ile hesaplanıyor (2,0 m/s², `quadplane.cpp:304`; formül `:4114-4146`).
  - POSITION2 koşulu: hedefe < 10 m, tilt hedef açıda ve yer hızı < 9 m/s (`ArduPlane/quadplane.cpp:2727-2735`).
- `Q_BACKTRANS_MS` (3000, `:448`) pitch sınırını kademeli açıyor (`ArduPlane/quadplane.cpp:4588-4595`).

---

## 3. Geçişte yükseklik tutma ve alçak geçiş

| Aşama | Yüksekliği kim tutuyor? | Kaynak | Referans |
|---|---|---|---|
| `AIRSPEED_WAIT` | VTOL motorlar, kapalı döngü Z kontrolcüsüyle (`hold_hover → run_z_controller`). Tırmanma talebi: auto-throttle modlarda TECS irtifa hatası × 0,1, 2 s'lik rampayla, `Q_WP_SPD_UP/DN` ile sınırlı. TECS da paralel çalışıyor (pitch ±3°). | EKF (inertial nav) | `ArduPlane/quadplane.cpp:1595-1600`, `:1434-1456`, `:1040-1066` |
| `TIMER` | **Yalnızca TECS** (pitch ±8°). VTOL gazı açık döngüde azalıyor. | EKF | `ArduPlane/quadplane.cpp:1673`, `:4694` |
| `DONE` | Yalnızca TECS | EKF | `ArduPlane/quadplane.cpp:1684` |

- TECS irtifa hatası: `target_altitude.amsl_cm - adjusted_altitude_cm()`. Terrain-following açıksa terrain verisi kullanılıyor (`ArduPlane/altitude.cpp:389-399`).
- LAND aşaması dışında TECS'e verilen yükseklik `relative_altitude` (`ArduPlane/Plane.cpp:844-847`).
- **Mesafe sensörü geçişte yükseklik kontrolüne girmiyor.** Tek dolaylı etkisi `Q_ASSIST_ALT` üzerinden (§4).

**5–10 m'de geçişe kod izin veriyor mu?**
- **Evet.** `SLT_Transition::update` ve `VTOL_update` içinde yükseklik, mesafe sensörü ya da terrain kontrolü yok (`ArduPlane/quadplane.cpp:1478-1716`).
- `Q_NAVALT_MIN` yalnızca VTOL kalkışında yatay navigasyonu kısıtlıyor (`ArduPlane/quadplane.cpp:3212-3218`).

**Alçak geçişin riskleri:**
1. **`Q_ASSIST_ALT`, geçiş yüksekliğine eşit ya da büyükse** assist hiç kapanmıyor. Geçiş `AIRSPEED_WAIT`'te takılıyor ve tilt `Q_TILT_MAX`'a geri kalkıyor (`ArduPlane/VTOL_Assist.cpp:104-113`; `quadplane.cpp:1497-1502`, `:1584`).
2. **`TIMER`'da Z kontrolü yok.** Motorlar ileri dönerken yükseklik TECS'e (±8° pitch) kalıyor. 5 m'de hata payı küçük (çıkarım).
   - SITL koşu 2'de geçiş 10,1–13,3 m arasında sorunsuz bitti. İleri uçuş ayağında en düşük yükseklik 9,7 m oldu. 5 m'de deneme yapmadım.
3. **`Q_OPTIONS` bit 0 (LEVEL_TRANSITION)** tırmanmayı ≤ 0 ile sınırlamayı tiltrotorlara uygulamıyor (`ArduPlane/quadplane.cpp:1597-1599`). Yalnızca roll sınırı uygulanıyor (`:4485-4494`).

---

## 4. Dikey motorların ileri uçuşta yardıma girmesi (assist)

### 4.1 Tetikleyiciler (`ArduPlane/VTOL_Assist.cpp:59-144`)

| Tetik | Koşul | Gecikme | Mesaj | Referans |
|---|---|---|---|---|
| Genel kapı | Silahlı, aux ile kapatılmamış. Şunlardan biri doğru: (`does_auto_throttle() && !throttle_suppressed`), gaz > 0 ya da `is_flying()`. Flare'de değil. | — | — | `:61-80` |
| `Q_ASSIST_SPEED ≤ 0` | **Hız, irtifa ve açı kontrollerinin hepsi kapalı.** Yalnızca zorla assist kalıyor. Varsayılan 0 pre-arm hatası veriyor; −1 bilinçli olarak kapatmak demek. | — | PreArm "Q_ASSIST_SPEED is not set" | `:84-90`; `ArduPlane/AP_Arming_Plane.cpp:209-213`; param açıklaması `quadplane.cpp:99` |
| Hız | `aspeed < Q_ASSIST_SPEED`. `Q_OPTIONS` bit 12 açıksa sentetik hız sayılmıyor. | **Yok** (histerezis de yok) | Ayrı mesaj yok. Yalnızca INFO "Transition started airspeed %.1f" | `:94-95`; `quadplane.cpp:1500` |
| İrtifa | `relative_ground_altitude(ASSIST) < Q_ASSIST_ALT`. Parametre `AP_Int16`, **tam metre**. | `Q_ASSIST_DELAY` (0,5 s) tetik, 1,0 s bırakma | WARNING "Alt assist %.1fm" | `:104-113`; `VTOL_Assist.h:23`; `quadplane.cpp:409`, `:421` |
| Açı | Roll/pitch, limitlerin +5° dışında **ve** hedeften ≥ `Q_ASSIST_ANGLE` (30°) sapmış | 0,5 s / 1,0 s | WARNING "Angle assist r=%d p=%d" | `:115-138`; `quadplane.cpp:232` |
| Zorla | `Q_OPTIONS` bit 7 ya da `RCx_OPTION=82` HIGH | — | INFO "QAssist: Force enabled" (aux) | `quadplane.cpp:813-816`; `RC_Channel_Plane.cpp:60-77` |
| Mod uygunluğu | Mod bazlı bir bayrak **yok**. MANUAL/ACRO/TRAINING dışındaki her sabit kanat modunda çalışıyor. | — | — | `ArduPlane/quadplane.cpp:1768-1783` |

### 4.2 `Q_ASSIST_ALT` yüksekliği nereden geliyor?

`Plane::relative_ground_altitude()`, öncelik sırasıyla (`ArduPlane/altitude.cpp:111-157`):

| Sıra | Kaynak | Koşul | Referans |
|---|---|---|---|
| 1 | Harici HAGL (`MAV_CMD_SET_HAGL`) | Değer zaman aşımına uğramamış olmalı. Yalnızca flash'ı > 1 MB olan kartlarda derleniyor. | `altitude.cpp:113-119`, `:897-923`; `GCS_MAVLink_Plane.cpp:858`; `libraries/GCS_MAVLink/GCS_config.h:138-139` |
| 2 | Mesafe sensörü | `RNGFND_LANDING` bit 0 (All) ya da bit 2 (Assist) açık ve `in_range` | `altitude.cpp:121-125`, `:162-173`; varsayılan 0: `Parameters.cpp:783` |
| 3 | 0 | VTOL final inişte sensör `OutOfRangeLow` (yalnızca QRTL/AUTO) | `altitude.cpp:127-134` |
| 4 | Terrain | Terrain-following açık | `altitude.cpp:136-143` |
| 5 | İnişte hedef wp'ye göre yükseklik | QRTL/AUTO VTOL inişi | `altitude.cpp:145-153` |
| 6 | Home'a göre EKF (baro) yüksekliği | Diğerlerinin hiçbiri yoksa | `altitude.cpp:156` |

Mesafe sensörünün "in range" kapısı (`ArduPlane/altitude.cpp:750-797`):
- 10 iyi örnek gerekiyor. Her biri **ilk okumadan** maksimum menzilin %5'inden fazla farklı olmalı.
- Maksimum menzilin %20'sinden büyük bir sıçrama sayacı sıfırlıyor.
- Tek bir kötü örnek `in_range` bayrağını hemen düşürüyor.
- Çıkarım: 40 m menzilli bir sensörde ilk okumadan 2 m fark gerekiyor. Su yüzeyinden dönüş kesildiğinde yükseklik kaynağı sıra 6'ya, yani baroya düşüyor.

### 4.3 Assist tiltrotorda ne yapıyor?

- Durum `AIRSPEED_WAIT`'e zorlanıyor, VTOL motorlar `THROTTLE_UNLIMITED` oluyor ve `hold_hover()` çalışıyor (`ArduPlane/quadplane.cpp:1493-1506`, `:1543`, `:1600`).
- **Rotorlar geri kalkıyor:** tilt en fazla `Q_TILT_MAX` (45°) olacak şekilde sınırlanıyor. Kalkış hızı `Q_TILT_RATE_UP` (40°/s) (`ArduPlane/tiltrotor.cpp:326-331`, `:159-182`).
- Auto-throttle modlarda pitch ±3° ile sınırlanıyor. `ahrs.set_fly_forward(false)` çağrılıyor (`quadplane.cpp:4668-4703`; `Plane.cpp:549-552`).
- Bırakma iki adımlı:
  - Önce assist koşulu kalkmalı **ve** hava hızı `AIRSPEED_MIN`'in üstüne çıkmalı. `Q_ASSIST_SPEED` değil, `AIRSPEED_MIN`.
  - Sonra `TIMER` ve `DONE` geliyor (`quadplane.cpp:1584`).
- Assist bittiğinde mesaj yok.

### 4.4 Yüzey etkisinde güvenlik ağı olarak kullanılabilir mi?

| Senaryo | Değerlendirme | Gerekçe |
|---|---|---|
| Hız kaybı | **Kısmen**, son çare olarak | Gecikmesiz tetikleniyor. Ama araya şunlar giriyor: motor devir alma (`AP_MOTORS_SPOOL_UP_TIME_DEFAULT` 0,5 s, `libraries/AP_Motors/AP_MotorsMulticopter.h:25`), 2 s'lik tırmanma rampası (`quadplane.cpp:1450-1454`) ve tilt geri kalkarken düşen ileri itki (çıkarım). Uzun sürerse `Q_TRANS_FAIL` QLAND'a götürür. |
| Ani yükseklik kaybı | **Hayır** | `Q_ASSIST_ALT` tam metre. 1–3 m'de seyirde iki seçenek var, ikisi de kötü: (a) değer ≥ seyir yüksekliği olursa assist sürekli açık kalır, geçiş biter bitmez geri döner; (b) değer < seyir yüksekliği olursa 0,5 s gecikme + devir alma + tilt süresi gerekir. Tırmanma hedefi de TECS irtifa hatası, "sudan uzaklaş" değil (`quadplane.cpp:1437-1440`). |
| Dalgada mesafe sensörü kesintisi | **Riskli** | Yükseklik kaynağı baroya düşüyor (§4.2). Bu, beklenmeyen tetiklemeye ya da tetiklememeye yol açabilir (çıkarım). |
| Rotor-su teması | **Doğrulanmadı** | Assist rotorları en fazla `Q_TILT_MAX`'a kadar dikleştiriyor. 1–3 m'de pervane-su açıklığı geometriye bağlı; kodda bununla ilgili bir kontrol yok. |

---

## 5. VTOL iniş

### 5.1 Mesafe sensörü nasıl kullanılıyor?

- Sensör yalnızca `RNGFND_LANDING` bit 0 ya da bit 1 (TakeoffAndLanding) açıksa kullanılıyor. Varsayılan 0, yani **kapalı** (`ArduPlane/Parameters.cpp:783`; `ArduPlane/defines.h:198-204`).
- Yön `RNGFND_LND_ORNT`, varsayılan PITCH_270 (`Parameters.cpp:1246`).
- Okuma 50 Hz'de alınıyor ve araç açısına göre düzeltiliyor (`ArduPlane/altitude.cpp:724-748`).
- `in_range` olduktan sonra ham değer filtresiz kullanılıyor (`:755`).

| Kullanım | Davranış | Referans |
|---|---|---|
| Final'e geçiş | `h < Q_LAND_FINAL_ALT` (6 m) ve önceki okumayla fark < 5 m → `LAND_FINAL` | `ArduPlane/quadplane.cpp:3604-3623` |
| İniş hızı | 6 m ile 12 m arasında `Q_LAND_FINAL_SPD` (0,5) ile `Q_WP_SPD_DN` (1,5) arasında doğrusal geçiş. Final'de 0,5 m/s'ye kilitleniyor. | `ArduPlane/quadplane.cpp:1269-1290`; `quadplane.cpp:123`, `:132`; `AC_WPNav.cpp:13`, `:83` |
| QLAND | Final kontrolü ve iniş hızı hızlı döngüde | `ArduPlane/mode_qloiter.cpp:150-170` |
| İleri gaz | Final'de `OutOfRangeLow` ise sıfırlanıyor | `ArduPlane/quadplane.cpp:3882-3890` |
| Mesaj | INFO "Rangefinder engaged at %.2fm" (QLAND/QRTL/AUTO VTOL inişinde bir kez) | `ArduPlane/altitude.cpp:775-791` |
| Sabit kanat düzeltmesi | `rangefinder_correction()` ve `get_landing_height()` VTOL inişte kullanılmıyor; yalnızca sabit kanat LAND aşamasında | `ArduPlane/altitude.cpp:690` |
| Sensör yokken | QLAND → home'a göre baro. QRTL/AUTO → hedef wp yüksekliğine göre EKF. | `ArduPlane/altitude.cpp:145-156` |

### 5.2 Yere ya da suya değme nasıl algılanıyor?

| Adım | Koşul | Referans |
|---|---|---|
| `should_relax()` | > 1 s boyunca gaz alt sınırda (`limit.throttle_lower && is_throttle_mix_min()`) ya da gaz < %1 | `ArduPlane/quadplane.cpp:1213-1231` |
| `land_detector(t)` | EKF yüksekliği pencere başından `Q_LAND_ALTCHG` (0,2 m)'den fazla değişirse pencere sıfırlanıyor. "İndi" demek için: pencere ≥ t **ve** gaz alt sınırda ≥ t + 1 s. | `ArduPlane/quadplane.cpp:3536-3565`; `Q_LAND_ALTCHG` `:467` |
| `check_land_complete()` | Yalnızca `LAND_FINAL`'da, `land_detector(4000)` ile. Sonra INFO "Land complete" ve disarm. | `ArduPlane/quadplane.cpp:3571-3597` |
| `LAND_DESCEND` | `land_detector(6000)` aracı yalnızca `LAND_FINAL`'a geçiriyor | `ArduPlane/quadplane.cpp:3618-3622` |
| Zaman aşımı | **Yok.** Bu yoldan başka bir VTOL "indi → disarm" yolu bulamadım (çıkarım). `LAND_DISARMDELAY` VTOL'da uygulanmıyor. | `ArduPlane/quadplane.cpp:3593` |

SITL koşu 1'de: "SIM Hit ground" 136,28 s → "Land complete" 142,53 s (6,25 s).

### 5.3 Dalgalı yüzeyde algılamayı yanıltabilecek şeyler

| # | Mekanizma (kodda var) | Olası etki | Referans | Durum |
|---|---|---|---|---|
| 1 | Pencere içinde EKF yüksekliği ±0,2 m'den fazla değişince sıfırlanıyor | Dalgayla inip çıkan, yüzen gövdede "Land complete" ve disarm hiç gelmeyebilir. Motorlar alt sınırda dönmeye devam eder. | `quadplane.cpp:3553-3557` | Çıkarım |
| 2 | Gaz alt sınırdan çıkınca `should_relax` sıfırlanıyor | Dalga gövdeyi itince Z kontrolcüsü gaz verirse sayaç baştan başlar | `quadplane.cpp:1221-1225` | Çıkarım |
| 3 | QLAND'da ivme > 3 m/s² ya da açı hatası > 30° → throttle mix max | Gövde çarpması `is_throttle_mix_min()` koşulunu bozar | `quadplane.cpp:4148-4150`, `:4180-4194` | Mekanizma kodda var, etkisi çıkarım |
| 4 | `in_range` sonrası filtresiz ham okuma | Köpük ya da dalga tepesi okuması erken `LAND_FINAL`'a yol açar. Sonuç yavaş iniş, erken disarm değil. | `altitude.cpp:755`; `quadplane.cpp:3611-3614` | Çıkarım |
| 5 | Kötü örnek gelince `in_range` hemen düşüyor | Yükseklik baroya düşüyor. Yeniden `in_range` için ilk okumadan farklı 10 örnek gerekiyor. | `altitude.cpp:759-772`, `:794-797` | Mekanizma kodda var |
| 6 | Sensör yokken final eşiği baroya göre | Baro yüksek okursa temas 1,5 m/s'ye kadar hızla olabilir | `quadplane.cpp:3611`, `:1283-1285` | Çıkarım |
| 7 | Final'de `set_touchdown_expected(true)` | EKF baro füzyonu yer etkisi moduna giriyor | `quadplane.cpp:2893-2897`; `mode_qloiter.cpp:165-167` | Sudaki etkisi doğrulanmadı |
| 8 | Kaldırma kuvveti (buoyancy) modeli | Kodda yok | — | — |

**İniş iptali:**
- `abort_landing()` yalnızca AUTO'da ve `LAND_DESCEND`/`LAND_FINAL` sırasında çalışıyor (`ArduPlane/quadplane.cpp:4839-4857`).
- Tetikleyiciler: `MAV_CMD_DO_GO_AROUND`, RC aux ya da Lua.
- İptalin başlangıcında ya da bitişinde STATUSTEXT gönderilmiyor.

---

## 6. Yeni yüzey etkisi modu

### 6.1 Doğal giriş noktası

**Doğal giriş noktası bir `Mode` alt sınıfı.** Aynı yaklaşım `squilter/ground_effect_mode` dalında da kullanılmış (bkz. `ANALIZ_ground_effect_mode.md`).

| Öğe | Not | Referans |
|---|---|---|
| Mod numarası | `Mode::Number` enum'u. En büyük değer `AUTOLAND = 26`, 30 rezerve. | `ArduPlane/mode.h:38-75` |
| Kancalar | `_enter`, `_exit`, `update` (saf sanal), `run` (varsayılan: stick mixing + stabilize, gaz yok), `navigate`, `does_auto_throttle`, `update_target_altitude`, `is_vtol_mode` (**false** kalmalı) | `ArduPlane/mode.h:87-192`; `mode.cpp:259-276` |
| Mod değişimi | `Plane::set_mode` → `Mode::enter()` → `quadplane.mode_enter()` → `_enter()` → `assisted_flight = should_assist(...)`. Giriş başarısız olursa WARNING "Flight mode change failed". | `ArduPlane/system.cpp:252-352`, `:322`; `mode.cpp:100`, `:156-161` |
| Geçiş durumunu sorgulama | `quadplane.transition->complete()` (`DONE` mu?) ve `quadplane.in_assisted_flight()` | `ArduPlane/transition.h:87`; `quadplane.h:108` |
| Mesafe sensörü verisi | `rangefinder_state.height_estimate`: 50 Hz, açı düzeltmeli. Bugün hiçbir sabit kanat seyir modu bunu yükseklik tutmak için kullanmıyor. | `ArduPlane/altitude.cpp:724-835`; `Plane.h:221` |
| Eklenmesi zorunlu `switch`'ler | `-Werror=switch` açık ve bu switch'lerde `default` yok. Mod eklenmeden derleme kırılır. | `Tools/ardupilotwaf/boards.py:417`; `ArduPlane/control_modes.cpp:9-102`; `events.cpp:26`, `:122`; `GCS_MAVLink_Plane.cpp:29`; `GCS_Plane.cpp:25` |
| Diğer | `Plane.h` içinde örnek nesne, `AVAILABLE_MODES` listesi | `ArduPlane/GCS_MAVLink_Plane.cpp:1311` |

İleri uçuştaki aracı devralmak:
- QuadPlane açısından yeni moda geçmek, bir sabit kanat modundan başka bir sabit kanat moduna geçmek demek. Geçiş `DONE` ise öyle kalıyor (çıkarım: `VTOL_update` yalnızca VTOL modunda çağrılıyor, `quadplane.cpp:1785-1793`).
- Q modundan doğrudan yeni moda geçilirse ileri geçiş, yeni modun içinde başlıyor.
- Bu yüzden `_enter()` içinde `transition->complete()` kontrol edip, geçiş bitmemişse modu reddetmek mantıklı.

### 6.2 QuadPlane koduyla çakışmalar (1–3 m'de seyir için)

| # | Çakışma | Neden önemli | Referans |
|---|---|---|---|
| 1 | Assist ve geçiş durum makinesi her sabit kanat modunda çalışıyor | VTOL motorlar ve tilt her an devreye girebilir, pitch ±3° ile sınırlanır | `ArduPlane/quadplane.cpp:1768-1783` |
| 2 | `Q_TRANS_FAIL` → QLAND/QRTL | Uzun assist, aracı moddan çıkarıp suya dikey indirir ya da 10 m'ye tırmandırır | `quadplane.cpp:1550-1575`; `mode_qrtl.cpp:26-49` |
| 3 | `Q_ASSIST_ALT` | Seyir yüksekliğine eşit ya da büyükse assist sürekli açık kalır | `VTOL_Assist.cpp:104-113` |
| 4 | Pitch ve TECS sınırları | Assist sırasında, `does_auto_throttle()` true ise pitch ±3°/±8° ile sınırlanır, TECS'e sentetik hava hızı verilir | `quadplane.cpp:4668-4703`, `:1534-1539` |
| 5 | TECS sahipliği | `does_auto_throttle()` true ise TECS 10 Hz'de pitch ve gaz yazar (`Plane.cpp:638-672`). Kendi gaz/pitch denetleyicisini yazan mod ya bunu false döndürmeli ya da TECS'i bilinçli yönetmeli (çıkarım). | `ArduPlane/Plane.cpp:224`, `:638-672` |
| 6 | Gaz bastırma | ≤ 5 m'de, stick 0 ve \|vz\| < 1 iken VTOL motorlar GROUND_IDLE'a alınıyor. Silahlı tiltrotorda etkisi en fazla bir döngü (ajan okuması, **doğrulanmadı**). | `quadplane.cpp:1856-1917` |
| 7 | RC failsafe | Kısa/uzun failsafe switch'lerinde yeni mod için bir eylem seçilmeli. Batarya failsafe'i QLAND'a geçirebilir. | `ArduPlane/events.cpp:26-103`, `:122-239`, `:272-285` |
| 8 | Tilt çıkışı | `tiltrotor.update()` her döngüde çalışıyor. Sabit kanatta tilt motorlarını ileri motor gibi sürüyor. | `quadplane.cpp:1803`; `tiltrotor.cpp:222-251` |
| 9 | `fly_forward` | Assist sırasında `false` oluyor; AHRS davranışı değişiyor | `ArduPlane/Plane.cpp:549-552` |
| 10 | Landing gear | `LGR_DEPLOY_ALT` > seyir yüksekliği ise gear seyirde açılır | `ArduPlane/takeoff.cpp:392-395` |
| 11 | İtki kaybı dedektörü | Assist sırasında da çalışıyor (`Q_THRST_LOSS_OPT` ile kapatılabilir) | `quadplane.cpp:573`, `:4914-4984` |

**Önerilen en küçük çözüm (uygulanmadı, yalnızca öneri):**
- Yeni modu `quadplane.cpp:1771-1773`'teki MANUAL/ACRO/TRAINING listesine eklemek, assist'i ve geçiş makinesini bu modda tamamen kapatır. Ama aynı zamanda güvenlik ağını da kaldırır.
- Assist kullanılmak isteniyorsa kendi koşullarıyla, bilinçli olarak tetiklenmeli.

---

## 7. GCS mesajları (STATUSTEXT)

| Olay | Metin | Önem | Referans | Not |
|---|---|---|---|---|
| Geçiş başladı (Q modundan) | "Transition airspeed wait" | INFO | `quadplane.cpp:1546` | Bir kez |
| Geçiş/assist başladı | "Transition started airspeed %.1f" | INFO | `quadplane.cpp:1500` | **Hız assist'inin tek işareti.** AUTO VTOL kalkışından sonra da bu mesaj çıkıyor (SITL). |
| Hava hızı eşiği aşıldı | "Transition airspeed reached %.1f" | INFO | `quadplane.cpp:1587` | |
| Geçiş bitti | "Transition FW done" / "Transition done" | INFO | `quadplane.cpp:1527` / `:1648` | Tiltrotorda genelde ilki |
| Geçiş başarısız | "Transition failed, exceeded time limit" | **CRITICAL** | `quadplane.cpp:1555` | Bir kez; yalnızca `Q_TRANS_FAIL > 0` iken |
| İrtifa assist'i | "Alt assist %.1fm" | WARNING | `VTOL_Assist.cpp:111` | Her aktivasyonda bir kez |
| Açı assist'i | "Angle assist r=%d p=%d" | WARNING | `VTOL_Assist.cpp:137` | Her aktivasyonda bir kez |
| Assist bitti | — | — | — | **Mesaj yok** |
| Assist aux anahtarı | "QAssist: Force enabled / Enabled / Disabled" | INFO | `RC_Channel_Plane.cpp:64`, `:69`, `:74` | |
| Olası motor/itki kaybı | "Potential VTOL Thrust Loss (%u)" | **EMERGENCY** | `quadplane.cpp:4979` | Koşullar: 1 s boyunca gaz ≥ %90, alçalma, küçük açı. Thrust boost'u açıyor. **Tek motor arızası mesajı bu.** |
| ESC/motor telemetri arızası | — | — | — | `libraries/AP_ESC_Telem/` içinde `send_text` yok; AP_Motors'ta multicopter için motor arızası mesajı yok |
| Kalkış başarısız | "Failed to complete takeoff within time limit" / "... excessive wind" | CRITICAL | `quadplane.cpp:3493` / `:3500` | Ardından QLAND |
| Kalkış tamamlandı | — | — | — | **Mesaj yok** |
| İniş aşamaları | "Land descend started", "Land final started", "Land complete" | INFO | `quadplane.cpp:3661`, `:3684`, `:3579` | QLAND'da "Land final started" gönderilmiyor (`mode_qloiter.cpp:151-160`) |
| Geri geçiş | "VTOL approach", "VTOL airbrake", "VTOL position1", "VTOL position2 started" | INFO | `quadplane.cpp:2263`, `:2478`, `:2470`, `:2732` | |
| Mod değişimi | — | — | `system.cpp:345` | STATUSTEXT değil, HEARTBEAT. Başarısızsa WARNING "Flight mode change failed" (`:322`) |

**STATUSTEXT dışındaki VTOL durumu:**
- `EXTENDED_SYS_STATE.vtol_state` şu değerleri alıyor (`ArduPlane/quadplane.cpp:4641-4665`; `GCS_MAVLink_Plane.cpp:1276-1286`):
  - `AIRSPEED_WAIT` ya da `TIMER` → `TRANSITION_TO_FW`. **Assist de bu değeri veriyor.**
  - `DONE` → `FW`.
  - Q modu → `MC`.
  - Yaklaşma/airbrake → `TRANSITION_TO_MC`.
- Bu mesaj varsayılan stream'lerde yok: `MSG_EXTENDED_SYS_STATE` ne `libraries/GCS_MAVLink/GCS_MAVLink_Parameters.cpp` stream listelerinde ne de `ArduPlane/` içinde geçiyor. Yalnızca gönderim kodu var (`libraries/GCS_MAVLink/GCS_Common.cpp:1169`, `:6746`). `SET_MESSAGE_INTERVAL` ile istenmesi gerekiyor.
- Log'da: `QTUN.Trn` geçiş durumunu, `QTUN.Ast` assist bit maskesini tutuyor (bit0 assist aktif, bit1 zorla, bit2 hız, bit3 irtifa, bit4 açı; `quadplane.cpp:3722-3754`).

---

## 8. SITL denemesi

**Kurulum:**
- `./waf configure --board sitl && ./waf plane`. Ek olarak `modules/littlefs` ve `modules/lwip` submodule'leri gerekti.
- Komut: `build/sitl/bin/arduplane -w --model quadplane-tilttri --speedup 5 -I0 --defaults Tools/autotest/default_params/quadplane.parm,Tools/autotest/default_params/quadplane-tilttri.parm`
- SITL şu uyarıyı verdi: "Warning model expected 4 motors and got 3".
- MAVLink'e pymavlink betiğiyle bağlandım, MAVProxy kullanmadım.

**Görev:**
- `NAV_VTOL_TAKEOFF` 10 m
- `WAYPOINT` 400 m kuzey, 15 m
- `WAYPOINT` 400 m K / 300 m D, 15 m
- `NAV_VTOL_LAND` home

**Parametre dosyasından gelen ilgili değerler:** `Q_ASSIST_SPEED=18`, `AIRSPEED_MIN=13`, `Q_TILT_MAX=45`, `Q_TILT_TYPE=0`, `Q_TRANS_FAIL=0`, `Q_TRANSITION_MS=5000`, `Q_OPTIONS=0`.

Zamanlar SITL saatine göre (boot'tan itibaren, s).

### Koşu 1: hazır parametreler (hiçbir değişiklik yok)

| t (s) | Olay | Önem | Metin / durum |
|---|---|---|---|
| 2,04 | MODE | | MANUAL → FBWA (SITL'de RC5=1800 → FLTMODE6; çıkarım) |
| 24,78 | MODE | | FBWA → AUTO |
| 24,78 | STATUSTEXT | INFO | Mission: 1 VTOLTakeoff |
| 24,78 | VTOL_STATE | | MC |
| 25,03 | ARMING | | ARMED |
| 30,28 | STATUSTEXT | INFO | Mission: 2 WP (kalkış tamam, 10,1 m) |
| 30,28 | STATUSTEXT | INFO | **Transition started airspeed 1.1** |
| 30,28 | VTOL_STATE | | TRANSITION_TO_FW |
| 55,03 | STATUSTEXT | INFO | Reached waypoint #2 dist 49m |
| 73,78 | STATUSTEXT | INFO | Mission: 4 VTOLLand; VTOL approach d=472.1 |
| 102,53 | STATUSTEXT | INFO | VTOL position1 v=15.4 d=90 sd=90 h=15.0 |
| 102,53 | VTOL_STATE | | TRANSITION_TO_MC |
| 109,53 | STATUSTEXT | INFO | VTOL position2 started v=5.7 d=9.7 h=14.9 |
| 115,28 | STATUSTEXT | INFO | Land descend started |
| 124,03 | STATUSTEXT | INFO | Land final started |
| 136,28 | STATUSTEXT | INFO | SIM Hit ground at 0.508050 m/s |
| 142,53 | STATUSTEXT | INFO | Land complete; Throttle disarmed |

**Bulgu:** ileri geçiş **hiç tamamlanmadı.**
- 30,3 s ile 102,6 s arasında dataflash log'unda `QTUN.Trn=0` (`AIRSPEED_WAIT`) ve `QTUN.Ast=5` vardı. Ast=5, "assist aktif + hız assist'i" demek. Bunu log'dan kendim doğruladım.
- Hava hızı tam gazla 14–15,5 m/s civarında kaldı: `AIRSPEED_MIN` (13) üstünde, `Q_ASSIST_SPEED` (18) altında.
- Tilt servosu bütün ayak boyunca 1500 µs'de kaldı, yani 45° = `Q_TILT_MAX` (çıkarım: 1000 dikey, 2000 tam ileri).
- VTOL motorlar 1340–1420 µs'de çalışmaya devam etti.
- "Transition airspeed reached", "Transition done" ya da "Transition failed" mesajlarından **hiçbiri gelmedi**. Görev buna rağmen sürdü.

### Koşu 2: yalnızca `Q_ASSIST_SPEED=12` (doğrulama)

| t (s) | Olay |
|---|---|
| 30,28 | Kalkış tamam (10,1 m); "Transition started airspeed 1.1" |
| 34,28 | "Transition airspeed reached 13.2" (log: `Trn` 0→1 34,52 s'de) |
| 35,53 | "Transition FW done"; VTOL_STATE FW (log: `Trn=2`, `Ast=0` 35,64 s'de) |
| 35,5–73,3 | İleri uçuş: 18–27,6 m/s, 9,7–17,4 m, tilt servosu 2000 µs |
| 73,78 | "VTOL airbrake v=19.1 d=128 sd=129 h=14.0"; TRANSITION_TO_MC |
| 79,03 / 81,78 | "VTOL position1" / "VTOL position2 started" |
| 87,28 / 96,78 | "Land descend started" / "Land final started" |
| 109,03 | SIM Hit ground 0,499 m/s |
| 115,28 | "Land complete"; disarm |

İki koşuda da motor kaybı, "Potential VTOL Thrust Loss", "Transition failed" ya da zaman aşımı mesajı görülmedi.

SITL düz bir yer zemininde uçuyor. Su yüzeyi, dalga ve kaldırma kuvveti modellenmiyor. §5.3'teki dalga etkileri bu yüzden SITL'de **test edilmedi**.

Ham log'lar (`messages.log`, `events.csv`, `flight.csv`, `.BIN`) oturumun geçici dizininde kaldı; repoya eklenmedi.

---

## Hazır olanlar

| İhtiyaç | ArduPlane 4.7.1'de durum |
|---|---|
| Dikey kalkış, birkaç metre dahil | Hazır (`NAV_VTOL_TAKEOFF`, GUIDED takeoff). Alt sınır yok. |
| Tiltrotor ileri geçiş | Hazır (`SLT_Transition` + `Tiltrotor`). Hava hızı + tilt koşullu. |
| Alçak irtifada geçiş | Kod engellemiyor; minimum yükseklik kontrolü yok |
| Geçiş başarısızlık eylemi | Hazır: `Q_TRANS_FAIL`, `Q_TRANS_FAIL_ACT` (−1/QLAND/QRTL), `Q_OPTIONS` bit 19 |
| Geri geçiş ve hassas VTOL iniş | Hazır (QRTL, `NAV_VTOL_LAND`, QLAND) |
| İnişte mesafe sensörü | Hazır, ama sınırlı: final'e geçiş ve iniş hızı için kullanılıyor (`RNGFND_LANDING`) |
| Dikey motorların yardımı | Hazır: hız, irtifa, açı ve zorla assist (`Q_ASSIST_*`, RC aux 82) |
| Harici yükseklik girişi | `MAV_CMD_SET_HAGL`, `relative_ground_altitude()`'ta mesafe sensöründen önce geliyor |
| GCS bildirimleri | Geçiş başla/bitti/başarısız, irtifa ve açı assist'i, itki kaybı. `EXTENDED_SYS_STATE.vtol_state`. Log'da `QTUN.Trn` ve `Ast`. |
| Mod altyapısı | `Mode` alt sınıfı, `transition->complete()`, `in_assisted_flight()`, `rangefinder_state` |

## Bizim yazmamız gerekenler

1. **Yüzey etkisi modu.** Mesafe sensörüyle suya göre yükseklik tutan bir `Mode` alt sınıfı.
   - Dalga filtresi gerekiyor; mevcut `rangefinder_state`, `in_range` olduktan sonra ham değer veriyor.
   - Sensör kesildiğinde ne yapılacağı belirlenmeli.
   - Pitch, gaz ve flap kontrolü yazılmalı. TECS'le ilişki de net olmalı: `does_auto_throttle()` ve `update_target_altitude()` kararları.
   - Mod numarası; zorunlu `switch` girişleri ve failsafe eylemleri eklenmeli (§6.1).
2. **Moda giriş ve çıkış mantığı.** Yalnızca `transition->complete()` iken girilebilmeli. Seyir yüksekliğinden suya kontrollü alçalma ve yüzey etkisinden çıkış (tırmanma) yazılmalı.
3. **QuadPlane ile çakışma politikası.** Yeni modda assist ve geçiş makinesi ne yapacak?
   - Seçenek a: Modu MANUAL/ACRO/TRAINING listesine eklemek. Assist tamamen kapanır.
   - Seçenek b: Assist'i açık bırakıp kendi tetik koşullarımızı yazmak.
   - `Q_ASSIST_ALT`, `Q_TRANS_FAIL` ve pitch sınırlarının bu modda devre dışı kalması garanti edilmeli.
4. **Yüzey etkisine özel güvenlik ağı.** Mevcut assist, hız ve yükseklik kaybına 1–3 m'de yeterince hızlı ve doğru yönde tepki vermiyor (§4.4). İhtiyaçlar:
   - Rangefinder tabanlı ve dalgaya dayanıklı bir "suya çok yakın" algısı.
   - Hız kaybında tilt'i geri kaldırmadan itki ekleyen bir tepki.
   - Pervane-su açıklığı kontrolü.
5. **Suya iniş/temas algılama.** Mevcut algılayıcı (±0,2 m / 4 s, zaman aşımı yok) dalgada disarm etmeyebilir. Gerekenler:
   - Suya özel bir "indi" kriteri.
   - Zaman aşımı.
   - Yüzerken motor davranışı.
   - Lua'da `set_land_descent_rate()` ve `abort_landing()` var, ama bir script'in iniş tamamlandı durumunu doğrudan işaretleyebildiği **doğrulanmadı**.
6. **Kalkışta suya göre yükseklik.** Kalkış yalnızca baroya bakıyor. Dalgalı yüzeyde mesafe sensörüyle doğrulama ya da kalkış zaman aşımı (`Q_TKOFF_FAIL_SCL`) gerekiyor.
7. **Parametre seti.**
   - `Q_ASSIST_SPEED`, `Q_TILT_MAX` ile ulaşılabilen hızın altında olmalı. Aksi halde geçiş hiç bitmiyor (SITL koşu 1).
   - `Q_TRANS_FAIL` ve `Q_TRANS_FAIL_ACT` su için bilinçli seçilmeli.
   - `RNGFND_LANDING` açılmalı.
   - `Q_LAND_FINAL_ALT`, `Q_LAND_ALTCHG` ve `Q_ASSIST_ALT` ayarlanmalı.
8. **GCS tarafı.**
   - Hız assist'i başlangıcı ve assist bitişi için mesaj yok. Gerekirse eklenmeli ya da `EXTENDED_SYS_STATE` ve `QTUN.Ast` izlenmeli.
   - ESC/motor arızası için "Potential VTOL Thrust Loss" dışında bildirim yok.
   - Yeni mod için STATUSTEXT'ler (giriş, sensör kaybı, çıkış) yazılmalı.
9. **Test.**
   - SITL'de su yüzeyi, dalga ve yüzey etkisi yok. Bunlar için ya özel bir SITL modeli (yüzey etkisi + dalga + buoyancy) ya da kontrollü saha testi gerekiyor.
   - 5 m'de geçiş denemesi yapılmadı.
