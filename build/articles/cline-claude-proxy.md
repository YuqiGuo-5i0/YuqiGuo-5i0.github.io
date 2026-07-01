<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
</div>

# Cline / Roo Code 接入 Claude 中转 API：VSCode 里跑 Agent

我在 VSCode 里用 Cline 和 Roo Code 折腾自动编码有小半年了，从一开始「让它帮我改个函数」到后来「丢一个需求让它自己跑一整个下午」，中间踩过的坑基本都跟一件事有关——token 怎么喂、往哪喂、喂得贵不贵。这篇就把接入 Claude 中转的完整配置、省钱姿势和常见报错捋一遍，全是我自己配出来能跑的东西，照着抄就行。

## 一、Cline / Roo Code 到底是什么

先说清楚这俩是啥，不然后面配置容易懵。

**Cline** 是 VSCode 里的一个扩展，本质是一个自动编码 Agent。你给它一句话需求，它会自己读你的项目文件、规划步骤、写代码、跑命令、看报错、再改——整个「读-想-写-验证」的循环它自己转，你在旁边点「同意」就行。**Roo Code** 是从 Cline 分叉出来的一个增强版，多了自定义模式、更细的权限控制、多角色（Architect / Code / Ask 之类）之类的东西，配置逻辑跟 Cline 几乎一模一样，所以下面的接入步骤两个通用。

它跟普通的「补全插件」最大的区别在于：它是**多轮、带上下文、会反复读文件**的。这意味着它是个不折不扣的 token 大胃王。一个稍微复杂点的任务，它可能来回读十几个文件、发起几十次请求，每次都把项目上下文重新塞进去。所以一旦你开始认真用 Agent 干活，「用哪个 API 通道」就不再是随便填个 Key 的小事，而是直接决定你**跑得顺不顺、账单疼不疼**的关键选择。

这也是为什么我最后没走官方直连、而是挂了个国内中转——高频长连接的场景对稳定性和延迟太敏感了。

## 二、接入步骤：两种配法

Cline / Roo Code 接 Claude 中转有两条路，一条走 OpenAI 兼容模式，一条走 Anthropic 原生模式。我两种都配过，下面分开讲，你挑一种就行。

打开 VSCode 侧边栏的 Cline（或 Roo Code）图标，点右上角的设置齿轮，进入 **API Configuration**。

### 方式 A：OpenAI Compatible（推荐先试这条）

在 **API Provider** 下拉里选 **OpenAI Compatible**，然后填：

- **Base URL**：`https://www.kingflow.ai/v1`
- **API Key**：你在 KingFlow 后台生成的 Key
- **Model ID**：`claude-opus-4-8`（重活）或 `claude-sonnet-4-6`（日常）

这条路的好处是通用性极强，几乎所有兼容 OpenAI 协议的客户端都能这么接，出问题也好排查。填完保存，随便发一句「列一下当前目录结构」测通就行。

### 方式 B：Anthropic 原生

如果你想让 Cline 走 Claude 的原生消息协议（对某些 Agent 特性支持更完整），在 **API Provider** 里选 **Anthropic**，然后：

- **Anthropic Base URL**：`https://www.kingflow.ai`（注意这里**不带** `/v1`，Anthropic 协议会自己拼 `/v1/messages`）
- **API Key**：同样是 KingFlow 后台的 Key
- **Model**：下拉里选或手填 `claude-opus-4-8` / `claude-sonnet-4-6`

两条路对应的鉴权其实就是环境变量那套的图形化版本：OpenAI 兼容模式对应 `OPENAI_API_KEY` + `base_url`；Anthropic 模式对应 `ANTHROPIC_AUTH_TOKEN` + `ANTHROPIC_BASE_URL`。Cline 帮你把这些封装进了设置面板，你不用手动导环境变量。

我个人的习惯：日常写代码用方式 A（省心、模型好切），需要跑长任务、想吃满 Prompt Cache 的时候用方式 B。

## 三、省钱：Agent 场景怎么把账单压下来

这一节是重点，因为 Agent 是真烧钱，配置对不对，成本能差出好几倍。

**第一件事，把 Prompt Cache 用起来。** Cline / Roo Code 这种 Agent 有个特点：每一轮请求都会把系统提示、项目文件、历史对话一整坨重新发一遍，输入 token 远远大于输出 token。如果每次都按新内容全量计费，那太亏了。Prompt Cache 的意思就是——那些重复的前缀内容缓存住，后面几轮命中缓存的部分**大幅降价**。对 Cline 这种反复读同一批文件的高频 Agent 来说,这个透传做得好不好,基本能把成本砍掉一大截（通常能省下相当可观的比例，具体看你任务的重复度）。走 KingFlow 这类完整透传 `cache_control` 的通道，你带缓存连发两次，第二次的 `cache_read` 用量就非零了，说明缓存真吃上了。

