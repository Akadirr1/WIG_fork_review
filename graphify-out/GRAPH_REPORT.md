# Graph Report - WIG_fork_review  (2026-10-08)

## Corpus Check
- 3 files · ~11,582 words
- Verdict: corpus is large enough that graph structure adds value.
- Unclassified: 1 file(s) not represented in the graph (top: (none) 1)

## Summary
- 78 nodes · 75 edges · 11 communities
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `8fd14671`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- `squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik
- c) Kontrol döngüsü
- f) Güvenlik
- e) Mevcut sistemlerle ilişki
- g) Güncel sürüme taşıma
- ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite
- 2. Tiltrotor ileri geçişi (hover → ileri uçuş)
- 4. Dikey motorların ileri uçuşta yardıma girmesi (assist)
- 1. VTOL kalkışı
- 5. VTOL iniş
- SITL Denemeleri 2: EKF'de Mesafe Sensörü Yüksekliği, Geçiş Hatası ve RC Kaybı

## God Nodes (most connected - your core abstractions)
1. `ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite` - 12 edges
2. ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik` - 12 edges
3. `SITL Denemeleri 2: EKF'de Mesafe Sensörü Yüksekliği, Geçiş Hatası ve RC Kaybı` - 8 edges
4. `c) Kontrol döngüsü` - 7 edges
5. `f) Güvenlik` - 7 edges
6. `2. Tiltrotor ileri geçişi (hover → ileri uçuş)` - 6 edges
7. `4. Dikey motorların ileri uçuşta yardıma girmesi (assist)` - 5 edges
8. `e) Mevcut sistemlerle ilişki` - 5 edges
9. `1. EKF yüksekliği mesafe sensöründen mi alıyor?` - 4 edges
10. `1. VTOL kalkışı` - 4 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Communities (11 total, 0 thin omitted)

### Community 0 - "`squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik"
Cohesion: 0.18
Nodes (10): a) Commit'ler (eskiden yeniye), b) Mod: ne ekleniyor, nasıl seçiliyor, Bizim araç için sonraki 5 adım, d) Yeni parametreler, h) SITL testi, i) Lisans (GPLv3, kısa hatırlatma, hukuki görüş değildir), Kısa özet, Mesafe sensörünü simüle etme (+2 more)

### Community 1 - "c) Kontrol döngüsü"
Cohesion: 0.29
Nodes (7): c) Kontrol döngüsü, Döngü hızı, Filtre, Hedef yükseklik, Neyi kontrol ediyor, PID kazançları, Sensör ve sürücüler

### Community 2 - "f) Güvenlik"
Cohesion: 0.29
Nodes (7): Dalga ve gürültü, f) Güvenlik, Failsafe ve `FS_LONG_ACTN`, Menzil dışına çıkarsa, Mesafe sensörü verisi kaybolursa, Mod kullanılmasa bile etkileyen yan etkiler, Sayısal hatalar

### Community 3 - "e) Mevcut sistemlerle ilişki"
Cohesion: 0.40
Nodes (5): Dönüşler, e) Mevcut sistemlerle ilişki, Kalkış ve iniş, TECS, Yol noktası navigasyonu (AUTO)

### Community 4 - "g) Güncel sürüme taşıma"
Cohesion: 0.50
Nodes (4): Değişen API'ler, g) Güncel sürüme taşıma, Çakışan dosyalar (merge-tree çıktısı, 22 dosya), Önerilen taşıma biçimi

