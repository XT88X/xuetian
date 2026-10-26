# OpenAI上线GPT-6.1 Terra：百万上下文，主打视觉推理与电脑操作

> OpenAI发布GPT6.1Terra | 收集时间：2026-10-27 03:05

## 一、事件概述

2026年10月26日，OpenAI（开放人工智能公司）开始向用户逐步推出GPT-6.1 Terra，同日微软Foundry（Microsoft Foundry，微软面向企业的AI模型与应用开发平台）宣布该模型正式可用（GA，Generally Available）。OpenAI同步发布GPT-6 Astra系统卡附录，专门说明GPT-6.1 Terra的能力边界与安全评估。

该模型被定位为“快速、经济的日常工作模型”，四项核心能力分别为：更强的视觉推理、更快更精准的电脑操作（computer use，指智能体识别屏幕内容并在浏览器、桌面软件中执行点击与输入）、清晰的写作，以及在长任务中向用户发送中间进度消息。规格上，它支持最高100万输入词元（token，模型处理文本的基本单位）的上下文窗口，接受文本与图像输入，可调节推理强度（含“无推理”模式）与回答详细度。

定价是其最大卖点：全球标准部署下，短上下文每百万词元输入1美元、输出5美元；长上下文输入2美元、输出7.5美元，明显低于旗舰模型GPT-6 Astra。OpenAI将两者定位为分工关系：Astra负责开放式、多步骤的规划与推理，Terra承担大批量、以屏幕和图像为基础的执行任务。OpenAI方面同时披露，同系列GPT-6.1 Sol在OSWorld 2.0（评估智能体完成真实电脑操作任务的基准测试）离线集上，以不足GPT-6 Sol一半的成本取得高7个百分点的成绩。

## 二、关键信息点

- 时间：2026年10月26日发布并逐步推送，同日登陆微软Foundry
- 地点：线上发布；通过微软Azure全球区域及美国、欧盟、亚太数据区部署
- 主要当事方：OpenAI、微软；同期竞争者包括Anthropic（人类学公司）、Mistral AI（米斯特拉尔人工智能公司）等
- 事件性质：科技产品发布／人工智能商业化
- 当前进展：模型已向用户分批开放，企业开发者可在Foundry中直接调用，无需更换SDK或接口结构

## 三、影响力分析

- 传播热度：AI开发者社区、模型聚合平台与财经媒体在发布当天集中讨论，“百万上下文”“1美元定价”“能自己操作电脑”成为传播关键词。
- 影响范围：企业自动化与流程外包（RPA）、电商运营、设计审图、工程制图、视频剪辑等行业；对OpenAI生态开发者与微软云客户影响最直接。
- 舆论倾向：开发者普遍欢迎“高性能+低价”组合；同时出现两类担忧，一是白领重复性岗位被替代，二是模型能力趋同、价格战压缩行业利润。
- 突出特点：把过去需要高价旗舰模型才能完成的视觉与屏幕任务，下放到每百万词元1美元级别，意味着AI智能体（AI Agent）的落地成本被大幅拉低，具备很强的行业改写潜力。

## 四、可创作方向

- **实测：让Terra自己操作电脑完成一天办公**
  - 角度说明：用填表、比价、整理报表、剪辑视频等真实任务展示“看得见、点得准”的能力边界
  - 适用形式：测评视频／图文实录
  - 目标受众：职场人、AI工具爱好者

- **每百万词元1美元，价格战打到什么程度了**
  - 角度说明：对比GPT-6 Astra、GPT-6.1 Sol、Claude、Mistral Large 4的定价与能力，解释毛利与算力成本
  - 适用形式：财经科普／图表长文
  - 目标受众：投资者、开发者、企业采购

- **AI智能体的“眼睛和手”是怎么练成的**
  - 角度说明：拆解视觉推理与computer use的技术路径，以及OSWorld类基准测试到底考什么
  - 适用形式：技术科普视频
  - 目标受众：技术爱好者、学生

- **哪些岗位会先被“屏幕智能体”改写**
  - 角度说明：从电商上架、表单录入、审图改图、视频粗剪四类高频任务切入，给出可落地的替代清单
  - 适用形式：行业分析长文
  - 目标受众：企业管理者、从业者

- **Astra负责想、Terra负责干：多模型编排怎么搭**
  - 角度说明：用具体工作流说明“规划模型+执行模型”的分工与成本优化方法
  - 适用形式：教程／案例拆解
  - 目标受众：开发者、产品经理

- **系统卡附录透露了哪些风险**
  - 角度说明：围绕屏幕操作带来的误触、权限、数据泄露与提示注入风险做安全科普
  - 适用形式：安全科普图文
  - 目标受众：企业安全负责人、普通用户

## 五、信息来源

- 微软Foundry官方博客：《GPT-6.1 Terra now generally available in Microsoft Foundry: build agents that see and act》（2026-10-26）https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/gpt-6-1-terra-now-generally-available-in-microsoft-foundry-build-agents-that-see/4567769
- OpenAI官方新闻页（含《Addendum to GPT-6 Astra System Card: GPT-6.1 Terra》，2026-10-26）https://openai.com/zh-Hans-CN/news/
- OpenRouter模型页：《GPT-6.1 Terra - API Pricing & Providers》（标注发布日期2026-10-26）https://openrouter.ai/openai/gpt-6.1-terra
- 微软Azure模型目录：gpt-6.1-terra https://ai.azure.com/catalog/models/gpt-6.1-terra
- 同花顺财经：《OpenAI宣布，其最新模型GPT-6.1 Terra目前正在向用户逐步推出》http://m.toutiao.com/group/7701041529124602409/
- OpenAI研究动态订阅源（含OSWorld 2.0对比数据）https://raw.githubusercontent.com/0xSMW/rss-feeds/main/feeds/feed_openai_research.xml
