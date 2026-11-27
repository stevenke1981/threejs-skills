# Three.js Skills 使用指南（繁體中文）

[English](README.md) · 繁體中文

這組技能讓 AI 程式助理（Claude Code、Codex、Antigravity CLI）掌握 **three.js r187**（`threejs-math-atmosphere`、`threejs-math-motion`、`threejs-math-particles-fields` 三個技能仍為 r186、單檔結構，尚待更新；其餘技能已改為精簡的 `SKILL.md` 加 `references/`） 的最新 API，以及產生電影級 3D 畫面所需的數學：噪聲、距離場、程序化建模、粒子模擬、運鏡與大氣散射。

- `skills/`：13 個 three.js 基礎技能（場景、幾何、材質、光影、著色器、後製、效能…）
- `math-skills/`：6 個數學技能，用數學函式建模與做特效

## 1. 安裝

### 一次裝到三個代理（建議）

用 [skills](https://skills.sh) CLI 一次安裝 19 個技能。`--full-depth` 不可省略，否則 CLI 只搜尋 `skills/`，找不到 `math-skills/`：

```bash
npx skills add stevenke1981/threejs-skills --full-depth -g -y --copy -s '*' -a claude-code -a codex -a antigravity-cli
```

| 參數 | 作用 |
| --- | --- |
| `-g` | 安裝到使用者層級，所有專案都能用 |
| `--copy` | 複製檔案而非建立符號連結（Windows 建立連結常需要管理員權限） |
| `-s '*'` | 安裝全部技能；只要某幾個就改成 `-s threejs-math-noise -s threejs-math-motion` |
| `-a ...` | 指定代理，可只留其中一個 |

各代理讀取技能的位置：

| 代理 | 全域技能目錄 |
| --- | --- |
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.agents/skills/` |
| Antigravity CLI（agy） | `~/.gemini/config/skills/` |

只想裝到目前專案：拿掉 `-g`，在專案根目錄執行。

### 手動安裝（PowerShell）

```powershell
git clone https://github.com/stevenke1981/threejs-skills.git

foreach ($dir in "$HOME\.claude\skills", "$HOME\.agents\skills", "$HOME\.gemini\config\skills") {
  New-Item -ItemType Directory -Force $dir | Out-Null
  Copy-Item -Recurse -Force threejs-skills\skills\*      $dir
  Copy-Item -Recurse -Force threejs-skills\math-skills\* $dir
}
```

安裝後請重新啟動代理，技能才會載入。之後要更新：`npx skills update -g`。

## 2. 技能如何被使用

不需要記指令。代理會比對你的需求和每個技能的描述，自動載入相符的技能。想指定某個技能，直接在需求中寫出它的名稱，例如「用 threejs-math-noise 做…」。

### 數學技能

| 技能 | 內容 | 適合的需求 |
| --- | --- | --- |
| `threejs-math-noise` | 雜湊、value / 梯度 / simplex / Worley 噪聲、fBm、脊狀噪聲、domain warping、curl noise | 「用噪聲生成山脈地形」「大理石紋理」「飄動的火焰」 |
| `threejs-math-sdf-raymarching` | SDF 基本形體、平滑布林運算、sphere tracing、軟陰影、AO、碎形、與網格的深度合成 | 「融合變形的金屬球」「Mandelbulb 碎形」「純著色器場景」 |
| `threejs-math-procedural-geometry` | 參數曲面、曲線標架與掃掠、超級公式、L-system、解析法線、marching cubes | 「用公式建貝殼」「沿曲線長出藤蔓」「L-system 樹」 |
| `threejs-math-particles-fields` | 向量場、curl 流、Euler / Verlet / RK4 與穩定性、奇異吸引子、GPGPU 與 TSL compute | 「十萬顆粒子沿旋渦流動」「Lorenz 吸引子」 |
| `threejs-math-motion` | 緩動、與幀率無關的阻尼、彈簧、四元數、弧長參數化的攝影機軌道、鏡頭晃動 | 「攝影機沿軌道等速運鏡」「平滑跟隨」「手持晃動」 |
| `threejs-math-atmosphere` | Rayleigh / Mie 散射、Beer–Lambert、體積霧與體積雲、god rays、色調映射 | 「黃昏天空」「體積雲」「霧中光束」 |

### 基礎技能

`threejs-fundamentals`、`threejs-geometry`、`threejs-materials`、`threejs-lighting`、`threejs-textures`、`threejs-animation`、`threejs-loaders`、`threejs-shaders`、`threejs-postprocessing`、`threejs-interaction`、`threejs-performance`、`threejs-react-three-fiber`、`threejs-asset-pipeline`。數學技能會在需要時連到它們。

## 3. 範例需求

一個需求可以同時用到多個技能，代理會一併載入。

**奇幻山谷開場鏡頭**（即下方參考範例）

> 用 three.js 做電影開場：脊狀 fBm 噪聲生成的環形河谷，黃昏 Sky 天空加指數霧，攝影機沿封閉 Catmull-Rom 曲線等速飛行，加一點噪聲手持晃動。

→ `noise`、`procedural-geometry`、`atmosphere`、`motion`

**科幻能量核心**

> 用 SDF raymarching 畫會融合變形的能量球，表面用 domain warping 擾動，周圍有 curl noise 驅動的 GPU 粒子環繞，最後加 bloom。

→ `sdf-raymarching`、`noise`、`particles-fields`，加上 `threejs-postprocessing`

**WebGPU 體積雲**

> 用 WebGPURenderer 與 TSL 做體積雲，密度用 mx_fractal_noise_float，加 bayer16 抖動消除條紋。

→ 每個技能都有「WebGPU and TSL」段落，代理會寫 TSL 而非 GLSL

**除錯**

> 我的 raymarching 場景地平線出現一條錯色的帶子，為什麼？

→ 代理會比對技能的 Common Mistakes：命中容差隨距離 `t` 增大，固定的材質判斷門檻會在遠處失效。

## 4. 參考範例：電影感山谷

[`examples/cinematic-valley/index.html`](examples/cinematic-valley/index.html) 是單一 HTML 檔，從 CDN 載入 three@0.186.1，程式碼附繁體中文註解，每段都標出取材自哪個技能：

| 程式段落 | 用到的數學 | 技能 |
| --- | --- | --- |
| 地形高度 | `ImprovedNoise` 的 fBm 與脊狀噪聲、smoothstep 河谷遮罩 | `threejs-math-noise` |
| 地形網格 | 改頂點高度、重算法線、依坡度與高度上色 | `threejs-math-procedural-geometry` |
| 天空與霧 | Sky 散射模型、Henyey-Greenstein 光暈、`FogExp2` 濃度換算、ACES 色調映射 | `threejs-math-atmosphere` |
| 運鏡 | 弧長參數 `getPointAt`、`MathUtils.damp` 指數阻尼、噪聲晃動、`Timer` | `threejs-math-motion` |

執行：模組腳本需要透過 HTTP 載入，請在 repo 根目錄啟動本機伺服器：

```bash
npx serve .
# 或
python -m http.server 8000
```

再用瀏覽器開啟 `http://localhost:3000/examples/cinematic-valley/`（`python` 則是 `http://localhost:8000/examples/cinematic-valley/`）。

可以試著修改：

- `sunDir` 的仰角（目前 9°）：越低越接近日落，天空越紅
- `FogExp2` 的 `2.146 / 520`：分母是霧濃度達 99% 的距離
- `SPEED`：攝影機每秒前進的世界單位；因為用了弧長參數，速度在整條軌道上一致
- `k`（阻尼衰減率）：越大轉向越跟手，越小越像穩定器

範例中有一個值得注意的細節：`timer.update()` 沒有傳入 `setAnimationLoop` 的 timestamp。建地形約需 1 秒，第一幀的 timestamp 可能早於 `Timer` 建立的時間，傳入後會得到負的 delta，讓攝影機往回跑。`threejs-math-motion` 的 Common Mistakes 也記錄了這一點。

## 5. 驗證方式

- 每個 `javascript` 區塊都能解析為 ES 模組；所有從 `three`、`three/webgpu`、`three/tsl`、`three/addons/*` 匯入的名稱都存在於 three@0.186.1
- 數學技能中每個完整範例都在 Chrome（WebGL 2）實際執行；WebGPU / TSL 範例在 WebGL 2 後備路徑上編譯執行，沒有錯誤或警告，並檢查過畫面
- CPU 端的數字（積分器穩定性、吸引子範圍、阻尼與彈簧的幀率無關性、曲線標架、marching cubes 半徑）都在 Node 中實測
- 參考範例在 Chrome 中執行無錯誤或警告，並檢查過畫面

發現與 r186 不符的內容，請開 issue 並附上技能名稱、行號與你使用的 three.js 版本。
