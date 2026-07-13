<h1 align="center">🛡️ Haune</h1>

<h3 align="center">Anti-detect Chromium 148 với phòng thủ fingerprint ở tầng source.</h3>

<p align="center">
Không phải JS shim. Không phải preset flag. Đây là một bản Chromium <strong>tự build</strong>
với các bề mặt fingerprint được vá ở <strong>C++</strong> và biên dịch thẳng vào binary —
hệ thống antibot chấm điểm nó như một trình duyệt bình thường vì nó <em>đúng là</em> trình
duyệt bình thường.
</p>

<p align="center">
<em>Chromium 148.0.7778.288 · official ThinLTO build · launcher Node + .NET · Windows</em>
</p>

<p align="center">
<img src="https://img.shields.io/badge/Chromium-148.0.7778.288-4285F4" alt="Chromium 148">
<img src="https://img.shields.io/badge/build-official%20ThinLTO-success" alt="official build">
<img src="https://img.shields.io/badge/platform-Windows-0078D6" alt="Windows">
<img src="https://img.shields.io/badge/API-.NET%208%20%C2%B7%20Node-512BD4" alt=".NET 8 / Node">
<img src="https://img.shields.io/badge/FingerprintJS-~75%25%20%E2%89%88%20Cloak-8A2BE2" alt="FingerprintJS ~75% = Cloak">
<img src="https://img.shields.io/badge/license-Proprietary%20%C2%B7%20All%20Rights%20Reserved-red" alt="Proprietary license">
</p>

<p align="center">
<img src="docs/assets/demo-sannysoft.png" width="620" alt="Haune passing bot.sannysoft.com — all green, real Chrome UA, coherent WebGL">
<br><em>bot.sannysoft.com — all green: real <code>Chrome/148</code> UA, no headless leak, coherent WebGL (real GPU passthrough).</em>
</p>

<p align="center"><a href="#-tiếng-việt">🇻🇳 Tiếng Việt</a> · <a href="#-english">🇬🇧 English</a> · <a href="LICENSE">📄 License</a></p>

---

## 🇻🇳 Tiếng Việt

### Haune là gì?

Haune là trình duyệt chống-nhận-diện của riêng bạn, làm theo đúng cách CloakBrowser và
BotBrowser làm: một bản fork Chromium mà các API mang fingerprint (canvas, WebGL, WebGPU,
audio, fonts, GPU, screen, WebRTC, User-Agent, navigator) được sửa ở **tầng source**, kèm
một launcher mỏng tổng hợp **danh tính mạch lạc, tái lập được từ seed** và lái binary đó.

- **Ở tầng source, không phải JavaScript.** Mọi spoof trả về qua code path gốc của trình
  duyệt — `Function.prototype.toString` vẫn báo `[native code]`, nên các detector kiểu
  stealth-plugin không tìm thấy gì để cờ.
- **Một seed = một thiết bị ổn định.** UA ↔ platform ↔ GPU ↔ fonts ↔ timezone luôn nhất
  quán với nhau (sai lầm số 1 của anti-detect là để chúng mâu thuẫn). **Hơn 6 tỉ** danh
  tính duy nhất, không trùng, từ bộ sinh seed 128-bit.
- **Sạch mặc định (ngang Cloak).** Cấu hình mặc định trình bày một thiết bị thật, mạch lạc,
  và farble canvas theo seed ở tầng *render* — không có dấu vết chỉnh sửa readback.
- **Vượt Cloak ở 2 vector:** spoof **WebGPU adapter** native và điều khiển
  **`speechSynthesis` voices** (CloakBrowser không có cả hai switch này).

> Dành cho nghiên cứu quyền riêng tư, QA anti-detect, và kiểm thử multi-account/scraping
> **được phép** trên môi trường bạn sở hữu hoặc có văn bản cho phép. Chi tiết kỹ thuật ở `docs/`.

### Bắt đầu nhanh

**Launcher Node:**

