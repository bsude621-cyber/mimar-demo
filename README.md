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
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.vercelignore)
_tools/             npm ffmpeg/ffprobe — DAĞITILMAZ
```

Dağıtılan toplam ~21 MB. Ziyaretçi başına indirilen: 1 hero (0.7–1.4 MB webm)
+ 120 kare (4–7 MB).

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
| Mobil kare seyreltme | JS `STEP = isMobile ? 2 : 1` |
| DPR tavanı | `resize()` içindeki `Math.min(devicePixelRatio, 2)` |
| Palet | `:root` → `--night --copper --frost` |

---

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
