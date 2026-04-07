# DeerFlow Harness 学习指南

这份文档只聚焦 `backend/packages/harness/deerflow/` 里的核心 harness 代码，目标不是“看完所有文件”，而是先建立正确的运行心智模型，再按主链路逐层展开。

## 先说结论

如果你把后端分成两层，你的判断大体是准确的：

- `backend/app/` 更像接入层和适配层，负责 FastAPI gateway、IM channel、HTTP 路由、服务生命周期。
- `backend/packages/harness/deerflow/` 是核心运行时，负责 agent 组装、配置加载、工具系统、sandbox、subagent、skills、memory、runtime store/stream bridge。

但有一个重要补充：

- “配置”并不只在 `app/`。
- 真正的全局配置入口在 `config/app_config.py`，`app/` 只是消费这些配置并把它们接到 gateway 和 channel 生命周期里。

所以更准确的表述是：

- `app/` = 外围接入层
- `deerflow/` = 核心 harness 层
- `deerflow/config/` = 核心配置模型与加载入口

## 先建立整体地图

建议先把 `deerflow/` 理解成 6 个核心块，再去看细节：

1. `config/`
   应用配置、模型配置、工具配置、sandbox 配置、memory 配置等都从这里进入系统。

2. `agents/`
   定义 lead agent、middleware 链、线程状态、checkpointer、memory 更新逻辑，是整个 harness 的“大脑中枢”。

3. `tools/`
   管理 DeerFlow 可暴露给模型的工具，包括 builtin tools、MCP tools、ACP tools、延迟搜索工具。

4. `sandbox/`
   提供文件系统和命令执行的隔离边界，以及与 agent middleware 的衔接。

5. `runtime/`
   负责 run 生命周期、流式桥接、store、序列化，是“把 agent 跑起来”的基础设施层。

6. `skills/` 和 `subagents/`
   前者是技能包加载与管理，后者是并行任务和子 agent 扩展能力。

其他目录可以暂时放在第二优先级：

- `models/`: 模型工厂与 provider 适配
- `mcp/`: MCP 工具接入
- `guardrails/`: 护栏抽象
- `uploads/`: 文件上传配套逻辑
- `community/`: 社区扩展工具，不是理解主链路的第一入口
- `reflection/`, `tracing/`, `utils/`: 支撑性模块

## 最推荐的学习顺序

不要按文件名从上到下硬读。正确方式是沿着“配置 -> 组装 agent -> 注入工具和中间件 -> 执行 runtime -> 接入层调用”这条主线走。

### 第 1 步：先看系统从哪里启动

先读：

- `../../langgraph.json`
- `agents/lead_agent/agent.py`

你要搞清楚两件事：

- LangGraph 的 graph 入口是谁
- 真正创建 lead agent 的函数是谁

读完后你应该能回答：

- 为什么项目最终会落到 `make_lead_agent`
- runtime 请求里的 `configurable` 参数是怎么影响 agent 行为的

重点关注：

- `make_lead_agent`
- `_build_middlewares`
- `_resolve_model_name`

## 第 2 步：搞懂全局配置是怎么进入系统的

接着读：

- `config/app_config.py`
- `config/model_config.py`
- `config/tool_config.py`
- `config/sandbox_config.py`
- `config/memory_config.py`
- `config/subagents_config.py`

这一层的目标不是记住所有字段，而是搞清楚配置流向：

- `config.yaml` 如何被定位和加载
- 环境变量如何注入配置
- 哪些子系统会从全局配置里取值

读这一层时，你要特别建立一个意识：

- DeerFlow 的大量行为不是硬编码在 agent 里，而是通过配置和 feature 开关驱动的。

## 第 3 步：把 agent 的组装过程吃透

然后读：

- `agents/factory.py`
- `agents/features.py`
- `agents/thread_state.py`
- `agents/lead_agent/prompt.py`

这里有两条线：

- `create_deerflow_agent`: 更偏 SDK/纯参数工厂
- `make_lead_agent`: 更偏应用层/配置驱动工厂

这是理解这个项目非常关键的一步。

你需要分清：

