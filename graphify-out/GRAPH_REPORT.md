# Graph Report - WIG_fork_review  (2026-10-05)

## Corpus Check
- 1 files · ~3,585 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 34 nodes · 33 edges · 6 communities
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `fb84ad58`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- `squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik
- c) Kontrol döngüsü
- f) Güvenlik
- e) Mevcut sistemlerle ilişki
- g) Güncel sürüme taşıma
- h) SITL testi

## God Nodes (most connected - your core abstractions)
1. ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik` - 12 edges
2. `c) Kontrol döngüsü` - 7 edges
3. `f) Güvenlik` - 7 edges
4. `e) Mevcut sistemlerle ilişki` - 5 edges
5. `g) Güncel sürüme taşıma` - 4 edges
6. `h) SITL testi` - 3 edges
7. `Kısa özet` - 1 edges
8. `a) Commit'ler (eskiden yeniye)` - 1 edges
9. `b) Mod: ne ekleniyor, nasıl seçiliyor` - 1 edges
10. `Sensör ve sürücüler` - 1 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Communities (6 total, 0 thin omitted)

### Community 0 - "`squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik"
Cohesion: 0.25
Nodes (7): a) Commit'ler (eskiden yeniye), b) Mod: ne ekleniyor, nasıl seçiliyor, Bizim araç için sonraki 5 adım, d) Yeni parametreler, i) Lisans (GPLv3, kısa hatırlatma, hukuki görüş değildir), Kısa özet, `squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik

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

### Community 5 - "h) SITL testi"
Cohesion: 0.67
Nodes (3): h) SITL testi, Mesafe sensörünü simüle etme, Önerilen test adımları

## Knowledge Gaps
- **27 isolated node(s):** `Kısa özet`, `a) Commit'ler (eskiden yeniye)`, `b) Mod: ne ekleniyor, nasıl seçiliyor`, `Sensör ve sürücüler`, `Filtre` (+22 more)
  These have ≤1 connection - possible missing edges or undocumented components. (Counts symbols only; 28 node(s) total have ≤1 connection when file, concept and rationale nodes are included.)

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik` connect ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik` to `c) Kontrol döngüsü`, `f) Güvenlik`, `e) Mevcut sistemlerle ilişki`, `g) Güncel sürüme taşıma`, `h) SITL testi`?**
  _High betweenness centrality (0.884) - this node is a cross-community bridge._
- **Why does `c) Kontrol döngüsü` connect `c) Kontrol döngüsü` to ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik`?**
  _High betweenness centrality (0.335) - this node is a cross-community bridge._
- **Why does `f) Güvenlik` connect `f) Güvenlik` to ``squilter/ground_effect_mode` Dalı Analizi: WIG Aracımıza Taşınabilirlik`?**
  _High betweenness centrality (0.335) - this node is a cross-community bridge._
- **What connects `Kısa özet`, `a) Commit'ler (eskiden yeniye)`, `b) Mod: ne ekleniyor, nasıl seçiliyor` to the rest of the system?**
  _27 weakly-connected nodes found - possible documentation gaps or missing edges._