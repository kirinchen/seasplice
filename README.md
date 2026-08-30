# seasplice

> 宣告式影片剪輯 DSL:一份 timeline JSON → 一次編譯成**單條** ffmpeg 指令。
> 給 agent 用的剪輯語言 —— 改 JSON 重編,不重跑一串 CLI、不落中間檔。

**狀態:📋 Spec 階段**(只有規格,尚無實作)。規格見 [`doc/SPEC.md`](doc/SPEC.md)。

---

## 為什麼

AI 影片工廠的剪輯通常長成一串各自獨立的命令式工具:
`cut → concat → strip_audio → …`,每步各自重編碼、各落一個中間檔。
反悔一步 = 重跑下游全部;`clip1-raw / clip1 / clip2-nochain / clip2` 這種檔名就是代價。

Creatomate / Shotstack 證明了另一條路:**剪輯決策用 JSON 寫,渲染只在最後做一次**。
seasplice 把這條路做成本機、免費、確定性的版本:

| | 命令式工具鏈 | seasplice |
|---|---|---|
| 反悔成本 | 重跑下游全部 | 改 JSON 重編 |
| 編碼次數 | 每步一次(累積失真) | 一次 |
| 中間檔 | 每步一個 | 無 |
| agent 友善 | 要記 N 支 CLI 的參數 | 一份 schema,產 JSON → validate → dry-run |
| 擴功能 | 再寫一支 CLI | 加一個 op |

## 定位(一句話)

**只做「確定性組裝」**:concat、trim、loop、疊 BGM、字卡、encode —— 沒有創意判斷、可 CI、可重跑的那一層。
創意剪輯(節奏、B-roll 選擇、轉場情緒)不是它的事,那是人 + NLE 的事。

## 用法草圖(目標介面,尚未實作)

```bash
seasplice validate timeline.json          # schema + 語意檢查(來源存在、時間不重疊…)
seasplice plan     timeline.json          # dry-run:印出將執行的單條 ffmpeg 指令 + 逐步說明
seasplice render   timeline.json --run    # 真跑
```

```jsonc
{
  "version": "0",
  "output": { "path": "out/film.mp4", "size": "1080x1920", "fps": 24 },
  "sources": {
    "a": "clips/a.mp4",
    "b": "clips/b.mp4",
    "bgm": "audio/song.mp3"
  },
  "video": [
    { "src": "a", "trim": { "start": 0.5 } },
    { "src": "b", "trim": { "start": 0, "dur": 3.2 } }
  ],
  "audio": { "mode": "bgm", "src": "bgm", "volume": 0.3, "loop": true }
}
```

## 設計原則

1. **Deterministic-first** —— 同一份 JSON + 同一批來源 → 同一條 ffmpeg 指令。LLM 只負責產 JSON,不碰渲染。
2. **單條指令** —— 整個 timeline 編成一張 `filter_complex` 圖,不落中間檔。
3. **dry-run 預設** —— `plan` 永遠先於 `render`;agent 看得懂 plan 才 `--run`。
4. **schema 守門** —— JSON Schema 在 `schemas/`;錯誤訊息寫給 agent 看(說哪個欄位、為什麼、怎麼改)。
5. **backend 可換** —— JSON 是介面,ffmpeg 是第一個 backend;MoviePy / FCPXML exporter 是之後的事。

## 非目標

- 不做 GUI、不做時間軸編輯器
- 不做創意剪輯(自動選 B-roll、自動節奏)
- 不做雲端渲染服務
- v0 不做 overlay / 轉場 / 字幕(需求出現再加 op)

## 關聯

- 首個 consumer:作者自己的 AI 影片工廠(private),走「自動 render 路」(無創意判斷的確定性組裝)

## License

MIT