```powershell
# Mặc định sạch: danh tính mạch lạc, GPU giữ thật (passthrough), canvas farble theo seed.
node launcher/haune.js --seed 12345 https://abrahamjuliot.github.io/creepjs/

# Qua proxy — timezone / geolocation / WebRTC IP tự khớp exit IP của proxy (GeoIP):
node launcher/haune.js --seed 12345 --proxy http://user:pass@host:port https://demo.fingerprint.com/web-scraping
```

Mỗi seed ghi profile đầy đủ ra `~/.haune/profiles/<seed>/fingerprint.json`.

| Flag | Tác dụng |
|---|---|
| `--seed <n>` | Seed danh tính (cùng seed → cùng thiết bị). |
| `--proxy <url>` | Proxy + auto GeoIP (timezone, `--fingerprint-location`, WebRTC IP). |
| `--country <ISO2>` | Ép quốc gia locale/timezone (bị GeoIP của proxy override). |
| `--full-spoof` | Phát mọi value-override (screen/GPU/webgpu/voices/…). Xem đánh đổi bên dưới. |
| `--spoof-gpu` | Spoof cả chuỗi GPU renderer (≈ rate của Cloak — xem dưới). |
| `--headless` | Chạy headless (mặc định là headful). |

**Thư viện .NET 8:**

```csharp
using Haune;

await using var browser = await HauneLauncher.LaunchAsync(new LaunchOptions {
    Seed    = 12345,
    Proxy   = "http://user:pass@host:port",  // GeoIp + WebRTC tự bật khi có proxy
    Headless = false,
});
var page = await browser.NewPageAsync();
await page.GotoAsync("https://demo.fingerprint.com/web-scraping");
```

`LaunchOptions` mô phỏng API của CloakBrowser: `FullSpoof`, `SpoofGpu`, `CoherentGpu`,
`CoherentFonts`, `GeoIp`, `WebRtcLeakProtection`, `FontsDir`, `HumanConfig`, `Brand`, `Os`…
Nhập liệu giống người (chuột Bézier, gõ từng ký tự, scroll) qua `NewHumanPageAsync()`.

### Mặc định = cấu hình *sạch nhất*

FingerprintJS-commercial `browserTampering` cờ hai thứ: **noise readback** sau render (kiểu
farble canvas/WebGL cổ điển — ngay cả fingerprint-chromium gốc cũng rớt) **và** các
**flag value-override** rõ ràng (danh tính "quá bị điều khiển"). Mặc định của Haune bỏ cả
hai: chỉ phát seed + coherence geo/timezone và **giữ GPU thật** (passthrough kiểu Cloak),
đồng thời vẫn farble canvas theo seed bằng jitter sub-pixel ở tầng **render** (render thật,
`getImageData` reread-diff = 0).

| Cấu hình | FingerprintJS (IP proxy fresh, A/B 3 vòng) |
|---|---|
| **Haune mặc định (sạch)** | **≈ 75% CLEAN — ngang CloakBrowser** |
| Haune `--full-spoof` (value overrides) | ~0% (sạch trên mọi detector FREE; dùng khi cần điều khiển identity theo pool) |
| Haune `--spoof-gpu` | ~67% (= Cloak, vốn cũng spoof GPU) |

**Trần trung thực:** không tool công khai nào — kể cả fingerprint-chromium, Camoufox, hay
Cloak — spoof GPU cross-vendor MÀ vẫn pass FingerprintJS 100%; render WebGL được raster ở
GPU process và lộ vendor thật. Haune giữ GPU thật mặc định (rate cao nhất) và khi bạn bật
spoof GPU thì ép về đúng vendor của máy (`--coherent-gpu`, mặc định bật) để render vẫn mạch lạc.

### Kết quả kiểm thử

Đo headful trên binary from-source. Detector free/phổ biến: **ngang Cloak.**

