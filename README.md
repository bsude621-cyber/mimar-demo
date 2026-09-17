# Mimarlık Demo Sitesi (Kütahya) — 6 Senaryo

Tek dosyalık statik demo. Build yok, bağımlılık yok.

**Hepsini bir arada görmek için:** [`senaryolar.html`](senaryolar.html)

## Senaryolar

3 hero videosu × 2 scroll videosu = 6 kombinasyon. `index.html` URL parametresiyle seçilir:

| | Hero (`?h=`) | Scroll (`?s=`) |
|---|---|---|
| **1 / a** | Mavi saat, betonarme cephe, sıcak iç ışık | Portikodan atriyuma yürüyüş |
| **2 / b** | Çizim masası, beyaz maket, lamba ışığı | Maketten gerçek iç mekâna |
| **3** | Işıklı atriyum, ahşap tavan, gün ışığı | — |

```
index.html?h=1&s=a   ← varsayılan
index.html?h=2&s=b   ← anlatı bütünlüğü en güçlü olan
```

Scroll videosuna göre sahne metinleri de değişir (`COPY` sabiti, `index.html` içinde):
`s=a` → EŞİK / IŞIK / DETAY · `s=b` → MAKET / ÖLÇEK / İÇERİ

Sitenin sağ altındaki **senaryo anahtarı** ile videolar arasında geçiş yapabilirsin;
scroll konumunu koruduğu için aynı noktada A/B karşılaştırması yapılabilir.

---

## Çalıştırma

`file://` ile açma — `frames/` fetch'i CORS'a takılır. Local server şart:

```bash
python -m http.server 8021
```

`http://localhost:8021/senaryolar.html`

---

## Klasör yapısı

```
index.html          demo site (senaryo parametreli)
senaryolar.html     6 senaryo karşılaştırma galerisi
media/
  hero1..3.mp4/.webm   16 sn dikişsiz boomerang loop, 1600x900, sessiz
  preview/h1..3.mp4    galeri önizlemeleri (560px, ~200 KB)
  preview/sa,sb.mp4    galeri önizlemeleri
frames/a/0001..0120.jpg  scroll-scrub kareleri (1440px)
frames/b/0001..0120.jpg
media/m/hero1..3.mp4/.webm   MOBİL hero (960px, 0.2–0.65 MB)
frames/am/, frames/bm/       MOBİL kareler (720px, 2 karede bir → 60 adet)
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.vercelignore)
_tools/             npm ffmpeg/ffprobe — DAĞITILMAZ
```

Dağıtılan toplam ~24 MB. Ziyaretçi başına indirilen:

| | hero | kareler | toplam |
|---|---|---|---|
| Masaüstü | 0.7–1.4 MB (webm) | 120 × ~62 KB = 4.4–7.4 MB | 5–9 MB |
| Mobil (Chrome, webm) | 0.21–0.44 MB | 60 × ~19 KB = 1.1 MB | **~1.35 MB** |
| Mobil (iOS, mp4) | 0.36–0.65 MB | 1.1 MB | **1.5–1.8 MB** |

Mobilde kareler sayfa açılışında değil, ilk kaydırmada (IntersectionObserver +
scroll/touch) inmeye başlar; hiç kaydırmayan ziyaretçi sadece hero'yu indirir.

---

## Videolar nasıl işlendi

ffmpeg sistemde yoktu, npm ile kuruldu:

```bash
cd _tools && npm i @ffmpeg-installer/ffmpeg @ffprobe-installer/ffprobe
```

Ham çıktılar: 1920×1080, 24 fps, 8.04 sn, 193 kare, AAC sesli. Watermark yok.

### Hero → dikişsiz loop (boomerang)

Basit `concat` yaparsan dönüş noktasında kare tekrar eder ve 1 karelik takılma olur.
Ters klipten ilk ve son kare atılmalı:

```bash
# 1) ileri
ffmpeg -y -i _raw/hero_raw_alt1.mp4 -an -vf "scale=1600:-2" \
  -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p _tools/tmp/f1.mp4
# 2) geri (n=0 ve n=192 atılır → 191 kare)
ffmpeg -y -i _tools/tmp/f1.mp4 -an \
  -vf "reverse,select='between(n\,1\,191)',setpts=N/FRAME_RATE/TB" \
  -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p _tools/tmp/r1.mp4
# 3) birleştir  (list1.txt: file 'f1.mp4' / file 'r1.mp4')
ffmpeg -y -f concat -safe 0 -i _tools/tmp/list1.txt -an \
  -c:v libx264 -crf 25 -preset slow -pix_fmt yuv420p -movflags +faststart media/hero1.mp4
# 4) webm
ffmpeg -y -i media/hero1.mp4 -an -c:v libvpx-vp9 -crf 36 -b:v 0 -row-mt 1 \
  -deadline good -cpu-used 3 media/hero1.webm
```