- 什么是“纯组装能力”
- 什么是“带全局配置和默认行为的应用装配”

如果这一层看懂了，你后面看 middleware、tools、subagents 都不会散。

## 第 4 步：按顺序读 middleware 链

接着集中读：

- `agents/middlewares/`
- `guardrails/middleware.py`
- `sandbox/middleware.py`

建议优先顺序：

1. `thread_data_middleware.py`
2. `uploads_middleware.py`
3. `tool_error_handling_middleware.py`
4. `memory_middleware.py`
5. `todo_middleware.py`
6. `title_middleware.py`
7. `subagent_limit_middleware.py`
8. `loop_detection_middleware.py`
9. `clarification_middleware.py`

这一层的学习目标是回答：

- 每个 middleware 在请求生命周期的哪个位置生效
- 哪些 middleware 是基础设施型
- 哪些 middleware 是产品能力型
- 为什么 `ClarificationMiddleware` 被放在靠后甚至最后

建议你一边读 `agents/lead_agent/agent.py` 里的 `_build_middlewares`，一边回到具体 middleware 文件看实现，不要孤立阅读。

## 第 5 步：搞懂工具系统如何暴露给模型

重点读：

- `tools/tools.py`
- `tools/builtins/`
- `mcp/tools.py`
- `community/` 下与你当前关注点相关的工具实现

这里要搞清楚四件事：

- 工具配置来自哪里
- builtin tool 和配置工具如何拼装
- 何时追加 MCP 工具
- 何时追加 ACP agent 工具与 subagent 工具

学习时建议先看：

- `tools/tools.py`
- `tools/builtins/task_tool.py`
- `tools/builtins/clarification_tool.py`
- `tools/builtins/view_image_tool.py`

如果你当前正研究图像或搜索相关能力，再进入：

- `community/image_search/tools.py`
- `community/jina_ai/`
- `community/ddg_search/`
- `community/firecrawl/`
- `community/tavily/`

但请注意：

- `community/` 更适合作为“能力插件层”来读，不要把它当成主入口。

## 第 6 步：把 sandbox 当成安全边界来理解

重点读：

- `sandbox/sandbox.py`
- `sandbox/sandbox_provider.py`
- `sandbox/security.py`
- `sandbox/tools.py`
- `sandbox/local/local_sandbox_provider.py`
- `sandbox/local/local_sandbox.py`

这一层不要只把它看成“执行命令的地方”，而要理解它是：

- 文件系统访问控制边界
- 工具调用的执行环境
- host bash 是否允许暴露给模型的安全判断来源

你要重点回答：

- 为什么 `tools/tools.py` 会依赖 `sandbox/security.py`
- sandbox middleware 和工具层是怎么配合的
- local provider 与 community provider 的边界在哪里

## 第 7 步：理解 runtime 是如何托管一次运行的

重点读：

- `runtime/__init__.py`
- `runtime/runs/manager.py`
- `runtime/runs/worker.py`
- `runtime/runs/schemas.py`
- `runtime/stream_bridge/`
- `runtime/store/`
- `runtime/serialization.py`

如果你想真正理解 gateway 为什么能提供 runs 接口，这一层必须吃透。

学习目标：

- run record 是怎么创建、追踪和结束的
- stream bridge 怎么把 agent 事件桥接到上层接口
- store 为什么独立于 checkpointer
- 哪些数据属于线程状态，哪些数据属于运行时托管元数据

## 第 8 步：再读 subagents 和 skills

先看：

- `subagents/registry.py`
- `subagents/config.py`
- `subagents/executor.py`
- `subagents/builtins/`

再看：

- `skills/loader.py`
- `skills/parser.py`
- `skills/manager.py`
- `skills/installer.py`
- `skills/validation.py`

理解这两块时要有一个区分：

- subagent 解决的是“任务拆分与执行扩展”
- skills 解决的是“能力包发现、安装、启停、解析”

很多人第一次看这套代码会把二者混在一起，这是阅读上的常见误区。

## 第 9 步：最后再回头看 app 层

当你已经看懂上面主链路后，再回头看：