| Detector | Kết quả |
|---|---|
| bot.sannysoft.com | ✅ 0 đỏ |
| CreepJS | ✅ 0 lie, 0% stealth, getter native |
| pixelscan.net | ✅ không proxy / không automation, geo khớp |
| browserscan.net/bot-detection | ✅ Normal (4/4) |
| bot-detector.rebrowser.net | ✅ xanh (identity brand Chrome) |
| deviceandbrowserinfo.com | ✅ "You are human!" |
| reCAPTCHA v3 | ✅ 0.9 (human, verify server-side) |
| Cloudflare Turnstile | ✅ cấp token |
| `navigator.webdriver` | ✅ `false` (getter native) |
| UA / TLS (JA3/JA4) | ✅ Chrome 148 thật, không lộ "Headless"/brand |
| FingerprintJS browserTampering | ≈ 75% (= Cloak) ở cấu hình mặc định sạch |

### Cách hoạt động

Luồng switch: `components/ungoogled/ungoogled_switches.{cc,h}` định nghĩa `--fingerprint*` →
`RenderProcessHostImpl` forward xuống renderer → Blink đọc tại mỗi điểm inject. Tất cả biên
dịch vào binary; không gì inject qua JavaScript.

Các vector fingerprint (đều native, verify live):

- **Canvas** — jitter sub-pixel theo seed ở tầng render trên text/path/image (không phát
  hiện được; readback nguyên vẹn). Solid-fill vuông giữ nét, nên không có anti-alias bất thường.
- **WebGL / GPU** — override chuỗi `UNMASKED_*`; render thật giữ mạch lạc (cùng vendor).
- **WebGPU adapter**, **speechSynthesis voices** — mạch lạc với GPU WebGL / locale.
- **Audio** — micro-gain ở tầng render, sample rate giữ số nguyên chính xác.
- **Fonts** — hiển thị theo profile + tùy chọn bundle `--fingerprint-fonts-dir`.
- **Screen, deviceMemory, hardwareConcurrency, platform, brand, UA-CH** — seed-derive, mạch lạc.
- **WebRTC** — IP candidate/SDP ghim theo exit của proxy (hoặc chặn); không lộ IP nội bộ.
- **Timezone / geolocation** — khớp exit IP của proxy.

### Build từ source

```powershell
cd scripts
. .\env.ps1
.\apply-patches.ps1      # restack bộ patch lên Chromium 148
.\build.ps1              # hoặc: autoninja -C out\Release chrome
```

`args/args.release.gn` là bản build **official** hạng Cloak/BotBrowser
(`is_official_build=true`, ThinLTO, `dcheck_always_on=false`, tắt fieldtrial-testing). Trên
máy 48 GB, link ThinLTO được giới hạn qua `build/config/compiler` `lldltojobs=8` để tránh
OOM. Diff tái lập đầy đủ so với 148 gốc: `patches/haune-trackb-full.patch`.

### Bố cục

```
args/        gn build config (release = bản official ship được)
patches/     patch Chromium tầng source + snapshot full-diff
scripts/     env / build / apply-patches / packaging (PowerShell)
launcher/    wrapper Node + fingerprint/ (data.js dataset, synth.js bộ sinh mạch lạc)
dotnet/      thư viện .NET 8 "Haune" (nền Playwright) + samples
docs/        kiến trúc, wiring, known-issues, status
```

### Sử dụng có trách nhiệm

Haune chỉ dành cho nghiên cứu & kiểm thử quyền-riêng-tư-fingerprint được phép. Giữ văn bản
cho phép cho mọi môi trường bạn test. Không dùng để vi phạm điều khoản của bất kỳ nền tảng
nào hay pháp luật.

### Bản quyền & giấy phép

© 2026 Haune Builder (github.com/haune2311). **All Rights Reserved.** Phần mềm này là
**ĐỘC QUYỀN** — KHÔNG ai được sao chép, clone, fork, chỉnh sửa, phân phối, host hay dùng lại
dưới bất kỳ hình thức nào nếu không có văn bản cho phép của chủ sở hữu. Xem đầy đủ ở
[`LICENSE`](LICENSE). (Các thành phần bên thứ ba như Chromium / fingerprint-chromium vẫn theo
giấy phép gốc của chúng.)

---

## 🇬🇧 English

### What is Haune?

Haune is your own fingerprint-resistant browser, built the way CloakBrowser and BotBrowser
are: a Chromium fork with the fingerprint-bearing APIs (canvas, WebGL, WebGPU, audio, fonts,
GPU, screen, WebRTC, User-Agent, navigator) modified at the **source level**, plus a thin
launcher that synthesizes a **coherent, seed-reproducible identity** and drives the binary.