Sonuç: 384 kare = tam 16.000 sn, 1600×900.

### Scroll → 120 kare

```bash
ffmpeg -y -i _raw/scroll_raw_alt1.mp4 -vf "fps=15,scale=1440:-2" -q:v 4 \
  -frames:v 120 "frames/a/%04d.jpg"
```

8 sn × 15 fps = 120 kare → JS'teki `FRAME_COUNT = 120` ile birebir.
`-frames:v 120` şart, yoksa 121. kare üretilip eşleşme kayabilir.
Sıfır-pad 4 hane (`%04d` ↔ `padStart(4,'0')`).

Watermark çıkarsa CSS ile kapatma, kaynakta sil:
`-vf "delogo=x=1715:y=875:w=175:h=165,fps=15,scale=1440:-2"`

---

## Ayar noktaları

| Ne | Nerede |
|---|---|
| Scroll hızı / uzunluğu | `.scene { height:520vh }` (mobil `400vh`) — büyük = yavaş scrub |
| Kare sayısı | JS `FRAME_COUNT` (ffmpeg fps ile senkron olmalı) |
| Metin sahne zamanları | `band()`/`bell()`: `0.14–0.26`, `0.30–0.56`, `0.58–0.78`, kart `0.80–0.92` |
| Sahne metinleri | JS `COPY` sabiti (scroll videosuna göre iki set) |
| Mobil kare seyreltme | JS `STEP = isMobile ? 2 : 1` + `DIR` (`frames/am`, `frames/bm`) |
| Mobil alt bar yüksekliği | CSS `--mbar-h` (mobilde 70px; `body` padding'i ve hero/kart konumları buna bağlı) |
| DPR tavanı | `resize()` içindeki `Math.min(devicePixelRatio, 2)` |
| Palet | `:root` → `--night --copper --frost` |

---

## Mobil uyum (Eylül 2026)

Site masaüstü için kurulmuştu; telefonda kullanılabilir hale getirildi.
Masaüstü görünümü değişmedi — değişikliklerin tamamı `@media (max-width: …)` içinde,
`1440`'ta önceki hâliyle birebir aynı (`_tools/tmp/mobil/1440-hero.png`).

### Ne değişti

1. **Mobil sabit alt bar** (`.mbar`, ≤860px): **Ara** + **WhatsApp** yan yana, 48px yükseklik,
   `env(safe-area-inset-bottom)` destekli. `body`'ye `--mbar-h` kadar alt boşluk eklendi;
   hero CTA'ları, scrub kartı ve scroll ipucu bu yüksekliğe göre yukarı alındı.
2. **Mobil menü** zaten vardı (hamburger ≤1160px), erişilebilirlik tamamlandı:
   `aria-expanded` + `aria-label` güncellemesi, **Esc** ile kapanma ve odağın düğmeye dönmesi,
   menü içinde odak tuzağı (Tab döngüsü), açıldığında ilk linke odak, arka plan kaydırma kilidi,
   ekran büyüyünce kilidin bırakılması. Dokunma hedefi: hamburger 44×44, menü linkleri 57px.
3. **Yazı boyutları** (≤768px): gövde metni ≥16px, etiket/ikincil metin ≥13px, satır yüksekliği ≥1.5.
   8–12,5px'e düşen tüm yerler düzeltildi (`brand-sub`, `kick`, `proj-meta`, `stat .lbl`,
   `tile-lbl`, `scrub-specs .l`, `btn`, footer linkleri, demo anahtarı).
   Başlıktaki `brand-sub` etiketi mobilde gizlendi (aynı metin footer'da 13px olarak duruyor),
   böylece yapışkan başlık da alçaldı.
4. **Dokunma hedefleri**: tüm link/düğme/sekme mobilde ≥44×44 px (görsel boyut aynı,
   alan padding/min-height ile büyütüldü).
5. **iOS video kuralı**: hero artık `<source>` listesi kullanmıyor. Format `canPlayType` ile
   seçiliyor (Safari → mp4), **tek `src`** veriliyor, `error` olayında diğer formata düşülüyor,
   ikisi de açılmazsa element kaldırılıp `.hero-fallback` gösteriliyor. Otomatik oynatma
   engellenirse ilk dokunuş/tıklamada başlatılıyor.
6. **Mobil medya seti**: hero 960px (`media/m/`), scroll kareleri 720px + 2 karede bir
   (`frames/am`, `frames/bm`). Kareler `IntersectionObserver` ile geç yükleniyor.
   Ziyaretçi başına inen veri 4,4–7,4 MB'tan **~1,35 MB'a** (iOS'te ≤1,8 MB) düştü.
7. **Demo senaryo anahtarı** mobilde katlanabilir: varsayılan olarak 44px'lik "SENARYO" hapı,
   alt barın üstünde duruyor, açılınca segmentler 44px hedeflerle görünüyor,
   menü açıkken gizleniyor. Hiçbir CTA ile çakışmıyor.
8. **Güvenli alan**: `viewport-fit=cover` + nav/section/footer/alt barda `env(safe-area-inset-*)`.
9. **Yatay mod** (ör. 812×375): mobil kurallar `(max-height:520px) and (orientation:landscape)`
   ile orada da geçerli; hero yüksekliği ve başlık ölçüsü küçültüldü, açıklama 2 satıra
   kısaltıldı, böylece başlık + iki CTA ilk ekranda kalıyor.
10. `senaryolar.html` (iç demo galerisi) aynı 13/16px ve 44px kurallarına göre elden geçirildi.

### Ölçümler (ölçüldü, tahmin değil)

| Genişlik | Yatay taşma | 44px altı hedef | 13px altı yazı | satır yüksekliği <1.5 | konsol |
|---|---|---|---|---|---|
| 320×700 | yok | 0 | 0 | 0 | temiz |
| 360×780 | yok | 0 | 0 | 0 | temiz |
| 375×812 | yok | 0 | 0 | 0 | temiz |
| 390×844 | yok | 0 | 0 | 0 | temiz |
| 414×896 | yok | 0 | 0 | 0 | temiz |
| 812×375 (yatay) | yok | 0 | 0 | 0 | temiz |
| 1440×900 | yok | — (masaüstü) | — | — | temiz |

`senaryolar.html` 375×812: taşma yok, 44px altı 0, 13px altı 0.

Elle test edildi: hamburger menü (aç/kapa/Esc/odak), alt bar linkleri, senaryo anahtarı
(katlanma + senaryo değiştirme), hero videosu (mobil `media/m/` seti yükleniyor),
scroll-scrub (mobil kare seti canvas'a çiziliyor).

Ekran görüntüleri: `_tools/tmp/mobil/` → `375-hero.png`, `375-hizmetler.png`,
`375-iletisim.png`, `375-senaryolar.png`, `1440-hero.png`.

### Mobil seti yeniden üretmek

```bash
FF=_tools/node_modules/@ffmpeg-installer/win32-x64/ffmpeg.exe
# kareler: 720px, 2 karede bir (dosya adı orijinal indeksi korur: 0001, 0003, …)
for s in a b; do for i in $(seq 1 2 120); do n=$(printf "%04d" $i);
  "$FF" -y -i frames/$s/$n.jpg -vf "scale=720:-2" -q:v 5 frames/${s}m/$n.jpg; done; done
# hero: 960px
for h in 1 2 3; do
  "$FF" -y -i media/hero$h.mp4 -an -vf "scale=960:-2" -c:v libx264 -crf 29 -preset slow \
    -pix_fmt yuv420p -movflags +faststart media/m/hero$h.mp4
  "$FF" -y -i media/m/hero$h.mp4 -an -c:v libvpx-vp9 -crf 42 -b:v 0 -row-mt 1 \
    -deadline good -cpu-used 4 media/m/hero$h.webm
done
```

### Bilinen kalan konular

- Yatay modda hero açıklaması 2 satıra kırpılıyor (`-webkit-line-clamp`) ve `hero-kick`
  gizleniyor; dikey moda dönünce tam metin geri geliyor.
- Scroll-scrub kareleri ilk kaydırmada inmeye başladığı için çok hızlı kaydıran ziyaretçi
  sahnenin ilk ~1 saniyesinde prosedürel placeholder görebilir.
- Mobil kareler 720px: 3x DPR telefonlarda tam ekran scrub'da hafif yumuşama olur —
  bant genişliği tercihi bilinçli.
- Emülasyonla ölçüldü (Chrome, 320–414 + yatay). Gerçek iOS Safari'de video/alt bar
  davranışı satış öncesi bir kez telefonda açılarak doğrulanmalı.

## Müşteriye teslim ederken

1. Senaryoyu seç, `HERO` / `SCRL` sabitlerini o değere sabitle (URL parametresini kaldır).
2. `<div class="demo-bar">` bloğunu, `.demo-bar` CSS'ini ve `demoBar()` JS fonksiyonunu sil.
3. `senaryolar.html`'i ve kullanılmayan hero/frames setlerini sil.
4. Firma adı, telefon, WhatsApp, e-posta, adres, `data-count` değerlerini değiştir
   (liste `index.html` başındaki yorum bloğunda).
5. Proje kartlarında `.proj-art` SVG'lerini gerçek fotoğrafla değiştir:
   `<div class="proj-art">…</div>` → `<img src="projeler/1.jpg" alt="...">`

---

*Mekanik kaynağı: `emlak-video-hero/NASIL-YAPILDI.md` (scroll-scrub reçetesi) + `insaat-web` (hero + palet).*
