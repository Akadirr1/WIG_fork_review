# SITL Denemeleri 2: EKF'de Mesafe Sensörü Yüksekliği, Geçiş Hatası ve RC Kaybı

**Kaynak:** ArduPilot `ArduPlane-stable`, commit `dbe792162d` (ArduPlane V4.7.1). Önceki çalışma: `ANALIZ_VTOL_TILTROTOR.md`.
**Model:** SITL `quadplane-tilttri`, `--speedup 5`, varsayılan parametreler `quadplane.parm` + `quadplane-tilttri.parm`.
**Yöntem:**
- Derleme ve pymavlink betiği önceki denemeyle aynı.
- Betiğe yalnızca bir tetikleyici eklendi: belirli bir uçuş anında (kalkış tırmanışı, seyir, son alçalma) parametre değiştiriyor.
- ArduPilot koduna dokunulmadı, yalnızca parametre değiştirildi.

> **Referanslar ve zaman:** Bütün `dosya:satır` referansları ArduPilot kök dizinine göre ve `dbe792162d` commit'ine aittir. Zamanlar SITL saatiyle (boot'tan itibaren, s) verildi. Yükseklik, EKF'nin home'a göre yüksekliği (`GLOBAL_POSITION_INT.relative_alt`); aksi yazılmadıkça bu değer kullanıldı. **Çıkarım:** koddan ya da log'dan yapılmış ama doğrudan ölçülmemiş sonuç.

---

## Kısa özet

| # | Soru | Sonuç |
|---|---|---|
| 1 | `EK3_RNG_USE_HGT=70` iken EKF dikey kalkış ve inişte yüksekliği mesafe sensöründen alıyor mu? | **Evet.** Kalkışta 0,8 m'den itibaren sensörü kullanıyor. Geçişte yatay hız ~2 m/s'yi geçince (12,2 m) baroya dönüyor. İnişte 9,4 m'de, hız < 1 m/s iken yeniden sensöre geçiyor. Geçişlerde EKF yüksekliğinde **sıçrama yok** (en fazla 0,2 m). |
| 1b | Kalkışta baroya +5 m sapma | EKF sapmayı yok saydı. Sapma `BOf`'ta öğrenildiği için baroya dönüşte sıçrama olmadı; seyirde kalan hata 0,26 m. |
| 1c | Seyirde baroya +3 m sapma (ek deneme) | EKF seyirde 3 m hatalı uçtu. Hata, EKF'nin arazi tahminine yazıldığı için inişte sensöre geçmek onu **düzeltmedi**: araç yere değdiğinde EKF 3,0 m gösteriyordu. |
| 2 | `Q_ASSIST_SPEED=18`, `Q_TRANS_FAIL=15` | **Evet.** Geçiş başladıktan tam **15,0 s** sonra CRITICAL "Transition failed, exceeded time limit" geldi ve araç **QLAND**'e geçti. |
| 3 | RC kaybı aşamaya göre | **Kalkışta beklenen QLAND çıkmadı**, görev sürdü. Uzun failsafe 5 s sonra geliyor ve o anda kalkış bitmişti. `FS_LONG_TIMEOUT=2` ile QLAND oldu. Seyirde `FS_LONG_ACTN=0` gereği görev devam etti. İnişte mod değişmedi. |
| 4 | Seyirde `FS_LONG_ACTN=2` (Glide) | FBWA'ya geçti ve gaz kesildi. Hava hızı süzülüşte 12 m/s'nin altına düşmediği için **havada assist girmedi**; araç 14 m/s ile yere değdi. **Assist yere değdikten sonra devreye girdi.** `Q_TRANS_FAIL=0` iken araç yerde silahlı kaldı. `Q_TRANS_FAIL=15` iken 15 s sonra QLAND oldu ve disarm etti. |

---

## Ortak ayarlar

Bütün denemelerde `common.parm` dosyası defaults'a eklendi:

| Parametre | Değer | Not |
|---|---|---|
| `RNGFND1_TYPE` | 100 | SITL mesafe sensörü |
| `RNGFND1_ORIENT` | 25 | PITCH_270, aşağı bakıyor |
| `RNGFND1_MAX` / `RNGFND1_MIN` | 20 / 0,2 m | Parametreler metre cinsinden (`libraries/AP_RangeFinder/AP_RangeFinder_Params.cpp:107`, `:115`) |
| `Q_ASSIST_SPEED` | 12 | Deneme 2 hariç |

