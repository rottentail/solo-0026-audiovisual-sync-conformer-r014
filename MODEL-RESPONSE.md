# 这两个读数各自证明了什么

## 读数的实际来源

`internal/media/signal.go:11-44` 里只有两个原始量：

- `audioEventMS`：在 `detectAudioOnset` 中按 5 ms RMS 窗找到首个“前面安静、当前越过阈值”的窗，再在样本级向前找到首个 `|sample| >= peak*0.12` 的点，最后 `precise*1000/(rate*channels)` 整数截断（`signal.go:97-106`）。
- `videoEventMS`：`detectPictureFlash` 算每帧平均亮度，取中位数为基线，`baseline+0.55*delta` 为阈值，取首个越阈值的**帧号**，再 `index*1000*fpsDen/fpsNum`（`signal.go:150-160`）。

`offset = audioEvent - videoEvent`（正数表示音频晚），`signal.go:28`。

`drift` 不是独立测量量：

```
drift = round(offset * 1e6 / (max(audioEvent, videoEvent) * 1000))     signal.go:29-33
```

即 `drift ≈ offset / 较晚事件时刻 × 1000`。它是同一对数字的确定性函数，自由度仍然是 1，不是 2。并且这个式子有两处实现偏差：

1. 量纲差了 1000 倍。125 ms 偏移、标记在 500 ms，真实比值是 250000 ppm，返回 250；标记在 60 s，真实约 2083 ppm，返回 2。
2. 分母用的是“标记事件时刻”，不是两次测量之间的间隔或整条带子长度。同一个物理延迟，标记放在片头还是片尾，`drift_ppm` 会差几十倍。

## offset 能证明的

只证明一件事：**在检测到的这一对事件所处的那一个时刻，音频内容相对画面内容晚了（或早了）N 毫秒**，而且这个 N 的分辨率受限于两端检测器：

- 音频端：5 ms 窗 + 1 ms 整数截断 + 阈值首次越点，偏差约 ±1~5 ms；
- 画面端：事件落在**帧起点**，24 fps 下量化误差 ±41.7 ms，29.97 fps ±33.4 ms，比多数真实偏移还大；
- 两端都只取“第一个”满足条件的事件（`signal.go:79-93`、`signal.go:151-156`），不验证声画事件是否同一个物理事件。所以它也不证明“偏移来自时钟”，更不证明“闪光是那声拍手”。

`offset` 的含义还取决于标记放在带子什么位置：若真实原因只有速率失配 r，标记在 t 处量到的 offset 就是 `r·t`。标记在 60 s 处读到 6 ms，并不代表“延迟 6 ms”，它只代表“60 s 处累计差 6 ms”。

## drift_ppm 能证明的

在这个实现里，**它不能证明任何关于漂移的东西**：

- 它完全由 offset 和事件时刻推导，没有任何第二处采样，因此无法区分“常数延迟”和“速率失配”。
- 偏移为 0 时恒为 0，与真实是否存在速率失配无关（例如标记恰在片头，真漂移 100 ppm，读数为 0）。
- 一片带子只有一对标记时，“漂移”在数学上无定义，此处只是用 `offset/span` 做了一个没有物理依据的线性外推。
- `DriftPPM` 是非指针字段、`drift_ppm` 无 `omitempty`（`internal/domain/model.go:54`），所以失败的分析也会输出 `drift_ppm: 0`，容易被下游当成“无漂移”。
- 另一个相关但独立的问题：`service.go:116-123` 先写入事件时刻，再在 `service.go:125` 因置信度/越界失败，`failAnalysis` 会把这些字段一起持久化。也就是说 PROJECT_SPEC “失败分析不暴露可用事件对”的约定实际没有守住，失败记录里的 `offset_ms` / `drift_ppm` 仍可读。

`confidence` 也不是对齐质量：它是 `(audioProminence+videoProminence)/2`（`signal.go:34`），音频端是峰值与底噪比的 log（`signal.go:116`），画面端是 `delta/80`（`signal.go:161`），衡量的是幅度显著度，不是“这声和这闪是同一事件”的置信度。

---

# 相同读数、不同场景、不同校正

下面这些场景在当前代码里可以产生完全相同的 `offset_ms` / `drift_ppm`，但正确校正方式互斥：