- **Source-level, not JavaScript.** Every spoof returns through the browser's native code
  paths — `Function.prototype.toString` still reports `[native code]`, so stealth-plugin
  detectors find nothing to flag.
- **One seed = one stable device.** UA ↔ platform ↔ GPU ↔ fonts ↔ timezone are all kept
  internally consistent (the #1 anti-detect mistake is incoherence). **6 billion+** unique,
  non-colliding identities from a 128-bit seeded generator.
- **Clean by default (Cloak-parity).** The default launch presents a real, coherent device
  and farbles canvas per-seed at *render* level — no detectable readback tampering.
- **Ahead of Cloak on two vectors:** native **WebGPU adapter** spoofing and
  **`speechSynthesis` voices** control (CloakBrowser ships neither switch).

> For **authorized** privacy research, anti-detect QA, and multi-account/scraping testing
> you own or have written permission to test. See `docs/` for the engineering details.

### Quick start

**Node launcher:**

```powershell
# Clean default: coherent identity, GPU kept real (passthrough), canvas farbled by seed.
node launcher/haune.js --seed 12345 https://abrahamjuliot.github.io/creepjs/

# Through a proxy — timezone / geolocation / WebRTC IP auto-match the proxy exit (GeoIP):
node launcher/haune.js --seed 12345 --proxy http://user:pass@host:port https://demo.fingerprint.com/web-scraping
```

Every seed writes its full profile to `~/.haune/profiles/<seed>/fingerprint.json`.

| Flag | Effect |
|---|---|
| `--seed <n>` | Identity seed (same seed → same device). |
| `--proxy <url>` | Proxy + auto GeoIP (timezone, `--fingerprint-location`, WebRTC IP). |
| `--country <ISO2>` | Force locale/timezone country (overridden by proxy GeoIP). |
| `--full-spoof` | Emit all pool-controlled value overrides (screen/GPU/webgpu/voices/…). See trade-off below. |
| `--spoof-gpu` | Spoof the GPU renderer string too (≈ Cloak's rate — see below). |
| `--headless` | Run headless (default is headful). |

**.NET 8 library:**

```csharp
using Haune;

await using var browser = await HauneLauncher.LaunchAsync(new LaunchOptions {
    Seed    = 12345,
    Proxy   = "http://user:pass@host:port",  // GeoIp + WebRTC pin auto-enable with a proxy
    Headless = false,
});
var page = await browser.NewPageAsync();
await page.GotoAsync("https://demo.fingerprint.com/web-scraping");
```

`LaunchOptions` mirrors the CloakBrowser API shape: `FullSpoof`, `SpoofGpu`, `CoherentGpu`,
`CoherentFonts`, `GeoIp`, `WebRtcLeakProtection`, `FontsDir`, `HumanConfig`, `Brand`, `Os`…
Human-like input (Bézier mouse, per-char typing, scroll) via `NewHumanPageAsync()`.

### The default is the *cleanest* config

FingerprintJS-commercial `browserTampering` flags two things: post-render **readback noise**
(the classic canvas/WebGL farble — even vanilla fingerprint-chromium fails it) **and**
explicit **value-override flags** (a coherent-looking but "too-controlled" identity).

Haune's **default** drops both: it emits only the seed + geo/timezone coherence and keeps the
**GPU real** (Cloak-style native passthrough), while still farbling canvas per-seed through a
native **render-level** sub-pixel jitter (genuine render, `getImageData` reread-diff = 0).

| Config | FingerprintJS (fresh rotated proxy IPs, 3-round A/B) |
|---|---|
| **Haune default (clean)** | **≈ 75% CLEAN — matches CloakBrowser** |
| Haune `--full-spoof` (value overrides) | ~0% (clean on all FREE detectors; use when pool-controlled identity matters more) |
| Haune `--spoof-gpu` | ~67% (= Cloak, which also spoofs GPU) |

**Honest ceiling:** no public tool — including fingerprint-chromium, Camoufox, or Cloak — does
cross-vendor GPU spoofing *and* passes FingerprintJS 100%; the WebGL render is rasterized in
the GPU process and reveals the real vendor. Haune keeps the GPU real by default (best pass
rate) and, when you opt into GPU spoofing, constrains it to the machine's **own vendor**
(`--coherent-gpu`, default on) so the render stays coherent.

### Detection results

Measured headful on the from-source binary. Free/common detectors: **parity with Cloak.**

| Detector | Result |
|---|---|
| bot.sannysoft.com | ✅ 0 red |
| CreepJS | ✅ 0 lies, 0% stealth, native getters |
| pixelscan.net | ✅ no proxy / no automation, geo matches |
| browserscan.net/bot-detection | ✅ Normal (4/4) |
| bot-detector.rebrowser.net | ✅ green (Chrome-brand identities) |
| deviceandbrowserinfo.com | ✅ "You are human!" |
| reCAPTCHA v3 | ✅ 0.9 (human, server-verified) |
| Cloudflare Turnstile | ✅ token issued |
| `navigator.webdriver` | ✅ `false` (native getter) |
| UA / TLS (JA3/JA4) | ✅ real Chrome 148, no "Headless"/brand leak |
| FingerprintJS browserTampering | ≈ 75% (= Cloak) in the clean default |

### How it works

Switches flow: `components/ungoogled/ungoogled_switches.{cc,h}` define `--fingerprint*` →
`RenderProcessHostImpl` forwards them to renderers → Blink reads them at each injection site.
Everything is compiled into the binary; nothing is injected via JavaScript.

Fingerprint vectors (all native, verified live):

- **Canvas** — render-level per-seed sub-pixel jitter on text/path/image draws (undetectable;
  readback untouched). Axis-aligned solid fills stay crisp so no impossible anti-aliasing.
- **WebGL / GPU** — `UNMASKED_*` renderer/vendor override; real render kept coherent (same-vendor).
- **WebGPU adapter**, **speechSynthesis voices** — coherent with the WebGL GPU / locale.
- **Audio** — render-level micro-gain, sample-rate kept an exact integer.
- **Fonts** — profile-scoped visibility + optional `--fingerprint-fonts-dir` bundling.
- **Screen, deviceMemory, hardwareConcurrency, platform, brand, UA-CH** — seed-derived, coherent.
- **WebRTC** — candidate/SDP IP pinned to the proxy exit (or blocked); no private-IP leak.
- **Timezone / geolocation** — matched to the proxy exit IP.

### Build from source

```powershell
cd scripts
. .\env.ps1
.\apply-patches.ps1      # restack the patch set onto Chromium 148
.\build.ps1              # or: autoninja -C out\Release chrome
```

`args/args.release.gn` is a Cloak/BotBrowser-class **official** build
(`is_official_build=true`, ThinLTO, `dcheck_always_on=false`, fieldtrial-testing off). On a
48 GB machine the ThinLTO link is capped via `build/config/compiler` `lldltojobs=8` to avoid
OOM. Full reproducible diff vs stock 148: `patches/haune-trackb-full.patch`.

### Layout

```
args/        gn build configs (release = shippable official build)
patches/     source-level Chromium patches + full-diff snapshot
scripts/     env / build / apply-patches / packaging (PowerShell)
launcher/    Node wrapper + fingerprint/ (data.js datasets, synth.js coherent generator)
dotnet/      .NET 8 "Haune" library (Playwright-based) + samples
docs/        architecture, wiring, known-issues, status
```

### Responsible use

Haune is for authorized fingerprint-privacy research and testing only. Keep written
authorization for every environment you test. Do not use it to violate any platform's terms
or any law.

### License

© 2026 Haune Builder (github.com/haune2311). **All Rights Reserved.** This software is
**PROPRIETARY** — no copying, cloning, forking, modification, redistribution, hosting, or
reuse in any form without the copyright holder's written permission. See [`LICENSE`](LICENSE)
for the full terms. (Third-party components such as Chromium / fingerprint-chromium remain
governed by their own upstream licenses.)