- `RNGFND_LANDING` her denemede varsayılan değeri 0'da kaldı. Yani QuadPlane iniş mantığı sensörü doğrudan kullanmadı; sensör yalnızca EKF üzerinden etki etti (deneme 1).
- Görev öncekiyle aynı:
  - `NAV_VTOL_TAKEOFF` 10 m
  - Yol noktası 400 m K, 15 m
  - Yol noktası 400 m K / 300 m D, 15 m
  - `NAV_VTOL_LAND` home
- SITL zemini su değil, CMAC çevresindeki arazi verisi. Bu yüzden mesafe sensörü seyirde arazi engebesini de görüyor.

---

## 1. EKF yüksekliği mesafe sensöründen mi alıyor?

### Hangi log alanında görünüyor?

EKF3'ün aktif yükseklik kaynağı (`activeHgtSource`) **hiçbir log alanına doğrudan yazılmıyor.** `XKF1`–`XKF5`, `XKFS` ve `XKF4.SS` alanlarının hiçbiri bu değişkeni içermiyor (`libraries/AP_NavEKF3/AP_NavEKF3_Logging.cpp`). Kaynağı iki dolaylı ama koda dayalı yoldan çıkardım:

| İz | Neden kaynağı gösteriyor | Referans |
|---|---|---|
| `XKF5.BOf` (baro ofseti) | `calcFiltBaroOffset()` yalnızca kaynak **baro değilken** çağrılıyor. `BOf` değişiyorsa kaynak mesafe sensörü, 0,5 s'den uzun süre sabitse kaynak baro. | `AP_NavEKF3_PosVelFusion.cpp:1296-1300`; `AP_NavEKF3_Measurements.cpp:806-810`; log `AP_NavEKF3_Logging.cpp:213` |
| Geçiş koşulları | Sensöre geçiş: yükseklik < 0,7 × (`RNGFND1_MAX` × `EK3_RNG_USE_HGT`/100) = **9,8 m**, yatay hız < max(`RNG_USE_SPD`−1, `RNG_USE_SPD`/2) = **1 m/s**. Baroya dönüş: yükseklik > **14 m** ya da yatay hız > **2 m/s**. | `AP_NavEKF3_PosVelFusion.cpp:1221-1258`; `EK3_RNG_USE_HGT` varsayılan −1 `AP_NavEKF3.cpp:488`; `EK3_RNG_USE_SPD` varsayılan 2,0 `:532` |

Log'dan bulduğum geçiş anları bu eşiklerle birebir uyuşuyor.

### 1a: `EK3_RNG_USE_HGT=70` (baro sapması yok)

| t (s) | Olay | Yükseklik | Not |
|---|---|---|---|
| 24,83 | AUTO, ARMED, "Mission: 1 VTOLTakeoff" (INFO) | 0 | EKF kaynağı baro (yerde sensör < 0,2 m) |
| 26,89 | **EKF → mesafe sensörü** | 0,8 m | Yatay hız 1,3 m/s |
| 30,08 | "Mission: 2 WP", "Transition started airspeed 1.1" (INFO) | 10,1 m | Kalkış, EKF sensör modundayken bitti |
| 31,48 | **EKF → baro** | 12,2 m | Yatay hız 2,3–2,8 m/s; EKF yüksekliğinde sıçrama ≤ 0,21 m |
| 35,58 | "Transition FW done" (INFO) | 13,2 m | |
| 87,33 | "Land descend started" (INFO) | ~16 m | |
| 92,68 | **EKF → mesafe sensörü** | 9,4 m | Yatay hız 0,18 m/s; sıçrama ≤ 0,04 m |
| 97,08 | "Land final started" (INFO) | 6 m | |
| 109,08 | SIM yere temas (0,50 m/s) | 0 | |
| 109,18 | EKF → baro | 0 | Sensör < `RNGFND1_MIN`, veri taze değil → baroya dönüş (`AP_NavEKF3_PosVelFusion.cpp:1275-1293`) |
| 115,33 | "Land complete", disarm (INFO) | 0 | |

"Sıçrama" ölçüsü: EKF yüksekliğinin (`XKF1.PD`) her adımdaki değişimi ile dikey hız × Δt arasındaki en büyük fark, geçiş anının ±1 s'si içinde.

### 1b ve 1c: baroya sapma