### Community 5 - "ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite"
Cohesion: 0.15
Nodes (12): 3. Geçişte yükseklik tutma ve alçak geçiş, 6.1 Doğal giriş noktası, 6.2 QuadPlane koduyla çakışmalar (1–3 m'de seyir için), 6. Yeni yüzey etkisi modu, 7. GCS mesajları (STATUSTEXT), 8. SITL denemesi, ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite, Bizim yazmamız gerekenler (+4 more)

### Community 6 - "2. Tiltrotor ileri geçişi (hover → ileri uçuş)"
Cohesion: 0.33
Nodes (6): 2.1 Geçiş nerede çalışıyor?, 2.2 Durum makinesi, 2.3 Geçiş hangi koşullarda tamamlanmış sayılıyor?, 2.4 Geçiş başarısız olursa, 2.5 Geri geçiş (ileri uçuş → hover), kısaca, 2. Tiltrotor ileri geçişi (hover → ileri uçuş)

### Community 7 - "4. Dikey motorların ileri uçuşta yardıma girmesi (assist)"
Cohesion: 0.40
Nodes (5): 4.1 Tetikleyiciler (`ArduPlane/VTOL_Assist.cpp:59-144`), 4.2 `Q_ASSIST_ALT` yüksekliği nereden geliyor?, 4.3 Assist tiltrotorda ne yapıyor?, 4.4 Yüzey etkisinde güvenlik ağı olarak kullanılabilir mi?, 4. Dikey motorların ileri uçuşta yardıma girmesi (assist)

### Community 8 - "1. VTOL kalkışı"
Cohesion: 0.50
Nodes (4): 1.1 Yükseklik hangi kaynaktan ölçülüyor?, 1.2 Kalkış ne zaman bitmiş sayılıyor?, 1.3 Su üstünde birkaç metrelik kalkış mümkün mü?, 1. VTOL kalkışı

### Community 9 - "5. VTOL iniş"
Cohesion: 0.50
Nodes (4): 5.1 Mesafe sensörü nasıl kullanılıyor?, 5.2 Yere ya da suya değme nasıl algılanıyor?, 5.3 Dalgalı yüzeyde algılamayı yanıltabilecek şeyler, 5. VTOL iniş

### Community 10 - "SITL Denemeleri 2: EKF'de Mesafe Sensörü Yüksekliği, Geçiş Hatası ve RC Kaybı"
Cohesion: 0.17
Nodes (11): 1. EKF yüksekliği mesafe sensöründen mi alıyor?, 1a: `EK3_RNG_USE_HGT=70` (baro sapması yok), 1b ve 1c: baroya sapma, 2. Geçiş takılınca QLAND, 3. RC kaybı aşamaya göre (`SIM_RC_FAIL=1`), 4. Seyirde Glide (`FS_LONG_ACTN=2`) ve dikey motor yardımı, Bizim için çıkanlar, Hangi log alanında görünüyor? (+3 more)

## Knowledge Gaps
- **60 isolated node(s):** `Kısa özet`, `Ortak ayarlar`, `Hangi log alanında görünüyor?`, `1a: `EK3_RNG_USE_HGT=70` (baro sapması yok)`, `1b ve 1c: baroya sapma` (+55 more)
  These have ≤1 connection - possible missing edges or undocumented components. (Counts symbols only; 63 node(s) total have ≤1 connection when file, concept and rationale nodes are included.)

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik` connect ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik` to `c) Kontrol döngüsü`, `f) Güvenlik`, `e) Mevcut sistemlerle ilişki`, `g) Güncel sürüme taşıma`?**
  _High betweenness centrality (0.160) - this node is a cross-community bridge._
- **Why does `ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite` connect `ArduPlane 4.7.1'de Tiltrotor VTOL: WIG İçin Fizibilite` to `1. VTOL kalkışı`, `5. VTOL iniş`, `2. Tiltrotor ileri geçişi (hover → ileri uçuş)`, `4. Dikey motorların ileri uçuşta yardıma girmesi (assist)`?**
  _High betweenness centrality (0.144) - this node is a cross-community bridge._
- **Why does `c) Kontrol döngüsü` connect `c) Kontrol döngüsü` to ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik`?**
  _High betweenness centrality (0.060) - this node is a cross-community bridge._
- **What connects `Kısa özet`, `Ortak ayarlar`, `Hangi log alanında görünüyor?` to the rest of the system?**
  _60 weakly-connected nodes found - possible documentation gaps or missing edges._