| 场景 | 读数表现 | 需要的校正 | 当前 `AlignAudio` 会怎样 |
|---|---|---|---|
| 录音机输入缓冲 / 无线话筒固定延迟 | 全片常数 offset，drift 随标记位置变 | 整条刚性平移 `-offset` | ✅ 正确 |
| 声画时钟速率失配（如音频晶振偏 100 ppm） | 标记越靠后 offset 越大；同一物理原因在不同标记位置读数不同 | 需要时变校正（重采样/变比率），纯平移只在检测点正确 | ❌ 只平移，残差随时间线性增长：100 ppm × 1 h = 360 ms |
| 录音机提前开录（片头空白/预卷） | 大常数 offset | 平移并在头部删除多余额度 | ⚠️ 平移可以，但正偏移时尾部补零（`align.go:27-32`），输出长度不变，等于丢尾 |
| 闪光与声音有物理传播延迟（机位距离） | 常数小 offset（约 3 ms/m） | 刚性平移 | ✅ 数值上可，但语义上不是时钟问题，不应写进“漂移” |
| Y4M 曾抽帧/变帧率但头里仍写标称 FPS | 画面事件时间由帧号×标称帧率算出，本身就错 | 必须修画面节奏；移音频无法解决 | ❌ 会把画面时基错误吸收成音频偏移 |
| 24 fps 量化：真实偏移 83.3 ms 与 125 ms 的同一闪帧 | 都落到同一帧，读数相同 | 需要亚帧定位或更长基线 | ❌ 误差被当成真实延迟写进 receipt |
| 双声道有一路延迟若干样本 | 用两声道平均 RMS 检测，读出小 offset | 需要逐声道校正 | ❌ 全局平移，一路修好一路更差 |
| 周期性节拍 + 多次闪光 | 检测器各取第一个，可能配错，读数与真值模周期同余 | 需要事件配对/顺序匹配 | ❌ 按错误配对平移 |
| 片头/片尾各一次标记的真实漂移 | 只有一个 offset 字段 | 需要用斜率而不是位移 | ❌ |

关键点：**offset 是“检测点处的位移”，drift 是“用位移除以检测点时刻”，两者都不是“校正模型”**。而 `AlignAudio`（`internal/media/align.go:24-32`）只有 `frameDelta` 一个自由度，`DriftPPM` 从未被 conform 读取（全仓库只有 `model.go`、`service.go:120`、`signal.go:38` 三处写，无读取方）。所以漂移读数目前对校正完全没有影响。

另外两个上限都拦不住漂移：`max_offset_ms`（默认 1000）和 `max_correction_ms`（默认 500）限制的是毫秒位移，带子时长不参与判断。100 ppm 在 60 s 上只有 6 ms，轻松通过，但刚性平移后仍有 6 ms 残差；在 2 h 上残差 720 ms。

`render-conform` 的自检同样无法暴露这一点：`cmd/syncflowcheck/main.go:282-288` 对输出重新跑**同一个单点检测器**并要求 `|offset| ≤ 2 ms`。刚性平移正是这个检测器假设的模型，所以它必然通过，无法区分“校正正确”和“只在一个点上正确”。

---

# 可行的漂移测量方案对比

| 方案 | 做法 | 精度 | 对现有历史分析 | 对 conform / bundle | 主要代价 |
|---|---|---|---|---|---|
| A. 双锚点线性拟合 | 片头片尾各一个声画标记，两点 offset 连线的斜率即 ppm | 受帧量化限制：24 fps、60 s 间隔约 1390 ppm 分辨率，需亚帧质心或更长时间基线才有实用价值 | 需要 `signal.go` 支持多事件；单标记带子退化为无漂移定义，必须允许“未测”而不是 0 | 需要新的校正策略（重采样），receipt 需记录比率 | 检测器改动最大 |
| B. 波形互相关（已知测试音/拍手） | 在样本域做互相关估计起止两点的分数延迟 | 亚毫秒，最好 | 新代码路径，旧分析记录仍是单点 | 同上 | 需要可辨识信号；不适用于任意素材 |
| C. 用容器时长差做粗估 | `audio.FrameCount/SampleRate` vs `Video.FrameCount*fpsDen/fpsNum`（`wav.go:136`、`y4m.go:75`） | 整数截断各 1 ms，60 s 带子约 17 ppm | 无需改检测器，纯派生量 | 零影响（不进 bundle） | 无法分离“起点差”与“速率差”；只在两端共享同一时间跨度时有意义 |
| D. 校正后残差斜率 | 先按 offset 平移，在输出上再做多锚点检测，斜率即漂移 | 等于 A/B 的检测精度 | 可在不可变素材上重算，但必须新建 analysis_id（同 id 幂等且终态） | 与现有 `audio-aligned.wav` 天然兼容，是最容易验证的方案 | 需要两次检测 |
| E. 嵌入式 LTC/时码 | 解码音频时码轨 | 最高 | 需要新格式支持，超出当前单 WAV 边界 | 大改 | 超出 baseline |

