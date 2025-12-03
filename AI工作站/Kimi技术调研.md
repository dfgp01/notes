
| 产品        | 功能  | 模型      | 模态                         | 版本   | 程序入口                                                                                     | 组织                | 推理类型      | 规模/精度        |
| --------- | --- | ------- | -------------------------- | ---- | ---------------------------------------------------------------------------------------- | ----------------- | --------- | ------------ |
| Kimi 智能助手 | 聊天  | Kimi-LM | Language ↔ Language（纯文本对话） | v1.5 | Web 端（kimi.moonshot.cn）<br> PC 客户端（Windows / macOS）<br>小程序（微信 / 支付宝）<br>手机 App（iOS / 安卓） | 月之暗面（Moonshot AI） | 推理（链式思考）  | 200 B / FP16 |
|           | 文生图 | Kimi-VL | Language → Vision（文本→图像）   | v1.5 | Web 端（kimi.moonshot.cn）<br> PC 客户端（Windows / macOS）<br>手机 App（iOS / 安卓）                  | 月之暗面（Moonshot AI） | 非推理（单次扩散） | 18 B / FP16  |
|           |     |         |                            |      |                                                                                          |                   |           |              |

### 为什么表格里同时出现 Kimi-LM / Kimi-VL，而官方又说「基座是 Kimi-K2」？
 
Kimi-LM、Kimi-VL 是产品对普通用户的命名习惯
– LM = Language Model，负责纯文本（聊天、总结、问答）
– VL = Vision-Language，负责图生文、文生图
 
Kimi-K2 是技术代号，指 2024-12 发布的第二代 MoE 万亿基座；它下面再细分：
– K2-Instruct（短链，默认）
– K2-Thinking（长链，深度思考）
也就是说，Kimi-LM 现在就是 Kimi-K2-Instruct，只是对外还沿用旧简称；Kimi-VL 则是把 K2 视觉编码器加进来，对外继续叫 VL。
所以表格里写「Kimi-LM」没错，它背后就是 K2 权重；写「Kimi-K2」更精确，但怕用户看不懂，产品层仍保留 LM / VL 的牌子。

### API 里为什么又冒出 kimi-k2-thinking / kimi-thinking-preview？
 月之暗面在开放平台上把「长链版」单独挂出来收费，于是给了一个显式 model 名，方便开发者显式调用。
 
命名规则：
- kimi-k2  → 普通 K2-Instruct（短链，低延迟）
- kimi-k2-thinking  → 同基座，但训练里加了 Long-CoT，推理会输出 reasoning_content
- kimi-thinking-preview  是早期灰度名，后续会统一到  kimi-k2-thinking 
所以不是额外一款异构模型，只是同一 K2 权重的「长链模式」被单独定价、单独挂牌。

### thinking 一定要手动开「深度思考」吗？不开就回落到 K2-Instruct？
 
产品侧：
- App/Web 的「深度思考」开关默认关 → 走的 K2-Instruct
- 打开开关 → 后台路由到 K2-Thinking，先流式吐思考过程，再给最终答案
 
API 侧：
- 你不传  model="kimi-k2-thinking"  就默认  kimi-k2 （也就是 K2-Instruct）
- 想调用长链必须显式指名，没有「自动 fallback」

一句话速记 名字三级跳：
对外产品名（LM / VL） → 技术基座名（K2） → 平台 API 名（k2 / k2-thinking）
本质同一套权重，只是「短链 / 长链」两种调用策略，开关或 model 参数决定到底用哪一档


### 开发者视角：Kimi 开放模型速查表

| base_id | model_name | 能力关键词 | 上下文长度 | 最大输出 | 是否含思维链 | 单价 (input/output) | 计费币种 | 备注 |
|---|---|---|---|---|---|---|---|---|
| k2 | kimi-k2 | 通用聊天、总结、工具调用 | 128 k | 4 k | 否 | 1.0 元 / 1.0 元 / MTok | CNY | 默认短链，延迟最低 |
| k2-thinking | kimi-k2-thinking | 数学、逻辑、代码推理 | 128 k | 8 k | 是（先吐 reasoning_content） | 1.5 元 / 3.0 元 / MTok | CNY | 长链模式，速度略慢 |
| vl-3 | kimi-vl-3 | 图→文、文→图、OCR | 8 k | 2 k | 否 | 1.2 元 / 1.2 元 / MTok | CNY | 视觉编码器 448×448 |
| tts-1 | kimi-tts-1 | 文本→语音 | 4 k | 音频 10 min | — | 0.015 元 / 千字符 | CNY | 仅支持中文语音 |
| embedding-2 | kimi-embedding-2 | 向量化 | 2 k | 2048 dim | — | 0.0005 元 / 1k tokens | CNY | 维度 2048，余弦归一化 |

说明
1. base_id：对应官方 /v1/models 返回的  base_id ，用于判断同属一个基座权重。
2. model_name：调用时必传的  model="xxx"  字段，大小写敏感。
3. 单价单位均为「每 1M tokens」；TTS 按字符、Embedding 按 token。
4. 上下文长度与最大输出均为官方上限，实际受账号等级 RPM/TPM 限制。
5. 思维链字段「是」代表会流式返回  reasoning_content ，需要前端单独解析。

后续维护
1. 官方新增模型 → 直接 append 一行，不必改表头。 
2. 价格调整 → 只改单价列即可。
3. 产品侧表格继续保留 LM/VL 昵称，与这张开发表互相对应，互不干扰