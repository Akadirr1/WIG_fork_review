# QGroundControl, ArduPlane tabanlı insansız WIG aracı için yer kontrol istasyonu (YKİ) olarak: kod düzeyinde analiz

| | |
|---|---|
| Repo | `github.com/mavlink/qgroundcontrol` |
| Analiz edilen etiket | **`v5.1.5`**. `v*.*.*` biçimindeki en yüksek etiket; daha yeni olan tek etiket `v5.2.0-dev` ve bu bir geliştirme etiketi. |
| Commit | **`3a67d31f0c36bf3fe38ec52970d250a89d0aaf67`** (2026-09-30, "fix(iOS): pin dated CA bundle snapshot …") |
| MAVLink tanımları | QGC'nin sabitlediği `mavlink/mavlink` commit'i `c409cf690454db6d3e004bd14173bc6c7ff1e0ff`, dialect `all` (`cmake/CustomOptions.cmake:123-125`). Mesajların dialect'te bulunup bulunmadığı bu commit'teki XML'den kontrol edildi. |
| Analiz tarihi | 2026-10-02 |
| Kapsam | Yalnızca yazılım. Fiziksel donanım ve ergonomi kapsam dışı. |

**Yöntem ve okuma kuralları**

- Repo, `v5.1.5` etiketinde blobless ve shallow (`--filter=blob:none --depth 1`) olarak klonlandı.
- Önce klasör yapısı çıkarıldı (bölüm 0). graphify yalnızca ilgili klasörlerde çalıştırıldı: `src/Vehicle`, `src/FlyView`, `src/FirmwarePlugin` (PX4 hariç), `src/Utilities/Audio`, `src/MAVLink`, `src/Toolbar`, `src/Comms` (MockLink hariç). graphify yalnızca C++ dosyalarını işledi: 217 dosya, 6707 node, 12345 edge, 280 community. QML dosyalarını graphify işlemediği için bu dosyalar doğrudan okundu.
- Kaynak takibi gerektiğinde `src/API`, `src/Settings`, `src/QmlControls`, `src/FactSystem`, `src/MainWindow`, `src/AnalyzeView`, `src/MissionManager`, `src/Gimbal` ve `src/FlightMap/Widgets` altındaki dosyalardan yalnızca ilgili olanlara bakıldı. Tüm repo okunmadı.
- `dosya:satır` atıflarının hepsi repo köküne göredir. Tablolarda kısa yazılan biçimler (`:123`, `V:123`) her bölümün başında açıklanan dosyaya aittir.
- Uzun biçimdeki 319 atıf (`src/...:N`) bir script ile kontrol edildi: dosyaların hepsi var ve satır numaraları dosyanın sınırları içinde. Bunun dışında kritik iddiaların bir kısmı kaynakta elle tekrar okundu.
- **"doğrulanmadı"** şu anlama gelir: ilgili davranış bu repodaki koddan kanıtlanamıyor. Çoğunlukla ArduPilot firmware'inin gerçekte hangi mesajı gönderdiği bu durumdadır. **"kodda bulunamadı"** ise belirtilen yerlerde arandığı halde ilgili kodun olmadığı anlamına gelir.
- Kod çalıştırılmadı. Bütün bulgular statik kod okumasına dayanır.

## 0. Klasör yapısı (ilgili kısımlar)

```
qgroundcontrol @ v5.1.5
├── src/
│   ├── Vehicle/              Vehicle, VehicleLinkManager (comm lost), StandardModes, InitialConnectStateMachine
│   │   └── FactGroups/       Telemetri FactGroup'ları ve *.json metadata (birim, açıklama)
│   ├── FirmwarePlugin/       FirmwarePlugin (temel), APM/ (APMFirmwarePlugin, ArduPlaneFirmwarePlugin, APM*Indicator.qml)
│   ├── FlyView/              Fly View QML: TelemetryValuesBar, VehicleWarnings, PreFlight*Check, GuidedActions
│   ├── Toolbar/              Üst çubuk göstergeleri: MainStatus, FlightMode, GPS, Battery, RSSI, ESC, RemoteID...
│   ├── MAVLink/              StatusTextHandler, SysStatusSensorInfo, LibEvents (health/arming events)
│   ├── Comms/                LinkManager, MAVLinkProtocol (tlog kaydı, paket kaybı sayacı)
│   ├── Utilities/Audio/      AudioOutput (QTextToSpeech, tek ses çıkışı)
│   ├── QmlControls/          FactValueGrid, InstrumentValueData (değer seçici)
│   ├── Settings/             *.SettingsGroup.json (varsayılan ayarlar)
│   ├── API/                  QGCCorePlugin (varsayılan grid düzeni, toolbar)
│   ├── FactSystem/           Fact, FactGroup, FactMetaData (birim dönüşümü, NaN gösterimi)
│   ├── MainWindow/           Kritik mesaj popup'ı
│   └── AnalyzeView/          MAVLink Inspector, Vibration, Onboard Logs
├── resources/  translations/  test/  cmake/  ...
```

graphify'ın en çok bağlantı alan düğümleri: `Vehicle` (579 edge), `FirmwarePlugin` (99), `APMFirmwarePlugin` (88) ve `VehicleFactGroup` (83). Analiz bu düğümlerin çevresinde yoğunlaştı.

## Yönetici özeti

- **Göstergeler:** Fly View'in varsayılan değer gridinde Plane için 8 değer var: Alt (Rel), Distance to Home, Climb Rate, Ground Speed, AirSpd, Thr, Flight Time ve Flight Distance. Üst çubukta solda Main Status (arm, hazır olma, comm lost) ve Flight Mode bulunur. Sağda 11 gösterge vardır ve her biri yalnızca kendi koşulu sağlandığında görünür. Kullanıcı gride yaklaşık 183 sabit Fact ekleyebilir; bunlara ek olarak dinamik batarya ve ESC grupları da vardır.
- **ArduPlane modları:** 0-26 arasındaki numaralar tanınıyor (9 numara RESERVED). Tanınmayan bir mod numarası geldiğinde ekranda numarasız, düz **"Unknown"** yazar. Firmware `AVAILABLE_MODES` gönderirse yeni bir mod adı QGC'de hiçbir değişiklik yapmadan görünür.
- **Uyarılar:** Resmi bir öncelik sistemi yok; yalnızca Error, Warning ve Normal olmak üzere üç sınıf var. Severity 0-3 olan mesajlar sarı bir popup açar ve popup'a tıklamak kabul anlamına gelir. Mesaj geçmişi sınırsızdır. EKF, titreşim, failsafe, geofence ve rangefinder için Fly View'de **görsel uyarı yok**.
- **Ses:** Tek ses yolu TTS. Şu olaylarda konuşur: STATUSTEXT (severity ≤ NOTICE veya `#` önekli), mod değişimi, arm/disarm, batarya `charge_state` geçişi, geofence ihlali ve link kaybı/geri gelmesi. Bip veya alarm sesi çalınmaz.
- **Bayatlık:** Araçtan 3.5 sn boyunca hiç mesaj gelmezse üst çubuk kırmızıya döner ve "Comms Lost" yazar, ayrıca TTS konuşur. Değerler gri olmaz, son hallerinde **donuk kalır**.
- **Kayıt:** tlog varsayılan olarak açık, ama yalnızca araç arm edilmiş oturumları kaydeder. Kayıt yeri `<Documents>/QGroundControl/Telemetry/yyyy-MM-dd hh-mm-ss.tlog`.

---

## A. Göstergeler

### A1. Fly View üst araç çubuğu göstergeleri ve varsayılan telemetri değerleri

Kaynak: QGroundControl v5.1.5 (3a67d31f0c36bf3fe38ec52970d250a89d0aaf67). Yollar repo köküne göredir. Yalnızca kodda görülenler yazıldı. Değerlerin beslendiği MAVLink mesajı için `src/Vehicle/`, `src/API/`, `src/Gimbal/` ve `src/FlightMap/Widgets/` altındaki ilgili dosyalara da bakıldı (istenen klasör listesinin dışında, yalnızca kaynak izlemek için). ArduPilot'un ilgili mesajı gerçekten yayınlayıp yayınlamadığı (firmware tarafı) kodda görülemez, "doğrulanmadı" olarak işaretlendi.

#### 1. Üst araç çubuğu (Fly View toolbar)

##### 1.1 Çubuğun yapısı ve gösterge listesinin nereden geldiği

- Çubuk `src/Toolbar/FlyViewToolBar.qml`. Soldan sağa: QGC logo butonu (`:77-84`), `MainStatusIndicator` (`:86-90`), araç varsa ve comm lost ise "Disconnect" butonu (`:93-98`), `FlightModeIndicator` (`:100-104`, `visible: _activeVehicle`), orta panelde `GuidedActionConfirm` (`:118-125`), sağda `FlyViewToolBarIndicators` (`:138`). Parametre indirme ilerlemesi çubuğun üstüne biner (`:184-186`).
- Sağ taraftaki göstergeler iki `Repeater` ile yüklenir: önce `QGroundControl.corePlugin.toolBarIndicators` (`src/Toolbar/FlyViewToolBarIndicators.qml:23-32`, model `:25`), sonra `_activeVehicle.toolIndicators` (`:34-44`, model `:36`). Her ikisinde de görünürlük `visible: item.showIndicator` (`:30`, `:42`). Yani her göstergenin görünme koşulu kendi QML dosyasındaki `showIndicator` property'sidir.
- `corePlugin.toolBarIndicators` yalnızca `RTKGPSIndicator.qml` içerir (`src/API/QGCCorePlugin.cc:333-342`, öğe `:337`).
- `Vehicle::toolIndicators()` firmware plugin'e gider (`src/Vehicle/Vehicle.cc:2581-2584`). Temel liste `src/FirmwarePlugin/FirmwarePlugin.cc:197-220` içinde, sırayla: VehicleGPS, GPSResilience, TelemetryRSSI, RCRSSI, Battery, RemoteID, Gimbal, Esc, Joystick, MultiVehicleSelector (`:202-211`), ayrıca yalnızca `QT_DEBUG` derlemesinde GCSControl (`:212-214`).
- ArduPilot: `APMFirmwarePlugin::toolIndicators` temel listeyi alır ve sona `APMSupportForwardingIndicator.qml` ekler (`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:631-642`, ekleme `:638`). `ArduPlaneFirmwarePlugin` bu fonksiyonu override etmiyor (`src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.{cc,h}` içinde `toolIndicators` geçmiyor; grep ile doğrulandı). Sonuç: ArduPlane için aday liste = temel 10 gösterge + SupportForwarding (+ debug derlemede GCSControl) + RTK (yalnızca araç yokken).
- Kullanılmayan dosyalar: `ArmedIndicator.qml`, `ModeIndicator.qml`, `FlightModeMenuIndicator.qml` yalnızca `src/Toolbar/CMakeLists.txt:8,13,24` içinde geçiyor, Fly View çubuğundan referans edilmiyor (grep ile doğrulandı). Yani çubukta ayrı bir "Armed/Disarmed" kutusu yok. Arm/Disarm `MainStatusIndicator` açılır sayfasından yapılıyor (aşağıda satır 14).

##### 1.2 Gösterge tablosu (ArduPlane için)

"AP Plane notu" sütunu yalnızca QGC kodundaki koşula dayanır. Firmware'in ilgili mesajı yollayıp yollamadığı "doğrulanmadı".

| # | Gösterge | Görünme koşulu (dosya:satır) | Ekranda gösterdiği | Birim | Kaynak Fact/property | Beslendiği MAVLink mesajı | Tıklayınca açılan sayfa | AP Plane notu |
|---|---|---|---|---|---|---|---|---|
| 1 | Vehicle GPS (uydu sayısı + HDOP) | `src/Toolbar/VehicleGPSIndicator.qml:9` `showIndicator: _activeVehicle.gps.telemetryAvailable` (bayrak, ilk GPS_RAW_INT/HIGH_LATENCY ile true: `src/Vehicle/FactGroups/VehicleGPSFactGroup.cc:83,96,111`). Uydu/HDOP kolonu ayrıca `src/Toolbar/GPSIndicator.qml:56` `!isNaN(gps.hdop.value)` | Uydu sayısı (`GPSIndicator.qml:62`), HDOP 1 ondalık (`:68`). İkon opaklığı `gps.count.value >= 0 ? 1 : 0.5` (`:48`), count hiç negatif olmadığı için pratikte hep 1 | uydu adedi, HDOP birimsiz | `vehicle.gps.count`, `vehicle.gps.hdop` | `GPS_RAW_INT` (`satellites_visible`, `eph/100`): `VehicleGPSFactGroup.cc:68-84` (satır `:76-77`) | `GPSIndicatorPage.qml` (`GPSIndicator.qml:75-82`): "Vehicle GPS Status" başlığı (`src/Toolbar/GPSIndicatorPage.qml:87`): Satellites (`:91-93`), GPS Lock = `gps.lock.enumStringValue` (`:96-98`; enum: None/No Fix/2D/3D/3D DGPS/RTK float/RTK fixed/Static, `src/Vehicle/FactGroups/GPSFact.json` `lock`), HDOP (`:100-103`), VDOP (`:105-108`), Course Over Ground (`:110-113`), GPS Error (`gps.systemErrors`, yalnızca >0, `:115-119`). "RTK GPS Status" grubu yalnızca `gpsRtk.connected` iken (`:122-145`). Genişletilmiş sayfada RTK base ayarları (`:149-267`) | Görünür. GPS2 (`GPS2_RAW`, `src/Vehicle/FactGroups/VehicleGPS2FactGroup.cc:13`) çubukta ayrı gösterilmiyor |
| 2 | RTK GPS | `src/Toolbar/RTKGPSIndicator.qml:8` `!_activeVehicle && _rtkConnected` | Aynı `GPSIndicator` temeli, "RTK" yazısı dikey (`GPSIndicator.qml:31-38`) | | `QGroundControl.gpsRtk.connected` | Yerel RTK base GPS (MAVLink değil) | Aynı `GPSIndicatorPage` | Araç bağlıyken gizli, WIG için ilgisiz |
| 3 | GPS Resilience (jamming/spoofing/auth) | `src/Toolbar/GPSResilienceIndicator.qml:30-34`: `gpsAggregate` içinde authentication/spoofing/jamming state'lerinden biri 0 ile 255 arasında (0 ve 255 hariç) | Kimlik doğrulama ikonu (`:37-47`) ve girişim ikonu (`:50-60`), renk durum koduna göre (`:62-81`) | enum | `vehicle.gpsAggregate.{authenticationState,spoofingState,jammingState}` | `GNSS_INTEGRITY` (development mesajı): `VehicleGPSFactGroup.cc:60-62,114-133` | "GPS Resilience Status" + GPS 1 / GPS 2 ayrıntıları (`GPSResilienceIndicator.qml:96-179`) | Başlangıç değeri 255 (`VehicleGPSFactGroup.cc:37-39`), mesaj gelmedikçe gizli kalır. ArduPilot GNSS_INTEGRITY yayınlıyor mu: doğrulanmadı |
| 4 | Telemetry RSSI | `src/Toolbar/TelemetryRSSIIndicator.qml:16,20` `radioStatus.lrssi.rawValue !== 0` | Yalnızca ikon (`:22-31`), sayı yok | dBm | `vehicle.radioStatus.lrssi` | `RADIO_STATUS`: `src/Vehicle/FactGroups/RadioStatusFactGroup.cc:22-52` (3DR Si1k için dönüşüm `:36-45`). Başka sysid'den gelse bile link araca aitse geçer: `src/Vehicle/Vehicle.cc:526-531` | "Telemetry RSSI Status": Local RSSI dBm (`:47-50`), Remote RSSI dBm (`:52-55`), RX Errors (`:57-60`), Errors Fixed (`:62-65`), TX Buffer (`:67-70`, json birimi %, `src/Vehicle/FactGroups/RadioStatusFact.json:32`), Local/Remote Noise (`:72-80`, dBm: `RadioStatusFact.json:38,44`) | Yalnızca RADIO_STATUS gelirse (telemetri radyosu varsa) görünür. lrssi tam 0 ise gizli |
| 5 | RC RSSI | `src/Toolbar/RCRSSIIndicator.qml:15` `supports.radio && _rcRSSIAvailable`; `:18` `0 < rcRSSI <= 100`. `supportsRadio()` varsayılan true (`src/FirmwarePlugin/FirmwarePlugin.h:262`), yalnızca ArduSub false (`src/FirmwarePlugin/APM/ArduSubFirmwarePlugin.h:86`) | Sinyal çubuğu ikonu (`:37-59`; `src/Toolbar/SignalStrength.qml:16-28` eşikler 20/40/60/80/95 %) | % (0-100) | `vehicle.rcRSSI` (low-pass filtre 0.9/0.1, 255 = bilinmiyor): `src/Vehicle/FactGroups/VehicleFactGroup.cc:51-77`; `Vehicle.cc:1385-1387` | `RC_CHANNELS.rssi`: `Vehicle.cc:601-603,1336-1390`. APM için önce ölçek çevrilir 0-254 → 0-100 (`APMFirmwarePlugin.cc:1126-1146`, satır `:1135-1137`; yalnızca ArduPilot bileşeninden gelen mesajlarda: `:302-311`) | "RC RSSI Status": RSSI % (`:30-31`) | RC_CHANNELS.rssi 255 ise (alıcı RSSI vermiyorsa) gizli. Firmware'in RSSI doldurması: doğrulanmadı |
| 6 | Battery | `src/Toolbar/BatteryIndicator.qml:17` `batteries.count > 0` | Pil ikonu + yüzde ve/veya voltaj. Renk/ikon `chargeState` ve eşiklere göre (`:211-260`). Metin: yüzde varsa yüzde, yoksa voltaj, yoksa chargeState, yoksa "n/a" (`:262-284`). Kaç değerin gösterileceği `valueDisplay` ayarı: 0 Percentage, 1 Voltage, 2 ikisi (`:24-26`, `:330-345`); varsayılan 0 (`src/Settings/BatteryIndicator.SettingsGroup.json:11`, `"default": false` = 0). Eşikler varsayılan 80 ve 60 % (aynı dosya, threshold1/threshold2 `default`). Birden fazla pil varsa en düşüğü gösterme varsayılan açık (`consolidateMultipleBatteries`, `:180`) | % ve V | `vehicle.batteries[i].{percentRemaining, voltage, chargeState}` | `BATTERY_STATUS`: `src/Vehicle/FactGroups/BatteryFactGroupListModel.cc:19-25,103-143` (voltaj = hücre voltajları toplamı `:113-132`, yüzde `:140`, akım `/100` `:138`). High latency için ayrı yol `:15-18,83-101`. SYS_STATUS'tan pil okuyan kod yok (grep ile doğrulandı) | "Battery Status" grubu (pil başına): Charge State, Remaining (süre ve %), Voltage, Consumed mAh, Temperature, Function (`BatteryIndicator.qml:350-431`; her satırın kendi `visible` koşulu `:360-367`). Genişletilmiş sayfa: Battery Display ayarları (`:433-551`), APM'ye özel "Low Voltage Failsafe" ve "Critical Voltage Failsafe" (BATT_FS_LOW_ACT, BATT_LOW_VOLT, BATT_LOW_MAH, BATT_FS_CRT_ACT, BATT_CRT_VOLT, BATT_CRT_MAH: `src/FirmwarePlugin/APM/APMBatteryIndicator.qml:14-72`, yükleme `BatteryIndicator.qml:553-556`, kaynak seçimi `APMFirmwarePlugin.cc:1398-1401`), "Vehicle Power" kurulum butonu (`:558-571`) | Yalnızca BATTERY_STATUS gelince görünür (BATT_MONITOR kapalıysa firmware mesaj yollamayabilir: doğrulanmadı) |
| 7 | Remote ID | `src/Toolbar/RemoteIDIndicator.qml:16` `remoteIDManager.available`; `available` yalnızca `OPEN_DRONE_ID_ARM_STATUS` ilk geldiğinde true (`src/Vehicle/RemoteIDManager.cc:74-93`, `:57`) | Durum ikonu (HEALTHY/WARNING/ERROR/UNAVAILABLE: `RemoteIDIndicator.qml:31-36`) | | `vehicle.remoteIDManager.*`, `remoteIDSettings.*` | `OPEN_DRONE_ID_ARM_STATUS` | "RemoteID Status": ARM STATUS, RID COMMS, GCS GPS, BASIC ID, OPERATOR ID (`src/Toolbar/RemoteIDIndicatorPage.qml:95-194`), Emergency butonu (`:220-245`), Self ID ayarları (`:317-380`) | RID cihazı yoksa gizli |
| 8 | Gimbal | `src/Toolbar/GimbalIndicator.qml:16` `gimbalController.gimbals.count` | Aktif gimbal adı, retract durumu, pitch ("P: ...") (`:70,87,97,103`) | derece | `vehicle.gimbalController.activeGimbal` | `GIMBAL_MANAGER_INFORMATION`, `GIMBAL_MANAGER_STATUS`, `GIMBAL_DEVICE_ATTITUDE_STATUS`: `src/Gimbal/GimbalController.cc:61-73` | Aktif gimbal, komutlar (Center, Tilt 90, Point Home, Retract, kontrol al/bırak: `GimbalIndicator.qml:136-220`), ekran kontrolü ve zoom hızı ayarları (`:238-311`) | Gimbal yoksa gizli |
| 9 | ESC | `src/Toolbar/EscIndicator.qml:14` `escs.count > 0` | Çevrimiçi motor sayısı (`:88`) ve "OK"/"ERR" (`:92-96`). OK koşulu: online motor sayısı == `count` ve online motorların `failureFlags` değeri 0 (`:41-55`) | adet | `vehicle.escs[i].{count, info, failureFlags, rpm, voltage, current, temperature, errorCount}` | `ESC_INFO` ve `ESC_STATUS`: `src/Vehicle/FactGroups/EscStatusFactGroupListModel.cc:19,27,79-134` | "ESC Status Overview" + her motor için RPM, Temp, Voltage, Current, Errors (`src/Toolbar/EscIndicatorPage.qml:31-106`). Sıcaklık json'da "centi-celsius" (`src/Vehicle/FactGroups/EscStatusFactGroup.json:97`), sayfa birimi olduğu gibi yazıyor (`EscIndicatorPage.qml:88`) | ESC telemetri mesajı gelmezse gizli. ArduPilot'un yayını: doğrulanmadı |
| 10 | Joystick | `src/Toolbar/JoystickIndicator.qml:14` `joystickManager.activeJoystick` (araç bağımsız) | Joystick ikonu, araç için etkin değilse turuncu (`:256-264`) | | `joystickManager.activeJoystick` | MAVLink yok (yerel cihaz) | Cihaz ayrıntıları (ad, tip, bağlantı, pil, VID/PID vb., `:27-245`) | Joystick bağlıysa görünür |
| 11 | Multi-vehicle seçici | `src/Toolbar/MultiVehicleSelector.qml:13,15` `vehicles.count > 1` | Araç seçici | | `multiVehicleManager.vehicles` | | Araç listesi (`:45`) | Tek araçta gizli |
| 12 | GCS Control (yalnızca debug derleme) | `src/Toolbar/GCSControlIndicator.qml:15` `firstControlStatusReceived`; listede yalnızca `QT_DEBUG` (`FirmwarePlugin.cc:212-214`) | | | | `Vehicle.cc:3294` civarı kontrol durumu | Kontrol devri sayfaları (`:54,122,379`) | Release derlemede yok |
| 13 | APM Support Forwarding | `src/FirmwarePlugin/APM/APMSupportForwardingIndicator.qml:15` `linkManager.mavlinkSupportForwardingEnabled` | Yeşil forwarding ikonu (`:32-40`) | | `mavlinkSettings.forwardMavlinkAPMSupportHostName` (`:26`) | | "Mavlink traffic is being forwarded to a support server" + sunucu adı (`:18-29`) | Yalnızca APM, ayar açıksa. (Dosya başlığında kopya yorum "Telemetry RSSI" yazıyor, `:7-8`) |

Sol taraftaki göstergeler (araç bağlı olduğunda her zaman çubukta):