我的建议：以 **A + 亚帧闪光质心 + 样本域起音** 为主，**C 作为先验/合理性校验**，并在 conform 侧保留现有整数平移作为“起点项”，另加一个比率项。单锚点带子必须把漂移置为“未测”（可空），而不是 0。

---

# 对历史分析与校正包的兼容性影响

## 1. 已持久化的 analysis 记录（`analyses/<id>.json`）

- `drift_ppm` 已经以**旧公式（1000 倍偏差、分母是标记时刻）**写进磁盘，而记录里没有 `analyzer_version` 或 `drift_method` 字段。一旦改公式，新写入和旧存证的 `drift_ppm` 不再同尺度，跨记录聚合/比较会失真，且无法从记录本身判断是哪套公式。
- 素材是不可变且保留的（`takes/<id>/`），所以可以用**新的 analysis_id** 重算；但同一 id 的 `CreateAnalysis` 是按 `(TakeID, Policy)` 幂等的（`storage/repository.go:170-174`），不能覆盖终态记录。历史记录只能在“新 id + 显式版本标记”下被重解释。
- 失败分析里已经写入了事件时刻和 `drift_ppm`，若下游把 `drift_ppm` 当权威量，历史失败记录会携带一个旧尺度数值。

## 2. Policy 与校验

- `SyncPolicy` 用结构体 `==` 比较。给 `SyncPolicy` 增加“漂移模式”字段后：新请求与零值不同的模式会正确 409；但**回滚到旧版本**时，旧结构体反序列化会忽略新字段，再把零值与请求零值比较，可能静默返回一条模式不同的既有记录。
- HTTP 请求解析是 `DisallowUnknownFields`（`internal/httpapi/server.go:184`），新客户端向旧服务发送 `drift_mode` 会直接 400，回滚窗口内不可用；响应侧是普通 `json.Unmarshal`，新增响应字段向后兼容。

## 3. conform receipt 与 bundle

- `ConformReceipt.SchemaVersion` 在 `service.go:264` 硬编码为 1，字段只有 `measured_offset_ms` / `applied_offset_ms` / `applied_samples` / 三个 digest（`model.go:82-94`）。**没有任何字段能表达重采样比率、第二锚点、带子时长或残差**。加入时变校正必须递增 schema 并新增字段（如 `applied_rate_ppm`、`drift_method`、`anchor_span_ms`），同时保留旧字段语义，才能让旧 bundle 继续可读。
- 注意现有 `AppliedSamples` 实际存的是**帧数**（`align.go:39` 返回 `frameDelta`），字段名与内容不符。新的漂移字段必须显式标注单位，否则会重复这个歧义。
- ZIP 是确定性打包（`bundle.go:34-48`，Store + 固定 mtime），一旦引入重采样，`audio-aligned.wav` 字节全变，`bundle_digest` 全变。旧 bundle 自身仍自洽（`ReadConformBundle` 校验 `BundleDigest`/`BundleBytes`），但新旧方案产出的 bundle 不可直接按字节比较，历史审计需要带方案版本。
- 校正上限语义：`correction_limit_exceeded` 现在只比较毫秒位移（`service.go:215`、`align.go:15`）。若要支持漂移校正，需要一个按“位移上限 + 速率上限”的双约束，否则 `max_correction_ms: 250` 这种请求会放过一条残差可达数百毫秒的时变校正。
- `ConformJob` 的幂等键是 `(AnalysisID, MaxCorrectionMS)`（`storage/repository.go:237-239`）；若 `max_correction_ms` 语义改成“时间预算”，同 id 重试会 409，需要显式版本化这个键。

## 4. workflow check

`cmd/syncflowcheck/main.go:207-215、286-295` 断言 `offset==125`、`videoEvent==375`、`receipt.AppliedOffsetMS==125`，并用同一检测器验证对齐后 `|offset|≤2`。校准素材只有**一个闪光、一个起音**（`synthetic.go:14-15`），所以在任何新方案下它的漂移都不可辨识。新方案必须定义“单锚点 → 漂移未测”的返回，并把这些断言改成“起点残差 ≤ 阈值”，否则要么无法通过，要么用假漂移值掩盖单点假设。

---

**一句话结论**：`offset` 只证明检测点处的一次相对位移，且受一帧量化和单事件配对假设限制；`drift_ppm` 在当前实现里不是独立测量，只是 `offset/标记时刻` 的换算（还差 1000 倍），既不能证明漂移，也没有进入校正路径。要区分“常数延迟 / 起点差 / 速率失配 / 画面节奏错误”，必须新增至少第二个锚点或独立时长依据，并把漂移作为可空、带方法版本的字段，同时以 schema 递增方式扩展 receipt，保留现有 `offset`/`applied_samples` 语义不动。
