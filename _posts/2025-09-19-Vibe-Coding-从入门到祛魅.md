---
layout: post
categories: coding life
---

凌晨5点又早醒了，刷到这篇推文：[Cursor 再次调价，Coding 产品的包月模式，真的搞不下去了](https://mp.weixin.qq.com/s/1Uxyarhwyi1Tyq1yNnS_Pw)，然后问chatgpt：“如何看待这篇文章的分析，对预算有限的个人开发者来说vibe coding是否注定不可持续？如何面对“由奢入俭难”的落差又不至于使生产力落后vibe coding太远？”
chatgpt认为长期确实不可持续，除非算力成本下降（更高效模型、更便宜 GPU、芯片国产化/开源化）或更多边际变现渠道（不是只靠订阅费，而是借助企业 API 收益来补贴 C 端），建议通过**开源本地模型 + 少量深思熟虑的付费 API调用 + 传统工具**来弥补生产力落差。

除了算力成本带来的不可持续性，[Steve Krouse的博文](https://blog.val.town/vibe-code)还提醒我以下几点：
- AI可能会生成大量重复的“屎山代码”（我遇到过Trae把条件1或条件2调用的同一个函数重复写了两遍）
- AI有幻觉，生成的代码可能需要花更长时间debug
- “Vibe coding is on a spectrum of how much you understand the code. The more you understand, the less you are vibing.”（而且我个人体感还有指数级倾向：不懂技术的过于宽泛的需求很容易让AI跑偏，然后沿着跑偏路上更多摸不着头脑的问题追问多次才发觉）

但把思路逆转过来，这样不失为一种好事：人类的技术能力和逻辑能力依然至关重要，不必太过担心迅速被AI取代。

简要回顾被vibe coding冲击的这两周：
从$3订阅Trae的首月优惠开始，借助它的builder（一个自动分析需求，拆解成可执行的步骤并选用工具逐个实施的AI agent）配合chat，先后：实现了gth2upf的一个新功能，修改了blog的风格（自定义首页背景图、favicon，分页和评论功能），结合[Bohrium平台课程](https://www.bohrium.com/courses/5920545182?tab=courses)和仓库（[AI4S-agent-tools](https://github.com/deepmodeling/AI4S-agent-tools)和[ABACUS-agent-tools](https://github.com/deepmodeling/ABACUS-agent-tools)）在自己机器上成功跑通了基于MCP tools的AI agents，并大致摸清了这类项目的整体架构和各个概念之间的关系，最后丢给builder一个简要的设计文档让它帮我从零写一个项目，不过到目前为止还没调试通。

不难发现，AI的核心作用在于提供高度相关的信息以打开思路，而这实际上并不需要vibe，chatgpt/copilot就足够完成以上任务。Agent相比LLM本身多出的是“问题拆解与工具选择”的智能，但我对这种智能的个人体感是，（至少在coding这一领域，到目前为止）它无法比肩人类的逻辑推理能力。

举一个昨天尝试[ABACUS-agent-tools](https://github.com/deepmodeling/ABACUS-agen-tools)遇到的实际案例：

首先这件事AI帮助最大的一点是，它很快从一大堆报错信息中定位到关键：
```
3 validation errors for Schema
enum.0: Input should be a valid string [type=string_type, input_value=1, input_type=int]
enum.1: Input should be a valid string [type=string_type, input_value=2, input_type=int]  
enum.2: Input should be a valid string [type=string_type, input_value=4, input_type=int]
```
我定睛一看，取值1|2|4，这不就是nspin吗？然而AI想当然地判断是依赖库的问题，甚至都没想到在代码里将这三个数组合搜索一下。（不排除是因为订阅模式下厂家为了降低成本，优先自动选择较小的模型）

我找到nspin那一行提醒它，它说Literal可能在有些老版本的库中不支持，建议这样改来验证问题是否出在这一行：
```python
    nspin: Literal[1, 2, 4] = 1,    #error
    nspin: int = 1,    # suggest change to debug
```
但我当即注意到，就在上方两行，同样用了Literal却没出问题：
```python
    job_type: Literal["scf", "relax", "cell-relax", "md"] = "scf",
```
于是尝试改成下面这样，问题解决:
```python
    nspin: Literal[1, 2, 4] = 1,    #error
    nspin: Literal["1", "2", "4"] = "1",    #ok
```
别人的机器上第一种写法也是没问题的，所以可能确实和依赖库版本有关，AI的判断也没错。但这件事让我更加坚定地认为，**人类的思维能力依然至关重要，且短期内不可替代**。

LLM兴起时，有人意识到“提问能力变得很重要”，而如何提问的背后正是人类如何用逻辑对知识/问题进行分析与整合，从而准确找出疑点并描述清楚的过程。

经历了这两周的体验，我已基本对vibe coding祛魅，准备好等月度订阅到期后重新回归chatgpt和copilot，继续用思考和提问的循环建立对专业领域和世界的自洽认知。
“创业出海”这种不切实际的幻想也可以放下了，不过“indie hacker”这个愿景还是值得保留的。