| Deneme | Sapma | Uygulandığı an | EKF kaynağı o an | Sonuç |
|---|---|---|---|---|
| 1b | `SIM_BARO_GLITCH=5`, `SIM_BAR2_GLITCH=5` | 28,08 s, kalkışta 4,6 m | Mesafe sensörü | EKF baroyu izlemedi, sensörü izledi. `BOf` 28,1–31,4 s arasında 0 → 4,74 m'ye çıktı. 31,46 s'de baroya dönüşte sıçrama 0,07 m. Seyirde EKF hatası 0,26 m (5 − 4,74). İnişte sensöre dönüş 92,86 s'de oldu, sıçrama 0,006 m. |
| 1c (ek) | `SIM_BARO_GLITCH=3`, `SIM_BAR2_GLITCH=3` | 40,78 s, seyirde 11,6 m | Baro | EKF yüksekliği ~2 s içinde baroyla birlikte 3 m kaydı; TECS aracı gerçekte 3 m alçakta uçurdu. 91,16 s'de sensöre geçiş **düzeltme yapmadı** (sıçrama 0,005 m). Yere temas 103,53 s'de olurken EKF 2,97 m gösteriyordu (sensör 0,12 m). "Land final started" EKF'de 6,2 m'de geldi; gerçek yükseklik ~3,3 m idi. |

**1c neden düzeltmiyor? (koddan)**
- Sensör modunda ölçülen yükseklik `range − terrainState` (`AP_NavEKF3_PosVelFusion.cpp:1333-1337`).
- `terrainState`'i ayrı bir tek durumlu filtre tahmin ediyor. Bu filtre yalnızca kaynak **baro iken** çalışıyor (`AP_NavEKF3_OptFlowFusion.cpp:54-58`, `:88-91`).
- Seyirde baro hatası bu yüzden "arazi yüksekliği"ne yazıldı: `XKF5.TOfs` 0,1 m'den −2,9 m'ye gitti (`AP_NavEKF3_Logging.cpp:202`).
- Sensöre geçince EKF bu yanlış araziye göre ölçüyor. Mutlak yükseklik baro hatasını taşımaya devam ediyor; yalnızca araziye göre yükseklik (`XKF5.HAGL`) doğru kalıyor.

**Sonuç:**
- Hız 2 m/s'nin altındayken EKF kalkış ve inişte yüksekliği mesafe sensöründen alıyor ve baroya dönüş sıçramasız.
- Ancak seyirde biriken baro hatasını inişte düzeltmiyor (1c).

---

## 2. Geçiş takılınca QLAND

**Parametreler:** `Q_ASSIST_SPEED=18`, `Q_TRANS_FAIL=15`, `Q_TRANS_FAIL_ACT` varsayılan (0 = QLAND).

