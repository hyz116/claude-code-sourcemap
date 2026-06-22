# Flag vs Hardcode（何时 A/B 测试，何时直接写死）

> Claude Code 用一条清晰判据决定"轻量化优化"是该 flag 测试还是直接硬编码：**输出是否回流主模型推理链**。`tengu_plum_vx3`（WebSearch 的 Haiku 替换）是这条判据的活样本。

## 核心机制

```
某处想用 Haiku 替换主模型？
  │
  ├─ 输出仅供用户/UI 看 / 用作纯工具值
  │     │
  │     └─► 直接硬编码用 Haiku（写死调用 getSmallFastModel）
  │
  └─ 输出会作为 tool_result / context 喂回主模型
        │
        └─► 必须 flag 测试（getFeatureValue_CACHED_MAY_BE_STALE）
              │
              ├─► 实验成功 → 注释 "is always true" 后硬编码
              ├─► 实验失败 → 删除路径，回滚
              └─► 实验进行中 → flag 名加版本号（vx2/vx3）继续迭代
```

判据来自一个具体观察：错误**回流到主推理链**会污染整个会话决策；错误**只到 UI**最多影响一次显示。

## 在 Claude Code 中的体现

### `tengu_plum_vx3` 的具体差异（`tools/WebSearchTool/WebSearchTool.ts:262-291`）

```ts
const useHaiku = getFeatureValue_CACHED_MAY_BE_STALE('tengu_plum_vx3', false)

queryModelWithStreaming({
  thinkingConfig: useHaiku ? { type: 'disabled' } : context.options.thinkingConfig,
  options: {
    model: useHaiku ? getSmallFastModel() : context.options.mainLoopModel,
    toolChoice: useHaiku ? { type: 'tool', name: 'web_search' } : undefined,
    ...
  }
})
```

flag 同时控制三件事，构成一个"激进优化包"：

| 维度 | flag-off（默认） | flag-on |
|---|---|---|
| 模型 | 用户主模型（Opus/Sonnet） | Haiku |
| extended thinking | 继承主会话 | 强制 disabled |
| toolChoice | undefined（模型可拒绝） | 强制 `{type:'tool', name:'web_search'}` |

**关键细节**：`toolChoice` 强制——他们不信 Haiku 能可靠判断"该不该搜"，只信它能把意图翻译成工具调用。**Haiku 的角色被降级为 formatter，不是 router。**

### 风险分级：A 类（写死）vs B 类（flag 测）

| 类别 | 调用点 | 失败代价 | 处理方式 |
|---|---|---|---|
| A | 会话标题（`utils/sessionTitle.ts:87`） | 标题难看 | 写死 |
| A | WebFetch 摘要（`tools/WebFetchTool/utils.ts:503`） | 用户自己判断 | 写死 |
| A | token 估算（`services/tokenEstimation.ts:277`） | 数字不准、重试 | 写死 |
| A | API key ping（`services/api/claude.ts:541`） | 失败重试 | 写死 |
| **B** | **WebSearch 查询改写（`WebSearchTool.ts:280`）** | **污染主推理链** | **flag 测试** |

A 类共同点：输出**不回流主模型**。B 类（目前已知唯一）：搜索结果作为 `tool_result` 直接喂回主循环。

### 毕业机制：另一个 plum 兄弟的归宿

代码里留下另一个证据（`services/compact/microCompact.ts:288`）：

```ts
// Legacy microcompact path removed — tengu_cache_plum_violet is always true.
```

`tengu_cache_plum_violet` 已经毕业——实验成功后硬编码为 true，但代码注释保留了"曾是 flag"的痕迹。

GrowthBook flag 的三条终局：

```
flag 实验 ──┬─► 毕业（硬编码 true，注释留痕）
            ├─► 死亡（删除路径）
            └─► 仍在测试（flag 还活着，如 tengu_plum_vx3）
```

`vx3` 这个后缀说明**至少第 3 次迭代**——前两版（vx1、vx2）失败或被取代。结论尚未稳定。

### Anthropic 的 flag 命名约定

观察 `tengu_*` flag 的命名：

| Flag | 类型 |
|---|---|
| `tengu_plum_vx3` | codename（颜色+物体） |
| `tengu_cache_plum_violet`（已毕业） | codename 同色系 |
| `tengu_satin_quoll` | codename |
| `tengu_amber_quartz_disabled` | codename + 状态 |
| `tengu_slate_thimble` | codename |
| `tengu_explore_agent` | descriptive |
| `tengu_bridge_repl_v2` | descriptive + 版本 |
| `tengu_otk_slot_v1` | descriptive + 版本 |

两类命名对应两类实验：

- **codename 型**（颜色+物体）：敏感实验，从外部看不出在测什么——避免竞争情报泄露
- **descriptive 型**：已公开特性的开关，名字直接告诉你它管什么

`plum_vx3` 用 codename + 版本号，承认"还在试错且不希望外部察觉"。

## 设计原则

| 原则 | 含义 |
|---|---|
| **按"错误是否污染下游"分级风险** | 输出回流主推理 → flag 测；纯内部管道 → 直接写死。比"重要性"更可操作 |
| **降级小模型时剥夺其决策权** | 用 `toolChoice` 强制工具调用，让 Haiku 只做翻译不做判断。"小模型替换"的安全模式 |
| **打包改用一个 flag** | 多个改动语义耦合时（这里：模型+thinking+toolChoice 同属"轻量化"主题）捆绑测，用 1 个 flag 替代 2ⁿ 个分支，加速迭代——代价是**质量回归无法归因到具体改动** |
| **flag 默认值保守** | `defaultValue: false` 表示主模型路径是安全 fallback；激进优化需明确 flip |
| **flag 命名承认实验状态** | `vx3` 比 `_optimized` 诚实，未来开发者一眼看出"还在试错" |
| **保留毕业痕迹** | 注释里留 `is always true` 标记，让后人知道这条路径曾是 flag、何时凝固 |
| **codename 隐藏意图** | 颜色+物体随机命名，让逆向工程者看不出在测什么。这是 SaaS 公司的标准做法 |

## 进一步追问的钩子

1. **`vx1` 和 `vx2` 失败在哪** — 代码里没留下，但版本号已暴露存在过。如果能拿到 GrowthBook 后台数据，可以反推 Haiku 替换的失败模式
2. **是否还有其他 B 类调用点** — 当前只发现 WebSearch 一个会回流主推理的 Haiku 替换点。如果未来加新的，应该都走 flag 路径
3. **打包测的可解释性危机** — 如果 vx3 实验成功了，他们怎么知道是哪一项（model/thinking/toolChoice）贡献了主要收益？这影响下次设计实验的方法

## 关联

- 上层概念：[[model-routing]]（这条判据指导第 ② 层"后台碎活"中哪些走 flag、哪些直接写死）
- 兄弟概念：[[opusplan-tradeoff]]（同样是模型路由动态化的设计取舍，但 opusplan 是 user-driven，flag 是 Anthropic-driven）
- 相关实体：`tools/WebSearchTool/WebSearchTool.ts`、`services/analytics/growthbook.ts`（`getFeatureValue_CACHED_MAY_BE_STALE`）、`services/compact/microCompact.ts:288`（毕业痕迹示例）
- 综合分析：[2.TOOLS.md](../2.TOOLS.md)（WebSearch 在工具体系的位置）