- `../../app/gateway/app.py`
- `../../app/gateway/deps.py`
- `../../app/gateway/services.py`
- `../../app/channels/service.py`
- `../../app/channels/manager.py`

这时你会发现 app 层主要做的是：

- 启动服务
- 挂路由
- 初始化 runtime 单例
- 接收 HTTP 或 IM channel 输入
- 把请求翻译成 harness 能理解的运行参数

也就是说，app 层很重要，但它主要是“承载核心能力”，不是“定义核心能力”。

## 一个高效的实战阅读法

如果你想在最短时间内吃透 harness，不要只看实现，建议采用下面的节奏：

1. 先沿主链路读一遍，不纠缠细节。
2. 第二遍只追配置流：配置从哪里来，流到哪里去。
3. 第三遍只追工具流：工具如何被注册、筛选、暴露、调用。
4. 第四遍只追状态流：thread state、checkpointer、store、run manager 各管什么。
5. 第五遍再补充可选扩展：skills、subagents、community、mcp。

这个顺序比“按目录顺序把文件全看完”有效得多。

## 推荐你边读边验证的测试

学习 DeerFlow 最快的方法之一，是把测试当成“行为说明书”。

建议优先看这些测试：

- `../../../tests/test_lead_agent_model_resolution.py`
- `../../../tests/test_lead_agent_prompt.py`
- `../../../tests/test_lead_agent_skills.py`
- `../../../tests/test_custom_agent.py`
- `../../../tests/test_guardrail_middleware.py`
- `../../../tests/test_dangling_tool_call_middleware.py`
- `../../../tests/test_llm_error_handling_middleware.py`
- `../../../tests/test_checkpointer.py`
- `../../../tests/test_checkpointer_none_fix.py`
- `../../../tests/test_gateway_services.py`
- `../../../tests/test_channels.py`
- `../../../tests/test_channel_file_attachments.py`

推荐方法不是一口气全跑，而是：

- 读完一个模块，就找对应测试
- 先看测试名，再看断言，再回实现

这样你会更快抓到作者真正想保证的行为。

## 我建议你当前就按下面这个顺序开始

如果你现在准备正式学，我建议从这 12 个文件开始，顺序不要改：

1. `../../langgraph.json`
2. `config/app_config.py`
3. `agents/lead_agent/agent.py`
4. `agents/factory.py`
5. `agents/thread_state.py`
6. `tools/tools.py`
7. `sandbox/middleware.py`
8. `sandbox/security.py`
9. `runtime/runs/manager.py`
10. `runtime/stream_bridge/base.py`
11. `subagents/executor.py`
12. `skills/loader.py`

这 12 个文件读完，你对 DeerFlow harness 的主干理解就会比较稳。

## 读代码时最该追的 8 个问题

每读一个模块，都强迫自己回答这 8 个问题：

1. 这个模块的输入是什么。
2. 这个模块的输出是什么。
3. 它依赖哪些全局配置。
4. 它修改了哪些运行时状态。
5. 它属于主链路还是扩展链路。
6. 它失败时由谁兜底。
7. 它和 middleware、tool、runtime 三者中的哪一层有关。
8. 有没有对应测试能证明你的理解。

如果这 8 个问题都答得出来，这个模块基本就算学透了。

## 一个简化后的心智模型

你可以先把 DeerFlow harness 暂时压缩成下面这条链：

`config` 提供配置 -> `lead_agent` 组装 agent -> `middlewares` 塑造运行行为 -> `tools`/`sandbox` 提供可执行能力 -> `runtime` 托管运行生命周期 -> `app` 把 HTTP 和 channel 请求接进来

这条主链抓住了，后面的 skills、mcp、community、uploads 都只是往上挂能力，不会再显得乱。

## 最后给你的学习建议

第一次学习时，请克制住两种冲动：

- 不要一开始就钻 `community/`，它会把你的注意力带到插件细节里。
- 不要一开始就把 `app/` 当业务核心，它更多是集成层而不是算法核心。

你应该先吃透：

- 配置入口
- lead agent 装配
- middleware 链
- 工具装配
- runtime 托管

这五部分吃透以后，再去看 gateway、channels、skills、subagents，阅读成本会明显下降。