**第二件事，模型分级用，别一把梭 opus。** 我的分工是这样：

- **claude-opus-4-8**：大重构、跨文件的复杂逻辑、需要它「想清楚再动手」的难任务。贵但是稳，关键时刻不含糊。
- **claude-sonnet-4-6**：日常写功能、改 bug、写测试，绝大多数活它都够用，性价比在线。
- **claude-haiku-4-5**：格式整理、简单问答、批量小改这种高频低难度的活，用它成本压得很低。

Cline / Roo Code 支持保存多套 API Configuration，我就存了三套，任务开始前扫一眼难度切一下模型。一个 KingFlow 的 Key 就能路由这些模型，改 Model ID 就切，不用维护好几套 Key。

**第三件事**，Cline 里把「自动读整个 workspace」的激进设置收一收，只让它读相关目录，能少喂不少无用上下文。

## 四、常见配置报错排查

按踩坑频率排个序，基本覆盖 90% 的接不通：

- **401 / Invalid API Key**：九成是 Key 复制时带了空格，或者用错了模式的字段（把 OpenAI 的 Key 填进了 Anthropic 那栏）。重新从后台复制，注意别选中前后空白。
- **404 Not Found**：Base URL 拼错了。OpenAI 兼容模式必须带 `/v1`，Anthropic 模式**不能**带 `/v1`——这俩最容易搞反，搞反了就 404。
- **模型不存在 / model not found**：Model ID 拼写问题，确认是 `claude-opus-4-8` 这种当前在售的名字，别用老早以前那种带长串日期后缀的旧模型名，那些早就下线了。
- **请求超时 / 一直转圈**：如果你之前挂着机场代理，高频长连接容易被识别掉线，反而不稳。走国内直连的中转通常不用再叠代理，TTFT 一般一两秒就有响应；真超时了先把系统代理关掉再试。
- **429 / 限流**：短时间并发打太猛。Agent 任务本身请求密集，稍微等一下或降低并发即可，后台也能查到用量明细帮你定位。
- **返回内容明显变蠢**：怀疑被掉包降级了（拿小模型冒充大模型）。选正规、模型保真的通道，后台能查调用明细的那种，遇到这种情况能对得上账。

## 五、为什么用 KingFlow

不吹，就说我实际在意的几点：

- **走官方 `/v1/messages` 协议**，不是逆向反代某某编辑器，Anthropic 那边更新了也不容易突然挂掉。
- **Prompt Cache 完整透传**，这个对 Cline / Roo Code 这种输入远大于输出的 Agent 是刚需，直接关系到账单。
- **国内直连**，节点在国内，首字延迟通常在一两秒级别，不用自己折腾代理，也就不会因为代理被识别而掉线。
- **一个 Key 多模型**，opus / sonnet / haiku 改个 Model ID 就切，还能顺手路由 gpt-5.5、deepseek-v4、glm-5.1 这些，想横向对比很方便。
- **用量透明**，后台能查日志、余额、token 明细，对账不抓瞎；人民币小额充值，新人注册有额度可以先测后充。

## 六、FAQ

**Q1：Cline 和 Roo Code 的配置能通用吗？**
能。两者 API 设置面板长得几乎一样，Base URL、Key、Model 这套填法完全一致，上面的步骤两个都适用。区别在 Roo Code 多了些模式和权限选项，不影响接入。

**Q2：方式 A 和方式 B 到底选哪个？**
没有绝对答案。图省事、想快速切换模型选 A（OpenAI Compatible）；想要 Claude 原生协议的完整特性、跑长任务吃满缓存选 B（Anthropic）。我自己两套都存着换着用。

**Q3：为什么我一跑复杂任务 token 就哗哗掉？**
这是 Agent 的正常特性——它会反复读文件、多轮迭代，输入量天然大。压成本靠三招：吃满 Prompt Cache、按难度分级选模型、收紧上下文读取范围。别用 opus 去干 haiku 能干的活。

**Q4：之前挂着代理接总是超时，怎么办？**
高频长连接的 Agent 场景，机场/VPS 代理很容易被识别然后频繁掉线。走国内直连的中转就不用再叠一层代理，把系统代理关掉直连试试，通常延迟和稳定性都会明显好转。
