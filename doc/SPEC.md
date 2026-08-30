# seasplice SPEC(v0 草案)

> 真相源:本檔定義 timeline JSON 的語意與編譯規則。schema 檔(`schemas/timeline.v0.schema.json`)是本檔的機器版,兩者不一致以本檔為準並修 schema。
> 狀態:📋 草案,2026-08-30。尚無實作。

---

## 0. 一句話

**timeline JSON → 一條 ffmpeg 指令。** 中間不落檔、只編碼一次、同輸入同輸出。

## 1. 範圍

### 做
- 確定性組裝:trim、concat、loop-to-duration、音軌處理(去除 / 保留 / 疊 BGM)、統一 fps / 解析度、encode 參數
- 驗證(schema + 語意)、dry-run plan、真跑
- backend registry(ffmpeg 先)

### 不做(v0)
- overlay / 字卡 / 字幕 / 轉場 / 速度變化 → 留 v1,以「加一個 op」的方式進來
- 多 video track 疊合(picture-in-picture)→ v1
- 創意判斷(自動選段、自動節奏)→ 永遠不做,那是上游 agent 或人的事
- GUI / 雲端

## 2. 核心模型

```
Timeline
├── output     : 成片規格(path / size / fps / codec / crf / faststart)
├── sources    : id → 檔案路徑(唯一的 I/O 邊界)
├── video      : 有序段落列表(順序 = 時間順序,首尾相接;v0 單軌)
└── audio      : 成片音軌策略(strip | keep | bgm)
```

- **sources 是唯一 I/O 邊界**:timeline 內任何地方引用媒體只能用 source id,不能寫路徑。這讓「來源存在性 / 格式探測」集中在一處做。
- **video 是段落列表,不是絕對時間軸**:v0 刻意不引入 `start_at` 絕對定位,避免重疊 / 空洞的語意問題。段落依序接起來,總長 = Σ 各段長度。
- **audio 是策略不是軌道**:v0 成片只有一條音軌,由策略決定內容。

## 3. Schema(v0)

```jsonc
{
  "version": "0",                       // 必填,固定 "0"

  "output": {
    "path": "out/film.mp4",             // 必填;相對路徑以 timeline.json 所在目錄為基準
    "size": "1080x1920",                // 選填;省略 = 取第一段來源的解析度
    "fps": 24,                          // 選填;省略 = 取第一段來源的 fps
    "codec": "libx264",                 // 選填,預設 libx264
    "crf": 18,                          // 選填,預設 18
    "pix_fmt": "yuv420p",               // 選填,預設 yuv420p
    "faststart": true                   // 選填,預設 true(-movflags +faststart)
  },

  "sources": {                          // 必填,至少一個
    "<id>": "path/to/file"              // id: ^[a-z][a-z0-9_-]*$
  },

  "video": [                            // 必填,至少一段
    {
      "src": "<id>",                    // 必填,指向 sources
      "trim": { "start": 0.5, "dur": 3.2 },   // 選填;start 預設 0;dur 省略 = 到來源結尾;二選一可給 "end"
      "loop_to": 20                     // 選填;把本段重播到指定秒數(與 trim 可並用:先 trim 再 loop)
    }
  ],

  "audio": {                            // 選填;省略 = { "mode": "strip" }
    "mode": "strip" | "keep" | "bgm",
    // mode=keep:保留各段來源音軌並隨 concat 接起來(來源缺音軌 → 驗證錯)
    // mode=bgm:
    "src": "<id>",                      // bgm 必填
    "volume": 0.3,                      // 選填,預設 1.0
    "offset": 0,                        // 選填,BGM 從第幾秒開始播,預設 0
    "loop": true,                       // 選填,預設 true;BGM 短於成片時重播
    "fade_out": 1.0                     // 選填,結尾淡出秒數,預設 0
  }
}
```

### 3.1 語意規則(schema 管不到、驗證器要管)
| 規則 | 錯誤訊息要說 |
|---|---|
| `video[].src` / `audio.src` 必須在 `sources` | 哪一段、id 是什麼、sources 有哪些 |
| `sources` 路徑必須存在且 ffprobe 讀得到 | 哪個 id、解析後絕對路徑 |
| `trim.start` ≥ 0,`dur` > 0,`start + dur` ≤ 來源長度 | 來源實際長度是多少 |
| `trim` 不可同時給 `dur` 和 `end` | — |
| `loop_to` 必須 > trim 後段長 | 段長多少、loop_to 多少 |
| `audio.mode=keep` 時每段來源都要有音軌 | 哪個 id 沒有 |
| `output.size` 格式 `WxH`,W/H 皆偶數 | yuv420p 要求偶數 |

## 4. 編譯規則(ffmpeg backend)

### 4.1 一條指令
輸出永遠是**一個** `ffmpeg` argv:`-i` × N + 一張 `-filter_complex` 圖 + `-map` + encode 參數 + output。沒有 shell pipeline、沒有暫存檔。

### 4.2 每個 source 先 normalize(最重要的坑)
`concat` filter 要求所有輸入的解析度、fps、SAR、pix_fmt、timebase 一致,否則直接炸或畫面錯位。所以**每個 video 段進圖前一律經過 normalize 鏈**:

```
[i:v] trim=start:end, setpts=PTS-STARTPTS,
      scale=W:H:force_original_aspect_ratio=decrease, pad=W:H:(ow-iw)/2:(oh-ih)/2,
      setsar=1, fps=FPS, format=yuv420p  [vN]
```