| # | Gösterge | Görünme koşulu | Ekranda gösterdiği | Kaynak property | Beslendiği MAVLink mesajı | Tıklayınca açılan sayfa |
|---|---|---|---|---|---|---|
| 14 | Main Status (araç durumu, arm) | Her zaman (`FlyViewToolBar.qml:86-90`) | Etiket (`src/Toolbar/MainStatusIndicator.qml:45-108`): araç yoksa "Disconnected - Click to manually connect"; `communicationLost` ise "Comms Lost" (kırmızı); armed ise "Flying"/"Landing"/"Armed" (yeşil); disarmed ise "Ready"/"Not Ready" (yeşil/sarı). Disarmed hazır olma mantığı: `healthAndArmingCheckReport.supported` ise onu kullanır (`:73-84`), değilse `readyToFlyAvailable` + `readyToFly` (`:85-92`), o da yoksa `allSensorsHealthy && autopilotPlugin.setupComplete` (`:93-101`). Mesaj ikonu: `messageCount > 0` iken, uyarıda turuncu, hatada kırmızı (`:110-133`). VTOL etiketi yalnızca VTOL'de (`:141-158`) | `vehicle.armed`, `flying`, `landing`, `readyToFly`, `allSensorsHealthy` | Armed: `HEARTBEAT.base_mode` (`Vehicle.cc:1277-1300`), ARMING_REQUIRE=0 durumunda `SYS_STATUS` motor çıkış biti (`:1117-1122`, `:1289-1296`). Flying (APM): `HEARTBEAT.system_status` ACTIVE/CRITICAL/EMERGENCY ve armed (`APMFirmwarePlugin.cc:265-278`). Landing/flying ayrıca `EXTENDED_SYS_STATE` ile de güncellenir (`Vehicle.cc:1030-1056`); QGC bunu APM'den 1 Hz ister (`APMFirmwarePlugin.cc:431-433`). readyToFly: `SYS_STATUS` PREARM_CHECK biti (`Vehicle.cc:1084-1095`) | Açılır sayfa (`MainStatusIndicator.qml:169-182`): Arm/Disarm `QGCDelayButton` (`:197-214`), Primary Link seçici (birden çok link varsa, `:216-245`), "Vehicle Messages" listesi (`:248-261`), "Sensor Status" (sensör adı/durumu, `SYS_STATUS`'tan, health-and-arming-checks desteklenmiyorsa, `:263-284`), "Overall Status" (health/arming check problemleri, destekliyse, `:286-296`). Genişletilmiş: APM'ye özel "Ground Control Comm Loss Failsafe" (FS_GCS_ENABLE, FS_GCS_TIMEOUT) ve "Failsafe Options" (FS_OPTIONS bitmask) (`src/FirmwarePlugin/APM/APMMainStatusIndicator.qml:14-31` ve `:33-61`; yükleme `MainStatusIndicator.qml:389-392`, kaynak seçimi `APMFirmwarePlugin.cc:1404-1405`), Force Arm (`MainStatusIndicator.qml:394-406`), Vehicle Parameters/Configuration butonları (`:408-436`) |
| 15 | Flight Mode | `FlyViewToolBar.qml:103` `visible: _activeVehicle` | Mod adı büyük yazı (`src/Toolbar/FlightModeIndicator.qml:42-48`) | `vehicle.flightMode` | `HEARTBEAT.base_mode/custom_mode` (`Vehicle.cc:1302-1306`); ArduPlane mod tablosu `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.cc:10-38` (MANUAL, CIRCLE, STABILIZE, TRAINING, ACRO, FBWA, FBWB, CRUISE, AUTOTUNE, AUTO, RTL, LOITER, TAKEOFF, AVOID_ADSB, GUIDED, INITIALIZING, QSTABILIZE, QHOVER, QLOITER, QLAND ...) | Mod butonları listesi `activeVehicle.flightModes` (`FlightModeIndicator.qml:148-191`), gizli modları düzenleme (`:97-120`, `:175-189`), onay için basılı tutma ayarı (`:229-231`). APM'nin genişletilmiş "Return to Launch" grubu yalnızca multirotor için yüklenir (`APMFirmwarePlugin.cc:1402-1403`; `APMFlightModeIndicator.qml:12`), Plane'de yok |

Not (ArduPlane için çubuk düzeni): Main Status, Flight Mode, sonra sırayla yalnızca koşulu sağlayanlar: Vehicle GPS, GPS Resilience, Telemetry RSSI, RC RSSI, Battery, Remote ID, Gimbal, ESC, Joystick, Multi-vehicle, Support Forwarding.

#### 2. Fly View varsayılan değerleri

##### 2.1 Alt sağ panel: telemetri değer çubuğu + instrument panel

- `src/FlyView/FlyViewWidgetLayer.qml:79-88` `FlyViewBottomRightRowLayout` yerleştirir. İçinde `TelemetryValuesBar` (`src/FlyView/FlyViewBottomRightRowLayout.qml:9-14`, `settingsGroup: factValueGrid.telemetryBarSettingsGroup` `:12`, yani "TelemetryBarUserSettings", `src/QmlControls/HorizontalFactValueGrid.cc:4`) ve `FlyViewInstrumentPanel` (`:16-19`, `visible: corePlugin.options.flyView.showInstrumentPanel && _showSingleVehicleUI`; `showInstrumentPanel` varsayılan true: `src/API/QGCOptions.h:35`).
- `src/FlyView/TelemetryValuesBar.qml` yalnızca `HorizontalFactValueGrid`'i sarar (`:52-54`); sağ tık veya basılı tutma ile kilit açılıp kullanıcı hücreleri düzenler (`:57-78`).
- Kullanıcı düzeni `QSettings`'te `"<settingsGroup>-<vehicleClass>"` anahtarında saklanır (`src/QmlControls/FactValueGrid.cc:300-303`). Kayıt yoksa (ya da sürüm uyuşmazsa) varsayılanlar yüklenir (`:317-325`, `:355-358`) ve `QGCCorePlugin::factValueGridCreateDefaultSettings` çağrılır. Araç tipi değişince yeniden yüklenir (`:61`).

##### 2.2 Varsayılan Fact listesi (tanım yeri: `src/API/QGCCorePlugin.cc:186-298`)

Bu, ArduPlane (VehicleClassFixedWing) için "tek araç" çubuğu varsayılanıdır: `includeFWValues` true (`:226`), 4 kolon x 2 satır (`:230-236`), yazı boyutu masaüstünde Medium (`:188-192`).

Fact'ler "Vehicle" grubunda, yani `Vehicle` nesnesinin kendisinde (`Vehicle : public VehicleFactGroup`, `src/Vehicle/Vehicle.h:86`; `src/QmlControls/InstrumentValueData.cc:72-73`). Tanımları `src/Vehicle/FactGroups/VehicleFact.json`.

| Değer (kolon, satır) | Etiket | Birim (json) | Kaynak Fact | Beslendiği mesaj / hesap | Varsayılan tanım satırı (`QGCCorePlugin.cc`) | Fact json satırı (`VehicleFact.json`) |
|---|---|---|---|---|---|---|
| 1,1 | Alt (Rel) (ikon arrow-thick-up) | "vertical m" | `altitudeRelative` | `ALTITUDE.altitude_relative` varsa o (`VehicleFactGroup.cc:147-158`, `:154`), yoksa `GLOBAL_POSITION_INT.relative_alt/1000` (`Vehicle.cc:861-873`, `:871`) | `:241-245` | `:70-74` |
| 1,2 | Distance to Home | m | `distanceToHome` | Araç koordinatı ile home arası mesafe, MAVLink alanı değil hesap: `Vehicle.cc:2613-2628` (`:2616`). Home yoksa NaN | `:247-251` | `:98-102` |
| 2,1 | Climb Rate | m/s | `climbRate` | `VFR_HUD.climb`: `src/Vehicle/FactGroups/VehicleFactGroup.cc:208-226` (`:215`) | `:256-260` | `:63-67` |
| 2,2 | Ground Speed | m/s | `groundSpeed` | `VFR_HUD.groundspeed` (`VehicleFactGroup.cc:214`) | `:262-266` | `:49-53` |
| 3,1 | "AirSpd" | m/s | `airSpeed` | `VFR_HUD.airspeed` (`VehicleFactGroup.cc:213`) | `:272-275` | `:56-60` |
| 3,2 | "Thr" | % | `throttlePct` | `VFR_HUD.throttle` (`VehicleFactGroup.cc:216`) | `:277-280` | `:170-173` |
| 4,1 | Flight Time (birim gösterilmez) | elapsedSeconds | `flightTime` | QGC içi kronometre: armed olunca başlar (`Vehicle.cc:1621-1637`, `:1223`, `:1229`), MAVLink yok | `:286-290` | `:159-162` |
| 4,2 | Flight Distance | m | `flightDistance` | İz noktalarından toplanır: `src/Vehicle/TrajectoryPoints.cc:20` → `Vehicle.cc:2903-2906`; armed olunca sıfırlanır (`:1625`) | `:292-296` | `:91-95` |

Notlar:
- Hız/irtifa birimleri: json'daki birim ham birimdir. Ekranda kullanıcının AppSettings birim tercihine göre dönüşüm uygulanıp uygulanmadığı bu çalışmada izlenmedi: doğrulanmadı.
- Multi-vehicle kartı için ayrı varsayılan: Alt (Rel) ve (FW ise) AirSpd (`QGCCorePlugin.cc:194-224`).
- Varsayılanı kullanıcı değiştirmiş olabilir, `FactValueGrid::resetToDefaults` ayarları siler (`FactValueGrid.cc:98-103`).
- Yeni hücre eklenince varsayılan olarak `AltitudeRelative` gelir (`FactValueGrid.cc:246-253`).

##### 2.3 Attitude / compass widget'ı (instrument panel)

- `src/FlyView/FlyViewInstrumentPanel.qml:6-9` hangi widget'ın yükleneceğini `flyViewSettings.instrumentQmlFile2` ile seçer. Seçenekler: Integrated Compass & Attitude (varsayılan), Horizontal Compass & Attitude, Large Vertical (`src/Settings/FlyView.SettingsGroup.json:107-115`, varsayılan `:112`).
- Varsayılan `src/FlightMap/Widgets/IntegratedCompassAttitude.qml`: roll göstergesi (`:27-32`, `vehicle.roll.rawValue`), pitch göstergesi (`:34-44`, `-vehicle.pitch.rawValue`, 90° döndürülmüş), ortada `QGCCompassWidget` (`:53-58`).
- Compass widget'ı (`src/FlightMap/Widgets/QGCCompassWidget.qml`): heading (`vehicle.heading`, `:23`, `:70`), altta heading yazısı derece (`:127`). Ek göstergeler yalnızca `showAdditionalIndicatorsCompass` ayarı açıksa (varsayılan false, `FlyView.SettingsGroup.json:32-35`; `QGCCompassWidget.qml:29`): COG ibresi `gps.courseOverGround` (yer hızı >= 0.5 ise, `:32-37`, `:74-83`), sonraki WP yönü `headingToNextWP` (`:44-46`, `:85-94`), home yönü "L" işareti `headingToHome` (`:40-42`, `:96-121`). `lockNoseUpCompass` varsayılan false (`:30`, `:61`; json `:40-43`).
- `QGCAttitudeWidget.qml` (diğer stil) heading'i üç haneli yazar, veri yoksa "OFF" (`:119-125`).
- Kaynak: roll/pitch/heading `ATTITUDE` (`VehicleFactGroup.cc:129-145`, `:108-127`; yaw 0-360'a çevrilir ve tam sayıya kesilir `:118-126`) ya da `ATTITUDE_QUATERNION` gelirse o (`:160-193`, quaternion alındıktan sonra ATTITUDE yok sayılır `:135-137`). Birim derece (`VehicleFact.json:7-25`). Heading VFR_HUD'dan değil attitude yaw'dan gelir.

##### 2.4 VehicleWarnings.qml (ekran ortasında uyarı kutusu)

`FlyViewWidgetLayer.qml:169-172` ortalar ve en üst z'ye koyar. Kutu `visible: _noGPSLockVisible || _prearmErrorVisible` (`src/FlyView/VehicleWarnings.qml:12`). Üç etiket gösterebilir (siyah yazı, yarı saydam beyaz zemin `:10`):
1. "No GPS Lock for Vehicle" (`:27`), koşul `requiresGpsFix && !coordinate.isValid` (`:15`). `requiresGpsFix` = `SYS_STATUS.onboard_control_sensors_present` içinde GPS biti (`Vehicle.h:539`, `Vehicle.cc:1103-1107`). `coordinate` `GPS_RAW_INT` (fix >= 3D ve GLOBAL_POSITION_INT yoksa) veya `GLOBAL_POSITION_INT` ile gelir, lat/lon 0/0 yok sayılır (`Vehicle.cc:835-857`, `:861-887`).
2. Son prearm hata metni `vehicle.prearmError` (`:35`) ve 3. sabit metin "The vehicle has failed a pre-arm check. In order to arm the vehicle, resolve the failure." (`:46`). Koşul `!armed && prearmError && !healthAndArmingCheckReport.supported` (`:16`). `prearmError`, STATUSTEXT "PreArm" ile başlıyorsa set edilir (`Vehicle.cc:3428-3441`, aynı metin 10 sn içinde tekrarlanırsa atlanır), 35 sn sonra silinir (`Vehicle.h:963`, `Vehicle.cc:2261-2273`). Health-and-arming-check event desteği varsa (`Vehicle.cc:3431`) PreArm STATUSTEXT'leri düşürülür ve bu kutu çıkmaz. ArduPilot için desteğin true olup olmadığı doğrulanmadı.

Ayrıntı için bkz. C.

##### 2.5 Fly View'de diğer parçalar (kısa)

Üst sağ: multi-vehicle paneli yalnızca birden fazla araçta (`src/FlyView/FlyViewTopRightPanel.qml:16`); tek araçta `FlyViewTopRightColumnLayout` (TerrainProgress ve kamera/foto-video kontrolü, `FlyViewTopRightColumnLayout.qml:9-33`, `FlyViewWidgetLayer.qml:67-77`). Bu çalışmada bunların değerleri izlenmedi.

#### 3. Doğrulanmayanlar (özet)

- ArduPilot'un BATTERY_STATUS, RADIO_STATUS, ESC_STATUS/ESC_INFO, GNSS_INTEGRITY, ODID_ARM_STATUS, GIMBAL_MANAGER_* mesajlarını WIG aracında gerçekten yayınlayıp yayınlamadığı (firmware/parametre bağımlı).
- Hız ve irtifa değerlerinin ekranda hangi birimde göründüğü (AppSettings birim dönüşümü).
- ArduPilot'ta `healthAndArmingCheckReport.supported` değerinin true olup olmadığı (EventManager tarafı incelenmedi); bu, Main Status "Ready/Not Ready" yolunu ve VehicleWarnings prearm kutusunu belirler.


### A2. Fly View değer gridine (TelemetryValuesBar) eklenebilecek TÜM değerler

Kaynak: QGC v5.1.5, commit 3a67d31f0c36bf3fe38ec52970d250a89d0aaf67. Tüm yollar repo köküne göredir. Salt okunur kod okuma; hiçbir şey çalıştırılmadı.

Kısaltmalar (tablolarda): `j<N>` = metadata JSON satırı (birim/shortDesc buradan), `s<N>` = Fact'i dolduran `setRawValue` satırı. "G" = Edit diyaloğuyla gride eklenebilir. "V" = varsayılan grid düzeninde yer alıyor. Satır numaraları grep -n / okuma ile görülerek yazıldı.

---

#### 1. Değer seçim UI'ı listeyi nereden alıyor

| Adım | Ne yapıyor | dosya:satır |
|---|---|---|
| Fly View alt bar | `TelemetryValuesBar` -> `HorizontalFactValueGrid`; ayar grubu "TelemetryBarUserSettings"; `specificVehicleForCard: null` = aktif araç | `src/FlyView/FlyViewBottomRightRowLayout.qml:9-13`, `src/QmlControls/HorizontalFactValueGrid.cc:4` |
| "Edit" (kilit aç) | Alt bardaki tıklama `settingsUnlocked = true` yapar | `src/FlyView/TelemetryValuesBar.qml:65-75` |
| Değer hücresine tıklama | `valueEditDialogFactory.open(...)` -> `InstrumentValueEditDialog` | `src/QmlControls/HorizontalFactValueGrid.qml:178, 184-193` |
| "Group" combobox | `model: instrumentValueData.factGroupNames` | `src/QmlControls/InstrumentValueEditDialog.qml:44-48` |
| "Value" combobox | `model: instrumentValueData.factValueNames` | `src/QmlControls/InstrumentValueEditDialog.qml:60-64` |
| Group listesi nasıl üretilir | `_vehicle->factGroupNames()` (Vehicle'ın alt-FactGroup haritasının anahtarları), baş harf büyütülür, başa sabit "Vehicle" eklenir | `src/QmlControls/InstrumentValueData.cc:325-335` |
| Value listesi nasıl üretilir | Seçili grubun `factGroup->factNames()` (her `_addFact` çağrısı sırasıyla), baş harf büyütülür. "Vehicle" seçiliyse grup = Vehicle'ın kendisi | `src/QmlControls/InstrumentValueData.cc:337-356`, `src/FactSystem/FactGroup.cc:116-131` |
| Group anahtarları sıralı | `factGroupNames()` = `QMap::keys()`, yani alfabetik. Dinamik gruplar sonradan eklenir, sinyal `factGroupNamesChanged` | `src/FactSystem/FactGroup.h:44`, `src/FactSystem/FactGroup.cc:133-143` |
| Fact'e çözümleme | `_vehicle->getFactGroup(grupAdı)->getFact(adı)`; ad camelCase'e çevrilir ("AltitudeRelative" -> "altitudeRelative") | `src/QmlControls/InstrumentValueData.cc:64-99`, `src/FactSystem/FactGroup.cc:72-100, 170-173` |
| Henüz var olmayan grup | Kayıtlı seçim varsa, grup sonradan görünürse `_lookForMissingFact` tamamlar; o zamana kadar hücre "–" gösterir | `src/QmlControls/InstrumentValueData.cc:29, 38-45`, `src/QmlControls/InstrumentValueValue.qml:30-36` |
| Gösterilen metin | `fact.enumOrValueString` + (showUnits ise) `fact.units`; `units` = `cookedUnits` (kullanıcı birim ayarına göre dönüştürülmüş) | `src/QmlControls/InstrumentValueValue.qml:30-34`, `src/FactSystem/Fact.h:52`, `src/FactSystem/Fact.cc:571` |
| Birim dönüşümleri | m/vertical m -> ft; m/s -> ft/s, mph, km/h, kn; C -> F; centi-celsius -> C; rad -> deg vb. Yalnızca tam eşleşen `rawUnits` dönüşür ("°C", "deg", "dBm", "rpm" dönüşmez) | `src/FactSystem/FactMetaData.cc:22-59`, `:966-968` ("vertical m" kullanıcıya hiç gösterilmez) |
| Ayarın saklanması | Grup ve fact adı QSettings'e yazılır/okunur | `src/QmlControls/FactValueGrid.cc:143-151` |
| Varsayılan düzen ne zaman uygulanır | Kayıtlı ayar yoksa veya sürüm 1 değilse | `src/QmlControls/FactValueGrid.cc:318-326, 355-358` |

Varsayılan grid düzeni (V) `src/API/QGCCorePlugin.cc:186-299`. FixedWing/VTOL/Airship sınıfı için `includeFWValues` (`:195` kart, `:226` ana grid `includeFWValues` testi; `QGCMAVLink.cc:218-219` MAV_TYPE_FIXED_WING -> FixedWing):
- Sütun 1: AltitudeRelative (`:242`), DistanceToHome (`:248`)
- Sütun 2: ClimbRate (`:257`), GroundSpeed (`:263`)
- Sütun 3 (yalnız FW/VTOL/Airship): AirSpeed (`:273`), ThrottlePct (`:278`)
- Son sütun: FlightTime (`:287`), FlightDistance (`:293`)
- Çoklu araç kartı (`specificVehicleForCard`): AltitudeRelative (`:207`) + AirSpeed (`:216`) veya GroundSpeed (`:220`)

Yeni bir hücre eklendiğinde ilk fact her zaman Vehicle/AltitudeRelative'dir (`src/QmlControls/FactValueGrid.cc:247-251`).

#### 2. Vehicle'ın FactGroup ağacı (ne eklenir, ne zaman görünür)

Ağaç `Vehicle` kurucusunda kurulur (`src/Vehicle/Vehicle.cc:311-330`, kayıt `:345-362`). `Vehicle` sınıfı kendisi `VehicleFactGroup`'tur (`src/Vehicle/Vehicle.h:86`), bu yüzden "Vehicle" grubu ayrıca `_addFactGroup` edilmez (`Vehicle.cc:344` yorum satırı), UI başa elle ekler (`InstrumentValueData.cc:332`).

Mesaj dağıtımı: `Vehicle.cc:584-585` dinamik grup oluşturma, `:588-590` her FactGroup'a `handleMessage`, `:592` Vehicle'ın kendisine `handleMessage`.

| UI grup adı | Kayıt anahtarı | Sınıf | Kayıt | Görünme zamanı |
|---|---|---|---|---|
| Vehicle | (Vehicle kendisi) | `VehicleFactGroup` (güncelleme 100 ms, `VehicleFactGroup.cc:11`) | `InstrumentValueData.cc:332` | Hep |
| Gps | `gps` | `VehicleGPSFactGroup` | `Vehicle.cc:345` | Hep |
| Gps2 | `gps2` | `VehicleGPS2FactGroup` (GPS grubundan türer) | `:346` | Hep |
| GpsAggregate | `gpsAggregate` | `VehicleGPSAggregateFactGroup` | `:347` | Hep |
| Wind | `wind` | `VehicleWindFactGroup` | `:348` | Hep |
| Vibration | `vibration` | `VehicleVibrationFactGroup` | `:349` | Hep |
| Temperature | `temperature` | `VehicleTemperatureFactGroup` | `:350` | Hep |
| Clock | `clock` | `VehicleClockFactGroup` | `:351` | Hep |
| Setpoint | `setpoint` | `VehicleSetpointFactGroup` | `:352` | Hep |
| DistanceSensor | `distanceSensor` | `VehicleDistanceSensorFactGroup` | `:353` | Hep (not: QML property adı `distanceSensors`, UI adı `DistanceSensor`) |
| LocalPosition | `localPosition` | `VehicleLocalPositionFactGroup` | `:354` | Hep |
| LocalPositionSetpoint | `localPositionSetpoint` | `VehicleLocalPositionSetpointFactGroup` | `:355` | Hep |
| EstimatorStatus | `estimatorStatus` | `VehicleEstimatorStatusFactGroup` (500 ms) | `:356` | Hep |
| Hygrometer | `hygrometer` | `VehicleHygrometerFactGroup` | `:357` | Hep |
| Generator | `generator` | `VehicleGeneratorFactGroup` | `:358` | Hep |
| Efi | `efi` | `VehicleEFIFactGroup` | `:359` | Hep |
| Rpm | `rpm` | `VehicleRPMFactGroup` | `:360` | Hep (not: `Vehicle.h:237-253` Q_PROPERTY listesinde `rpm` yok; QML'den `vehicle.rpm` yok ama C++ `getFactGroup` ile grid seçicide var) |
| Terrain | `terrain` | `TerrainFactGroup` | `:361` | Hep (`_terrainProtocolHandler` yalnız çevrimdışı olmayan araçta kurulur, `Vehicle.cc:332-334`) |
| RadioStatus | `radioStatus` | `RadioStatusFactGroup` | `:362` | Hep |
| Battery0, Battery1, ... | `battery<id>` | `BatteryFactGroup` (1000 ms) | `src/FactSystem/FactGroupListModel.cc:42-44` | İlk BATTERY_STATUS (id = mesajdaki `id`) veya HIGH_LATENCY/2 (id 0) gelince (`BatteryFactGroupListModel.cc:10-29`) |
| EscStatus0, EscStatus1, ... | `escStatus<index>` | `EscStatusFactGroup` | `FactGroupListModel.cc:42-44` | İlk ESC_INFO/ESC_STATUS gelince; bir mesajda `index..index+3` için 4 grup birden açılır (`EscStatusFactGroupListModel.cc:11-47`) |
| Gimbal<managerCompid><deviceId> | `gimbal<comp><dev>` | `Gimbal` | `src/Gimbal/GimbalController.cc:367` | Gimbal manager bilgisi/durumu/attitude tamamlanınca (`:340-367`) |
| Camera | `camera` | `VehicleCameraControl` (kameranın parametreleri dinamik Fact olur) | `src/Camera/VehicleCameraControl.cc:1168` | Kamera tanım dosyası yüklenince; bu analiz kapsamı dışı |
| ApmSubInfo | `apmSubInfo` | `APMSubmarineFactGroup` | `src/FirmwarePlugin/APM/ArduSubFirmwarePlugin.cc:192, 279-281` | YALNIZ ArduSub; ArduPlane'de yok |

APM notu: Firmware'e özel FactGroup kancası `FirmwarePlugin::factGroups()` (`src/FirmwarePlugin/FirmwarePlugin.h:346`, varsayılan nullptr), Vehicle bunları `Vehicle.cc:365-370`'te ekler. `src/FirmwarePlugin` içinde yalnız `ArduSubFirmwarePlugin` bunu override ediyor; `ArduPlaneFirmwarePlugin` / `APMFirmwarePlugin` içinde APM'e özel FactGroup YOK. Yani ArduPlane için grid listesi PX4 ile aynı genel kümedir.

---

#### 3. Grup: Vehicle (araç kök grubu, 32 Fact)

Metadata: `src/Vehicle/FactGroups/VehicleFact.json` (27 girdi). Fact tanımı: `src/Vehicle/FactGroups/VehicleFactGroup.h:90-121`, kayıt `VehicleFactGroup.cc:13-44`. Mesaj `case`'leri `VehicleFactGroup.cc:82-102` (ATTITUDE 82, ATTITUDE_QUATERNION 85, ALTITUDE 88, VFR_HUD 91, NAV_CONTROLLER_OUTPUT 94, RAW_IMU 97, RANGEFINDER 100). Setter dosyası `VehicleFactGroup.cc` ve kısmen `src/Vehicle/Vehicle.cc`.

| Değer (UI adı) | Birim (JSON ham) | Beslendiği MAVLink | Ekranda nerede | dosya:satır |
|---|---|---|---|---|
| Roll | deg | ATTITUDE (veya ATTITUDE_QUATERNION; quaternion bir kez gelirse ATTITUDE yok sayılır) | G; ayrıca artificial horizon: `src/FlightMap/Widgets/QGCAttitudeWidget.qml:15`, `IntegratedCompassAttitude.qml:30` | j7 / s`VehicleFactGroup.cc:124`; ATTITUDE işleyici `:129-143`, yok sayma `:135-137` |
| Pitch | deg | ATTITUDE / ATTITUDE_QUATERNION | G; horizon `QGCAttitudeWidget.qml:16`, `IntegratedCompassAttitude.qml:38` | j14 / s`:125` |
| Heading | deg | ATTITUDE / ATTITUDE_QUATERNION yaw (VFR_HUD.heading KULLANILMIYOR); tamsayıya kırpılır (`:122`). HIGH_LATENCY/2 ile de yazılır | G; pusula: `src/FlightMap/Widgets/QGCCompassWidget.qml:23` | j21 / s`:126`, `src/Vehicle/Vehicle.cc:931, 983` |
| RollRate | deg/s | YALNIZ ATTITUDE_QUATERNION (ATTITUDE işleyicisi hız yazmaz) | G | j28 / s`:188` |
| PitchRate | deg/s | YALNIZ ATTITUDE_QUATERNION | G | j35 / s`:189` |
| YawRate | deg/s | YALNIZ ATTITUDE_QUATERNION | G | j42 / s`:190` |
| GroundSpeed | m/s (kullanıcı hız birimine dönüşür) | VFR_HUD.groundspeed (NaN ise 0); HIGH_LATENCY/2 | G; V (FW sütun 2, `QGCCorePlugin.cc:263`); pusula `QGCCompassWidget.qml:25` | j49 / s`:214`, `Vehicle.cc:929, 981` |
| AirSpeed | m/s | VFR_HUD.airspeed (NaN ise 0); HIGH_LATENCY/2 | G; V (FW, `QGCCorePlugin.cc:273`) | j56 / s`:213`, `Vehicle.cc:928, 980` |
| AirSpeedSetpoint | (JSON'da YOK) | NAV_CONTROLLER_OUTPUT: `airSpeed - aspd_error` | G; PX4 tuning sayfası (`src/AutoPilotPlugins/PX4/PX4TuningComponentPlaneTECS.qml:22`) | s`:202`. Metadata yok: shortDesc boş, birim boş (aşağıda not 1) |
| ClimbRate | m/s | VFR_HUD.climb; HIGH_LATENCY/2 | G; V (`QGCCorePlugin.cc:257`) | j63 / s`:215`, `Vehicle.cc:930, 982` |
| AltitudeRelative | vertical m (-> m/ft) | ALTITUDE.altitude_relative varsa o; yoksa GLOBAL_POSITION_INT.relative_alt/1000; HIGH_LATENCY/2'de NaN | G; V (`QGCCorePlugin.cc:242`, kart `:207`); `GuidedActionsController.qml:233, 612` | j70 / s`:154`, `Vehicle.cc:871, 932, 984`; öncelik `VehicleFactGroup.cc:152`, `Vehicle.cc:870-873` |
| AltitudeAMSL | vertical m | ALTITUDE.altitude_amsl; yoksa GLOBAL_POSITION_INT.alt/1000; yoksa GPS_RAW_INT.alt/1000 (3D fix ve GLOBAL_POSITION_INT yoksa); HIGH_LATENCY/2 | G; `EditPositionDialog.qml:122` | j77 / s`:155`, `Vehicle.cc:854, 872, 933, 985` |
| AltitudeAboveTerr | vertical m | MAVLink DEĞİL: konum değişince çevrimiçi arazi yükseklik sorgusu (AMSL - arazi). Hiç sorgu sonucu gelmezse güncellenmez | G; `EditPositionDialog.qml:136` | j84 / s`src/Vehicle/TerrainQueryCoordinator.cc:213-214`, tetik `Vehicle.cc:268-269`, kısıtlama `TerrainQueryCoordinator.cc:165-186` |
| AltitudeTuning | (JSON'da YOK) | VFR_HUD.alt - ilk örnek ofseti (ilk VFR_HUD'da 0'dan başlar) | G; PX4 TECS sayfası `PX4TuningComponentPlaneTECS.qml:23` | s`:218-220`; ofset sıfırlama `Vehicle.cc:2761-2764`. Metadata yok |
| AltitudeTuningSetpoint | (JSON'da YOK) | NAV_CONTROLLER_OUTPUT: `altitudeTuning - alt_error` | G; `PX4TuningComponentPlaneTECS.qml:24` | s`:200`. Metadata yok |
| XTrackError | (JSON'da YOK) | NAV_CONTROLLER_OUTPUT.xtrack_error | G | s`:201`. Metadata yok |
| RangeFinderDist | (JSON'da YOK) | RANGEFINDER (id 173) `distance` (NaN ise 0). DISTANCE_SENSOR'dan DEĞİL | G (QML'de başka kullanım bulunamadı) | s`:243`, `:100-102`. Metadata yok |
| FlightDistance | m | MAVLink DEĞİL: yörünge noktaları arası toplam mesafe (`TrajectoryPoints.cc:20` -> `Vehicle::updateFlightDistance`); uçuş başlayınca sıfırlanır | G; V (`QGCCorePlugin.cc:293`) | j91 / s`Vehicle.cc:1625, 2905` |
| FlightTime | elapsedSeconds (tip; birim yok, hh:mm:ss biçimi) | MAVLink DEĞİL: uçuş zamanlayıcısı (1 sn); uçuş başlangıcı `Vehicle.cc:1223` | G; V (`QGCCorePlugin.cc:287`, showUnits=false) | j159 / s`Vehicle.cc:1626, 1636` |
| DistanceToHome | m | MAVLink DEĞİL: `coordinate().distanceTo(homePosition())` (konum veya HOME_POSITION değişince) | G; V (`QGCCorePlugin.cc:248`) | j98 / s`Vehicle.cc:2616, 2625`; bağlantı `:233-235` |
| TimeToHome | s | VFR_HUD'dan türetilir: distanceToHome(cooked) / groundspeed | G | j105 / s`:221-223` (not 2) |
| MissionItemIndex | birim yok | MAVLink DEĞİL: `MissionManager::currentIndex` (+1 ofset gerekirse) | G | j140 / s`Vehicle.cc:2656`, bağlantı `:248-249` |
| HeadingToNextWP | deg | MAVLink DEĞİL: aracın konumundan sıradaki görev öğesine azimut (<5 m ise NaN) | G; pusula `QGCCompassWidget.qml:26` | j145 / s`Vehicle.cc:2640, 2643` |
| DistanceToNextWP | m | NAV_CONTROLLER_OUTPUT.wp_dist | G | j152 / s`:203` |
| HeadingToHome | deg | MAVLink DEĞİL: konumdan home'a azimut (home'a >1 m ise) | G | j112 / s`Vehicle.cc:2618, 2621, 2626` |
| HeadingFromHome | deg | MAVLink DEĞİL: home'dan araca azimut | G | j119 / s`Vehicle.cc:2619, 2622, 2627` |
| HeadingFromGCS | deg | MAVLink DEĞİL: GCS konumundan araca azimut (GCS konumu gerekir) | G | j126 / s`Vehicle.cc:2664, 2667` |
| DistanceToGCS | m | MAVLink DEĞİL: GCS konumu ile araç arası | G | j133 / s`Vehicle.cc:2663, 2666` |
| Hobbs | string | MAVLink DEĞİL: kalıcı hobbs sayacı | G | j165 / s`Vehicle.cc:2691`, sayaç `:2705-2715` |
| ThrottlePct | % | VFR_HUD.throttle (int16'ya çevrilir) | G; V (FW, `QGCCorePlugin.cc:278`) | j170 / s`:216` |
| ImuTemp | °C (dönüşmez) | RAW_IMU.temperature * 0.01 (0 ise 0) | G | j176 / s`:233` |
| RcRSSI | % | RC_CHANNELS.rssi, 0.9/0.1 alçak geçiren filtre, 255 = bilinmiyor | G; `src/Toolbar/RCRSSIIndicator.qml:18, 31, 57`. APM'de RSSI önceden ölçeklenir (aşağıda APM notu) | j182 / s`:57, 75`, çağrı `Vehicle.cc:1387` (`case` `:601`) |

Notlar:
1. Metadata'sız beş Fact (airSpeedSetpoint, altitudeTuning, altitudeTuningSetpoint, xTrackError, rangeFinderDist): `VehicleFact.json` içinde karşılığı yok (JSON'daki 27 ad ile `.h`'deki 32 ad karşılaştırıldı). `Fact` kurucusu boş bir `FactMetaData` yaratır (`src/FactSystem/Fact.cc:22-23`), bu yüzden seçilebilirler ama `shortDescription` ve birim boş gelir. Edit diyaloğu seçimde etiketi `fact.shortDescription` ile doldurur (`InstrumentValueEditDialog.qml:52, 68`), yani etiket boş kalır. Ondalık hane sayısı davranışı doğrulanmadı.
2. TimeToHome: `_distanceToHomeFact.cookedValue()` (kullanıcının mesafe birimi, örn. ft) m/s hıza bölünüyor (`VehicleFactGroup.cc:221-223`). Mesafe birimi metre dışındaysa sonucun yanlış çıkması kod okumasıyla öngörülüyor; çalıştırılarak doğrulanmadı.
3. ATTITUDE işleyicisi yalnız roll/pitch/heading yazar (`:129-143`); açısal hızlar yalnız ATTITUDE_QUATERNION ile dolar. ArduPlane'in bu mesajı gönderip göndermediği bu repodan doğrulanamaz (doğrulanmadı).
4. Grid'e eklenmeyen ama kod konumu (`coordinate()`) alan Vehicle özellikleri (lat/lon, `Vehicle::_handleGlobalPositionInt`, `Vehicle.cc:860-880`) Fact değildir; enlem/boylam için gps grubuna bakın.

---

#### 4. Grup: Gps (`gps`) ve Gps2 (`gps2`)

`src/Vehicle/FactGroups/VehicleGPSFactGroup.cc` (kayıt `:12-28`), metadata `src/Vehicle/FactGroups/GPSFact.json` (güncelleme 1000 ms, `:10`). Gps2 aynı sınıftan türer ve aynı JSON'u kullanır (`VehicleGPS2FactGroup.h:10-13`, GNSS_INTEGRITY id = 1; GPS grubunda id = 0, `VehicleGPSFactGroup.h:77`).

`handleMessage` case'leri (gps): GPS_RAW_INT `:51`, HIGH_LATENCY `:54`, HIGH_LATENCY2 `:57`, GNSS_INTEGRITY `:60`. Gps2: GPS2_RAW `VehicleGPS2FactGroup.cc:13`, GNSS_INTEGRITY `:16` (aynı `_handleGnssIntegrity`, `VehicleGPSFactGroup.cc:114-133`).

| Değer | Birim | MAVLink (gps / gps2) | Ekranda nerede | dosya:satır (gps; gps2 setter `VehicleGPS2FactGroup.cc`) |
|---|---|---|---|---|
| Lat | birim yok (JSON'da yok; ham derece) | GPS_RAW_INT.lat*1e-7 / GPS2_RAW; HIGH_LATENCY/2 | G | j7 / s`:73, 91, 104` (gps2 `:29`) |
| Lon | birim yok | GPS_RAW_INT / GPS2_RAW; HIGH_LATENCY/2 | G | j13 / s`:74, 92, 105` (gps2 `:30`) |
| Mgrs | string | lat/lon'dan MGRS dönüşümü | G | j19 / s`:75, 93, 106` (gps2 `:31`) |
| Hdop | birim yok | GPS_RAW_INT.eph/100; HIGH_LATENCY2.eph/10 | G; araç çubuğu GPS göstergesi `src/Toolbar/GPSIndicator.qml:56, 68` | j24 / s`:77, 108` (gps2 `:33`) |
| Vdop | birim yok | GPS_RAW_INT.epv/100 | G | j30 / s`:78, 109` (gps2 `:34`) |
| CourseOverGround | deg | GPS_RAW_INT.cog/100 | G | j36 / s`:79` (gps2 `:35`) |
| Yaw | deg | GPS_RAW_INT.yaw/100 | G | j43 / s`:80` (gps2 `:36`) |
| Lock | enum (None, No Fix, 2D, 3D, 3D DGPS, RTK float, RTK fixed, Static) | GPS_RAW_INT.fix_type / GPS2_RAW.fix_type | G (enum metni gösterir); `GPSIndicatorPage.qml:97`, `PreFlightGPSCheck.qml:17` | j50 / s`:81` (gps2 `:37`) |
| Count | birim yok (uydu sayısı) | GPS_RAW_INT.satellites_visible (255 -> 0); HIGH_LATENCY/2'de 0 | G; `GPSIndicator.qml:48, 62` | j58 / s`:76, 94, 107` (gps2 `:32`) |
| SystemErrors | birim yok | GNSS_INTEGRITY.system_errors (development dialect) | G | j63 / s`:123` |
| SpoofingState | enum (Unknown, Not spoofed, Mitigated, Ongoing) | GNSS_INTEGRITY | G; `src/Toolbar/GPSResilienceIndicator.qml:129-165` | j68 / s`:124` |
| JammingState | enum | GNSS_INTEGRITY | G; aynı gösterge | j76 / s`:125` |
| AuthenticationState | enum (Unknown, Initializing, Error, Ok, Disabled) | GNSS_INTEGRITY | G; aynı gösterge | j84 / s`:126` |
| CorrectionsQuality | birim yok | GNSS_INTEGRITY | G | j92 / s`:127` |
| SystemQuality | birim yok | GNSS_INTEGRITY.system_status_summary | G | j97 / s`:128` |
| GnssSignalQuality | birim yok | GNSS_INTEGRITY | G | j102 / s`:129` |
| PostProcessingQuality | birim yok | GNSS_INTEGRITY | G | j107 / s`:130` |

Not: GNSS_INTEGRITY yan başlığı `development/mavlink_msg_gnss_integrity.h` altından geliyor (`VehicleGPSFactGroup.cc:5`); ArduPlane'in bu mesajı gönderip göndermediği doğrulanmadı. Gps2 grubunda GNSS_INTEGRITY dışı alanlar da listede görünür (gps ile aynı 17 ad).

#### 5. Grup: GpsAggregate (`gpsAggregate`)

`src/Vehicle/FactGroups/VehicleGPSAggregateFactGroup.cc` (kayıt `:18-21`), JSON olarak `GPSFact.json` verilir (`:16`). MAVLink'i doğrudan işlemez; gps ve gps2'nin GNSS_INTEGRITY sonuçlarını birleştirir (`bindToGps` `:33-46`, birleştirme `:116-132`). 5 sn bayatlama zamanlayıcısı (`VehicleGPSAggregateFactGroup.h:52`, `.cc:60-66`).

| Değer | Birim | Kaynak | Ekranda | dosya:satır |
|---|---|---|---|---|
| SpoofingState | enum | gps/gps2 en kötü değer | G; `GPSResilienceIndicator.qml:26` (`_gpsAggregate`) | j68 / s`.cc:129` |
| JammingState | enum | gps/gps2 en kötü değer | G | j76 / s`.cc:130` |
| AuthenticationState | enum | öncelik birleşimi | G | j84 / s`.cc:131` |
| IsStale | bool (JSON'da YOK) | 5 sn GNSS_INTEGRITY gelmezse true | G | `.h:67`, s`.cc:55, 65`. Metadata yok (not 1 ile aynı sonuç) |

---

#### 6. Grup: Wind (`wind`)

`src/Vehicle/FactGroups/VehicleWindFactGroup.cc`, metadata `WindFact.json`, `case`'ler: WIND_COV `:23`, HIGH_LATENCY `:26`, HIGH_LATENCY2 `:29`, WIND `:32`. QML'de başka kullanım bulunamadı (yalnız grid).

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| Direction | deg | WIND.direction; WIND_COV (atan2(y,x)); HIGH_LATENCY2.wind_heading*2 | G | j7 / s`:55, 70, 90` |
| Speed | m/s | WIND.speed; WIND_COV (sqrt(x²+y²)); HIGH_LATENCY2.windspeed/5. HIGH_LATENCY'de hava hızı (airspeed/5) buraya yazılıyor | G | j14 / s`:45, 56, 73, 91` |
| VerticalSpeed | m/s | WIND.speed_z; WIND_COV.wind_z | G | j21 / s`:75, 92` |

ArduPlane'in WIND veya WIND_COV gönderip göndermediği doğrulanmadı.

#### 7. Grup: Vibration (`vibration`)

`src/Vehicle/FactGroups/VehicleVibrationFactGroup.cc`, `VibrationFact.json`, VIBRATION için tek koşul `:23`.

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| XAxis / YAxis / ZAxis | birim yok | VIBRATION.vibration_x/y/z | G; Analyze > Vibration `src/AnalyzeView/Vibration/VibrationPage.qml:17-23` | j7 / j13 / j19; s`:30, 31, 32` |
| ClipCount1 / 2 / 3 | birim yok | VIBRATION.clipping_0/1/2 | G; `VibrationPage.qml:189-193` | j25 / j30 / j35; s`:33, 34, 35` |

#### 8. Grup: Temperature (`temperature`)

`src/Vehicle/FactGroups/VehicleTemperatureFactGroup.cc`, `TemperatureFact.json`. `case`'ler: SCALED_PRESSURE `:21`, SCALED_PRESSURE2 `:24`, SCALED_PRESSURE3 `:27`, HIGH_LATENCY `:30`, HIGH_LATENCY2 `:33`. QML'de grid dışı kullanım yok (araç `BatteryIndicator.qml:362` kendi batarya sıcaklığını okur, bu grup değil).

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| Temperature1 | C (-> F ayarı) | SCALED_PRESSURE.temperature/100 (baro); HIGH_LATENCY/2.temperature_air | G | j7 / s`:46, 56, 66` |
| Temperature2 | C | SCALED_PRESSURE2.temperature/100 | G | j14 / s`:76` |
| Temperature3 | C | SCALED_PRESSURE3.temperature/100 | G | j21 / s`:86` |

Not: Kod yalnız SCALED_PRESSURE*.temperature alanını /100 ile okuyor; bu sıcaklığın fiziksel olarak neyi ölçtüğü (baro çipi mi, hava mı) bu repodan doğrulanamaz.

#### 9. Grup: Clock (`clock`)

`src/Vehicle/FactGroups/VehicleClockFactGroup.cc`, `ClockFact.json`. MAVLink DEĞİL: `_updateAllValues` her 1 sn'de bilgisayar (GCS) saatini yazar (`:16-25`).

| Değer | Birim | Kaynak | Ekranda | dosya:satır |
|---|---|---|---|---|
| CurrentTime | string | GCS yerel saati | G | j7 / s`:18` |
| CurrentUTCTime | string | GCS UTC saati | G | j12 / s`:19` |
| CurrentDate | string | GCS tarihi (dil kısa biçimi) | G | j17 / s`:20` |

#### 10. Grup: Setpoint (`setpoint`)

`src/Vehicle/FactGroups/VehicleSetpointFactGroup.cc`, `SetpointFact.json`, tek mesaj ATTITUDE_TARGET (`:28`). Ekranda: APM çok-rotor tuning sayfası `src/AutoPilotPlugins/APM/APMAdvancedTuningCopterComponent.qml:30, 72, 114` (hız setpoint'leri) ve PX4 tuning sayfaları.

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| Roll | deg | ATTITUDE_TARGET.q -> euler | G | j7 / s`:38` |
| Pitch | deg | ATTITUDE_TARGET | G | j14 / s`:39` |
| Yaw | deg | ATTITUDE_TARGET (0..360'a getirilir `:40-43`) | G | j21 / s`:43` |
| RollRate | deg/s | ATTITUDE_TARGET.body_roll_rate | G | j28 / s`:45` |
| PitchRate | deg/s | ATTITUDE_TARGET.body_pitch_rate | G | j35 / s`:46` |
| YawRate | deg/s | ATTITUDE_TARGET.body_yaw_rate | G | j42 / s`:47` |

ArduPlane'in ATTITUDE_TARGET gönderip göndermediği doğrulanmadı.

#### 11. Grup: DistanceSensor (`distanceSensor`)

`src/Vehicle/FactGroups/VehicleDistanceSensorFactGroup.cc`, `DistanceSensorFact.json`, tek mesaj DISTANCE_SENSOR (`:25`). `current_distance` cm -> m (`:52`), mesajın `orientation` alanına göre yön Fact'ine yazılır (`:37-55`). Ekranda: proximity radar `src/FlyView/ProximityRadarValues.qml:7, 33`.

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| RotationNone (Forward) | m | DISTANCE_SENSOR, orientation = ROTATION_NONE | G; radar | j7 / s`:52` (eşleme `:38`) |
| RotationYaw45 (Forward/Right) | m | orientation = YAW_45 | G; radar | j15 / `:39` |
| RotationYaw90 (Right) | m | YAW_90 | G; radar | j23 / `:40` |
| RotationYaw135 (Rear/Right) | m | YAW_135 | G; radar | j31 / `:41` |
| RotationYaw180 (Rear) | m | YAW_180 | G; radar | j39 / `:42` |
| RotationYaw225 (Rear/Left) | m | YAW_225 | G; radar | j47 / `:43` |
| RotationYaw270 (Left) | m | YAW_270 | G; radar | j55 / `:44` |
| RotationYaw315 (Forward/Left) | m | YAW_315 | G; radar | j63 / `:45` |
| RotationPitch90 (Up) | m | PITCH_90 | G; radar | j71 / `:46` |
| RotationPitch270 (Down) | m | PITCH_270 (JSON etiketi "Down") | G; radar | j79 / `:47` |
| MinDistance | m | DISTANCE_SENSOR.min_distance/100 | G | j87 / s`:57` |
| MaxDistance | m | DISTANCE_SENSOR.max_distance/100 | G | j95 / s`:58` |

Önemli: Yönü bu 10 değerden biri olmayan sensör (örn. farklı rotasyon enumu) yalnız min/max'ı günceller, mesafe Fact'i güncellenmez (`:50-55`). Ayrıca Vehicle grubundaki `RangeFinderDist` ayrı bir mesajdan (RANGEFINDER) beslenir; ikisi birbirinin kopyası değildir. WIG yükseklik sensörünün hangi mesajı/yönü gönderdiği bu repodan doğrulanamaz (doğrulanmadı).

#### 12. Grup: LocalPosition (`localPosition`) ve LocalPositionSetpoint (`localPositionSetpoint`)

LocalPosition: `VehicleLocalPositionFactGroup.cc`, tek mesaj LOCAL_POSITION_NED (`:26`). LocalPositionSetpoint: `VehicleLocalPositionSetpointFactGroup.cc`, tek mesaj POSITION_TARGET_LOCAL_NED (`:26`). İkisi de aynı JSON'u kullanır: setpoint grubu `LocalPositionSetpointFact.json` ister (`.cc:5`) ama bu dosya yoktur; `LocalPositionFact.json` CMake ile bu ad takma adıyla paketlenir (`src/Vehicle/FactGroups/CMakeLists.txt:78-82`). Ekranda: yalnız PX4 tuning sayfaları (`src/AutoPilotPlugins/PX4/PX4TuningComponentCopterPosition.qml:37-55` vb.).

| Değer | Birim | MAVLink (localPosition / localPositionSetpoint) | Ekranda | dosya:satır (JSON `LocalPositionFact.json`) |
|---|---|---|---|---|
| X | m | LOCAL_POSITION_NED.x / POSITION_TARGET_LOCAL_NED.x | G | j7 / s`LocalPosition.cc:33`, `Setpoint.cc:33` |
| Y | m | .y | G | j14 / s`:34`, `:34` |
| Z | vertical m | .z (işaret çevrilmez, olduğu gibi kopyalanır) | G | j21 / s`:35`, `:35` |
| VX | m/s | .vx | G | j28 / s`:37`, `:37` |
| Vy | m/s | .vy | G | j35 / s`:38`, `:38` |
| Vz | m/s | .vz | G | j42 / s`:39`, `:39` |

Not: `Vy`/`Vz` adları JSON'da küçük harfli yazılmış (`shortDesc` "Vy", "Vz"); kayıt adları `vy`, `vz`, UI'da "Vy", "Vz" olarak görünür. ArduPlane'in LOCAL_POSITION_NED / POSITION_TARGET_LOCAL_NED gönderip göndermediği doğrulanmadı.

#### 13. Grup: EstimatorStatus (`estimatorStatus`)

`src/Vehicle/FactGroups/VehicleEstimatorStatusFactGroup.cc` (500 ms güncelleme, `:5`), `EstimatorStatusFactGroup.json`, tek mesaj ESTIMATOR_STATUS (`:33`). QML'de grid dışı kullanım bulunamadı. Bayraklar (`flags & ESTIMATOR_*`) bool Fact olur, oranlar doğrudan kopyalanır. Fact adı `goodAttitudeEsimate` yazım hatalıdır ama JSON ve `.h` (`:57`) ile tutarlıdır.

| Değer | Birim | MAVLink alanı | Ekranda | dosya:satır |
|---|---|---|---|---|
| GoodAttitudeEsimate | bool | flags & ESTIMATOR_ATTITUDE | G | j7 / s`:40` |
| GoodHorizVelEstimate | bool | ESTIMATOR_VELOCITY_HORIZ | G | j13 / s`:41` |
| GoodVertVelEstimate | bool | ESTIMATOR_VELOCITY_VERT | G | j19 / s`:42` |
| GoodHorizPosRelEstimate | bool | ESTIMATOR_POS_HORIZ_REL | G | j25 / s`:43` |
| GoodHorizPosAbsEstimate | bool | ESTIMATOR_POS_HORIZ_ABS | G | j31 / s`:44` |
| GoodVertPosAbsEstimate | bool | ESTIMATOR_POS_VERT_ABS | G | j37 / s`:45` |
| GoodVertPosAGLEstimate | bool | ESTIMATOR_POS_VERT_AGL | G | j43 / s`:46` |
| GoodConstPosModeEstimate | bool | ESTIMATOR_CONST_POS_MODE | G | j49 / s`:47` |
| GoodPredHorizPosRelEstimate | bool | ESTIMATOR_PRED_POS_HORIZ_REL | G | j55 / s`:48` |
| GoodPredHorizPosAbsEstimate | bool | ESTIMATOR_PRED_POS_HORIZ_ABS | G | j61 / s`:49` |
| GpsGlitch | bool | ESTIMATOR_GPS_GLITCH | G | j67 / s`:50` |
| AccelError | bool | ESTIMATOR_ACCEL_ERROR | G | j73 / s`:51` |
| VelRatio | birim yok | vel_ratio | G | j79 / s`:52` |
| HorizPosRatio | birim yok | pos_horiz_ratio | G | j86 / s`:53` |
| VertPosRatio | birim yok | pos_vert_ratio | G | j93 / s`:54` |
| MagRatio | birim yok | mag_ratio | G | j100 / s`:55` |
| HaglRatio | birim yok | hagl_ratio | G | j107 / s`:56` |
| TasRatio | birim yok | tas_ratio | G | j114 / s`:57` |
| HorizPosAccuracy | birim yok (JSON'da birim yok; MAVLink tarafında m) | pos_horiz_accuracy | G | j121 / s`:58` |
| VertPosAccuracy | birim yok | pos_vert_accuracy | G | j128 / s`:59` |

ArduPlane'in ESTIMATOR_STATUS gönderip göndermediği doğrulanmadı.

#### 14. Grup: Terrain (`terrain`)

`src/Vehicle/FactGroups/TerrainFactGroup.cc` (kayıt `:6-7`), `TerrainFactGroup.json`. Grup kendi mesajını işlemez; `TerrainProtocolHandler` TERRAIN_REPORT'u (`case` `src/Vehicle/TerrainProtocolHandler.cc:37`) yazar. Ekranda: `src/FlyView/TerrainProgress.qml:16-18`.

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| BlocksPending | birim yok | TERRAIN_REPORT.pending | G; TerrainProgress | j7 / s`TerrainProtocolHandler.cc:67` |
| BlocksLoaded | birim yok | TERRAIN_REPORT.loaded | G; TerrainProgress | j14 / s`:68` |

#### 15. Grup: Hygrometer (`hygrometer`)

`src/Vehicle/FactGroups/VehicleHygrometerFactGroup.cc`, `HygrometerFact.json`, HYGROMETER_SENSOR (`:21`). QML'de grid dışı kullanım yok.

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| Temperature | deg (dönüşmez) | HYGROMETER_SENSOR.temperature/100 | G | j7 / s`:34` |
| Humidity | % | HYGROMETER_SENSOR.humidity | G | j14 / s`:35` |
| Hygrometerid | birim yok | HYGROMETER_SENSOR.id (tek grup; birden çok sensör aynı Fact'e yazar) | G | j21 / s`:36` |

#### 16. Grup: Generator (`generator`)

`src/Vehicle/FactGroups/VehicleGeneratorFactGroup.cc`, `GeneratorFact.json`, GENERATOR_STATUS (`:39`; hepsi `:52-62`). 0xFFFF/INT_MAX "bilinmiyor" değerleri NaN yapılır. QML'de grid dışı kullanım yok.

| Değer | Birim | MAVLink alanı | Ekranda | dosya:satır |
|---|---|---|---|---|
| Status | birim yok (bayrak maskesi) | status | G | j7 / s`:52` |
| GenSpeed | rpm | generator_speed | G | j12 / s`:53` |
| BatteryCurrent | A | battery_current | G | j18 / s`:54` |
| LoadCurrent | A | load_current | G | j25 / s`:55` |
| PowerGenerated | W | power_generated | G | j32 / s`:56` |
| BusVoltage | V | bus_voltage | G | j39 / s`:57` |
| RectifierTemp | °C (dönüşmez) | rectifier_temperature | G | j46 / s`:58` |
| BatCurrentSetpoint | A | bat_current_setpoint | G | j52 / s`:59` |
| GenTemp | °C (dönüşmez) | generator_temperature | G | j59 / s`:60` |
| Runtime | sec | runtime | G | j65 / s`:61` |
| TimeMaintenance | sec | time_until_maintenance | G | j71 / s`:62` |

#### 17. Grup: Efi (`efi`) — içten yanmalı motor / ECU

`src/Vehicle/FactGroups/VehicleEFIFactGroup.cc`, `EFIFact.json`, EFI_STATUS (`:53`; hepsi `:66-84`). QML'de grid dışı kullanım yok. ArduPlane'in EFI_STATUS göndermesi için EFI sürücüsü gerektiği kod okumasından çıkarılamaz (doğrulanmadı).

| Değer | Birim | MAVLink alanı | Ekranda | dosya:satır |
|---|---|---|---|---|
| Health | birim yok | health (INT8_MAX -> NaN) | G | j7 / s`:66` |
| EcuIndex | A (JSON'da "A" yazıyor; alan adıyla uyumsuz görünüyor) | ecu_index | G | j12 / s`:67` |
| Rpm | birim yok | rpm | G | j19 / s`:68` |
| FuelConsumed | cm^3 | fuel_consumed | G | j25 / s`:69` |
| FuelFlow | cm^3/min | fuel_flow | G | j32 / s`:70` |
| EngineLoad | % | engine_load | G | j39 / s`:71` |
| ThrottlePos | % | throttle_position | G | j46 / s`:72` |
| SparkTime | ms | spark_dwell_time | G | j53 / s`:73` |
| BaroPress | kPa | barometric_pressure | G | j60 / s`:74` |
| IntakePress | kPa | intake_manifold_pressure | G | j67 / s`:75` |
| IntakeTemp | °C (dönüşmez) | intake_manifold_temperature | G | j74 / s`:76` |
| CylinderTemp | °C (dönüşmez) | cylinder_head_temperature | G | j81 / s`:77` |
| IgnTime | deg | ignition_timing | G | j88 / s`:78` |
| InjTime | ms | injection_time | G | j95 / s`:79` |
| ExGasTemp | °C (dönüşmez) | exhaust_gas_temperature | G | j102 / s`:80` |
| ThrottleOut | % | throttle_out | G | j109 / s`:81` |
| PtComp | birim yok | pt_compensation | G | j116 / s`:82` |
| IgnVoltage | V | ignition_voltage | G | j122 / s`:83` |
| FuelPressure | kPa | fuel_pressure | G | j129 / s`:84` |

#### 18. Grup: Rpm (`rpm`)

`src/Vehicle/FactGroups/VehicleRPMFactGroup.cc`, `RPMFact.json`. `case`'ler: RAW_RPM `:27`, RPM `:30`. Dikkat: `Vehicle.h`'de `rpm` için Q_PROPERTY yok (`Vehicle.h:237-253`), yalnız C++ `rpmFactGroup()` (`Vehicle.h:566`); grid seçicisi `getFactGroup` ile ulaşır.

| Değer | Birim | MAVLink | Ekranda | dosya:satır |
|---|---|---|---|---|
| Rpm1 | rpm | RAW_RPM, index 0, `frequency` | G | j7 / s`:44` |
| Rpm2 | rpm | RAW_RPM, index 1 | G | j14 / s`:47` |
| Rpm3 | rpm | RAW_RPM, index 2 | G | j21 / s`:50` |
| Rpm4 | rpm | RAW_RPM, index 3 | G | j28 / s`:53` |
| RpmSensor1 | rpm | RPM.rpm1 | G | j35 / s`:67` |
| RpmSensor2 | rpm | RPM.rpm2 | G | j42 / s`:68` |

Not: RAW_RPM.frequency doğrudan "rpm" birimli Fact'e yazılıyor (`:44`); alan tanımı bu repoda olmadığı için frekans/rpm dönüşümü doğrulanmadı. ArduPlane'in bu mesajları göndermesi doğrulanmadı.

#### 19. Grup: RadioStatus (`radioStatus`)

`src/Vehicle/FactGroups/RadioStatusFactGroup.cc`, `RadioStatusFact.json`, RADIO_STATUS (`:22`; hepsi `:50-56`). 3DR Si1k telemetri radyosunda (sysid '3', compid 'D') RSSI ham registerdan dBm'e çevrilir (`:42-44`). Ekranda: `src/Toolbar/TelemetryRSSIIndicator.qml:19`.

| Değer | Birim | MAVLink alanı | Ekranda | dosya:satır |
|---|---|---|---|---|
| Lrssi | dBm | rssi | G; araç çubuğu | j7 / s`:50` |
| Rrssi | dBm | remrssi | G; araç çubuğu | j13 / s`:51` |
| RxErrors | birim yok | rxerrors | G | j19 / s`:52` |
| Fixed | birim yok | fixed | G | j24 / s`:53` |
| TxBuffer | % | txbuf | G | j29 / s`:54` |
| LNoise | dBm | noise | G | j35 / s`:55` |
| RNoise | dBm | remnoise | G | j41 / s`:56` |

#### 20. Dinamik grup: Battery0, Battery1, ... (`battery<id>`)

`src/Vehicle/FactGroups/BatteryFactGroupListModel.cc` (Fact kaydı `:39-49`, güncelleme 1000 ms `:37`), `BatteryFact.json`. Grup oluşturma: BATTERY_STATUS (id = `batteryStatus.id`, `:19-25`), HIGH_LATENCY/2 (id 0, `:15-18`). Grup işleme `case`'leri `:69, 72, 75`. Başlangıçta değerler NaN (`:54-61`). Araç çubuğu bataryası `src/Toolbar/BatteryIndicator.qml:17-37` (APM'in kendi göstergesi `src/FirmwarePlugin/APM/APMBatteryIndicator.qml`).

| Değer | Birim | MAVLink alanı | Ekranda | dosya:satır |
|---|---|---|---|---|
| Id | birim yok | BATTERY_STATUS.id (kurucuda `:51`) | G | j7 (JSON) / `FactGroupWithId.cc:6` |
| BatteryFunction | enum (n/a, All Flight Systems, Propulsion, Avionics, Payload) | battery_function | G (enum metni) | j12 / s`:134` |
| BatteryType | enum (n/a, LIPO, LIFE, LION, NIMH) | type | G | j20 / s`:135` |
| Voltage | v (JSON'da küçük "v") | voltages[] hücre toplamı (mV/1000, `voltages_ext` dahil) | G; araç çubuğu | j28 / s`:137` |
| PercentRemaining | % | battery_remaining; HIGH_LATENCY.battery_remaining; HIGH_LATENCY2.battery | G; araç çubuğu | j35 / s`:88, 98, 140` |
| MahConsumed | mAh | current_consumed | G | j42 / s`:139` |
| Current | A | current_battery/100 | G | j49 / s`:138` |
| Temperature | C (-> F ayarı) | temperature/100 (INT16_MAX -> NaN); araç çubuğu `BatteryIndicator.qml:362, 419` | G | j56 / s`:136` |
| InstantPower | W | voltage * current (türetilmiş) | G | j63 / s`:143` |
| TimeRemaining | s | time_remaining (0 -> NaN) | G | j70 / s`:141` |
| TimeRemainingStr | string | timeRemaining'den türetilir (hh:mm:ss) | G | j77 / s`:151, 158` |
| ChargeState | enum (n/a, Ok, Low, Critical, Emergency, Failed, Unhealthy, Charging) | charge_state | G | j82 / s`:142` |

#### 21. Dinamik grup: EscStatus0, EscStatus1, ... (`escStatus<index>`)

`src/Vehicle/FactGroups/EscStatusFactGroupListModel.cc` (Fact kaydı `:57-65`), `EscStatusFactGroup.json`. Grup oluşturma `_shouldHandleMessage` `:11-47`; işleme `case`'leri `:82` (ESC_INFO), `:85` (ESC_STATUS). ESC göstergesi `src/Toolbar/EscIndicatorPage.qml:83, 88`. Kod gözlemi: `_shouldHandleMessage` içinde ESC_INFO için de `mavlink_esc_status_t` ile çözüm yapılıyor (`:19-26`); ESC_INFO alan düzeninin buna uyup uymadığı doğrulanmadı. ArduPlane'de ESC telemetrisi yoksa bu gruplar hiç oluşmaz.

| Değer | Birim | MAVLink alanı | Ekranda | dosya:satır |
|---|---|---|---|---|
| Id | birim yok | ESC index | G | j7 / s`:67` |
| Rpm | birim yok | ESC_STATUS.rpm[idx] | G; `EscIndicatorPage.qml:83` | j12 / s`:129` |
| Current | A | ESC_STATUS.current[idx] | G | j19 / s`:130` |
| Voltage | V | ESC_STATUS.voltage[idx] | G | j28 / s`:131` |
| Count | birim yok | ESC_INFO.count | G | j36 / s`:106` |
| ConnectionType | enum (PPM, Serial Bus, One Shot, I2C, CAN-Bus, DShot) | ESC_INFO.connection_type | G | j42 / s`:107` |
| Info | birim yok | ESC_INFO.info | G | j50 / s`:108` |
| FailureFlags | bitmask | ESC_INFO.failure_flags[idx] | G | j56 / s`:109` |
| ErrorCount | birim yok | ESC_INFO.error_count[idx] | G | j100 / s`:110` |
| Temperature | centi-celsius (-> C) | ESC_INFO.temperature[idx] | G; `EscIndicatorPage.qml:88` | j92 / s`:111` |

#### 22. Dinamik gruplar: Gimbal ve Camera (kısa)

- `gimbal<managerCompid><deviceId>` (`src/Gimbal/GimbalController.cc:367`): `src/Gimbal/GimbalFact.json` metadata (`Gimbal.cc:8`), Fact'ler `Gimbal.cc:54-59` (gimbalRoll, gimbalPitch, gimbalYaw, gimbalAzimuth: deg; deviceId; managerCompid JSON'da YOK). Besleyen: GIMBAL_DEVICE_ATTITUDE_STATUS (`GimbalController.cc:71`) ve manager mesajları (`:65, :68`). Yorum (`:366`): "yeni gimbal telemetrisi fly view panelinde seçilebilsin diye" grup eklenir. WIG'de gimbal yoksa görünmez.
- `camera` (`src/Camera/VehicleCameraControl.cc:1168`): kamera parametre Fact'leri; kamera tanımı yüklenmezse yok. Ayrıntı bu analiz kapsamı dışı (doğrulanmadı).
- `apmSubInfo`: yalnız ArduSub (`ArduSubFirmwarePlugin.cc:192`; `SubmarineFact.json`: cameraTilt, tetherTurns, lights1/2, pilotGain, inputHold, rangefinderDistance, rangefinderTarget, rollPitchToggle). ArduPlane'e uygulanmaz.

---

#### 23. APM / ArduPlane'e özgü davranışlar (bu listeyi etkileyenler)

1. APM'e özel FactGroup yok (ArduSub hariç, yukarıda). Grid listesi firmware'e göre değişmez; yalnız hangi Fact'lerin dolduğu araçtan gelen mesajlara bağlıdır.
2. RC RSSI: APM RC_CHANNELS.rssi değerini (0-254) QGC'nin beklediği 0-100'e çevirip mesajı yeniden kodlar (`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:1126-1145`; yönlendirme `:304-312`); sonra `Vehicle::_handleRCChannels` -> `updateRCRSSI` (`Vehicle.cc:1387`). Dönüşüm yalnız ArduPilot bileşeninden gelen mesajlara uygulanır (`:301-302`).
3. Akış hızları: bağlanınca APM için `MAV_DATA_STREAM_*` istekleri gönderilir (varsayılan: RAW_SENSORS 2, EXTENDED_STATUS 2, RC_CHANNELS 2, POSITION 3, EXTRA1 10, EXTRA2 10, EXTRA3 3 Hz; `src/Settings/APMMavlinkStreamRate.SettingsGroup.json:16-76`) ve `apmStartMavlinkStreams` varsayılan true (`src/Settings/Mavlink.SettingsGroup.json:24-31`). Kodu: `APMFirmwarePlugin.cc:385-421`. Ayrıca yalnız HOME_POSITION ve EXTENDED_SYS_STATE için 1 sn aralık istenir (`:429-433`). Hangi akışın hangi MAVLink mesajını taşıdığı ArduPilot firmware bilgisidir; bu repodan doğrulanamaz (doğrulanmadı). Bu nedenle "ArduPlane X mesajını gönderir mi" soruları yukarıda "doğrulanmadı" bırakıldı.
4. VTOL/Plane sınıfı: Varsayılan gridde AirSpeed/ThrottlePct yalnız `vehicleClass()` FixedWing/VTOL/Airship ise gelir (`QGCCorePlugin.cc:195, 226`, sınıf eşlemesi `src/MAVLink/QGCMAVLink.cc:218-221`). WIG aracın heartbeat'inde hangi MAV_TYPE'ı bildirdiği bu repodan doğrulanamaz (doğrulanmadı); MAV_TYPE_FIXED_WING ise FW varsayılanları uygulanır.
5. İki farklı rangefinder yolu: Vehicle/RangeFinderDist (RANGEFINDER mesajı, `VehicleFactGroup.cc:100-102, 243`) ve DistanceSensor/RotationPitch270 vb. (DISTANCE_SENSOR mesajı, `VehicleDistanceSensorFactGroup.cc:25`). Biri dolu, diğeri boş olabilir. Yükseklik hesabı yapan "AltitudeAboveTerr" ise bu iki sensörden hiçbirini kullanmaz; çevrimiçi arazi sorgusuna dayanır (`TerrainQueryCoordinator.cc:165-214`).

#### 24. Özet: eklenebilir toplam değer sayısı (kodla sayıldı; header ve JSON karşılaştırmasıyla)

| Grup | Fact sayısı |
|---|---|
| Vehicle | 32 |
| Gps | 17 |
| Gps2 | 17 |
| GpsAggregate | 4 |
| Wind | 3 |
| Vibration | 6 |
| Temperature | 3 |
| Clock | 3 |
| Setpoint | 6 |
| DistanceSensor | 12 |
| LocalPosition | 6 |
| LocalPositionSetpoint | 6 |
| EstimatorStatus | 20 |
| Hygrometer | 3 |
| Generator | 11 |
| Efi | 19 |
| Rpm | 6 |
| Terrain | 2 |
| RadioStatus | 7 |
| Battery<id> (her batarya) | 11 `_addFact` + 1 `id` = 12 |
| EscStatus<index> (her ESC) | 9 `_addFact` + 1 `id` = 10 |
| Gimbal (varsa) | 6 |
| **Sabit gruplar toplamı (Vehicle..RadioStatus)** | **183** |

Toplam, her Fact'in bir kez sayılmasıyla elde edildi (`Vehicle` 32 + gps 17 + gps2 17 + gpsAggregate 4 + wind 3 + vibration 6 + temperature 3 + clock 3 + setpoint 6 + distanceSensor 12 + localPosition 6 + localPositionSetpoint 6 + estimatorStatus 20 + hygrometer 3 + generator 11 + efi 19 + rpm 6 + terrain 2 + radioStatus 7). Batarya, ESC ve gimbal grupları araç mesaj yayınlayınca dinamik açıldığı için ayrıca sayılır.

Doğrulanmayanlar (tekrar): ArduPlane'in hangi MAVLink mesajlarını (ATTITUDE_QUATERNION, WIND, ESTIMATOR_STATUS, DISTANCE_SENSOR, RANGEFINDER, EFI_STATUS, RPM vb.) gerçekten yayınladığı; WIG aracın MAV_TYPE'ı; metadata'sız Fact'lerin ondalık hane davranışı; TimeToHome birim hatasının çalışma anındaki etkisi.


## B. ArduPilot'a özel davranış: uçuş modları

Kaynak: QGroundControl v5.1.5 (3a67d31f0c36bf3fe38ec52970d250a89d0aaf67). Tüm yollar repo köküne göredir. Plane plugin'i yalnızca `MAV_TYPE_FIXED_WING` ve VTOL tiplerinde seçilir (`src/FirmwarePlugin/APM/APMFirmwarePluginFactory.cc:43-54`), yani WIG aracı `MAV_TYPE_FIXED_WING` ile bağlanırsa `ArduPlaneFirmwarePlugin` kullanılır.

#### B1. ArduPlane'in tanıdığı modlar

Üç yerde tanımlı: enum `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.h:7-35`, isim sabitleri `ArduPlaneFirmwarePlugin.h:58-83`, isim tablosu `ArduPlaneFirmwarePlugin.cc:10-38`, seçilebilirlik listesi `ArduPlaneFirmwarePlugin.cc:40-68`. "Seçilebilir" sütunu `canBeSet` alanıdır; yalnızca `canBeSet=true` olanlar dropdown'a girer (`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:83-93`).

| # | QGC'de görünen ad | Seçilebilir | enum (.h) | isim sabiti (.h) | seçilebilirlik satırı (.cc) |
|---|---|---|---|---|---|
| 0 | Manual | evet | :8 | :58 | :42 |
| 1 | Circle | evet | :9 | :59 | :43 |
| 2 | Stabilize | evet | :10 | :60 | :44 |
| 3 | Training | evet | :11 | :61 | :45 |
| 4 | Acro | evet | :12 | :62 | :46 |
| 5 | FBW A | evet | :13 | :63 | :47 |
| 6 | FBW B | evet | :14 | :64 | :48 |
| 7 | Cruise | evet | :15 | :65 | :49 |
| 8 | Autotune | evet | :16 | :66 | :50 |
| 9 | (RESERVED_9, tabloda yok, adı yok) | hayır | :17 | - | - |
| 10 | Auto | evet | :18 | :67 | :51 |
| 11 | RTL | evet | :19 | :68 | :52 |
| 12 | Loiter | evet | :20 | :69 | :53 |
| 13 | Takeoff | evet | :21 | :70 | :54 |
| 14 | Avoid ADSB | evet | :22 | :71 | :55 |
| 15 | Guided | evet | :23 | :72 | :56 |
| 16 | Initializing | hayır (`false`) | :24 | :73 | :57 |
| 17 | QuadPlane Stabilize | evet | :25 | :74 | :58 |
| 18 | QuadPlane Hover | evet | :26 | :75 | :59 |
| 19 | QuadPlane Loiter | evet | :27 | :76 | :60 |
| 20 | QuadPlane Land | evet | :28 | :77 | :61 |
| 21 | QuadPlane RTL | evet | :29 | :78 | :62 |
| 22 | QuadPlane AutoTune | evet | :30 | :79 | :63 |
| 23 | QuadPlane Acro | evet | :31 | :80 | :64 |
| 24 | Thermal | evet | :32 | :81 | :65 |
| 25 | Loiter to QLand | evet | :33 | :82 | :66 |
| 26 | Autoland | evet | :34 | :83 | :67 |

Ortak (APMFirmwarePlugin) taban kod:
- Taban enum `APMCustomMode` yalnızca AUTO=3, GUIDED=4, RTL=6, SMART_RTL=21 (`src/FirmwarePlugin/APM/APMFirmwarePlugin.h:10-18`). Bunlar Plane numaraları DEĞİL; `_convertToCustomFlightModeEnum` ile Plane numarasına çevrilir (`ArduPlaneFirmwarePlugin.cc:182-196`: AUTO->10, GUIDED->15, RTL->11, SMART_RTL->RTL(11)).
- Taban sınıfın kendi mini listesi (Guided/RTL/Smart RTL/Auto, hepsi `canBeSet=true`) `APMFirmwarePlugin.cc:36-51`; Plane ctor'u bunu `updateAvailableFlightModes` ile tamamen ezer (`ArduPlaneFirmwarePlugin.cc:69` -> `FirmwarePlugin.cc:437-445` listeyi ve `_modeEnumToString`'i `clear()` edip yeniden kurar).
- Önemli gözlem: `ArduPlaneFirmwarePlugin.cc:10-38` (`_setModeEnumToModeStringMapping`) işlevsel olarak gereksiz. Satır 69'daki `_updateFlightModeList` aynı map'i `availableFlightModes` listesinden yeniden üretiyor (`FirmwarePlugin.cc:439-443`). Yeni mod için gerçekten gerekli olan tek tablo `.cc:40-68` listesi. Eşleme tablosunu yine de tutarlılık için güncellemek zararsız.
- `fixedWing`/`multiRotor` bayrakları `ArduPlaneFirmwarePlugin.cc:172-180` içinde set ediliyor (ikisi de `true`); src altında okuyan kod bulunamadı (grep: yalnızca atamalar), yani etkisiz.
- `advanced` bayrağı da `APMFirmwarePlugin::flightModes()` içinde kullanılmıyor (`APMFirmwarePlugin.cc:83-93` sadece `canBeSet` bakıyor).
- UI'da gizleme: Toolbar `FlightModeIndicator.qml:97-118` mod listesini `apmHiddenFlightModes<VehicleClass>` ayarıyla filtreliyor (isim bazlı kara liste). FixedWing varsayılanı `src/Settings/FlightMode.SettingsGroup.json:62-66`: "Circle,Training,Acro,FBW B,Cruise,Autotune,QuadPlane Stabilize,Guided,QuadPlane Hover,QuadPlane Loiter,QuadPlane Land,QuadPlane RTL,QuadPlane AutoTune,QuadPlane Acro,Thermal". Kara liste olduğu için yeni mod varsayılan olarak GÖRÜNÜR. Not: bu liste isimlerle eşleşir; firmware'den AVAILABLE_MODES ile farklı isim gelirse (örn. "FBWB") varsayılan gizleme eşleşmez ve mod görünür hale gelir.

#### B2. Tanınmayan custom_mode

- Kod yolu (Plane): `Vehicle::_handleHeartbeat` HEARTBEAT'ten `_custom_mode` alır (`src/Vehicle/Vehicle.cc:1302-1313`) -> `Vehicle::flightMode()` (`Vehicle.cc:1468-1471`) -> `APMFirmwarePlugin::flightMode()` (`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:95-103`): `QString flightMode = "Unknown"` (:97), `CUSTOM_MODE_ENABLED` bayrağı varsa `_modeEnumToString.value(custom_mode, "Unknown")` (:99-100), yoksa yine "Unknown" (:102).
- Sonuç: Plane için tanınmayan mod numarası sadece düz "Unknown" olarak gösterilir. Numara yazılmaz ("Unknown 25" gibi bir biçim YOK). Karşılaştırma: taban sınıf `FirmwarePlugin::flightMode` `"Custom:0x%1"` üretir (`src/FirmwarePlugin/FirmwarePlugin.cc:68`) ama APM bunu override ettiği için kullanılmaz.
- Yan etki 1: iki farklı tanınmayan mod arasında geçişte `previousFlightMode != flightMode()` hep false (ikisi de "Unknown"), `flightModeChanged` sinyali çıkmaz (`Vehicle.cc:1311-1313`).
- Yan etki 2: tanınmayan moddayken dropdown'daki hiçbir düğme "aktif" olmaz; `isGuidedMode` vb. string karşılaştırmaları false (`APMFirmwarePlugin.cc:644-647`).
- Toolbar etiketi: `src/Toolbar/FlightModeIndicator.qml:44` (`activeVehicle.flightMode`), diğer göstergeler `Toolbar/FlightModeMenuIndicator.qml:46`, `Toolbar/ModeIndicator.qml:11`, `QmlControls/FlightModeMenu.qml:10`.
- Ses/uyarı: Mod adı değişince sesli duyuru var. `flightModeChanged` -> `Vehicle::_handleFlightModeChanged` (bağlantı `Vehicle.cc:126`, işlev `Vehicle.cc:1788-1795`) -> `_say(tr("%1 %2 flight mode"))` (:1792) -> `AudioOutput::instance()->say(text.toLower())` (`Vehicle.cc:1728-1731`). Aynı ad tekrar duyurulmaz (`_lastAnnouncedFlightMode`, :1790). Görsel popup/uyarı bu yolda bulunamadı (doğrulanmadı: başka yerde mod değişimine bağlı popup olup olmadığı tüm repoda aranmadı, yalnızca belirtilen dizinlerde).
- Speakable notu: `src/FirmwarePlugin/FirmwarePlugin.h:133` yorumu "Flight mode names must be human readable as well as audio speakable" diyor. "GROUND_EFFECT" yerine "Ground Effect" gibi bir ad seslendirme için daha uygun.

#### B3. AVAILABLE_MODES / CURRENT_MODE (standard modes) desteği

Evet, destekleniyor ve ArduPilot için de koşulsuz çalışıyor.
- İstek: `InitialConnectStateMachine` bağlantıda `RequestStandardModes` durumunu çalıştırır (`src/Vehicle/InitialConnectStateMachine.cc:69-72`, geçiş :188-189, işlev :362-368), firmware tipine bakılmadan. `StandardModes::request()` -> `requestMode(1)` ile `REQUEST_MESSAGE(AVAILABLE_MODES, index)` tek tek ister (`src/Vehicle/StandardModes.cc:114-136`).
- Yanıt işleme: `StandardModes::gotMessage` her mesajdan `name`, `standard_mode`, `custom_mode`, `properties` (NOT_USER_SELECTABLE -> `cannotBeSet`, ADVANCED) okur (`StandardModes.cc:30-36`) ve `FirmwareFlightMode` oluşturur (`StandardModes.cc:72-80`). Son mod gelince `ensureUniqueModeNames()` (:84, :99-112) ve `firmwarePlugin()->updateAvailableFlightModes(_modeList)` (:85) çağrılır, `modesUpdated` yayılır (:86).
- Plane'de bu çağrı `ArduPlaneFirmwarePlugin::updateAvailableFlightModes` (`ArduPlaneFirmwarePlugin.cc:172-180`) -> `_updateFlightModeList` (`FirmwarePlugin.cc:437-445`) ile hem dropdown listesini (`_flightModeList`) hem custom_mode->isim map'ini (`_modeEnumToString`) firmware'in gönderdiğiyle DEĞİŞTİRİR. Yani statik tablo (B1) yalnızca yedek (firmware cevap vermezse; hata yolu `StandardModes.cc:91-95`).
- UI yenilenmesi: `modesUpdated` -> `flightModesChanged` ve `flightModeChanged(flightMode())` yeniden yayılır (`Vehicle.cc:276-281`), böylece HEARTBEAT daha önce geldiyse etiket güncellenir.
- Dinamik değişim: `AVAILABLE_MODES_MONITOR` (seq değişirse yeniden istek) `Vehicle.cc:706-715` ve `StandardModes.cc:138-145`; bağlantı kurulumu sırasında yok sayılır (`Vehicle.cc:709`).
- Kanıt (QGC tarafı): `MockLink` ArduPlane için AVAILABLE_MODES döndürüyor (`src/Comms/MockLink/MockLink.cc:3329-3345`, Plane listesi :95-123) ve test altyapısı var (`test/Vehicle/StandardModesTest.cc`, `test/Vehicle/InitialConnectTest.cc`).
- Sonuç: QGC'ye dokunmadan yeni mod adı gösterilebilir, KOŞULLA ki firmware AVAILABLE_MODES (+ HEARTBEAT custom_mode ile aynı custom_mode numarası) gönderiyor olsun. ArduPilot firmware'in AVAILABLE_MODES gönderip göndermediği QGC kodundan doğrulanamaz: doğrulanmadı (firmware kodu kapsam dışı).
- Özel isim ezmeleri: standard_mode değeri 0 olmayan modlarda ad QGC tarafından değiştirilir (`StandardModes.cc:37-63`): POSITION_HOLD->"Position", ORBIT->"Orbit" (ve seçilemez), CRUISE->"Cruise", ALTITUDE_HOLD->"Altitude", SAFE_RECOVERY->"Safe Recovery", MISSION->"Mission", LAND->"Land", TAKEOFF->"Takeoff". Özel mod için `standard_mode = MAV_STANDARD_MODE_NON_STANDARD (0)` gönderilirse firmware'in adı aynen kullanılır (varsayım: 0 değerinin non-standard anlamı QGC kodunda doğrulanmadı; switch'te 0 için dal yok, yani ad değişmez, bu doğrulandı).
- Dikkat: `Mission` adı (standard mode MISSION) gelirse `missionFlightMode()` yine custom_mode numarasından çözülür (`APMFirmwarePlugin.cc:659-662`, :61) ve isim eşleşmesi tutarlı kalır; ama FlightMode.SettingsGroup.json'daki varsayılan gizleme isimleri ("FBW B" vb.) tutmaz (B1).
- CURRENT_MODE: yalnızca `intended_custom_mode` okunur (`Vehicle.cc:716-718`, :1317-1334) ve `_custom_mode_user_intention` olarak saklanır. `Vehicle::flightMode()` hâlâ HEARTBEAT `_custom_mode` kullanır (`Vehicle.cc:1468-1471`). `effectiveCustomMode()` (`Vehicle.h:513`) yalnızca `MAVLinkEventManager` içinde, health/arming-check mode-group seçimi için kullanılıyor (`src/MAVLink/LibEvents/MAVLinkEventManager.cc:128-129`); mod adının üretilmesinde kullanılmıyor. CURRENT_MODE'un `standard_mode`/`custom_mode` alanları kullanılmıyor. Mod adı CURRENT_MODE'dan türetilmez.
- Mod ayarlama: `setFlightMode` APM'de ad -> custom_mode araması yapar (`APMFirmwarePlugin.cc:105-126`, büyük/küçük harf duyarsız) ve `MAV_CMD_DO_SET_MODE` gönderir (`Vehicle.cc:1496-1501`, `APMFirmwarePlugin.h:39`).

#### B4. "GROUND_EFFECT" için somut adım listesi

Önce seçim: (a) firmware AVAILABLE_MODES gönderiyorsa QGC değişikliği şart değil (B3); (b) gönderilmiyorsa ya da yedek isteniyorsa aşağıdaki adımlar. Mod numarası firmware ile birebir aynı olmalı (boş numara seçimi firmware tarafında; QGC'de 9 `RESERVED_9` olarak ayrılmış, `ArduPlaneFirmwarePlugin.h:17`).

1. `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.h:34` sonrası (enum kapanışı :35 öncesi): `GROUND_EFFECT = <N>,`
2. `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.h:83` sonrası: `const QString _groundEffectFlightMode = tr("Ground Effect");` (tr() -> çeviri, aşağıda 8. adım)
3. `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.cc:36` sonrası (isim tablosu): `{ APMPlaneMode::GROUND_EFFECT, _groundEffectFlightMode },` (işlevsel olarak gereksiz ama tutarlılık için, bkz. B1 gözlemi)
4. `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.cc:67` sonrası (settable listesi): `{ _groundEffectFlightMode, APMPlaneMode::GROUND_EFFECT, true, true },` -> `true` = kullanıcı seçebilir (`FirmwareFlightMode` kurucusu `FirmwarePlugin.h:28-34`). Bu satır olmadan mod ne adlanır ne dropdown'a girer (`FirmwarePlugin.cc:443`).
5. Guided / RTL / Auto eşlemeleri: `_convertToCustomFlightModeEnum` (`ArduPlaneFirmwarePlugin.cc:182-196`) yalnızca taban AUTO/GUIDED/RTL/SMART_RTL için; GROUND_EFFECT bunlardan biri değilse DEĞİŞİKLİK GEREKMEZ. Yalnızca özel modun bu rollerden birini oynaması isteniyorsa (örn. WIG RTL'i GROUND_EFFECT olsun) ilgili case'in döndüğü değeri değiştirin.
6. Pause/Takeoff/Stabilize: `ArduPlaneFirmwarePlugin.cc:157-170` Plane için Takeoff->TAKEOFF, Stabilized->STABILIZE, Pause->LOITER sabit. WIG için "pause" davranışı farklıysa (Loiter yerine başka mod) burayı değiştirin; yoksa dokunmayın. `landFlightMode()` Plane'de override EDİLMEMİŞ (boş string döner, `FirmwarePlugin.h:161`; sadece Copter override ediyor, `ArduCopterFirmwarePlugin.cc:187-190`); WIG'de "land" modu varsa `ArduPlaneFirmwarePlugin.h:50-53` bölümüne `QString landFlightMode() const override;` eklenmeli (kullanım: `FlyView/GuidedActionsController.qml:338`, `Vehicle.cc:2540`).
7. Gizleme: `src/Settings/FlightMode.SettingsGroup.json:65` değiştirmeyin; yeni mod varsayılan görünür. Kara listeye eklenmesi istenirse tam görünen adı ("Ground Effect") listeye ekleyin. Mevcut kullanıcı ayarları kullanıcıda saklandığı için eski kurulumları etkilemez.
8. Çeviri: `translations/qgc.ts` `lupdate` ile otomatik üretilir (mevcut kayıt örneği: `translations/qgc.ts:4226-4235`, "Loiter to QLand"/"Autoland"). Elle düzenleme gerekmez; yeni string `tr()` ile çıkar. Çeviri yoksa İngilizce ad kalır.
9. Opsiyonel (QGC dışı tutarlılık):
   - Log analiz ekranı: `src/AnalyzeView/LogViewer/APMDataFlash/APMDataFlashLogParser.cc:56-82` ve `src/AnalyzeView/LogViewer/APMDataFlash/LogViewerDataFlashParser.cc:62-72` kendi Plane mod tablolarını tutar; yeni numara yoksa "Mode <N>" yazar (`APMDataFlashLogParser.cc:113`, `LogViewerDataFlashParser.cc:85`). Eklemek için `{<N>, QStringLiteral("GROUND_EFFECT")}`.
   - Test/simülasyon: `src/Comms/MockLink/MockLink.cc:95-123` (Plane AVAILABLE_MODES listesi).
10. Toolbar için kod değişikliği gerekmez: gösterge ve dropdown `activeVehicle.flightMode` / `activeVehicle.flightModes` (`Toolbar/FlightModeIndicator.qml:44,150`) isimleri dinamik okur. APM'e özel genişletilmiş gösterge yalnızca multiRotor için (`APMFirmwarePlugin.cc:1402-1403`), Plane'e etkisi yok.
11. Sesli duyuru otomatik: yeni mod adı `Vehicle.cc:1792` ile seslendirilir; ad İngilizce okunabilir olsun.

Not (kapsam dışı öneri): "Unknown" yerine numara göstermek için `APMFirmwarePlugin.cc:97-102` değiştirilebilir (örn. `QStringLiteral("Unknown %1").arg(custom_mode)`); uygulanmadı, yalnızca öneri.

#### B5. Plane'e özel diğer davranışlar

- Capability: fixedWing için yalnızca `TakeoffVehicleCapability` (guided takeoff yok) `APMFirmwarePlugin.cc:64-81` (:67-68). Tüm APM için Set/Pause/Guided/ROI capability :66.
- Guided takeoff: multirotor/VTOL dışı `_guidedModeTakeoff` reddedilir (`APMFirmwarePlugin.cc:1008-1010`); fixed wing `startTakeoff` ile Takeoff moduna geçip arm eder (`APMFirmwarePlugin.cc:1047-1064`, `takeOffFlightMode` -> `ArduPlaneFirmwarePlugin.cc:157-160`).
- Mission başlatma: fixedWing/VTOL için önce Auto'ya geçilir, sonra arm (`APMFirmwarePlugin.cc:1066-1089`); diğer araçlar Guided+arm+MISSION_START (:1090-1105).
- Min. takeoff irtifası: `PILOT_TKO_ALT_M`/`PILOT_TKOFF_ALT` (VTOL için Q_ varyantları), `APMFirmwarePlugin.cc:974-1006`.
- Guided/goto: `gotoFlightMode() = guidedFlightMode()` (`APMFirmwarePlugin.h:41`); `isGuidedMode` = mod adı Guided mı (`APMFirmwarePlugin.cc:644-647`, :664-667). Goto `MAV_CMD_DO_REPOSITION` ile, fixed wing için `forwardFlightLoiterRadius` yarıçap ve işaret (yön) olarak gönderilir (`APMFirmwarePlugin.cc:775-830`, yorum :799-804, flag `MAV_DO_REPOSITION_FLAGS_CHANGE_MODE` :818); desteklenmezse mission-item yöntemi (:832-836).
- Pause: `pauseVehicle` -> `pauseFlightMode()` (`APMFirmwarePlugin.cc:747-750`) = Plane'de LOITER (`ArduPlaneFirmwarePlugin.cc:167-170`); `setGuidedMode(false)` de pause'a düşer (:738-745).
- RTL: `guidedModeRTL` -> `rtlFlightMode()` (`APMFirmwarePlugin.cc:841-844`, :649-652). SmartRTL Plane'de RTL'e eşlenir (`ArduPlaneFirmwarePlugin.cc:191-192`).
- Land: Plane için `landFlightMode`/`guidedModeLand` override yok (`ArduCopterFirmwarePlugin.h:48,55` yalnızca Copter'da).
- Airspeed: `AIRSPEED_MIN/MAX` paramları ile sınırlar ve `MAV_CMD_DO_CHANGE_SPEED` (`APMFirmwarePlugin.cc:1242-1281`); 4.5 parametre yeniden adlandırması `ArduPlaneFirmwarePlugin.cc:74-77`, `remapParamNameHigestMinorVersionNumber` :152-155 (4 -> 7).
- Irtifa değişimi: `guidedModeChangeAltitude` Guided'a geçer ve SET_POSITION_TARGET_LOCAL_NED yollar (`APMFirmwarePlugin.cc:846-890`).
- FollowMe: `APMFirmwarePlugin::sendGCSMotionReport` (`APMFirmwarePlugin.cc:1169-1180`); `followFlightMode()` Plane'de override YOK (boş string, `FirmwarePlugin.h:182`; yalnızca Copter `ArduCopterFirmwarePlugin.cc:197-200` ve Rover `ArduRoverFirmwarePlugin.cc:88-91`), `FollowMe::_isFollowFlightMode` bu değere bakar (`src/FollowMe/FollowMe.cc:202-205`). Dolayısıyla Plane'de follow modu tanımsız.
- Mission komutları: Plane için NAV_LAND/NAV_TAKEOFF (`APMFirmwarePlugin.cc:556-569`), plane özel komut JSON'u `APM-MavCmdInfoFixedWing.json` (`APMFirmwarePlugin.cc:583-584`).
- Flying tespiti: HEARTBEAT system_status ile (`APMFirmwarePlugin.cc:270-278`), mod bağımsız; `landing` durumu EXTENDED_SYS_STATE'ten (`Vehicle.cc:1039-1052`), mod adına bağlı değil.
- Parametre meta verisi: Plane için `APMParameterFactMetaData.Plane.<major>.<minor>.json` (`APMFirmwarePlugin.cc:682-736`).


## C. Alarmlar ve görsel uyarılar

Tüm yollar repo köküne göredir (`src/...`). Kısaltmalar: SH = `src/MAVLink/StatusTextHandler.cc`, V = `src/Vehicle/Vehicle.cc`, MW = `src/MainWindow/MainWindow.qml`, MSI = `src/Toolbar/MainStatusIndicator.qml`, BI = `src/Toolbar/BatteryIndicator.qml`.

#### C.1 STATUSTEXT akışı (uçtan uca)

1. `Vehicle::_handleMessage` önce `_firmwarePlugin->adjustIncomingMavlinkMessage()` çağırır (V:564), sonra `MAVLINK_MSG_ID_STATUSTEXT` -> `m_statusTextHandler->mavlinkMessageReceived()` (V:672-674).
2. APM'e özel filtre: `APMFirmwarePlugin::_handleIncomingStatusText` (`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:243-263`). Yalnızca heartbeat'inde `autopilot==ARDUPILOTMEGA` olan component'lerden gelen mesajlara uygulanır (APMFirmwarePlugin.cc:282, 302-308).
   - Metin "Place vehicle" veya "Calibration successful" içeriyorsa severity `INFO`'ya düşürülür, yani popup çıkmaz (APMFirmwarePlugin.cc:249-251, 362-383). Re-encode MAVLink1 bayrağıyla yapılır (satır 372); id/chunk_seq extension alanlarına etkisi doğrulanmadı.
   - `^Frame: (\S*)` yalnızca `_coaxialMotors` değişkenini ayarlar, görsel etkisi yok (APMFirmwarePlugin.cc:254-260).
   - Başka APM'e özel STATUSTEXT önek/filtre işlemi bulunamadı. "PreArm" işlemi firmware-bağımsızdır ve `Vehicle.cc` içindedir (C.1.3).
3. Chunk birleştirme (`SH:255-333`):
   - `id==0`: tek parça mesaj, doğrudan yayınlanır (SH:270-276).
   - `id!=0`: component başına `ChunkedStatusTextInfo` biriktirilir. `chunk_seq` boşlukları " ... " ile doldurulur (SH:281-288, 324). Mesaj NUL ile biterse (`messageText.length() < 50`) birleşir (SH:261, 301-304).
   - Aynı component'ten farklı `id` gelirse eksik mesaj olduğu varsayılıp eldeki parçalar yayınlanır (SH:264-268). 1000 ms timeout varsa kalanlar yayınlanır (SH:38-40, 307-314).
4. `Vehicle::_textMessageReceived` (V:3419-3460):
   - PX4'e özel: `\t` ile biten mesaj atılır (V:3422).
   - `"PreArm"` ile başlayan (APM) veya `"preflight"` (PX4, severity>=CRITICAL) mesaj: health-and-arming-check (events) destekleniyorsa listeye bile alınmadan atılır (V:3428-3434). Desteklenmiyorsa `setPrearmError(text)` çağrılır (V:3441), aynı metin 10 sn içinde tekrarlanırsa hata yenilenmez (V:3437-3440).
   - Ses: `"#"` öneki kaldırılıp seslendirilir, veya `severity <= NOTICE` ise seslendirilir (V:3447-3456). **Ses de var, bkz. D.**
   - Sonra `handleHTMLEscapedTextMessage(text.toHtmlEscaped())` (V:3459).
5. Events yolu (PX4 ağırlıklı): `MAVLinkEventManager::_handleEvent` log level'ı MAV_SEVERITY'ye çevirir (`src/MAVLink/LibEvents/MAVLinkEventManager.cc:134-144`). `health`/`arming_check`/`calibration` grupları status text'e çevrilmez (satır 147-153). Diğerleri `Vehicle::_onStatusTextFromEvent` -> `handleHTMLEscapedTextMessage` yoluna girer (V:3413-3417). Bu yol `_say` ve PreArm işlemini atlar. ArduPlane'in bu protokolü (`health_and_arming_check`) destekleyip desteklemediği QGC kodundan doğrulanamaz: **doğrulanmadı**. Koşul `EventHandler.cc:171-175`.

#### C.2 Severity -> görsel eşleme (`SH:132-244`)

| MAV_SEVERITY | Liste stili / `MessageType` | Toolbar mesaj ikonu rengi | Popup | Dosya:satır |
|---|---|---|---|---|
| EMERGENCY, ALERT, CRITICAL, ERROR (0-3) | `<#E>` / Error | `colorRed` (`MSI:127-128`), yalnızca okunmamış en kötü tür | Var (kritik popup), `newErrorMessage` | SH:162-168, 241-243; V:3410, 3462-3470 |
| WARNING, NOTICE (4-5) | `<#I>` / Warning | `colorOrange` (`MSI:125-126`) | Yok | SH:171-175 |
| INFO, DEBUG (6-7) | `<#N>` / Normal | `qgcPal.text` (varsayılan renk) | Yok | SH:177-179; MSI:123 |

- Satır öneki: `[hh:mm:ss.zzz COMP:n] <EMERGENCY|ALERT|Critical|Error|Warning|Notice|Info|Debug>: metin`. COMP yalnızca birden fazla component mesaj yollarsa eklenir (SH:183-229).
- **Liste rengi bulgu:** `VehicleMessageList.formatMessage` `<#E>` ve `<#I>` için aynı rengi (`qgcPal.warningText`) kullanır. Yani listede Error ve Warning/Notice satırları renkle ayırt edilemez. Yalnızca INFO/DEBUG `qgcPal.text` olur (`src/Toolbar/VehicleMessageList.qml:25-27`). `warningText` değeri `#cc0808` (açık) / `#f85761` (koyu) (`src/QmlControls/QGCPalette.cc:44`). SH:157-160 yorumu "ERROR bigger/bolder/red, warning orange" der ama kod bunu uygulamaz.
- Kritik popup (`MW:478-491, 494-585`):
  - Zemin `alertBackground` `#eecc44` (sarı), kenarlık `alertBorder` `#808080`, yazı `alertText` siyah (`QGCPalette.cc:65-67`). Başlık etiketi "Vehicle Error" (`MW:540`). Çoklu araçta "Vehicle N: " öneki eklenir (V:3464-3469).
  - Kabul etme: popup'a tıklama (`MW:581-584`) `acknowledge()` çağırır (`MW:507-515`). Ek hata gelmişse ana durum çekmecesini açar (`flyView.dropMainStatusIndicatorTool`), gelmemişse `resetErrorLevelMessages()` çağrılır (yalnızca Error sayacı sıfırlanır, SH:107-130). `closePolicy: CloseOnPressOutside`, otomatik kapanma yok (`MW:502`).
  - **Tek mesaj kuralı:** popup açıkken gelen yeni kritik mesaj gösterilmez, yalnızca "Additional errors received" etiketi çıkar (`MW:482-485`, `557`).
  - **Bastırma koşulları:** video tam ekranken (`MW:482`) veya ana durum çekmecesi açıkken (`MW:479`, `MSI:164-165, 179-180`) popup hiç açılmaz. Tam ekran video dalında mesaj kaybolur, yalnızca bayrak set edilir.
  - C++ tarafında "PreArm"/"preflight" ile başlayan mesajlar popup'tan hariç tutulur (`src/QGCApplication.cc:411-415`). Popup `_showErrorsInToolbar` bayrağına bağlıdır (`QGCApplication.cc:320, 418`).

#### C.3 Geçmiş, kabul, öncelik

- **Geçmiş:** `StatusTextHandler::m_messages` (`StatusTextHandler.h:55`) bir `QList<StatusText*>`. `SH:236`'daki `append` için üst sınır yok (kod okumasıyla; bellek sınırı testi yapılmadı). Mesajlar araç nesnesi yaşadığı sürece tutulur. Silme: çöp kutusu butonu -> `clearMessages()` (`VehicleMessageList.qml:75-104`, `SH:75-85`).
- **Gösterim:** Ana durum etiketine tıklayınca çekmece açılır, "Vehicle Messages" grubu en yeni üstte listeler (`MSI:28-38, 135-138, 248-261`; `SH:65-73`). Liste açılınca `resetAllMessages()` çağrılır, yalnızca sayaçları ve ikonu sıfırlar, mesajları silmez (`VehicleMessageList.qml:31-36`). İkon, `messageCount>0` iken görünür (`MSI:120`). Not: `VehicleMessageList.qml:32` `_activeVehicle` null kontrolünden önce kullanılır, kontrol satır 33'te.
- **Öncelik:** Resmî öncelik sistemi yok. Üç sınıf var: Error/Warning/Normal ve "okunmamış en kötü tür" ikon rengi (`SH:368-381`).
- **Kullanıcı ayarı:** Mesaj eşiği/filtre ayarı yok (`src/Settings/*.json` içinde warn/alert/failsafe adlı ayar bulunamadı).

#### C.4 Uyarı tablosu

| Uyarı | Tetikleyen koşul | Görsel biçim | Öncelik / kabul / geçmiş | Dosya:satır |
|---|---|---|---|---|
| Kritik mesaj popup'ı | STATUSTEXT/event severity 0-3, "PreArm"/"preflight" hariç (ses de var, bkz. D) | Sarı kutu, siyah yazı, "Vehicle Error" başlığı, ortada üstte | Tek mesaj. Tıkla = kabul. Otomatik kapanmaz. Geçmişe de yazılır | MW:478-585; V:3462-3470; QGCApplication.cc:409-429 |
| Mesaj ikonu rengi | `messageCount>0` | Kırmızı/turuncu/beyaz ikon durum etiketinde | Liste açılınca sayaç sıfırlanır | MSI:110-133; SH:335-382 |
| "Comms Lost" | `vehicleLinkManager.communicationLost`: tüm linklerde >3500 ms boyunca araçtan hiçbir MAVLink mesajı gelmemiş (RADIO_STATUS sayılmaz, bkz. F.1), 1 sn'de bir kontrol (ses de var, bkz. D) | Etiket "Comms Lost", gradyan zemin `"red"`, ayrıca "Disconnect" butonu | Kabul yok. Heartbeat gelince kendiliğinden döner | MSI:48-51; FlyViewToolBar.qml:18, 51-56, 93-98; VehicleLinkManager.h:84-85; VehicleLinkManager.cc:111-157 |
| Link geçişi/yeniden kazanım | Çoklu link: primary link değişti | Modal app mesajı diyaloğu (`showAppMessage`) + ses | Geçmişe yazılmaz | VehicleLinkManager.cc:84-87, 128-132 |
| "Not Ready" / "Ready" | Events yok (ArduPlane için varsayım, doğrulanmadı): `readyToFlyAvailable` (SYS_STATUS `PREARM_CHECK` enabled bit'i) -> `readyToFly` sarı/yeşil. Yoksa `allSensorsHealthy && setupComplete` | Etiket + gradyan zemin: Not Ready = `"yellow"`, Ready = `"green"` | Bu dalda kırmızı Not Ready yok. Kırmızı yalnızca events `canArm=false` | MSI:73-103; V:1073-1115 |
| "Armed/Flying/Landing" renk | Armed iken zemin `"green"` (events varsa `hasWarningsOrErrors` -> sarı, `!canArm` -> kırmızı) | Gradyan zemin | Sensör sağlıksız olsa bile events yoksa yeşil kalır | MSI:52-71 |
| Sensor Status listesi | SYS_STATUS present/enabled/health bitleri, events yoksa | Çekmecede ad + "Error/Normal/Disabled" metni, renk yok. Sıra: hatalı, sağlıklı, kapalı | Yalnızca çekmece açılınca görünür. Uyarı/popup yok | MSI:263-284; SysStatusSensorInfo.cc:18-106; QGCMAVLink.cc:335-366 (GeoFence, RC receiver, Terrain, Battery, AHRS dahil) |
| "No GPS Lock for Vehicle" | `requiresGpsFix` (SYS_STATUS present GPS bit'i) `&& !coordinate.isValid` | Haritada ortada yarı saydam beyaz kutu, siyah büyük yazı | Kabul yok. Koordinat ilk geçerli konumda geçerli olur. `_coordinate`'i geçersiz kılan satır bulunamadı (V:850, 884, 923-926, 975 yalnızca atama), yani GPS sonradan kaybolursa bu banner geri gelmez (doğrulanmadı: tüm yollar izlenmedi) | VehicleWarnings.qml:12-16, 22-28; V.h:539; V:846-887, 1106 |
| PreArm hata bandı | `!armed && prearmError != "" && !healthAndArmingCheckReport.supported`. APM'de "PreArm..." mesajı (ses de var, bkz. D) | Aynı beyaz kutu: mesaj metni + "The vehicle has failed a pre-arm check..." | Kayıt 35 sn sonra silinir (kabul yok). Aynı metin 10 sn'de bir yenilenir. Mesaj ayrıca geçmişe de yazılır, popup'ta yok | VehicleWarnings.qml:16, 30-47; V:2261-2273, 3428-3443; V.h:963 |
| Batarya ikonu | Bkz. C.5 | İkon + % / V | Kabul yok | BI:204-348 |
| GPS ikonu | `gps.count >= 0` -> opak, değilse yarı saydam. Sat sayısı + HDOP metni (HDOP geçerliyse) | Renk her zaman `qgcPal.text`, eşik/uyarı rengi yok. Fix tipi yalnızca çekmecede (`gps.lock.enumStringValue`) | Yok | GPSIndicator.qml:48-69; GPSIndicatorPage.qml:92-102; VehicleGPSIndicator.qml:9 |
| GPS spoofing/jamming/auth | GNSS_INTEGRITY durumu (0<değer<255) | Ayrı ikon: yeşil/turuncu/kırmızı/sarı/gri | Yok | GPSResilienceIndicator.qml:30-34, 62-80 |
| RC kaybı | Ayrı uyarı yok. RC RSSI göstergesi `rssi>0 && <=100` değilse tamamen gizlenir (opacity yalnız göstergede) | İkon kaybolur | Yok. `PreFlightRCCheck` telemetri kontrolü `false` olarak devre dışı | RCRSSIIndicator.qml:15-18, 50; PreFlightRCCheck.qml:10 |
| Telemetry (radio) RSSI | `lrssi != 0` | Gösterge, renk eşiği yok | Yok | TelemetryRSSIIndicator.qml:16-20, 30 |
| EKF / ESTIMATOR_STATUS | `ESTIMATOR_STATUS` flags/oranları fact olarak saklanır | UI'da kullanılmıyor (QML'de referans bulunamadı) | Yok. `EKF_STATUS_REPORT` repoda hiç geçmiyor (grep sonucu boş) | VehicleEstimatorStatusFactGroup.cc:33-71 |
| Titreşim / clipping | `VIBRATION` fact'leri | Yalnızca Analyze > Vibration sayfası: X/Y/Z çubuğu, 30 ve 60'ta kırmızı çizgi, clip sayısı metni. FlyView'da uyarı yok | Yok | VibrationPage.qml:25-28, 60-80, 184-198 |
| Geofence ihlali | `FENCE_STATUS.breach_status==1`, 3 sn aralıkla | **Yalnızca ses**, görsel yok (bkz. D) | Yok | V:2866-2901 (özellikle 2896) |
| Failsafe | Heartbeat `system_status` (CRITICAL/EMERGENCY) yalnızca `flying` belirlemekte kullanılır. Görsel failsafe göstergesi yok. Failsafe, firmware'in yolladığı STATUSTEXT olarak (C.2 kuralları) görünür | - | - | APMFirmwarePlugin.cc:270-278; QML'de `failsafe` grep sonucu yalnızca Actuators |
| Terrain | `TerrainProgress` yalnızca yükleme ilerlemesi gösterir, uyarı değil | Küçük panel, yeşil çerçeve, `blocksPending/Loaded` | 15 sn sonra gizlenir | TerrainProgress.qml:21-45, 65-71; FlyViewTopRightColumnLayout.qml:12 |
| ADSB çakışma | `ADSBVehicle.alert` -> harita simgesi `AlertAircraft.svg`, değilse `AwarenessAircraft.svg`. `alert` yalnızca ADSB TCP link'inden gelir (`AlertAvailable`). MAVLink `ADSB_VEHICLE` yolu alert'ı hiç set etmez | Haritada simge değişimi, ses/popup yok | Yok | ADSBVehicleMapItem.qml:17, 30; ADSBTCPLink.cc:181-182; ADSBVehicleManager.cc:73-141; FlyViewMap.qml:284-296 |
| Yükseklik / hız uyarısı | FlyView/Toolbar/FlightMap/QmlControls/Vehicle içinde stall/overspeed/low altitude uyarı kodu bulunamadı | - | - | Arama sonucu boş |
| Pre-flight checklist | Varsayılan **kapalı** (`useChecklist=false`, `enforceChecklist=false`). Açıksa `FixedWingChecklist` kullanılır | Düğmeler: geçti `#86cc6a`, bekliyor `#f7a81f`, başarısız `#c31818` | Checklist `Passed` değilse (enforce) popup otomatik açılır; override yalnızca GPS sat sayısı için (`allowOverrideSatCount: true`) | App.SettingsGroup.json:184-200; FlyViewPreFlightChecklistPopup.qml:15-43; FixedWingChecklist.qml:21-35; PreFlightCheckButton.qml:36-50 |

Pre-flight checklist kontrol koşulları (FixedWing):
- Batarya: `batteries[0].percentRemaining < 40` -> başarısız (override yok, NaN = 0 sayılır) (`PreFlightBatteryCheck.qml:16-21`; `FixedWingChecklist.qml:21-24`).
- Sensörler: `sensorsUnhealthyBits & (MAG|ACCEL|GYRO|ABS_PRESSURE|DIFF_PRESSURE|GPS|AHRS)` (`PreFlightSensorsHealthCheck.qml:8, 45-52`). Metin: "Failure. ... issues. Check console." (satır 60-66).
- GPS: `gps.lock >= 3` ve `count > 9` (override'lı) (`PreFlightGPSCheck.qml:17-21`; `FixedWingChecklist.qml:29-32`).
- Ses: muted veya volume<=0 (`PreFlightSoundCheck.qml:79-82`, bkz. D).

#### C.5 Batarya eşikleri (kaynak ve renkler)

- **Kaynak:** Yalnızca `BATTERY_STATUS` (`BatteryFactGroupListModel.cc:19, 75-76, 103-142`) ve HIGH_LATENCY/2 (satır 83-98). `SYS_STATUS.voltage_battery/battery_remaining` kullanılmaz (grep sonucu boş). `percentRemaining==-1` -> NaN, `charge_state` doğrudan fact'e yazılır (`BatteryFactGroupListModel.cc:140-142`). Varsayılan `chargeState=UNDEFINED` (satır 60).
- **Renk mantığı (`BI:211-235`):** `charge_state` ana belirleyicidir, QGC kendisi low/critical eşiği hesaplamaz.
  - `OK`: yüzde `> threshold1` yeşil, `> threshold2` sarı-yeşil, aksi sarı. Yüzde NaN ise `qgcPal.text`.
  - `LOW`: `colorOrange`.
  - `CRITICAL/EMERGENCY/FAILED/UNHEALTHY`: `colorRed`.
  - `UNDEFINED` veya `CHARGING`: `qgcPal.text` (renksiz). ArduPlane `charge_state`'i doldurmazsa low/critical renkleri hiç oluşmaz, doğrulanmadı (firmware tarafı kapsam dışı).
- **İkon seçimi (`BI:237-259`):** OK/LOW/CRITICAL/EMERGENCY için ayrı SVG'ler. **Kod bulgusu:** `case OK` içinde yüzde NaN ise `return` yok, `case LOW`'a düşer ve turuncu ikon çıkar (`BI:239-250`). Yani `charge_state=OK` ama yüzde bilinmiyorsa ikon turuncu olur (renk ise `qgcPal.text`).
- **Kullanıcı eşikleri:** `threshold1` varsayılan 80, `threshold2` varsayılan 60, birim % (`src/Settings/BatteryIndicator.SettingsGroup.json:14-29`). Doğrulama: `threshold1` 16..99 ve `>threshold2`, `threshold2` >15 ve `<threshold1` (`src/Settings/BatteryIndicatorSettings.cc:52-84`). Ayar UI'ı: batarya çekmecesi "Coloring" satırı (`BI:460-550`). Diğer ayarlar `valueDisplay` (%/V/ikisi) ve `consolidateMultipleBatteries` (varsayılan true) (JSON:6-36).
- **Ses:** `charge_state` ilk kez LOW/CRITICAL/EMERGENCY/FAILED/UNHEALTHY'ye yükselince "warning" + "battery N level low/critical/..." söylenir (`V:1131-1188`). **Görsel karşılığı yalnızca ikon rengi, ses de var, bkz. D.** Batarya için popup veya geçmiş girdisi yok (mesaj listesine yazılmaz, `_say` doğrudan).
- **APM'e özel (batarya):** `APMBatteryIndicator.qml` yalnızca çekmecede `BATT_FS_LOW_ACT`, `BATT_LOW_VOLT/MAH`, `BATT_FS_CRT_ACT`, `BATT_CRT_VOLT/MAH` parametrelerini düzenletir. Bunlar firmware failsafe eşikleridir ve QGC'nin görsel rengini etkilemez (`src/FirmwarePlugin/APM/APMBatteryIndicator.qml:11-72`, bağlanış `APMFirmwarePlugin.cc:1398-1401`).
- Çoklu batarya: en düşük olan (yüzde/voltaj/charge_state) gösterilir (`BI:33-157`).

#### C.6 ArduPlane WIG için özet çıkarımlar

- Elde edilebilir görsel uyarı kanalları: (1) kritik popup (STATUSTEXT severity<=3), (2) durum etiketi/gradyan zemin (Comms Lost kırmızı, Not Ready sarı), (3) mesaj ikonu rengi, (4) PreArm bandı (35 sn), (5) batarya ikonu (charge_state'e bağlı), (6) GPS/RC göstergeleri (eşik rengi yok).
- Görsel karşılığı olmayanlar: geofence (yalnızca ses), failsafe, EKF/ESTIMATOR, vibration (yalnızca Analyze), yükseklik/hız, RC kaybı, terrain uyarısı. ADSB çakışması yalnızca TCP kaynaklı `alert` ile.
- Dikkat edilecek kenar durumlar: tam ekran video veya açık durum çekmecesi sırasında kritik popup bastırılır (MW:479-485); popup yalnızca ilk mesajı gösterir; mesaj listesinde Error ve Warning aynı renk (VehicleMessageList.qml:25-26); mesaj listesi sınırsız ve temizleme yalnızca elle.


## D. Sesli uyarılar

Kapsam: QGC v5.1.5 (3a67d31f). Tüm yollar repo köküne göre. Tek ses çıkış yolu `AudioOutput::say()` (QTextToSpeech). `QSoundEffect`, `QMediaPlayer` ile ses, `QApplication::beep()` veya başka bir "beep" çağrısı `src` altında bulunmadı (grep: `QSoundEffect|beep|playSound|\.wav`; `QMediaPlayer` yalnızca video alıcıda, `src/VideoManager/VideoReceiver/QtMultimedia/QtMultimediaReceiver.cc:20`). `resources/audio/alert.wav` dosyası var ama repoda hiçbir yerden referans verilmiyor (`alert.wav` grep'i yalnızca `.pre-commit-config.yaml:23` ve `.claudeignore:11` dizin istisnalarını buldu); yani çalınmıyor. "Beep" geçen diğer yerler: `src/Vehicle/Actuators/Actuators.cc:459` ve `ActuatorActions.cc:14` (araca ACTUATOR_CONFIGURATION_BEEP komutu, QGC ses çalmaz), `src/AutoPilotPlugins/APM/APMESCComponent.qml:159-161` (kullanıcıya metin, ESC'nin çıkardığı bip).

#### D.1 Konuşma tetikleyicileri

`Vehicle::_say(text)` her zaman `text.toLower()` ile `AudioOutput::say()` çağırır (`src/Vehicle/Vehicle.cc:1728-1731`). `VehicleLinkManager` çağrıları da `.toLower()` kullanır. `_vehicleIdSpeech()` birden fazla araç varsa `"Vehicle <id> "` döner, tek araçta boş string (`src/Vehicle/Vehicle.cc:1779-1786`).

| Olay | Metin (tam string) | Koşul | dosya:satır |
|---|---|---|---|
| STATUSTEXT (araçtan gelen serbest metin) | Gelen metnin kendisi (küçük harfe çevrilir, sonra abbreviation dönüşümü). Araç ID öneki EKLENMEZ. | `severity <= MAV_SEVERITY_NOTICE` (yani EMERGENCY, ALERT, CRITICAL, ERROR, WARNING, NOTICE) veya metin `#` ile başlıyorsa (`#` silinir, severity'den bağımsız); `skipSpoken` değilse | `src/Vehicle/Vehicle.cc:3447-3456` (`_say(text)` satır 3455); STATUSTEXT girişi `src/Vehicle/Vehicle.cc:672-674`, parçalı mesaj birleştirme `src/MAVLink/StatusTextHandler.cc:255-333` |
| Uçuş modu değişimi | `tr("%1 %2 flight mode")` -> "[Vehicle N ] <mod adı> flight mode" (ör. ArduPlane: "Manual", "FBW A", "RTL", "Auto", "Loiter", "Takeoff", "Autoland") | `flightModeChanged` sinyali ve mod adı `_lastAnnouncedFlightMode`'dan farklıysa | `src/Vehicle/Vehicle.cc:126` (connect), `1788-1795` (`_say` satır 1792); mod adları `src/FirmwarePlugin/APM/ArduPlaneFirmwarePlugin.h:58-83` |
| Arm / disarm | `"%1 %2"` -> "[Vehicle N ] armed" veya "[Vehicle N ] disarmed" (`tr("armed")`, `tr("disarmed")`) | `armedChanged` sinyali (her değişimde, dedupe yok; `_updateArmed` yalnızca `_armed != armed` ise emit eder) | `src/Vehicle/Vehicle.cc:127` (connect), `1797-1799`; emit `src/Vehicle/Vehicle.cc:1215-1219` |
| Batarya durumu LOW | önce `tr("warning")`, sonra "[Vehicle N ] battery [<id>] level low" (`tr("battery %1 level low")`) | BATTERY_STATUS `charge_state == LOW` ve bu batarya id için daha önce anons edilen en yüksek durumdan büyükse | `src/Vehicle/Vehicle.cc:1146-1150, 1178-1187` |
| Batarya CRITICAL | "warning" + "battery %1 level is critical" | `charge_state == CRITICAL`, aynı escalation koşulu | `src/Vehicle/Vehicle.cc:1152-1157, 1185-1186` |
| Batarya EMERGENCY | "warning" + "battery %1 level emergency" | `charge_state == EMERGENCY`, aynı koşul | `src/Vehicle/Vehicle.cc:1158-1163` |
| Batarya FAILED | "warning" + "battery %1 failed" | `charge_state == FAILED`, aynı koşul | `src/Vehicle/Vehicle.cc:1164-1169` |
| Batarya UNHEALTHY | "warning" + "battery %1 unhealthy" | `charge_state == UNHEALTHY`, aynı koşul | `src/Vehicle/Vehicle.cc:1170-1175` |
| Geofence ihlali | `breachTypeStr + " " + tr("fence breached")`; breachTypeStr: "minimum altitude" / "maximum altitude" / "boundary"; bilinmeyen tipte breachTypeStr boş kalır ("" + " fence breached"). Araç ID öneki YOK. | FENCE_STATUS mesajı `breach_status == 1`, `breach_type != NONE`, son anonstan >3000 ms geçmişse | `src/Vehicle/Vehicle.cc:2866-2901` (`_say` satır 2896); çağrı `src/Vehicle/Vehicle.cc:685` |
| Bağlantı geri geldi (tek link) | `tr("%1Communication regained")` | Daha önce `commLost` olan linkte heartbeat geri geldi, `_rgLinkInfo.count() <= 1` | `src/Vehicle/VehicleLinkManager.cc:36-53` (heartbeat girişi), `55-72`, `80-82` |
| Bağlantı geri geldi (çoklu link) | `tr("%1Communication regained on %2 link")`, %2 = "primary" / "secondary" | Aynı, `_rgLinkInfo.count() > 1` | `src/Vehicle/VehicleLinkManager.cc:68-69, 80-82` |
| Yeni primary linke geçiş (geri gelme sonrası) | `tr("%1Switching communication to new primary link")` (ayrıca `QGC::showAppMessage`) | Link geri gelince `_updatePrimaryLink()` true dönerse | `src/Vehicle/VehicleLinkManager.cc:75-78, 84-87` |
| Tek linkte bağlantı kaybı (çoklu link varken) | `tr("%1Communication lost on %2 link.")` (primary/secondary) | Link için son heartbeat'ten beri >3500 ms, high-latency link değil, ve `_rgLinkInfo.count() > 1`. Kontrol 1000 ms aralıklı timer ile | `src/Vehicle/VehicleLinkManager.cc:109-121`; sabitler `src/Vehicle/VehicleLinkManager.h:84-85` |
| Secondary linke geçiş (kayıp sırasında) | `tr("%1Switching communication to secondary link.")` (ayrıca `QGC::showAppMessage`) | `_commLostCheck` içinde `_updatePrimaryLink()` true | `src/Vehicle/VehicleLinkManager.cc:128-132` |
| Toplam bağlantı kaybı | `tr("%1Communication lost")` | Tüm linkler `commLost`, `_autoDisconnect` false (true ise araç kapatılır ve konuşma yok), ve `_communicationLost` daha önce false; `_communicationLostEnabled` true olmalı | `src/Vehicle/VehicleLinkManager.cc:102-104, 134-157` (`say` satır 153) |
| Ses testi | `tr("Audio test. Volume is %1 percent")`, %1 = ses seviyesi (1 ondalık) | Ayarlar > General "Test" butonu; önce mevcut konuşmayı keser | `src/Utilities/Audio/AudioOutput.cc:273-286`; buton `src/AppSettings/pages/General.SettingsUI.json:29-33`; slot `src/QmlControls/QGroundControlQmlGlobal.cc:346-349` |

Sesli olmayan olaylar (doğrulandı, `say` çağrısı yok): takeoff/land, mission item ulaşma, GPS kaybı/fix, RC kaybı, MAVLink events (PX4 tarzı) gelen statustext. `Vehicle::_onStatusTextFromEvent` yalnızca `handleHTMLEscapedTextMessage` çağırır, `_say` çağırmaz (`src/Vehicle/Vehicle.cc:3413-3417`). Geriye kalan `src` alanında başka `say(`/`_say(` çağrısı yok (grep ile taranan `.cc/.h/.qml`; `src/Utilities/StateMachine/QGCStateMachine.cc:5` yalnızca `#include "AudioOutput.h"`, çağrı yok).

#### D.2 STATUSTEXT seslendirmesi: severity, dönüşüm, araç ID

- Severity eşiği: `severity <= MAV_SEVERITY_NOTICE` (`src/Vehicle/Vehicle.cc:3450`). Sayısal değerler QGC'nin sabitlediği MAVLink tanımından doğrulandı: EMERGENCY=0 … NOTICE=5 … DEBUG=7 (mavlink@c409cf69 `message_definitions/v1.0/common.xml:3037-3060`). Konuşulmayanlar: INFO ve DEBUG, `#` önekli değilse.
- `#` öneki: metnin başındaki `#` silinir ve severity ne olursa olsun seslendirilir (`src/Vehicle/Vehicle.cc:3447-3449`). Silme `emit textMessageReceived` ve log'a da yansır (aynı `text` değişkeni, satır 3458-3459).
- PreArm özel durumu: `text.startsWith("PreArm")` (ArduPilot) veya "preflight" + severity>=CRITICAL (PX4). Araç health-and-arming-checks event'lerini destekliyorsa mesaj tamamen düşürülür (konuşma ve log yok) (`src/Vehicle/Vehicle.cc:3428-3434`). Desteklemiyorsa aynı metin 10 saniyede bir kez seslendirilir, aradakiler `skipSpoken` (`src/Vehicle/Vehicle.cc:3436-3443`, harita `src/Vehicle/Vehicle.h:1038`). Metin dönüşümü yapılmaz; "PreArm" sadece genel abbreviation tablosundan geçer: `_say` küçük harfe çevirdiği için "prearm" -> `_textHash["PREARM"]` -> "pre arm" (`src/Utilities/Audio/AudioOutput.cc:35`; arama büyük harfe çevirerek yapılır, satır 302).
- Dönüşüm zinciri `AudioOutput::say` -> `_fixTextMessageForAudio` (`src/Utilities/Audio/AudioOutput.cc:243, 288-298`), sırayla:
  1. `_replaceNegativeSigns`: sayıdan önceki `-` -> "negative " (satır 351-358)
  2. `_replaceAbbreviations`: token bazlı, sondaki instance rakamlarını ayırır ("EKF3" -> "E.K.F. 3") (satır 320-349). `_textHash` (büyük/küçük harf duyarsız, satır 19-48): ERR->error, POSCTL->Position Control, ALTCTL->Altitude Control, AUTO_RTL->auto return to launch, RTL->"return To launch", ACCEL, RC_MAP_MODE_SW, REJ, WP->waypoint, CMD, COMPID, PARAMS, ID->"I.D.", ADSB->"A.D.S.B.", EKF->"E.K.F.", PREARM->"pre arm", PITOT->"pee toe", SERVOX_FUNCTION, CNT, DNST, TKOFF, TERRN, AROT, TCAL, PWR, PERF, CFG, FBWA->"fly-by-wire A". `_spelledAcronyms` (yalnızca TAMAMEN BÜYÜK HARF eşleşir, harf harf "G.P.S." diye okunur, satır 51-57, mantık 300-318).
  3. `_replaceDecimalPoints`: "12.5" -> "12 point 5" (satır 360-374)
  4. `_replaceMeters`: sayıdan sonraki `m` -> " meters" (satır 376-390)
  5. `_convertMilliseconds`: "1500ms" -> "1 second and 500 millisecond"; 1000 ms altı dokunulmaz; >=60000 ms dakika/saniye (satır 392-419, regex satır 423)
- ÖNEMLİ bulgu: Bütün Vehicle/VehicleLinkManager çağrıları metni `toLower()` ile gönderdiği için (`src/Vehicle/Vehicle.cc:1730`, `src/Vehicle/VehicleLinkManager.cc:81,85,119,130,153`), `_spelledAcronyms` (büyük harf şartlı) bu yollarda hiç eşleşmez ("GPS" -> "gps" olarak TTS'e gider). Yalnızca `_textHash` girişleri çalışır (case-insensitive). Bu, kod okumasından çıkarım; çalışma zamanında test edilmedi.
- ArduPlane ile ilgili: uçuş modu adı "FBW A" ("fbw a") `_textHash["FBWA"]` ile eşleşmez (token "fbw" ve "a" ayrı), TTS'in kendi okumasına kalır. "RTL" -> "return To launch" (tablo değeri, satır 24). "Avoid ADSB" -> "avoid A.D.S.B.".
- Çeviri: `TextMod::Translate` bayrağı hiçbir çağrıcı tarafından verilmiyor (`say(text)` çağrıları ek parametresiz); bayrak verilse bile `tr("%1").arg(outText)` fiilen no-op (`src/Utilities/Audio/AudioOutput.cc:245-247`). Metinler `tr()` kaynak string'leri (İngilizce); TTS yerel ayarı en_US'e pinlenir (satır 187-190).
- Araç ID: yalnızca `_vehicleIdSpeech()` kullanan çağrılarda (uçuş modu, arm/disarm, batarya, bağlantı mesajları) ve yalnızca `MultiVehicleManager::instance()->vehicles()->count() > 1` ise "Vehicle <id> " eklenir (`src/Vehicle/Vehicle.cc:1779-1786`). STATUSTEXT (satır 3455) ve geofence (satır 2896) mesajlarına araç ID EKLENMEZ; çoklu araçta kaynağı ayırt edilemez.

#### D.3 Kuyruk, tekrar engelleme, rate limit

| Mekanizma | Davranış | dosya:satır |
|---|---|---|
| Kuyruk üst sınırı | `kMaxTextQueueSize = 20`. Kuyruk doluysa (>=20) mevcut konuşma `stop(Immediate)` ile kesilir, sayaç sıfırlanır (kuyruk boşaltılır), sonra yeni metin eklenir; `qCWarning` basar | `src/Utilities/Audio/AudioOutput.h:80`; `src/Utilities/Audio/AudioOutput.cc:254-270` |
| Sayaç takibi | `aboutToSynthesize` ile sayaç azalır; engine `Ready` olunca ve `errorOccurred`'da sıfırlanır | `src/Utilities/Audio/AudioOutput.cc:95-115` |
| Genel duplicate/rate-limit | YOK: `say()` aynı metni tekrar tekrar kuyruğa alır (genel dedupe yok) | `src/Utilities/Audio/AudioOutput.cc:225-271` |
| PreArm STATUSTEXT | Aynı tam metin 10 sn'de bir (yalnızca konuşma bastırılır; metin yine yayınlanır) | `src/Vehicle/Vehicle.cc:3436-3443` |
| Geofence | 3 sn'de bir; `lastUpdate` fonksiyon içi `static` (tüm araçlar için ortak). Ayrıca `breach_status != 1` iken `lastUpdate` sürekli güncellenir, yani ihlal biter bitmez yeniden başlarsa ilk anons 3 sn gecikir | `src/Vehicle/Vehicle.cc:2874-2900` |
| Uçuş modu | Aynı mod adı tekrar anons edilmez (`_lastAnnouncedFlightMode`) | `src/Vehicle/Vehicle.cc:1790-1793` |
| Batarya | Her batarya id için yalnızca durum kötüleşirse (charge_state > en yüksek anons edilen) bir kez; arm anında harita temizlenir; durum OK'ye dönerse sıfırlanır; periyodik tekrar YOK | `src/Vehicle/Vehicle.cc:1136-1176, 1226` |
| Bağlantı kaybı | `_communicationLost` bayrağı ile bir kez; geri gelince bayrak sıfırlanır | `src/Vehicle/VehicleLinkManager.cc:134-157, 91-97` |
| Mute/volume=0 | Yeni metin kuyruğa alınmaz; mute anında kuyruk dahil konuşma durdurulur | `src/Utilities/Audio/AudioOutput.cc:234-236, 214-221` |

#### D.4 Batarya düşük/kritik sesli uyarı

- Var, ama yüzde bazlı değil: yalnızca MAVLink BATTERY_STATUS `charge_state` alanına bağlı (LOW, CRITICAL, EMERGENCY, FAILED, UNHEALTHY). Konuşma sırası: "warning", ardından "[Vehicle N ] battery [id] level low/is critical/emergency/failed/unhealthy" (`src/Vehicle/Vehicle.cc:1131-1188`). Batarya id yalnızca birden fazla batarya varsa eklenir (`src/Vehicle/Vehicle.cc:1180-1184`).
- Tekrarlama periyodu: yok. Her durum geçişi için arm döngüsü başına tek anons. Arm olunca `_lowestBatteryChargeStateAnnouncedMap.clear()` (`src/Vehicle/Vehicle.cc:1226`).
- Dikkat: Bir batarya id'sinin ilk BATTERY_STATUS mesajı zaten LOW veya daha kötüyse, ilk değer "zaten anons edildi" kabul edilir (`src/Vehicle/Vehicle.cc:1136-1138`), dolayısıyla o durum için ilk mesajda anons YAPILMAZ (yalnızca daha da kötüleşirse konuşur). Arm anında harita temizlendikten sonra aynı mantık yeniden geçerli.
- `appSettings.batteryPercentRemainingAnnounce` (varsayılan 30 %) bir TTS eşiği DEĞİL: `src/Settings/AppSettings.h:27` yorumu "only used to calculate battery swaps"; tek kullanımı mission planlamada pil değişimi hesabı (`src/MissionManager/MissionFlightStatusCalculator.cc:43-44`). Ayar tanımı `src/Settings/App.SettingsGroup.json:97-107` (açıklaması "Announce..." diyor ama kodda yüzde ile konuşma yok).
- Firmware tarafından gönderilen "battery low/critical" STATUSTEXT'leri (ör. ArduPilot) severity <= NOTICE ise ayrıca D.2 yolundan seslendirilir. Firmware'in hangi severity/metin ve BATTERY_STATUS.charge_state gönderdiği bu repodan doğrulanmadı.

#### D.5 Susturma ve ses ayarları

| Ayar / mekanizma | Değer / davranış | dosya:satır |
|---|---|---|
| `audioMuted` (bool) | Varsayılan false. "Mutes all audio output without changing the volume level." | `src/Settings/App.SettingsGroup.json:122-130` (default satır 127); `src/Settings/AppSettings.h:29`; `src/Settings/AppSettings.cc:191` |
| `audioVolume` (double, %) | Varsayılan 100.0, min 0.0, userMax 100.0, 1 ondalık | `src/Settings/App.SettingsGroup.json:131-144` (default satır 136); `src/Settings/AppSettings.h:30`; `src/Settings/AppSettings.cc:192` |
| Ayar UI'si | General sayfasında slider + "enable" checkbox (`checked = !audioMuted`). Checkbox açılırken volume <= 0 ise volume 75'e çekilir. "Test" butonu, mute değilken ve volume > 0 iken etkin | `src/AppSettings/pages/General.SettingsUI.json:22-34` |
| Başlatma | `AudioOutput::instance()->init(audioVolume, audioMuted)` uygulama açılışında | `src/QGCApplication.cc:303-304`; `src/Utilities/Audio/AudioOutput.cc:82-139` |
| `say()` mute/ses kontrolü | `volume <= 0` veya `muted` ise `say()` hemen döner, metin kuyruğa alınmaz (sessizce atılır, sonra unmute edilince tekrar konuşulmaz) | `src/Utilities/Audio/AudioOutput.cc:234-236` |
| Mute/volume değişimi | `_setVolume()`: mute ise etkin ses 0; volume 0 olunca `_engine->stop(Immediate)` + kuyruk sayacı sıfırlanır (kuyruktaki metinler konuşulmaz); aksi halde `QTextToSpeech::setVolume(volume/100)` | `src/Utilities/Audio/AudioOutput.cc:145-151, 201-223` |
| Engine | `QTextToSpeech` (Qt TextToSpeech modülü). Birim testlerde "none" backend, aksi halde otomatik seçim (log kategorisi notlarında flite/android) | `src/Utilities/Audio/AudioOutput.cc:61-68, 9-16`; platform listesi: `src/Utilities/Audio/CMakeLists.txt:27` (yalnızca include). Hangi backend'in hangi platformda seçildiği bu repoda doğrulanmadı |
| Engine hazır değilse | Android gibi asenkron backend'lerde `Ready`'ye kadar init ertelenir; 10 sn içinde hazır olmazsa uyarı log'u. `_initialized` false iken `say()` uyarı basıp döner | `src/Utilities/Audio/AudioOutput.cc:94-138, 227-232`; `AudioOutput.h:83` |
| Konuşma yeteneği yoksa | `Capability::Speak` yoksa `say()` "Speech Not Supported" log'layıp döner | `src/Utilities/Audio/AudioOutput.cc:198, 238-241` |
| Dil/ses | Locale en_US varsa o seçilir; ilk mevcut voice pinlenir | `src/Utilities/Audio/AudioOutput.cc:181-199` |
| Pre-flight checklist | `PreFlightSoundCheck`: audioMuted veya audioVolume <= 0 ise "QGC audio output is disabled. Please enable it under application settings->general to hear audio warnings!" hatası (checklist öğesi); aksi halde manuel soru "QGC audio output enabled. System audio output enabled, too?". FixedWingChecklist'te kullanılıyor | `src/FlyView/PreFlightSoundCheck.qml:6-13`; `src/FlyView/FixedWingChecklist.qml:56` |
| Hız / pitch ayarı | Bulunmadı (`speechRate`/`pitch` grep sonucu boş) | doğrulandı: yok |

#### D.6 WIG (ArduPlane) için pratik notlar

- Konuşmanın tamamı ground-station tarafında araçtan gelen veriye bağlı: STATUSTEXT (severity <= NOTICE), mod/arm değişimi, BATTERY_STATUS.charge_state, FENCE_STATUS, heartbeat zaman aşımı (3.5 sn). Takeoff/land/mission item için ayrı sesli uyarı yok.
- ArduPlane'in WIG'e özgü STATUSTEXT'leri varsa (ör. ground-effect geçiş mesajları) severity <= NOTICE iken otomatik okunur; INFO (6) ve DEBUG (7) için firmware mesajının başına `#` konmalı. Bu davranış kod okumasından çıkarım, firmware mesajları doğrulanmadı.
- STATUSTEXT'te araç ID öneki yok ve genel duplicate bastırma yok; hızlı tekrarlayan bir ArduPlane mesajı kuyruğu 20'ye doldurur ve konuşma kesilip kuyruk sıfırlanır (`src/Utilities/Audio/AudioOutput.cc:255-259`). Yalnızca "PreArm" metinleri 10 sn'de bir sınırlanır.


## E. MAVLink mesajları

Kapsam: sadece `src/` (commit 3a67d31). Tüm yollar repo köküne göredir. Dağıtım mantığı: `Vehicle::_mavlinkMessageReceived` her mesajı önce firmware plugin'e, sonra tüm FactGroup'lara (`src/Vehicle/Vehicle.cc:588-590`), sonra Vehicle'ın kendi `switch`'ine (`src/Vehicle/Vehicle.cc:594-`) verir. Mesajın sysid'i aracınkiyle eşleşmiyorsa (RADIO_STATUS istisnası hariç) atılır (`src/Vehicle/Vehicle.cc:527-531`). MAVLink v1 çerçeveli mesajlar HEARTBEAT ve RADIO_STATUS dışında düşürülür (`src/Comms/MAVLinkProtocol.cc:131-134`), yani aşağıdaki mesajların hepsi v2 olarak gelmeli.

#### Özet tablo

| Mesaj | İşleniyor mu | İşleyen sınıf/fonksiyon | Kullanılan alanlar | Ekranda nerede | dosya:satır |
|---|---|---|---|---|---|
| DISTANCE_SENSOR | Evet | `VehicleDistanceSensorFactGroup::handleMessage` | `orientation` (seçici), `current_distance` (cm→m), `min_distance`, `max_distance` (cm→m). `id`, `type`, `covariance`, `horizontal_fov`, `quaternion` vb. okunmuyor | (a) Fact grid: grup "DistanceSensor" (rotationNone…rotationPitch270, minDistance, maxDistance). (b) Fly View proximity radar (sadece 8 yaw sektörü; pitch yok) | `src/Vehicle/FactGroups/VehicleDistanceSensorFactGroup.cc:21-61`; `src/FlyView/ProximityRadarValues.qml:7-19,30`; `src/FlightMap/MapItems/ProximityRadarMapView.qml:13`; `src/FlyView/ProximityRadarVideoView.qml:12` |
| RANGEFINDER (ArduPilot, 173) | Evet (iki yerde) | (1) `VehicleFactGroup::_handleRangefinder` (tüm araçlar) (2) `ArduSubFirmwarePlugin::_handleMavlinkMessage` (sadece ArduSub) | `distance` (NaN ise 0). `voltage` okunmuyor | Fact grid: grup "Vehicle" > `RangeFinderDist` (metadata yok, birim/etiket yok). ArduSub'da ayrıca "Submarine/Info" grubu `rangefinderDistance` | `src/Vehicle/FactGroups/VehicleFactGroup.cc:100-101,238-246`; `src/Vehicle/FactGroups/VehicleFactGroup.h:24,106`; `src/FirmwarePlugin/APM/ArduSubFirmwarePlugin.cc:261-267`; `src/Vehicle/FactGroups/SubmarineFact.json:43-48` |
| NAMED_VALUE_FLOAT | Sadece ArduSub'da Fact'e; diğer araçlarda yalnız MAVLink Inspector | `ArduSubFirmwarePlugin::_handleNamedValueFloat` ; Inspector: `QGCMAVLinkMessage::_extractDebugInstanceValue` | ArduSub: `name` ∈ {CamTilt, TetherTrn, Lights1, Lights2, PilotGain, InputHold, RollPitch, RFTarget}, `value`. Inspector: `name` instance anahtarı, `value` çizilebilir alan | ArduSub: "Submarine/Info" Fact'leri. Plane: sadece Analyze > MAVLink Inspector (liste + chart) | `src/FirmwarePlugin/APM/ArduSubFirmwarePlugin.cc:229-253,258-260`; `src/AnalyzeView/MAVLinkInspector/MAVLinkMessage.cc:152-160,346-356` |
| AIS_VESSEL | Hayır | `src/` içinde hiçbir referans yok | - | Yok. Sadece Inspector'da ham görünebilir (mesaj dialect'te var: mavlink@c409cf69 `common.xml:7639`) | grep: `src` içinde `AIS_VESSEL` sıfır sonuç |
| ESC_TELEMETRY_1_TO_4 (ArduPilot) | Hayır | `src/` içinde hiçbir referans yok. Benzer ama farklı mesajlar ESC_INFO ve ESC_STATUS işleniyor | (ESC_STATUS: `index`, `rpm[]`, `current[]`, `voltage[]`; ESC_INFO: `index`, `count`, `connection_type`, `info`, `failure_flags[]`, `error_count[]`, `temperature[]`) | ESC_INFO/ESC_STATUS için: üst toolbar `EscIndicator`. ESC_TELEMETRY_1_TO_4 için yok, sadece Inspector | `src/Vehicle/FactGroups/EscStatusFactGroupListModel.cc:18-32,79-90,93-114,116-132`; `src/Toolbar/EscIndicator.qml:14-55`; `src/FirmwarePlugin/FirmwarePlugin.cc:209` |
| EKF_STATUS_REPORT (ArduPilot) | Hayır | `src/` içinde referans yok. Benzer standart mesaj ESTIMATOR_STATUS işleniyor | (ESTIMATOR_STATUS: `flags` bitleri + `vel_ratio`, `pos_horiz_ratio`, `pos_vert_ratio`, `mag_ratio`, `hagl_ratio`, `tas_ratio`, `pos_horiz_accuracy`, `pos_vert_accuracy`) | ESTIMATOR_STATUS: sadece Fact grid, grup "EstimatorStatus" (özel QML yok). EKF_STATUS_REPORT: sadece Inspector | `src/Vehicle/FactGroups/VehicleEstimatorStatusFactGroup.cc:29-62` |
| VIBRATION | Evet | `VehicleVibrationFactGroup::handleMessage` | `vibration_x/y/z`, `clipping_0/1/2` | Analyze > Vibration sayfası (gauge + clip sayaçları); Fact grid grup "Vibration" | `src/Vehicle/FactGroups/VehicleVibrationFactGroup.cc:19-38`; `src/AnalyzeView/Vibration/VibrationPage.qml:17,21-23,189-197`; `src/API/QGCCorePlugin.cc:107-110` |
| RADIO_STATUS | Evet | `RadioStatusFactGroup::_handleRadioStatus` (+ sysid/link istisnaları) | `rssi`, `remrssi`, `noise`, `remnoise`, `rxerrors`, `fixed`, `txbuf` | Üst toolbar `TelemetryRSSIIndicator` (popup); Fact grid grup "RadioStatus" | `src/Vehicle/FactGroups/RadioStatusFactGroup.cc:18-59`; `src/Toolbar/TelemetryRSSIIndicator.qml:19-79`; `src/FirmwarePlugin/FirmwarePlugin.cc:204` |
| WIND (ArduPilot, 168) | Evet | `VehicleWindFactGroup::_handleWind` | `direction` (negatifse +360), `speed`, `speed_z` | Sadece Fact grid, grup "Wind". Özel QML göstergesi yok | `src/Vehicle/FactGroups/VehicleWindFactGroup.cc:32-34,80-95` |
| WIND_COV | Evet | `VehicleWindFactGroup::_handleWindCov` | `wind_x`, `wind_y` (yön ve hız hesaplanır), `wind_z` | Aynı "Wind" grubu | `src/Vehicle/FactGroups/VehicleWindFactGroup.cc:23-25,61-78` |
| SYS_STATUS | Evet | `Vehicle::_handleSysStatus` + `SysStatusSensorInfo::update` | `onboard_control_sensors_present/enabled/health` (üç bit alanı). Diğer alanlar bu fonksiyonda yok (doğrulanmadı: başka yerde okunuyor mu bakılmadı, istenmedi) | Üst toolbar "Main status" göstergesi + popup "Sensor Status"; PreFlight checklist "Sensors"; VehicleWarnings (GPS) ; GuidedActions (estimator origin) | `src/Vehicle/Vehicle.cc:639-641,1073-1129`; `src/MAVLink/SysStatusSensorInfo.cc:18-106`; `src/Toolbar/MainStatusIndicator.qml:85-100,263-284`; `src/FlyView/PreFlightSensorsHealthCheck.qml:8-33` |
| HOME_POSITION | Evet | `Vehicle::_handleHomePosition` | `latitude`, `longitude`, `altitude` (mm→m, AMSL). Diğer alanlar (x,y,z,q,approach) okunmuyor | Doğrudan harita işareti yok; `Vehicle.homePosition` mesafe/yön fact'lerini besler, harita fit ve gimbal ROI kullanır | `src/Vehicle/Vehicle.cc:595-597,1199-1213,2615-2619`; `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:298-299,429` |
| MISSION_CURRENT | Evet | `MissionManager::_handleMissionCurrent` | sadece `seq` | Fly View'da aktif waypoint vurgusu; `missionItemIndex` Fact'i (grid); heading-to-next-WP hesabı; Continue Mission | `src/MissionManager/MissionManager.cc:223-225,267-272`; `src/MissionManager/MissionController.cc:1739-1751`; `src/Vehicle/Vehicle.cc:248-249,2647-2657` |

#### Detaylar ve notlar

**DISTANCE_SENSOR**
- Hangi orientation: `MAV_SENSOR_ROTATION_NONE, YAW_45, YAW_90, YAW_135, YAW_180, YAW_225, YAW_270, YAW_315, PITCH_90, PITCH_270` (`VehicleDistanceSensorFactGroup.cc:37-48`). Bunların dışındaki orientation değerleri sessizce atlanır (döngü eşleşme bulamaz, `:50-55`). Aynı orientation'a ait birden fazla sensör (`id` farklı) olsa bile `id` okunmadığı için son gelen mesaj değeri ezer.
- Aşağı bakan rangefinder (PITCH_270): Fact olarak VAR. `rotationPitch270`, açıklaması "Down", birim m, 2 ondalık (`src/Vehicle/FactGroups/DistanceSensorFact.json:79-85`). Fly View'da hazır bir değer/yazı olarak gösterilmiyor: `ProximityRadarValues.qml` sadece None + 7 yaw orientation'ı okur, pitch90/pitch270 yok (`src/FlyView/ProximityRadarValues.qml:11-19,30-31`). Yani Fly View'da görmek için Fact grid'e (Instrument panel) "DistanceSensor > RotationPitch270" değeri elle eklenmelidir. `grep rotationPitch` sonucu QML'de sıfır; sadece JSON ve C++ tarafında geçiyor.
- Proximity overlay ilişkisi: radar katmanı `vehicle.distanceSensors` grubundan beslenir. Harita katmanı `FlyViewMap.qml:274-282`, video katmanı `FlyViewVideo.qml:157-160` içinde. Görünürlük koşulu `telemetryAvailable` (`ProximityRadarMapView.qml:13`, `ProximityRadarVideoView.qml:12`). `telemetryAvailable` herhangi bir DISTANCE_SENSOR gelince true olur (`VehicleDistanceSensorFactGroup.cc:60`), orientation'dan bağımsız. Sonuç: sadece PITCH_270 sensörü olan bir WIG aracında radar katmanı yine de belirir (sektörler boş/şeffaf kalır, `ProximityRadarMapView.qml:62-64`) ve harita katmanında `maxDistance`'a göre beyaz "detection limit" dairesi çizilir (`ProximityRadarMapView.qml:137-144`). `min/maxDistance` her DISTANCE_SENSOR mesajında (orientation fark etmeksizin) üzerine yazılır (`VehicleDistanceSensorFactGroup.cc:57-58`); gözlenen etki, aşağı bakan sensörün max değerinin dairenin yarıçapını belirlemesidir (kod okumasından çıkarım; çalıştırılarak doğrulanmadı).
- Video katmanı varsayılan 6 m ölçek kullanır (`ProximityRadarVideoView.qml:15`); alçak irtifa WIG için bu radar aşağı sensörü çizmediğinden anlamlı değil.
- OBSTACLE_DISTANCE ayrı bir yol: `Vehicle::_handleObstacleDistance` → `_objectAvoidance->update` (`src/Vehicle/Vehicle.cc:681-682,2859-2864`), overlay'ler `ObstacleDistanceOverlayMap/Video` (`src/FlyView/FlyViewMap.qml:234`, `src/FlyView/FlyViewVideo.qml:162`). DISTANCE_SENSOR ile kod paylaşımı yok.
- FactGroup güncelleme periyodu 1000 ms (`VehicleDistanceSensorFactGroup.cc:5`). UI değerleri bu hızda yenilenir, `VehicleFactGroup` ise 100 ms (`VehicleFactGroup.cc:11`). Bu, FactGroup kurucusundaki `updateRateMsecs` argümanından okundu; davranışı çalıştırarak doğrulamadım.

**RANGEFINDER**
- `VehicleFactGroup` içinde `rangeFinderDist` Fact'i var (`VehicleFactGroup.h:24,106`); `VehicleFact.json` içinde bu isim için kayıt yok (tüm `name` alanları `src/Vehicle/FactGroups/VehicleFact.json` içinde tarandı; `rangeFinderDist` geçmiyor). Sonuç: kısa açıklama (`shortDescription`) boş, birim yok; `Fact::shortDescription` metadata yoksa boş string döner. Grid'e eklenince etiketi elle yazmak gerekir (`InstrumentValueEditDialog.qml:44-72` etiketi `fact.shortDescription`'dan doldurur).
- Değer ham olarak metre varsayımıyla yazılıyor (`distance` doğrudan, `VehicleFactGroup.cc:243`); NaN ise 0 yapılır, yani "geçersiz" ile "0 m" ayırt edilemez, WIG için dikkat.
- Hiçbir QML dosyası `rangeFinderDist`'i doğrudan kullanmıyor (grep sadece C++ tanımlarını buldu).
- ArduPlane/Copter plugin'lerinde RANGEFINDER'a özel kod yok; sadece ArduSub'da ek Fact var (`ArduSubFirmwarePlugin.cc:261-267`).
- QGC `src/` içinde rangefinder parametreleri (`RNGFND*`) için özel QML/C++ yok (grep: .qml/.cc/.h/.json içinde sonuç yok).
- Firmware'in rangefinder'ı DISTANCE_SENSOR mu RANGEFINDER mı olarak yayınladığı `src/` ile doğrulanamaz: doğrulanmadı.

**NAMED_VALUE_FLOAT: Fact olarak grid'e eklenebilir mi?**
- Genel (ArduPlane dahil) yol yok. Mesaj sadece `ArduSubFirmwarePlugin` içinde, sabit 8 isim için, ArduSub'a ait Fact'lere yazılıyor (`ArduSubFirmwarePlugin.cc:229-253`, `adjustIncomingMavlinkMessage` içinden `:273-277`). Genel bir "isme göre dinamik Fact" mekanizması `src/` içinde yok (grep: NAMED_VALUE sadece ArduSub plugin, MAVLink Inspector ve MockLink içinde). Dolayısıyla Plane'de NAMED_VALUE_FLOAT değerleri Fact grid'e eklenemez.
- Inspector/plot: evet. Inspector her mesaj için `name` alanını instance anahtarı yapar, yani her `name` ayrı satır olur (`MAVLinkMessage.cc:77-83,152-160`); `value` (float) alanı seçilebilir ve grafiğe çizilebilir (float alan `MAVLinkMessage.cc:346-356`; `_selectable = true` varsayılan `MAVLinkMessageField.h:64`; chart'lar `MAVLinkInspectorPage.qml:323-336`).
- Not: ArduSub kodu `name`'i `QString(value.name)` ile okur (`ArduSubFirmwarePlugin.cc:234`); bu araçta kullanılmıyor.

**AIS_VESSEL**: `src/` içinde hiçbir işleyici/QML/UI yok (grep `AIS_VESSEL|ais_vessel` sıfır; `AIS_` yalnızca `src/FirmwarePlugin/APM/Rover.OfflineEditing.params:26` içinde bir parametre). Yakın alternatif olarak ADSB_VEHICLE ayrı bir yoldan işleniyor: `src/Vehicle/Vehicle.cc:663-664` (`ADSBVehicleManager`).

**ESC_TELEMETRY_1_TO_4**: `src/` içinde yok. ESC verisi için QGC ESC_INFO + ESC_STATUS kullanıyor ve her 4'lü grup için dinamik FactGroup üretiyor (`EscStatusFactGroupListModel.cc:11-52` içinde `_shouldHandleMessage`, index..index+3). Bu gruplar `Vehicle.escs` listesi üzerinden toolbar'da (`EscIndicator.qml:14-17`) gösteriliyor; `_addFactGroup` ile eklenmedikleri için (`Vehicle.cc:345-362` listesinde yok) Fact grid'in "Group" seçicisinde görünmedikleri çıkarımı yapılıyor (grid kodu `InstrumentValueData.cc:325-336` yalnızca `Vehicle::factGroupNames()`'i listeler); çalıştırılarak doğrulanmadı. Yan not: `_shouldHandleMessage` içindeki `ESC_INFO` dalı `mavlink_esc_status_t` ile decode ediyor (`EscStatusFactGroupListModel.cc:19-24`). Sonucu için bkz. "Kod okurken görülen olası hatalar".

**EKF_STATUS_REPORT**: `src/` içinde yok. ESTIMATOR_STATUS gruplaması var ama ArduPilot'ın EKF_STATUS_REPORT'u ile aynı mesaj değil; QGC'nin EKF flag/variance'ını ayrı bir mesajdan gösterdiği doğrulanmadı. Not: `EstimatorStatusFactGroup.json:7` içinde `"goodAttitudeEsimate"` yazım hatası var (C++ tarafı `goodAttitudeEstimate`), yani o Fact'in metadata'sı bağlanmayabilir (`FactGroup.cc:124` isimle eşler; sonuç doğrulanmadı).

**VIBRATION**: Vibration sayfasının "available" koşulu `xAxis` NaN değil olması (`VibrationPage.qml:17`). Sayfa Analyze menüsüne `QGCCorePlugin.cc:107-110` ile eklenir.

**RADIO_STATUS**: SiK radyolar için sysid `'3'`/compid `'D'` özel dönüşümü (`RadioStatusFactGroup.cc:42-44`); diğerleri int8 dBm kabul edilir (`:46-47`). RADIO_STATUS araç sysid'i ile eşleşmese bile bağlantı aracın kullandığı bir link ise geçirilir (`Vehicle.cc:529`); `VehicleLinkManager` bu mesajı "araç canlı" sinyali saymaz (`VehicleLinkManager.cc:37-40`). TelemetryRSSIIndicator `lrssi != 0` ise görünür (`TelemetryRSSIIndicator.qml:20`).

**WIND / WIND_COV**: Hiçbir QML `vehicle.wind`'i kullanmıyor (grep `.wind` QML sonuç yok). Fact isimleri `direction`, `speed`, `verticalSpeed`, birimler deg ve m/s (`src/Vehicle/FactGroups/WindFact.json:7-25`). Yön WIND'de doğrudan `direction`; WIND_COV'da `atan2(wind_y, wind_x)` (`VehicleWindFactGroup.cc:66-70`). Açı konvansiyonlarının uyumu (ArduPilot WIND yönü ile atan2 sonucu aynı anlamda mı) `src/` ile doğrulanamaz: doğrulanmadı. HIGH_LATENCY/HIGH_LATENCY2 de aynı gruba yazar (`:26-30`).

**SYS_STATUS bitleri**
- `_handleSysStatus` sadece `_defaultComponentId`'den gelen mesajı işler (`Vehicle.cc:1075-1077`).
- present/enabled/health: `_onboardControlSensors*` olarak saklanır, değişince `sensorsPresentBitsChanged` vb. sinyaller (`Vehicle.cc:1103-1115`); `unhealthy = enabled & ~health` (`:1124-1128`).
- `allSensorsHealthy = (enabled & health) == enabled` (`:1097-1101`). ArduPilot dahil, `PREARM_CHECK` enabled ise `readyToFly` buradan (`:1084-1095`).
- Sınıf: `SysStatusSensorInfo` (`src/MAVLink/SysStatusSensorInfo.cc:18-54`); sadece present bit'li sensörleri listeler, "Error / Normal / Disabled" sıralar (`:56-106`). Bit adları `QGCMAVLink::mavSysStatusSensorToString` içinde (`src/MAVLink/QGCMAVLink.cc:328-379`), örn. `LASER_POSITION` -> "Laser based position" (`:344`), `SENSOR_PROXIMITY` -> "Proximity" (`:362`), `TERRAIN` (`:358`). Tabloda olmayan bit "Unknown sensor" olur (`:376-378`).
- UI: (1) `MainStatusIndicator.qml:85-100` renk/metin (yeşil/sarı) `readyToFly`/`allSensorsHealthy`'den; (2) popup "Sensor Status" grubu SADECE `!_healthAndArmingChecksSupported` iken görünür (`MainStatusIndicator.qml:263-266,16`); HealthAndArmingCheck destekleyen firmware'de bu liste yerine başka panel gösterilir; (3) `PreFlightSensorsHealthCheck.qml:11-17` yalnız MAG, ACCEL, GYRO, ABS_PRESSURE, DIFF_PRESSURE, GPS, AHRS bitlerini kontrol eder, rangefinder/LASER_POSITION bitleri hiçbir QML kontrolünde yok; (4) `VehicleWarnings.qml:15` ve `GuidedActionsController.qml:143` GPS bitine bakar.
- WIG için sonuç: rangefinder sağlığı SYS_STATUS'ta hangi bitle bildirilirse bildirilsin QGC'de yalnızca yukarıdaki popup listesinde (HealthAndArmingCheck desteklenmiyorsa) ve genel `allSensorsHealthy` rengini etkileyerek görünür; rangefinder'a özel uyarı/pre-flight yok (kod taramasından; hangi bitin kullanıldığı firmware tarafı, doğrulanmadı).

**HOME_POSITION**: sadece `_defaultComponentId`'den (`Vehicle.cc:1201-1203`). ArduPilot'ta QGC 1 Hz aralık için `MAV_CMD_SET_MESSAGE_INTERVAL` ister (`APMFirmwarePlugin.cc:426-429`) ve kayıp tespiti için son geliş zamanını tutar (`:298-299`). Tüketiciler: `Vehicle.cc:2615-2619` (distanceToHome, headingToHome, headingFromHome Fact'leri), `MapFitFunctions.qml:23`, `CenterMapDropButton.qml:33`, `GimbalIndicator.qml:203`, `Viewer3D...`, `APMFirmwarePlugin.cc:1171,1199`, `TerrainQueryCoordinator.cc:123`. Fly View haritasında `vehicle.homePosition` ile çizilen doğrudan bir home işareti `src/` QML grep'inde bulunmadı (home işareti plan/mission öğesi üzerinden; doğrulanmadı).

**MISSION_CURRENT**: sadece `seq` okunur (`MissionManager.cc:267-272`) ve `_updateMissionIndex` (`:233-251`) ile `currentIndexChanged` yayınlar. Alınan indeks `MissionController::_currentMissionIndexChanged` ile Fly View'daki mission öğelerinin `isCurrentItem` bayrağına (`MissionController.cc:1739-1751`; QML `src/FlightMap/MapItems/MissionItemIndicator.qml:23,33`), `Vehicle::_updateMissionItemIndex` ile `missionItemIndex` Fact'ine (`Vehicle.cc:2647-2657`; JSON `VehicleFact.json:140`) ve `GuidedActionsController.qml:133,168` (Continue Mission) koşuluna gider. HIGH_LATENCY/HIGH_LATENCY2 `wp_num` de aynı yola girer (`MissionManager.cc:253-265`).

#### Grid'e (Instrument panel) Fact eklenebilirliği, ortak mekanizma
Fact grid'in "Group" seçicisi `Vehicle::factGroupNames()` + "Vehicle" grubunu listeler (`src/QmlControls/InstrumentValueData.cc:325-336`), "Value" listesi seçili grubun `factNames()`'idir (`:338-355`). Vehicle.cc'de eklenen gruplar: gps, gps2, gpsAggregate, wind, vibration, temperature, clock, setpoint, distanceSensor, localPosition, localPositionSetpoint, estimatorStatus, hygrometer, generator, efi, rpm, terrain, radioStatus (`src/Vehicle/Vehicle.cc:345-362`) + firmware plugin grupları (`:365-370`). Bu yüzden DISTANCE_SENSOR (her orientation + min/max), RANGEFINDER (Vehicle grubu), WIND, VIBRATION, ESTIMATOR_STATUS ve RADIO_STATUS alanları Fact grid'e eklenebilir. NAMED_VALUE_FLOAT (Plane), AIS_VESSEL, ESC_TELEMETRY_1_TO_4, EKF_STATUS_REPORT eklenemez.

#### MAVLink Inspector'da ham görünürlük
Genel cümle: MAVLink Inspector, işlenip işlenmediğine bakmaksızın `MAVLinkProtocol::messageReceived` sinyalinden gelen her mesajı (`src/AnalyzeView/MAVLinkInspector/MAVLinkInspectorController.cc:45-46,197-224`), sysid/compid/msgid (ve varsa instance alanı) bazında listeler, alanları `mavlink_get_message_info()` meta verisiyle ham değer olarak gösterir ve seçilen sayısal alanları 2 chart'ta çizer (`src/AnalyzeView/MAVLinkInspector/MAVLinkMessage.cc:239-260`, `MAVLinkInspectorPage.qml:323-336`). Menüde Analyze > "MAVLink Inspector" (`src/API/QGCCorePlugin.cc:102-106`, bir araç bağlı olmalı: `requiresVehicle = true`). Mesaj dialect'te tanımlı değilse (`mavlink_get_message_info` NULL) alanlar doldurulamaz (`MAVLinkMessage.cc:241-245`, uyarı log'u). Projenin varsayılan dialect'i `all` (`cmake/CustomOptions.cmake:125`); `all` dialect'i `ardupilotmega.xml` ve `common.xml`'i içerir (mavlink@c409cf69 `all.xml:7,18`). Bu commit'te AIS_VESSEL (`common.xml:7639`, id 301), ESC_TELEMETRY_1_TO_4 (`ardupilotmega.xml:1938`, id 11030), EKF_STATUS_REPORT (`ardupilotmega.xml:1732`, id 193), RANGEFINDER (`ardupilotmega.xml:1568`, id 173) ve WIND (`ardupilotmega.xml:1538`, id 168) tanımlı. Bu yüzden bunlar Inspector'da alanlarıyla birlikte çözümlenebilir. Bu mesajlar araca gelmesi (ArduPilot'un onları yayınlaması, v2 çerçeve) koşuluyla Inspector'da görünür. Yayın hızı için Inspector'dan `setMessageInterval` ile aralık ayarlanabiliyor (`MAVLinkInspectorController.cc:235-`).

Not: ArduPilot stream istekleri `MAV_DATA_STREAM_*` ile 7 stream üzerinden yapılır (`APMFirmwarePlugin.cc:395-420`; varsayılan hızlar `src/Settings/APMMavlinkStreamRate.SettingsGroup.json`, örn. ExtendedStatus 2 Hz `:21-28`, Extra1 10 Hz `:51-58`). Hangi mesajın hangi stream'de olduğu firmware tarafındadır, `src/` ile doğrulanamaz: doğrulanmadı.


Kaynak: QGroundControl v5.1.5 (3a67d31f0c36bf3fe38ec52970d250a89d0aaf67). Yollar repo köküne göredir. Yalnızca kodda görülenler yazıldı, belge/forum bilgisi kullanılmadı.

## F. Veri bayatlığı: telemetri gecikince ya da kesilince

#### F.1 Mekanizma özeti

- Comm lost tespiti **link başına** tutulur (`LinkInfo_t.commLost` ve link başına `heartbeatElapsedTimer`, `src/Vehicle/VehicleLinkManager.h:67-71`), ama UI'ya çıkan `communicationLost` **araç başına** tek bir bool'dur (`VehicleLinkManager.h:79`, her `Vehicle` kendi `VehicleLinkManager`'ına sahip: `src/Vehicle/Vehicle.cc:271`). Araç düzeyinde "comm lost" ancak o araca ait **tüm** link'ler comm lost olunca true olur (`VehicleLinkManager.cc:138-157`); herhangi bir link geri gelince false olur (`VehicleLinkManager.cc:91-97`).
- Zamanlayıcıyı "heartbeat" değil, **o araca ait herhangi bir MAVLink mesajı** sıfırlar: `Vehicle::_mavlinkMessageReceived` her mesajı `VehicleLinkManager::mavlinkMessageReceived`'a verir (`src/Vehicle/Vehicle.cc:535`), orada yalnızca `RADIO_STATUS` hariç tutulur (`VehicleLinkManager.cc:37-40`) ve `heartbeatElapsedTimer.restart()` çağrılır (`VehicleLinkManager.cc:49`). Yani adı "heartbeat" olsa da fiilen "3.5 sn boyunca araçtan hiç mesaj gelmedi" kontrolüdür. Başka sysid'den gelen mesajlar zaten `Vehicle.cc:524-532`'de elenir (RADIO_STATUS hariç).
- Kontrol 1 Hz bir `QTimer` ile yapılır, bu yüzden pratik tespit gecikmesi 3.5 sn + en fazla 1 sn.
- Yüksek gecikmeli (HIGH_LATENCY) link'ler timeout kontrolünden muaftır (`VehicleLinkManager.cc:111`, `:190`); WIG için geçerli değilse önemsiz.
- Kontrol, `communicationLostEnabled == false` iken tamamen atlanır (`VehicleLinkManager.cc:102-104`). Bunu kapatanlar: APM sensör kalibrasyonu (`src/AutoPilotPlugins/APM/APMSensorsComponentController.cc:298,344,355,364,373`) ve onboard log listeleme/indirme (`src/AnalyzeView/OnboardLogs/OnboardLogController.cc:922,933`). Yani log indirirken bağlantı kopsa bile "Comms Lost" gösterilmez.
- `autoDisconnect` true ise (yalnızca firmware upgrade sayfası ayarlıyor: `src/Vehicle/VehicleSetup/FirmwareUpgrade.qml:166,552`) toplam kayıpta araç kapatılır, comm lost bayrağı kalkmaz (`VehicleLinkManager.cc:147-151`). Varsayılan false (`VehicleLinkManager.h:81`).
- Comm lost, araç nesnesini **silmez**: araç, link `disconnected` sinyali gelince ya da kullanıcı Disconnect'e basınca (`_activeVehicle.closeVehicle()`, `src/Toolbar/FlyViewToolBar.qml:96`) silinir (`VehicleLinkManager.cc:231-249`, `:346-362`).

#### F.2 Tablo

| durum | süre/eşik | görsel | ses | dosya:satır |
|---|---|---|---|---|
| Link başına mesaj/heartbeat zaman aşımı | `_heartbeatMaxElpasedMSecs = 3500` ms; kontrol aralığı `_commLostCheckTimeoutMSecs = 1000` ms. Unit test'te 1500 / 250 ms | Tek link varsa kendi başına ayrı görsel yok (aşağıdaki toplam kayıpla aynı). `linkStatuses` listesinde o link için "Comm Lost" metni (link listesi UI'sı bu çalışmada incelenmedi: doğrulanmadı) | Yalnızca birden fazla link varsa: "Communication lost on primary/secondary link." (küçük harfe çevrilip okunur) | `src/Vehicle/VehicleLinkManager.h:84-85`, `:88-98`; `src/Vehicle/VehicleLinkManager.cc:107`, `:111-120`, `:412` |
| Toplam comm lost (aracın tüm link'leri sustu) | Aynı 3.5 sn; `_communicationLost = true` | Üst çubuk (toolbar) ana durum etiketi "Comms Lost" olur ve arka plan gradient rengi `"red"` olur; "Disconnect" butonu çıkar. Bkz. F.3 | "Communication lost" (araç sayısı > 1 ise "Vehicle N " öneki) | `VehicleLinkManager.cc:138-157` (ses: `:153`); `src/Toolbar/MainStatusIndicator.qml:37,47-51`; `src/Toolbar/FlyViewToolBar.qml:18,93-98` |
| Comm regained | İlk gelen mesajda anında | Durum etiketi normal akışa döner (Armed/Flying/Ready...), Disconnect butonu kaybolur | Tek link: "Communication regained"; çok link: "Communication regained on primary/secondary link"; primary değişirse ek olarak "Switching communication to new primary link" (+ ekranda app message) | `VehicleLinkManager.cc:50-52`, `:55-98` (ses `:81`, `:85`; mesaj `:86`) |
| Primary/secondary link geçişi (otomatik) | Her kontrol turunda ve comm regained'de `_updatePrimaryLink()`; mevcut primary comm lost olunca en iyi aktif link seçilir | `QGC::showAppMessage` ile "Switching communication to secondary link." mesajı | Aynı mesaj okunur | `VehicleLinkManager.cc:128-132`, `:304-344` |
| Primary link seçim önceliği | 1) comm lost olmayan USB direct seri link, 2) comm lost olmayan normal gecikmeli link, 3) mevcut high-latency primary, 4) herhangi high-latency | Yok | Yok | `VehicleLinkManager.cc:251-302` (`:255-265`, `:269-279`, `:282-286`, `:289-299`); yalnızca high-latency link'lerde `MAV_CMD_CONTROL_HIGH_LATENCY` gönderilir: `:324-341` |
| Primary link'in elle seçilmesi | Anında | Link adıyla; `primaryLinkChanged` | Yok | `VehicleLinkManager.cc:386-394` |
| Link kapanması (seri port çekildi, TCP/UDP kapandı) | Link `disconnected` sinyali | Son link giderse araç silinir (`allLinksRemoved`) ve toolbar "Disconnected - Click to manually connect" olur | Bu yolda ses kodu bulunamadı (yalnızca VehicleLinkManager.cc içinde baktım) | `VehicleLinkManager.cc:197`, `:231-249`; `MultiVehicleManager.cc:117`; `MainStatusIndicator.qml:104-106` |
| Video tam ekranı | comm lost olunca | Video tam ekrandan çıkar, comm lost iken tam ekrana alınamaz | Yok | `src/VideoManager/VideoManager.cc:486-488`, `:777-781` |
| FlyView mission-complete diyalog düğmeleri | comm lost iken | Bazı düğmeler gizlenir | Yok | `src/FlyView/FlyViewMissionCompleteDialog.qml:82,106` |
| Tek tek Fact bayatlığı ("stale"/geçersiz gösterim) | **Kodda bulunamadı** | Fact'ler son değerlerinde **donuk** kalır, gri/soluk/"--" olmaz | Yok | Bkz. F.4 |
| RADIO_STATUS (SiK telemetri RSSI) | Zaman aşımı yok | Son değer kalır | Yok | Bkz. F.5 |

#### F.3 Fly View / toolbar'da comm lost'ta görsel olarak ne değişiyor (kodda görülen tek şey)

- `MainStatusIndicator.qml:45-51`: `_communicationLost` ise metin `qsTr("Comms Lost")` (`:37`) ve `_mainStatusBGColor = "red"`. Bu renk `FlyViewToolBar.qml:19,53` içindeki sol gradient arka plana (`GradientStop { position: 0; color: _mainStatusBGColor }`) gider. Tüm durum etiketi öncelik sırasında comm lost ilk koşuldur, yani armed/flying bilgisinin önüne geçer.
- `FlyViewToolBar.qml:93-98`: "Disconnect" düğmesi yalnızca `_activeVehicle && _communicationLost` iken görünür.
- Başka toolbar öğesi (FlightModeIndicator, GPS/Battery/RSSI göstergeleri) comm lost'a bağlanmamış: `src/Toolbar` ve `src/FlyView` altında `communicationLost` aramasının sonuçları yalnızca yukarıdaki iki dosya + `FlyViewMissionCompleteDialog.qml` + `FlightDisplayViewVideo.qml:20`. Bu göstergelerin değerleri gri olmaz, son değerde donuk kalır (QML'de bayatlık koşulu bulunamadı).
- Not, olası ölü kod/hata: `src/FlyView/FlightDisplayViewVideo.qml:20` `globals.activeVehicle.communicationLost` okuyor, ama `communicationLost` `Vehicle`'da değil `VehicleLinkManager`'da tanımlı (`VehicleLinkManager.h:23`; `src/Vehicle/Vehicle.h` içinde bu isimde property yok). Aynı dosyada `_connected` başka yerde kullanılmıyor (grep tek satır). Etkisi yok görünüyor ama sonuç "doğrulanmadı" (QML çalıştırılmadı).
- Konuşan ses ayrıntısı D bölümünde; burada yalnızca: `AudioOutput::say` ses seviyesi 0 ya da muted ise hiçbir şey okumaz (`src/Utilities/Audio/AudioOutput.cc:234-236`), yani ses tamamen kullanıcı ayarına bağlı.

#### F.4 Fact düzeyinde bayatlık: baktığım yerler ve sonuç

Baktığım yerler: `src/FactSystem/FactGroup.{h,cc}`, `src/FactSystem/Fact.cc`, `src/Vehicle/FactGroups/*.cc` (GPS, RadioStatus, Vehicle), `src/Toolbar/VehicleGPSIndicator.qml`, `src/Vehicle/Vehicle.{h,cc}` ("elapsed"/"lastHeartbeat"/"stale" aramaları).

- `FactGroup` zamanlayıcısı **UI güncelleme hız sınırlayıcıdır, bayatlık dedektörü değil**: `_updateTimer` her `updateRateMsecs`'te `sendDeferredValueChangedSignal()` çağırır (`src/FactSystem/FactGroup.cc:39-46`, `:145-150`). Hızlar: `VehicleFactGroup` 100 ms (`src/Vehicle/FactGroups/VehicleFactGroup.cc:11`); `VehicleGPSFactGroup` ve `RadioStatusFactGroup` dahil çoğu grup 1000 ms (`VehicleGPSFactGroup.cc:10`, `RadioStatusFactGroup.cc:7`); `VehicleEstimatorStatusFactGroup` 500 ms (`VehicleEstimatorStatusFactGroup.cc:5`).
- `FactGroup::telemetryAvailable` ("hiç telemetri alınmadı" bayrağı): `_setTelemetryAvailable(true)` ile true olur (`FactGroup.cc:175-181`; örn. `RadioStatusFactGroup.cc:58`) ve **`_setTelemetryAvailable(false)` çağrısı kod tabanında bulunamadı**, yani bir kez true olduktan sonra comm lost'ta false'a dönmez. `showIndicator` olarak kullanan yerler: `VehicleGPSIndicator.qml:9`, `ProximityRadarValues.qml:7`.
- Veri yaşı: GPS mesajının yaşı, "son güncelleme zamanı" ya da "X sn önce" alanı **kodda bulunamadı** (`VehicleGPSFactGroup.cc` içinde `elapsed`/`age` yok; `GPS_RAW_INT` doğrudan `setRawValue`'lar, `:51-52`, `:68-81`).
- "–" gösterimi: `Fact::invalidValueString` yalnızca değer **NaN** ise (örn. hiç veri gelmedi/başlangıç) tek "–" ya da "–.–" üretir (`src/FactSystem/Fact.cc:387-390`, `:398-401`, `:412-413`, `:429-436`). Telemetri kesildiği için Fact NaN'a çekilmez; son geçerli değer kalır (Vehicle/FactGroup tarafında zaman aşımıyla NaN'a döndüren kod bulunamadı).
- Sonuç: Telemetri kesilirse tüm Fact'ler (irtifa, hız, GPS, batarya, vb.) **son değerinde donuk** kalır; tek ipucu üst çubuktaki "Comms Lost" ve kırmızı arka plandır. Bu, WIG gibi hızlı/alçak irtifa araçta operatör için risktir (karar notu: kodda tespit edilen davranış).
- Yan etki: CSV kaydı (`saveCsvTelemetry`) 1 Hz zamanlayıcıyla her saniye Fact'lerin son değerini yazar, comm lost'a bakmaz (`src/Vehicle/Vehicle.cc:172-173`, `:2823-2850`), yani kesinti sırasında bayat değerler yeni zaman damgasıyla CSV'ye yazılır. Tlog ise yalnızca gelen ham paketleri yazar (G.2).

#### F.5 RADIO_STATUS ve link kalitesi / paket kaybı

| özellik | ne yapıyor | UI'da nerede | dosya:satır |
|---|---|---|---|
| RADIO_STATUS alanları | rssi, remrssi, rxerrors, fixed, txbuf, noise, remnoise Fact'lere yazılır. SiK ("3D") için dBm dönüşümü | Toolbar'da `TelemetryRSSIIndicator` (ikon yalnızca `lrssi != 0` iken görünür); tıklayınca Local/Remote RSSI (dBm), RX Errors, Errors Fixed, TX Buffer, Local/Remote Noise listesi. **Renk/gri durumu ve zaman aşımı yok** | `src/Vehicle/FactGroups/RadioStatusFactGroup.cc:27-59`; `src/Toolbar/TelemetryRSSIIndicator.qml:16-20`, `:44-80`; araç araç listesi: `src/FirmwarePlugin/FirmwarePlugin.cc:204` |
| RADIO_STATUS comm lost'ı etkiler mi | Hayır: "SiK radyodan gelir, karşı uçta hayat olduğunu göstermez" diye heartbeat zamanlayıcısından hariç | - | `src/Vehicle/VehicleLinkManager.cc:37-40`; ayrıca farklı sysid'den gelen RADIO_STATUS yine de araca iletilir: `src/Vehicle/Vehicle.cc:526-532` |
| MAVLink sequence ile paket kaybı (link/kanal bazlı) | `_updateCounters`: kanal başına toplam alınan ve kayıp sayacı; kayıp = sıra numarası atlaması (mod 256). Yalnızca **MAVLink v2** paketlerde sayılır (`!isV1`). Aynı seq tekrarı (v1/v2 çifti) kayıp sayılmaz. Yüzde: `(anlık yüzde + önceki yüzde) * 0.5` ile yumuşatılmış hareketli ortalama | **Toolbar/Fly View'de gösterilmiyor.** Yalnızca Ayarlar > Telemetry sayfasında "Link Status (Current Vehicle)": toplam gönderilen (hesaplanan), toplam alınan, toplam kayıp, "Loss rate" (`toFixed(0) + '%'`), signing durumu | `src/Comms/MAVLinkProtocol.cc:135-139`, `:152-183` (`:176-182` yüzde); yayın her 31. mesajda: `:284-287`; `src/Vehicle/Vehicle.cc:2719-2727` (yalnızca `uasId == _systemID`); `src/AppSettings/MavlinkLinkStatus.qml:14-36`; `src/AppSettings/pages/Telemetry.SettingsUI.json:91` |
| Eski araç düzeyi kayıp sayacı | `Vehicle::_mavlinkMessageReceived` içinde aynı mantık `_messagesLost` ile (v1 dahil), "mavlinkLoss*"tan bağımsız. UI'da yalnızca ESP8266 kurulum sayfasında kullanılıyor. Not: taşma düzeltmesi `seq_received + 255` (256 olması gerekir), olası 1 paket sapması | ESP8266 bileşen sayfası | `src/Vehicle/Vehicle.cc:540-559` (`:552`); `src/AutoPilotPlugins/Common/ESP8266Component.qml:324,365` |
| "Kayıp yüzdesi eşiği aşınca uyarı" | **Kodda bulunamadı** (yalnızca gösterim var; uyarı/ses/renk yok) | - | Baktığım yerler: `MAVLinkProtocol.cc`, `Vehicle.cc`, `src/Toolbar`, `src/FlyView` |

## G. Kayıt: telemetri (tlog), CSV ve onboard log

#### G.1 Ayar tablosu

| ayar | varsayılan | açıklama | dosya:satır |
|---|---|---|---|
| `telemetrySave` | **true** (varsayılan açık) | "Save log after each flight": uçuş bitince tlog'u otomatik kaydet | `src/Settings/Mavlink.SettingsGroup.json:6-13` (`"default": true` satır 10). UI: `src/AppSettings/pages/Telemetry.SettingsUI.json:38` |
| `telemetrySaveNotArmed` | **false** | "Save logs even if vehicle was not armed": hiç arm edilmemiş oturumlar da kaydedilsin | `Mavlink.SettingsGroup.json:15-22` (`"default": false` satır 19). UI: `Telemetry.SettingsUI.json:41` |
| `saveCsvTelemetry` | **false** | Tüm Fact'leri 1 Hz CSV'ye yaz (`<tarih saat> vehicle<sysid>.csv`, tlog ile aynı `Telemetry` dizinine). Yalnızca armed iken, `telemetrySaveNotArmed` açıksa arm olmadan da başlar | `Mavlink.SettingsGroup.json:33-40`; `src/Vehicle/Vehicle.cc:2796-2821`, `:2823-2850`; UI `Telemetry.SettingsUI.json:44` |
| `savePath` (kök dizin) | `""` (boş); çalışma anında oluşturulur | Uygulama kayıt dizini; kullanıcı değiştirmediyse masaüstünde `Documents/<applicationName>`, Android/iOS'ta platform veri dizini | `src/Settings/App.SettingsGroup.json:233-241`; `src/Settings/AppSettings.cc:95-141` (masaüstü dalı `:133-140`, bkz. not) |
| `disableAllPersistence` | **false** | true ise hiçbir şey diske yazılmaz (tlog dahil) | `src/Settings/App.SettingsGroup.json:363-369`; tlog tarafı `src/Comms/MAVLinkProtocol.cc:322-324`, `:374` |
| `sendGCSHeartbeat` | **true** | QGC'nin 1 Hz GCS heartbeat göndermesi (`kGCSHeartbeatRateMSecs = 1000`) | `Mavlink.SettingsGroup.json:68-75`; `src/Vehicle/MultiVehicleManager.h:77`; `MultiVehicleManager.cc:265-269` |

Not (`savePath` satır numaraları): masaüstü varsayılan kök, `QGC::runningUnitTests() || simpleBootTest()` ise Temp, değilse `DocumentsLocation` (`AppSettings.cc:135-139`); `rootDir.filePath(appName)` ile `QCoreApplication::applicationName()` eklenir (`:98`, `:140`). `applicationName` = `QGC_APP_NAME` = "QGroundControl" (`cmake/CustomOptions.cmake:17`, `src/QGCApplication.cc:92`); daily build'de "QGroundControl Daily" (`src/QGCApplication.cc:90`).

#### G.2 Tlog davranışı (kodda görülen)

| konu | bulgu | dosya:satır |
|---|---|---|
| Kayıt var mı | Evet. `MAVLinkProtocol`, alınan her MAVLink mesajını (SETUP_SIGNING hariç, imza bloğu çıkarılarak) ve gönderilen baytları 8 byte big-endian mikrosaniye zaman damgasıyla geçici dosyaya yazar | `src/Comms/MAVLinkProtocol.cc:78-99` (gönderilen), `:224-244` (alınan) |
| Kaydı ne başlatır | İlk gelen HEARTBEAT (ya da HIGH_LATENCY/HIGH_LATENCY2) -> `_startLogging()`. Desktop'ta `telemetrySave` kapalı olsa da geçici dosyaya yazma **başlar** (`telemetrySave` kontrolü `_startLogging`'de yalnızca Android/iOS'ta); karar durdurma anında verilir | `MAVLinkProtocol.cc:253-271`, `:315-365` (`:326-330` yalnızca mobil) |
| Geçici dosya | Temp dizini: `FlightDataXXXXXX.mavlink` (benzersiz yol) | `src/Comms/MAVLinkProtocol.h:97-98`; `MAVLinkProtocol.cc:341-342` |
| "Sadece armed iken mi" | **Kayıt armed beklemeden başlar** (geçici dosyaya hep yazılır), ama **kalıcı kayıt** yalnızca `(arm görüldüyse || telemetrySaveNotArmed) && telemetrySave && !disableAllPersistence` ise yapılır; aksi halde geçici dosya silinir. "Arm görüldü" = herhangi bir alınan HEARTBEAT'in `base_mode` bitlerinden `MAV_MODE_FLAG_DECODE_POSITION_SAFETY` (armed biti) | `MAVLinkProtocol.cc:246-249` (arm tespiti), `:367-382` (karar; `:372-374` koşul, `:377` silme, `:381` bayrak sıfırlama) |
| Kayıt ne zaman biter | Son araç silinince (`MultiVehicleManager::vehicleRemoved` -> sayı 0 -> `_stopLogging`) ya da dosya yazma hatasında. **Comm lost tek başına kaydı bitirmez**; araç Disconnect edilip silinene ya da link kapanana kadar tlog sürer | `MAVLinkProtocol.cc:48-58` (`init`/`vehicleRemoved` bağlantısı), `:519-524`; `src/Vehicle/MultiVehicleManager.cc:169` |
| Kayıt bitince ne oluyor (kaydet/sor) | **Kullanıcıya sorulmaz**. Koşullar sağlanıyorsa otomatik olarak kalıcı dizine kopyalanır, geçici dosya silinir. Hata olursa ekranda uygulama mesajı çıkar | `MAVLinkProtocol.cc:417-497`; hata mesajları `:438-488` |
| Hedef dizin | `AppSettings::telemetrySavePath()` = `<savePath>/Telemetry` (`telemetryDirectory` sabiti çevrilebilir `QT_TRANSLATE_NOOP`, varsayılan "Telemetry"). `savePath` boşsa ya da dizin yoksa kayıt yapılmaz ve hata mesajı verilir | `src/Settings/AppSettings.h:112`; `AppSettings.cc:299-312`, `:329-332`; `MAVLinkProtocol.cc:499-517` |
| Dosya adı formatı | `yyyy-MM-dd hh-mm-ss.tlog`; aynı ada çakışırsa `yyyy-MM-dd hh-mm-ss.1.tlog`, `.2.tlog`, ... Uzantı sabiti `tlog` | `MAVLinkProtocol.cc:423-432`; `src/Settings/AppSettings.h:103` |
| Yazma biçimi | `QSaveFile` (atomik, 256 KiB tamponla), izinler: sahip r/w, grup r, diğer r | `MAVLinkProtocol.cc:445-493` |
| Kapanışta kurtarma | Uygulama açılışında Temp dizinindeki artık `*.mavlink` dosyaları bulunur: boş olanlar silinir, **dolu olanlar armed/NotArmed kontrolü yapılmadan** (ve `telemetrySave`'e bakılmadan) `Telemetry` dizinine kaydedilir | `MAVLinkProtocol.cc:384-401`; çağrı `src/QGCApplication.cc:350` |
| Log replay sırasında | Yeni log yazılmaz | `src/Comms/LogReplayLink.cc:152,168`; `MAVLinkProtocol.h:41` |
| Birden çok araç/link | Tek geçici dosya tüm link ve araçları yazar; oturum son araç silinene kadar sürer (araç başına ayrı dosya yok) | `MAVLinkProtocol.cc:224-244`, `:519-524` |

Çıkarım (WIG için, kodda görülen): varsayılan ayarlarla (`telemetrySave=true`, `telemetrySaveNotArmed=false`) arm edilmemiş yer testleri **kaydedilmez**; sadece bir kez arm görülen oturum, araç Disconnect/link kapanışında `Documents/<app>/Telemetry/yyyy-MM-dd hh-mm-ss.tlog` olarak saklanır.

#### G.3 Onboard log indirme (LogDownload) ve MAVLink log streaming (MAVLinkLogManager) farkı

| | Onboard log indirme | MAVLinkLogManager (MAVLink log streaming) |
|---|---|---|
| Sınıf / dosya | `OnboardLogController` (`src/AnalyzeView/OnboardLogs/OnboardLogController.{h,cc}`, `OnboardLogPage.qml`). v5.1.5'te eski "LogDownloadController" adı kodda bulunamadı, bu sınıf onun yerine var | `src/Vehicle/MAVLinkLogManager.{h,cc}`; `Vehicle` başına bir tane: `src/Vehicle/Vehicle.cc:340`, `:3506-3514` |
| Ne yapar | Araç **üzerindeki** (SD kart) log dosyalarını listeler ve bilgisayara indirir (MAVLink LOG_* mesajları ya da FTP; FTP capability varsa FTP, yoksa mesaj transport'u) | Uçuş sırasında araçtan **canlı akan** log verisini (`LOGGING_DATA` -> ULog) yerelde `.ulg` olarak yazar; ayrıca uzak sunucuya yükleme (upload) seçenekleri var |
| Firmware | Firmware'e bağımsız (kodda firmware kontrolü yok; FTP kapasitesi `MAV_PROTOCOL_CAPABILITY_FTP` ile seçilir) | **Yalnızca PX4**: `px4Firmware()` kontrolleri; ArduPlane'de çalışmaz. UI başlığı "MAVLink 2.0 Logging (PX4 Pro Only)" |
| Kaydedilen yer | `appSettings->logSavePath()` = `<savePath>/Logs` (indirme, dosya adı `.bin` eklenir) | Aynı `logSavePath()` (`<savePath>/Logs`) |
| Comm lost etkisi | Listeleme/indirme sırasında `communicationLostEnabled=false` yapar | Yok (bu sınıfta bu yönde kod aramadım) |
| Tlog ile ilişki | Ayrı mekanizma: aracın kendi dataflash/SD logu | Ayrı mekanizma: ULog; tlog'dan bağımsız |
| Dosya:satır | `OnboardLogController.cc:54-56`, `:466`, `:507-524`, `:915-938` | `MAVLinkLogManager.cc:286`, `:321-325`, `:580-623`, `:875-887`; `PX4LogControl.qml:9-14`, `src/API/QGCOptions.h:63,131` |

Sonuç (ArduPlane tabanlı WIG için): MAVLinkLogManager uygulanamaz (PX4-only). Uçuş verisi kaydı için gerçekte iki yol var: QGC tarafında **tlog** (G.2) ve araç tarafında **dataflash log**'un Onboard Logs sayfasından indirilmesi.

#### Doğrulanmadı / kodda bulunamadı listesi

- Fact bazında bayatlık (gri/"--"/yaş göstergesi): **kodda bulunamadı** (F.4'te listelenen yerlere bakıldı).
- `telemetryAvailable`'ı false'a döndüren kod: **kodda bulunamadı** (`grep` sonucu boş).
- Paket kaybı eşiğine bağlı uyarı/ses: **kodda bulunamadı**.
- Link başına "Comm Lost" metninin hangi QML'de gösterildiği (`linkStatuses` tüketicisi): bu çalışmada incelenmedi, **doğrulanmadı**.
- Comm lost/regained sesinin ayrıntıları (ses seviyesi, TTS motoru, metin işleme): D bölümünde.
- Gerçek `Documents/...` yolu platforma bağlıdır; uygulama çalıştırılarak **doğrulanmadı**. Uygulama adı "QGroundControl" (yukarıdaki nota bakın).

---

## Kod okurken görülen olası hatalar (kapsam dışı, bilgi için)

Aşağıdakiler kodu okurken fark edildi; hiçbiri çalıştırılarak doğrulanmadı.

| Gözlem | Olası etki | dosya:satır |
|---|---|---|
| `charge_state == OK` ve yüzde NaN olduğunda `case OK` dalında `return` yok, akış `case LOW`'a düşüyor | Batarya ikonu sebepsiz yere turuncu görünüyor | `src/Toolbar/BatteryIndicator.qml:239-250` |
| Mesaj listesinde `<#E>` ve `<#I>` aynı renkle (`warningText`) çiziliyor | Listede Error ile Warning/Notice satırları ayırt edilemiyor | `src/Toolbar/VehicleMessageList.qml:25-26` |
| `ESC_INFO`, `mavlink_esc_status_t` ile decode ediliyor. MAVLink XML'e göre `index` alanı ESC_INFO'da 42. bayttadır, ESC_STATUS'ta 56. bayttadır, ESC_INFO payload'ı ise 46 bayttır | ESC_INFO'nun `index` değeri hep 0 okunur; 4'ten fazla ESC olduğunda gruplama bozulabilir | `src/Vehicle/FactGroups/EscStatusFactGroupListModel.cc:19-24`; mavlink@c409cf69 `message_definitions/v1.0/common.xml:7574-7596` |
| Metadata adı `goodAttitudeEsimate` (yazım hatası), C++ tarafındaki Fact adı `goodAttitudeEstimate` | Bu Fact'in metadata'sı (açıklama vb.) bağlanmıyor olabilir | `src/Vehicle/FactGroups/EstimatorStatusFactGroup.json:7`; `VehicleEstimatorStatusFactGroup.cc:7` |
| `timeToHome = distanceToHome.cookedValue / vfrHud.groundspeed`: mesafe kullanıcı birimindeyken (ör. ft) hız m/s | Metrik olmayan birim ayarında Time to Home yanlış hesaplanır | `src/Vehicle/FactGroups/VehicleFactGroup.cc:221-223` |
| Paket kaybı taşma düzeltmesinde `seq_received + 255` kullanılıyor (256 olmalı) | Kayıp sayısında 1 paketlik sapma (yalnızca ESP8266 sayfasındaki sayaçta) | `src/Vehicle/Vehicle.cc:552` |
| `_say()` metni `toLower()` ile küçültüyor; `_spelledAcronyms` ise yalnızca tamamı BÜYÜK harf olan kelimeyle eşleşiyor | "GPS" gibi kısaltmalar harf harf okunmuyor, kelime olarak okunuyor | `src/Vehicle/Vehicle.cc:1730`; `src/Utilities/Audio/AudioOutput.cc:51-57,300-318` |
| QML, `Vehicle` nesnesinde olmayan `activeVehicle.communicationLost` property'sini okuyor | `_connected` değeri hiçbir yerde kullanılmıyor, etkisiz görünüyor | `src/FlyView/FlightDisplayViewVideo.qml:20` |

---

## WIG için QGC'de eksik görünenler

Bu liste yalnızca yukarıdaki kod bulgularına dayanır. Her maddede kanıtın bulunduğu bölüm belirtilmiştir.

**Göstergeler ve telemetri**

1. **Yüksekliği gösteren hazır bir öğe yok.** Aşağı bakan rangefinder (DISTANCE_SENSOR `PITCH_270`) değeri yalnızca bir Fact olarak var (`DistanceSensor > RotationPitch270`). Fly View'de varsayılan olarak gösterilmiyor, kullanıcının gride elle eklemesi gerekiyor (E). Proximity radar overlay yalnızca 8 yaw sektörünü çiziyor, aşağı yönü hiç göstermiyor (`src/FlyView/ProximityRadarValues.qml:11-19`).
2. **RANGEFINDER (173) değerinin etiketi ve birimi yok.** `rangeFinderDist` için JSON metadata tanımlı değil. NaN gelen değer 0'a çevriliyor, bu yüzden "geçersiz okuma" ile "0 m" birbirinden ayırt edilemiyor (`src/Vehicle/FactGroups/VehicleFactGroup.cc:238-246`) (E).
3. **Yüksekliğe veya rangefinder'a bağlı hiçbir uyarı yok.** Alçak irtifa, rangefinder kaybı veya rangefinder sağlığı için ne görsel ne de sesli uyarı bulunuyor. Pre-flight sensör kontrolü rangefinder bitine bakmıyor (`src/FlyView/PreFlightSensorsHealthCheck.qml:11-17`) (C, E).
4. **Varsayılan gridde ground-effect'e özgü bir değer yok.** Grid uçak varsayılanlarıyla geliyor: Alt (Rel), Airspeed, Throttle ve diğerleri (`src/API/QGCCorePlugin.cc:186-298`). Yalnızca WIG'e ait bir grid profili veya araç sınıfı yok (A).
5. **NAMED_VALUE_FLOAT Plane'de ekrana konamıyor.** Firmware'in özel değerleri (ör. WIG durum değişkenleri) yalnızca MAVLink Inspector'da görülebilir ve oradan grafiğe çizilebilir. Bu mesaj sadece ArduSub'da Fact'e dönüştürülüyor (E).
6. **Rüzgar için görsel bir öğe yok.** WIND ve WIND_COV mesajları işleniyor ama hiçbir QML bunları kullanmıyor; değerler yalnızca gride elle eklenebiliyor (E).
7. **ESC_TELEMETRY_1_TO_4 işlenmiyor.** ArduPilot dialect'inde tanımlı olduğu halde QGC yalnızca ESC_INFO/ESC_STATUS'u okuyor (E).
8. **AIS_VESSEL işlenmiyor.** Deniz üstünde çalışacak bir WIG için çevredeki gemiler haritada gösterilmiyor. Mesaj dialect'te var (common.xml:7639), ama `src` içinde hiç referans yok (E). Hava trafiği ADSB yoluyla gösteriliyor, ancak MAVLink üzerinden gelen ADSB_VEHICLE için çakışma uyarısı üretilmiyor (C).

**Durum kestirimi (EKF), sağlık ve failsafe**

9. **EKF_STATUS_REPORT işlenmiyor.** ArduPilot'un EKF flag ve variance bilgileri QGC'de gösterilmiyor. Benzer bir mesaj olan ESTIMATOR_STATUS işleniyor, ama ondan gelen veri de yalnızca grid Fact'i olarak kalıyor ve hiçbir uyarı üretmiyor (C, E).
10. **Titreşim ve clipping için Fly View'de uyarı yok.** Bu veriler yalnızca Analyze > Vibration sayfasında görülebiliyor (C).
11. **Failsafe durumunun kendine ait bir göstergesi yok.** Failsafe ancak firmware STATUSTEXT gönderirse görünür (C).
12. **Geofence ihlalinin görsel karşılığı yok.** İhlal yalnızca sesli olarak bildiriliyor (C, D).
13. **"No GPS Lock" uyarısı yalnızca bağlantının başında çıkıyor.** GPS uçuş sırasında kaybolursa bu uyarının geri geldiğini gösteren bir kod yolu bulunamadı; GPS göstergesinde de renk eşiği yok (C; tüm yollar izlenmediği için doğrulanmadı).
14. **RC kaybının ayrı bir uyarısı yok.** RSSI göstergesi sadece ekrandan kayboluyor (C).

**Uyarı yönetimi**

15. **Öncelik sistemi, ayarlanabilir eşik ve tekrarlayan alarm yok.** Uyarılar yalnızca Error, Warning ve Normal olmak üzere üç sınıfa ayrılıyor. Mesaj filtresi veya kullanıcının ayarlayabileceği bir alarm eşiği yok (C).
16. **Kritik popup tek mesaj gösteriyor ve bazı durumlarda hiç açılmıyor.** Video tam ekrandayken veya durum çekmecesi açıkken popup bastırılıyor. Bir popup açıkken gelen yeni kritik mesajlar ayrı gösterilmiyor (C).
17. **Bataryanın düşük veya kritik olduğu tamamen firmware'den gelen `charge_state` alanına bağlı.** QGC tarafında yüzde veya voltaja göre alarm verilmiyor. İlk BATTERY_STATUS mesajı zaten LOW veya daha kötü bir değerle gelirse bu durum sesli olarak bildirilmiyor. Batarya uyarıları tekrarlanmıyor (C, D).
18. **Bip veya alarm sesi yok, yalnızca TTS var.** `resources/audio/alert.wav` dosyası repoda duruyor ama hiçbir yerden kullanılmıyor. STATUSTEXT için genel bir tekrar engelleme yok; kuyruk 20 mesajı aşınca konuşma kesilip kuyruk sıfırlanıyor (D).

**Bağlantı ve bayatlık**

19. **Tek tek değerlerin bayatladığı gösterilmiyor.** Bağlantı kesildiğinde irtifa, hız ve GPS değerleri son halleriyle ekranda kalıyor; gri olmuyor ve "--" gösterilmiyor. Bayatlığın tek işareti üst çubuktaki "Comms Lost" yazısı. Comm lost eşiği 3500 ms sabit ve kullanıcı tarafından ayarlanamıyor (F).
20. **Link kalitesi Fly View'de görünmüyor.** Paket kaybı yüzdesi yalnızca Ayarlar > Telemetry sayfasında gösteriliyor. Bunun için bir eşik veya uyarı da yok (F).

**Uçuş modu**

21. **Tanınmayan bir mod numarası olarak yalnızca "Unknown" gösteriliyor ve numara yazılmıyor.** Firmware'e yeni eklenen bir mod (ör. GROUND_EFFECT), firmware `AVAILABLE_MODES` göndermiyorsa ve QGC'ye de eklenmediyse ekranda numarasız "Unknown" olarak çıkar. İki tanınmayan mod arasındaki geçiş ne görsel ne de sesli olarak bildirilir (B).
22. **Plane için `landFlightMode` ve `followFlightMode` tanımlı değil.** Guided land ve follow-me için Plane'e özel bir yol yok (B).

**Kayıt**

23. **Arm edilmemiş oturumlar varsayılan olarak kaydedilmiyor.** `telemetrySaveNotArmed=false` olduğu için yer testleri (ör. su üstünde taksi denemeleri) tlog'a yazılmıyor (G).
24. **CSV kaydı bağlantı kesildiğinde bayat değer yazmaya devam ediyor.** Kesinti sırasında son değerler yeni zaman damgalarıyla CSV'ye ekleniyor (F, G).
25. **MAVLink log streaming yalnızca PX4 için çalışıyor.** ArduPlane'de uçuş verisi QGC tarafında tlog ile, araç tarafında ise dataflash logunun indirilmesiyle alınabiliyor (G).