| t (s) | Mod | Mesaj / olay | Önem | Yükseklik | Hava hızı |
|---|---|---|---|---|---|
| 30,35 | AUTO | "Transition started airspeed 0.3" | INFO | 10,7 m | 0,6 |
| 30,40 | AUTO | `QTUN` Trn=0, Ast=5 (hız assist'i) | — | 10,7 m | 0,6 |
| 45,35 | AUTO → **QLAND** | "Transition failed, exceeded time limit" | **CRITICAL** | 14,3 m | 14,5 |
| 69,85 | QLAND | SIM yere temas (0,50 m/s) | — | 0 | — |
| 75,85 | QLAND | "Land complete", "Throttle disarmed"; DISARMED | INFO | 0 | — |

- Süre tam **15,0 s**, yani `Q_TRANS_FAIL`. Koddaki yol: `ArduPlane/quadplane.cpp:1550-1567`.
- QLAND boyunca "Land descend/final started" mesajı gelmedi; bu, önceki analizdeki tespitle uyumlu.
- **Sonuç:** Geçiş takılınca 15 s sonra CRITICAL mesajla QLAND'e geçiyor; araç 14,5 m/s hızla ileri giderken QLAND'e alındı ve 24,5 s sonra yere indi.

---

## 3. RC kaybı aşamaya göre (`SIM_RC_FAIL=1`)

**Kodda beklenen akış:**
- RC kesilir. `RC_FS_TIMEOUT` (1,0 s, `libraries/RC_Channel/RC_Channels_VarInfo.h:113`) sonra "Throttle failsafe on" gelir ve kısa failsafe başlar (`ArduPlane/radio.cpp:243`).
- AUTO'da kısa failsafe hiçbir şey yapmaz (`FS_SHORT_ACTN=0`; `ArduPlane/events.cpp:82`; varsayılan `Parameters.cpp:443`).
- Son RC'den `FS_LONG_TIMEOUT` (5 s, `Parameters.cpp:459`) sonra uzun failsafe gelir (`ArduPlane/system.cpp:380-383`). AUTO'da uzun failsafe şöyle davranır:
  - İniş dizisindeyse hiçbir şey yapmaz (`events.cpp:185-188`).
  - **O anda** VTOL kalkışındaysa QLAND'e geçer (`events.cpp:191-195`).
  - Diğer durumlarda `FS_LONG_ACTN`'e göre davranır (varsayılan 0 = Continue, `Parameters.cpp:450`; dallar `events.cpp:202-218`).
- Mesaj her durumda WARNING "RC Long Failsafe On: switched to <mod>" (`events.cpp:240`). Mod değişmese de gönderiliyor.

| Deneme | Kesinti anı | Kısa FS (+1 s) | Uzun FS | Mod sonucu | Bitiş |
|---|---|---|---|---|---|
| 3a: VTOL kalkışı | 28,03 s, 4,4 m | 29,04 "Throttle failsafe on", "RC Short Failsafe On" (WARNING), mod değişmedi | 33,03 "RC Long Failsafe On: switched to Auto" | **AUTO devam.** Kalkış 30,28 s'de, uzun FS'den önce bitti; geçiş, iki yol noktası, VTOL iniş tamamlandı. | 115,28 "Land complete", disarm |
| 3a': aynı + `FS_LONG_TIMEOUT=2` | 28,06 s, 4,5 m | 29,06 (aynı mesajlar) | 30,06 "RC Long Failsafe On: switched to QLand" | **AUTO → QLAND**, 9,5 m'de, kalkış hâlâ sürerken | 49,81 yere temas; 55,81 "Land complete", disarm |
| 3b: seyir | 40,78 s, 11,7 m, 27,6 m/s | 41,78 (aynı mesajlar) | 46,03 "...switched to Auto" | **AUTO devam** (`FS_LONG_ACTN=0`). Görev bitti, VTOL iniş yapıldı. | 115,28 "Land complete", disarm |
| 3c: VTOL inişi, son alçalma | 97,03 s, "Land final started" anında, 6,0 m | 98,03 (aynı mesajlar) | 102,03 "...switched to Auto" | **Değişiklik yok**, iniş sürdü | 109,03 yere temas; 115,28 "Land complete", disarm |

Bütün denemelerde disarm'dan sonra CRITICAL "PreArm: Radio failsafe on" geldi.

**Sonuç:**
- **Seyir ve iniş** kodda beklendiği gibi davrandı.
- **Kalkıştaki QLAND yalnızca uzun failsafe anında araç hâlâ VTOL kalkışındaysa** çalışıyor. 10 m'lik, ~5 s süren bir kalkışta RC kalkışın ortasında kesilirse kalkış uzun failsafe gelmeden biter ve araç görevine RC'siz devam eder.

---

## 4. Seyirde Glide (`FS_LONG_ACTN=2`) ve dikey motor yardımı

**Parametreler:**
- 4: `FS_LONG_ACTN=2`
- 4b: `FS_LONG_ACTN=2` ve `Q_TRANS_FAIL=15`
- İkisinde de `Q_ASSIST_SPEED=12`. RC, `FW done`'dan 5 s sonra kesildi.

| t (s) | Mod | Olay | Yükseklik | Hava hızı | `QTUN.Ast` |
|---|---|---|---|---|---|
| 40,78 | AUTO | RC kesildi | 11,7 m | 27,6 | 0 |
| 41,78 | AUTO | "Throttle failsafe on", "RC Short Failsafe On" (WARNING) | 12,3 m | 27,3 | 0 |
| 46,03 | AUTO → **FBWA** | "RC Long Failsafe On: switched to FBWA" (WARNING) | 13,9 m | 26,7 | 0 |
| 47,3–56,3 | FBWA | Gaz %0, süzülüş: düşüş ~2 m/s, hız 24 → 14 m/s | 16,7 → 1,4 m | 14,2 | **0** |
| 56,78 | FBWA | SIM yere temas, dikey 1,96 m/s; aynı anda "Transition started airspeed 11.7" (INFO) | 0 | 14 → 9,8 | 0 |
| 56,89 | FBWA | Assist başladı (hız assist'i) | 0 | 9,8 | **5** |
| 58,3 | FBWA | VTOL motorları 1438 µs, tilt dikeye (servo12 2000 → 1000) | 0 | 1,6 | 5 |
| ~62–161 | FBWA | VTOL motorları 1100 µs, araç silahlı; `vtol_state` TRANSITION_TO_FW | 0 | <1 | 1 |
| 161 | FBWA | Deneme durduruldu (tetikten 120 s sonra). QLAND yok, disarm yok. | 0 | — | 1 |
| **4b:** 56,56 | FBWA | Yere temas + "Transition started airspeed 11.7" | 0 | — | 5 |
| **4b:** 71,56 | FBWA → **QLAND** | "Transition failed, exceeded time limit" (**CRITICAL**), temastan 15,0 s sonra | 0 | 0,8 | 0 |
| **4b:** 78,06 | QLAND | "Land complete", "Throttle disarmed" | 0 | — | 0 |

**Koddan açıklama:**
- RC kaybında sabit kanat modlarında gaz girdisi 0'a çekiliyor (`ArduPlane/radio.cpp:224-226`). Bu yüzden FBWA'da gaz kesildi.
- Assist kapısında FBWA için `is_flying()` yeterli (`ArduPlane/VTOL_Assist.cpp:67-74`). Hız assist'i `aspeed < Q_ASSIST_SPEED` ile gecikmesiz tetikleniyor (`:94-95`).
- Süzülüşte hava hızı 14 m/s'nin altına inmediği için havada tetiklenmedi. Yere değip yavaşlayınca tetiklendi.
- Bir kez `AIRSPEED_WAIT`'e girince `assisted_flight` her döngüde true kalıyor (`quadplane.cpp:1589`). Durumdan çıkmak için hava hızı > `AIRSPEED_MIN` gerekiyor (`:1584`). Yerde bu hiç olmuyor.
- Bu yüzden `Q_TRANS_FAIL=0` iken süresiz kalıyor; `Q_TRANS_FAIL>0` iken QLAND'e geçiyor (`:1550-1567`), QLAND de iniş algılayıcısıyla disarm ediyor.
- 1100 µs, `Q_M_SPIN_ARM` (0,10) seviyesine karşılık geliyor (`libraries/AP_Motors/AP_MotorsMulticopter.h:16`). Motorlar boşta dönüyor, kaldırma üretmiyor (çıkarım).
- Bu düşük seviyeye neden indiğini (gaz bastırma mı, başka bir mekanizma mı) kodda izlemedim: **doğrulanmadı**.

**Sonuç:**
- `Q_ASSIST_SPEED`, süzülüşün suya değme hızından düşükse, Glide aracı suya süzülerek indiriyor.
- Ama temastan sonra assist devreye giriyor: VTOL motorları kısa bir an güç alıp tilt'i dikeye kaldırıyor, sonra araç silahlı ve boşta kalıyor.
- Disarm ancak `Q_TRANS_FAIL>0` ile, QLAND üzerinden geliyor.

**Dikkat (çıkarım, test edilmedi):**
- `Q_ASSIST_SPEED` süzülüşün temas hızından yüksek olsaydı assist havada tetiklenirdi.
- FBWA'da gaz 0 iken assist tırmanma talebi 0 (`quadplane.cpp:1441-1442`). Bu durumda VTOL motorları aracı **o yükseklikte tutar**, yani süzülüş suya oturmaz.

---

## Bizim için çıkanlar

1. `EK3_RNG_USE_HGT`, kalkış ve iniş için kullanılabilir. Ama yalnızca hız < 2 m/s iken devrede ve seyirdeki baro hatasını düzeltmiyor (1c). Yüzey etkisi modu EKF yüksekliğine değil, doğrudan mesafe sensörüne dayanmalı.
2. Seyirde RC kaybında Glide istiyorsak iki şey gerekiyor:
   - `Q_ASSIST_SPEED`, süzülüşün temas hızının altında olmalı.
   - Temastan sonra motorların durması için ya `Q_TRANS_FAIL > 0` (QLAND üzerinden disarm) ya da kendi kodumuz gerekiyor. Yoksa araç suda silahlı kalıyor.
3. Kalkışta RC kaybı korumasına güvenmek için `FS_LONG_TIMEOUT`, kalkış süresinden kısa olmalı. Alternatifi, kalkışı kendi kodumuzla bağlamak.
4. `Q_TRANS_FAIL`, assist'i de sınırlıyor. Seyirde uzun bir assist de QLAND'e götürür (önceki analiz §2.4).

**Ham veriler:**
- Oturumun geçici dizininde duruyor; repoya eklenmedi.
- İçerik: her deneme için `messages.log`, `events.csv`, `flight.csv`, `.BIN`; betikler `fly2.py`, `run.sh`, `ekf_hgt.py`, `tl.py`.