- scale + pad = letterbox / pillarbox 保比例,不裁不拉
- `setpts=PTS-STARTPTS` 讓 trim 後時間從 0 起算,concat 才不會留空洞
- 音軌(mode=keep)同理:`atrim, asetpts, aresample=48000, aformat=stereo`

### 4.3 loop_to
段落級 loop 用 `loop` filter 需知道幀數,不穩;v0 改用**輸入級** `-stream_loop -1` + normalize 鏈裡的 `trim=duration=loop_to`。代價:同一 source 若一段要 loop 一段不要,要當兩個 `-i` 進來(編譯器自動處理,對使用者透明)。

### 4.4 concat
```
[v0][v1]...[vN-1] concat=n=N:v=1:a=0 [vout]        # audio strip / bgm
[v0][a0][v1][a1]... concat=n=N:v=1:a=1 [vout][aout] # audio keep
```

### 4.5 audio 策略
| mode | 圖 |
|---|---|
| strip | 不 map 音軌,輸出加 `-an` |
| keep | 隨 concat 出 `[aout]`,`-map [aout]` |
| bgm | BGM 當額外 `-i`(`-stream_loop -1` 若 loop),`volume=V, atrim=0:total, afade=t=out:st=total-F:d=F` → `[aout]`;`-shortest` 保底 |

### 4.6 encode
`-c:v {codec} -crf {crf} -pix_fmt {pix_fmt} [-c:a aac -b:a 192k] [-movflags +faststart] -y {path}`

### 4.7 確定性
- 同一份 JSON + 同一批 source(路徑 + mtime + size)→ argv 逐字相同
- plan 輸出附 `sha256(argv)`,consumer 可拿它當 cache key

## 5. CLI

```
seasplice validate <timeline.json>            # 只驗證;exit 0/1;錯誤寫 stderr,agent 可讀
seasplice plan     <timeline.json> [--json]   # 驗證 + 印 argv(shell-quoted)+ 逐步中文說明;--json 給機器
seasplice render   <timeline.json> --run      # plan + 真跑;沒 --run 等同 plan(dry-run 預設)
seasplice probe    <file>                     # ffprobe 摘要(長度 / 解析度 / fps / 有無音軌),給 agent 填 JSON 前看
```

實作:Python 單檔 uv inline script(PEP 723),依賴只有 `jsonschema`;ffmpeg / ffprobe 走 subprocess。

## 6. Backend registry

```
Renderer(ABC): available() · build_plan(timeline) -> Plan · render(plan)
  ffmpeg   ← v0 唯一實作
  moviepy  ← 佔位(複雜合成需求出現才做)
  fcpxml   ← 佔位(exporter:同一份 JSON 出 NLE 交換檔,接人工收尾路)
```

JSON 不綁 backend;`plan --backend` 可切。

## 7. 與 aura-stream 既有工具的對應

| aura-stream `tools/` | seasplice 對應 | 去向 |
|---|---|---|
| `cut_clip.py`(trim + 壓 fps + 去音) | `video[].trim` + `output.fps` + `audio.mode=strip` | 退役成 op |
| `export_film.py`(多段裁頭尾接起來) | `video[]` 列表 | 退役成 op |
| `strip_audio.py` | `audio.mode=strip` | 退役成 op |
| `toolbox/render/ffmpeg_render.py`(loop + BGM) | `video[].loop_to` + `audio.mode=bgm` | 退役,registry 骨架可搬來 |
| `align_audio.py` / `beat_grid.py`(量測) | **不對應** —— 它們是「量出數字」的工具 | 保留;輸出改成可直接填進 JSON 的片段(如 `audio.offset`) |
| `extract_frame.py` / `silence_check.py` / `waveform.py` | 不對應(檢視 / 診斷) | 保留 |

## 8. Roadmap

| 版 | 內容 | 觸發條件 |
|---|---|---|
| **v0** | 本檔全部:schema、validate / plan / render、ffmpeg backend、normalize、strip/keep/bgm | 現在 |
| v1 | `overlay`(圖 / 文字 / 浮水印)、`xfade` 轉場、`speed`、多 video track | aura-stream 某頻道真的需要 |
| v2 | fcpxml / otio exporter(人工收尾路)、MoviePy backend | history-suspense 級重片開工 |

## 9. 已知風險

- **`-shortest` 語意**:ffmpeg 的 `-shortest` 在有 filter 時行為不直觀,bgm 模式要靠 `atrim` 明確截長度,不能只靠 `-shortest`
- **timebase 不一致**:normalize 鏈加 `settb=AVTB` 保險;concat 前若仍炸,是這裡
- **VFR 來源**(手機錄影常見):`fps=` filter 會轉 CFR,但 trim 秒數會有 ±1 幀誤差;可接受,文件講清楚
- **音訊 sample rate**:mode=keep 混不同 sample rate 的來源要 `aresample`,否則 concat 拒絕

## 10. 待拍板

- [ ] `trim` 用 `end` 還是 `dur` 當主要寫法(草案:兩者皆收,`dur` 優先文件化,對齊 `export_film` 的 `a.mp4:start:dur` 慣例)
- [ ] `output.size` 省略時取「第一段」還是「最大解析度」(草案:第一段,可預測)
- [ ] 是否 v0 就給 `video[].fps_override`(對齊 `cut_clip` 壓 24fps 省 fal 費用的需求)—— 草案:不給,`output.fps` 已覆蓋
