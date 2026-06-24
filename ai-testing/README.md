# AI + 测试面试题

---

## Q1. AI 生成的代码每次都不一致，怎么约束它？

### 问题描述

用 Cursor AI / ChatGPT / Copilot 写测试代码，每次生成的代码风格不同、命名不一致、框架选型随缘。怎么让 AI 产出稳定、统一的代码？

### 答案

你说的"约定"在 AI 编程里叫 **Rules / Instructions / Skills**。不同工具有不同的叫法：

| 工具 | 功能名 | 作用 |
|:---:|:---|:---|
| Cursor | `.cursorrules` | 项目级规则文件，AI 自动读取 |
| GitHub Copilot | `.github/copilot-instructions.md` | 指令文件 |
| Windsurf | `.windsurfrules` | 规则文件 |
| 通用 | `rules/` 目录 + `SKILL.md` | 结构化技能定义 |

---

### 一、最简方案：项目根目录放规则文件

在项目根目录创建 `.cursorrules`（Cursor）或 `rules/` 目录，AI 工具会自动读取并遵守。

**示例：接口自动化项目的规则文件**

```markdown
# 项目规则

## 框架约定
- 测试框架：Pytest
- 接口请求：requests 库
- 数据管理：YAML 文件（不硬编码）
- 报告生成：Allure

## 命名规范
- 测试文件：test_xxx.py
- 测试类：TestXxx
- 测试方法：test_xxx
- 关键字方法：_xxx（下划线开头）
- fixture 命名：小写 + 下划线

## 代码风格
- 断言用 assert，不用 self.assertEqual
- fixture scope 优先用 session（复用 token）
- 敏感信息从 config.yaml 读取，不硬编码
- 接口 URL 用 base_url + 路径拼接

## 日志规范
- 每个关键字自动打日志（通过代理/装饰器，不在方法内手动加）
- 关键操作记录入参和耗时

## 不推荐的做法
- 不用 unittest 风格
- 不用 time.sleep（用显式等待）
- 测试数据不写在 Python 文件里
```

AI 看到这个文件后，生成的代码会自动遵守这些规则。

---

### 二、进阶方案：结构化 Skills 定义

如果你用了支持 Skills 的工具（如 WorkBuddy），可以把规则写成结构化的 Skill 文件：

```markdown
# SKILL.md
---
name: api-automation-rules
description: 接口自动化框架的编码规范和约定
version: 1.0.0
rules:
  - id: FRAME-001
    description: 使用 Pytest 作为测试框架
    severity: error
  - id: FRAME-002
    description: fixture 默认使用 session 级别（复用登录 token）
    severity: warning
  - id: NAMING-001
    description: 关键字方法用 _xxx 命名
    severity: error
  - id: DATA-001
    description: 测试数据从 YAML 读取，不在代码里硬编码
    severity: error
  - id: LOG-001
    description: 关键字调用日志通过代理统一处理，不在每个方法内写 print
    severity: warning
  - id: ASSERT-001
    description: 断言使用原生 assert
    severity: error
---

## 代码生成规则

### 测试文件结构
```python
import pytest
import requests
import yaml

class TestXxx:
    @pytest.mark.parametrize("case", load_data("xxx.yaml"))
    def test_xxx(self, case):
        resp = api_client.post("/path", json=case["body"])
        assert case["expect"] in resp.text
```

### 例外情况
- 调试阶段可以临时放宽 rules，但提交代码前必须恢复
```

这样 AI 工具可以基于规则校验生成的代码，不符合规则的会报 warning/error。

---

### 三、实战：你的项目中可以用的约束

以你的接口自动化框架为例，建议定这几条规则：

#### 规则 1：YAML 数据驱动优先

```yaml
# ❌ AI 可能会生成的（不约束时）
def test_login():
    resp = requests.post("/auth/login", json={
        "username": "admin",
        "password": "123456",
        "role": "admin",
        "dept": "sale"
    })
    assert resp.status_code == 200

# ✅ 加了规则后 AI 会生成的
# test_data.yaml
login_cases:
  - case: "管理员登录"
    body: {username: "admin", password: "123456", role: "admin", dept: "sale"}
    expect: 200

@pytest.mark.parametrize("case", load_yaml("test_data.yaml")["login_cases"])
def test_login(case):
    resp = api_client.post("/auth/login", json=case["body"])
    assert resp.status_code == case["expect"]
```

#### 规则 2：URL 不硬编码

```python
# ❌ 不约束时 AI 可能生成
resp = requests.post("https://test-api.example.com/order/create")

# ✅ 加了规则后
resp = api_client.post("/order/create")  # base_url 从 config 读取
```

#### 规则 3：断言不用 unittest 风格

```python
# ❌ 不约束时 AI 可能生成
self.assertEqual(resp.status_code, 200)
self.assertTrue("成功" in resp.text)

# ✅ 加了规则后
assert resp.status_code == 200
assert "成功" in resp.text
```

---

### 四、多轮对话中约束 AI 的口头约定

除了文件规则，对话中也建议你约定：

**首次对话时给 AI 说清楚**：

> "这个项目用的是 Pytest 框架，数据驱动，YAML 管理测试数据。所有断言用 assert，URL 从配置文件读取不硬编码。关键字方法用下划线开头。你按这个风格写。"

**如果 AI 跑偏了及时纠正**：

> "不要用 unittest 风格，改用 assert。"
> "把用户名密码写到 YAML 里，不要在代码里写死。"
> "用 fixture 做前置登录，不用在每个用例里调 login。"

AI 在对话中会持续学习你的偏好，同一轮对话中后续生成会保持一致。

---

### 五、面试回答模板

> **"AI 生成的代码不一致是常见问题，我的做法是两层约束。第一层是项目级规则文件——在项目根目录放 .cursorrules 文件，写明框架约定、命名规范、代码风格，AI 会主动遵守。第二层是对话中的口头约定——每次跟 AI 协作时先说清楚项目的框架和风格偏好，万一生错了及时纠正，AI 会在同一轮对话中保持一致。**
>
> **比如我最近做接口自动化，用 Cursor 写测试用例。我在 .cursorrules 里写了：Pytest 框架、数据从 YAML 读取不硬编码、断言用 assert、关键字方法用 _ 开头。AI 生成的代码基本不用改就能直接用，风格也统一。"**

---

**关键词**：AI 编程规范 / .cursorrules / Skills / 结构化规则 / AI 代码约束 / 提示词工程

---

## Q2. 生成测试用例时约束什么？生成测试代码时约束什么？

### 问题描述

同样是让 AI 帮忙写测试，生成测试用例和生成测试代码是两个不同的场景，约束点也不一样。具体来说：生成用例时重点管什么？生成代码时重点管什么？

### 答案

你说得很对，这两件事的约束点完全不同：

| 维度 | 生成测试用例（设计） | 生成测试代码（实现） |
|:---:|:---|:---|
| **关注什么** | 业务场景覆盖全不全 | 代码框架对不对 |
| **输出产物** | 测试要点 / 用例表格 | Python 文件 / YAML 数据 |
| **约束重点** | 场景完整性、边界值、数据流向 | 框架约定、命名规范、代码风格 |
| **谁看得懂** | 测试 + 开发 + 产品 | 只有开发测试看得懂 |

---

### 一、生成测试用例时的约束

目标是让 AI 把**业务场景想全**，不要漏 Case。

#### 约束 1：场景维度

```markdown
## 用例生成规则
针对每个功能，必须覆盖以下维度：
1. 正向场景（正常流程，至少 1 条）
2. 异常场景（输入异常、状态异常、权限异常）
3. 边界值（最小值、最大值、临界值）
4. 并发场景（多人同时操作同一数据）
5. 数据一致性（操作后各模块数据是否同步）
6. 幂等（重复操作不产生副作用）
```

**示例**——给 AI 一个需求，让它生成用例：

```
需求：销售改价功能

约束规则：
1. 正向场景：正常改价（调高/调低）
2. 异常场景：改价为负数、改价时商品正在出库
3. 边界值：改价 0 元、改价 999999.99
4. 并发场景：50 人同时给同一商品改价
5. 数据一致性：改价后库存成本是否同步更新、历史订单毛利是否重算
6. 幂等：重复提交改价请求，价格不变

请按以上维度生成测试用例，输出表格格式。
```

AI 输出：

| 用例编号 | 场景 | 前置条件 | 操作步骤 | 预期结果 |
|:---:|:---|:---|:---|:---|
| TC001 | 正向-单商品调高价格 | 商品 SKU001 当前价 100 | 改价为 150 | 改价成功，成本同步更新 |
| TC002 | 正向-单商品调低价格 | 商品 SKU001 当前价 100 | 改价为 80 | 改价成功，成本同步更新 |
| TC003 | 异常-价格为负数 | 商品 SKU001 | 改价为 -10 | 系统拦截，提示价格非法 |
| TC004 | 边界-价格为 0 | 商品 SKU001 | 改价为 0 | 系统拦截或允许（按业务规则） |
| TC005 | 并发-多人改价 | SKU001 正在出库 | 同时 50 人改价 | 最终价格正确，库存不丢失 |
| TC006 | 一致性-成本同步 | 改价后 | 查询库存成本 | 成本已同步，金额正确 |
| TC007 | 幂等-重复提交 | 已改价为 150 | 再次提交改价 150 | 返回成功，价格不变 |

#### 约束 2：业务场景描述格式

```markdown
## 用例描述格式
每条测试用例必须包含：
- 用例编号（TCxxx）
- 场景描述
- 前置条件（数据/状态/环境）
- 操作步骤
- 预期结果
- 涉及模块（多个模块联动时需要标注）
```

#### 约束 3：异常场景必须包含线上历史问题

```markdown
## 异常场景补充
在生成异常场景时，参考以下历史问题类型：
- 数据不同步（A模块改了B模块没同步）
- 并发冲突（多人操作同一单据）
- 超时/重试（接口超时后状态不一致）
- 幂等缺失（重复请求导致重复数据）
```

---

### 二、生成测试代码时的约束

目标是让 AI 生成的代码**直接能跑、风格统一、不踩坑**。

#### 约束 1：框架和工具链

```markdown
## 框架与工具
- 测试框架：Pytest（不用 unittest）
- 接口请求：requests 库
- 数据管理：YAML 文件（不硬编码在 py 文件里）
- 断言：assert 原生语法（不用 self.assertEqual）
- 参数化：@pytest.mark.parametrize
- 报告：Allure
```

**不加约束时 AI 可能写成这样**（常见的坑）：

```python
# ❌ AI 不约束时爱写的风格
import unittest
import requests

class TestLogin(unittest.TestCase):
    def setUp(self):
        self.url = "https://test-api.example.com/auth/login"
        self.data = {"username": "admin", "password": "123456"}

    def test_login_success(self):
        resp = requests.post(self.url, json=self.data)
        self.assertEqual(resp.status_code, 200)
```

**加了约束后**：

```python
# ✅ 加了约束后的风格
import pytest
import requests
import yaml

@pytest.mark.parametrize("case", yaml.safe_load(open("data/login.yaml"))["cases"])
def test_login(case, api_client):
    resp = api_client.post("/auth/login", json=case["body"])
    assert case["expect"]["status"] == resp.status_code
    assert case["expect"]["msg"] in resp.text
```

#### 约束 2：数据管理

```markdown
## 数据管理规则
- 测试数据统一放在 data/ 目录下的 YAML 文件
- YAML 文件结构：每个用例一个 entry，包含 body 和 expect
- 敏感信息（密码/token）从 config.yaml 或 .env 读取
- URL 路径不写完整 URL，用 base_url + 路径拼接
```

```yaml
# data/login.yaml —— 约束后 AI 会这样生成
cases:
  - case: "管理员登录"
    body:
      username: "admin"
      password: "123456"
    expect:
      status: 200
      msg: "登录成功"

  - case: "密码错误"
    body:
      username: "admin"
      password: "wrong"
    expect:
      status: 401
      msg: "密码错误"
```

#### 约束 3：fixture 使用规范

```markdown
## fixture 规则
- 通用 fixture（登录/初始化）写在 conftest.py 里
- scope 优先用 session（复用 token，减少登录次数）
- 模块级的前置条件在对应模块的 conftest.py 里定义
- 不在每个测试文件里重复写登录逻辑
```

#### 约束 4：代码风格

```markdown
## 代码规范
- 函数/变量命名：小写 + 下划线
- 测试类命名：Test + 大驼峰
- 关键字方法命名：_ + 小写 + 下划线
- 每个测试方法不超过 30 行
- 不用的 import 不要留
- 方法加 docstring 说明测试场景
```

#### 约束 5：断言粒度

```markdown
## 断言规则
- 状态码断言：assert resp.status_code == 200
- 业务断言：assert "预期内容" in resp.text
- 数据一致性断言：两个接口返回的数据做 cross-check
- 不使用过于宽泛的断言（如 assert True）
```

```python
# ❌ 太宽泛
assert resp.status_code != 500

# ✅ 精确
assert resp.status_code == 200
assert resp.json()["order_status"] == "PAID"
assert resp.json()["amount"] == 100.00
```

---

### 三、一句话版速查

```
生成测试用例时约束："场景要全、维度要够、异常要覆盖历史问题"
生成测试代码时约束："框架统一、数据分离、不硬编码、风格一致"
```

---

### 四、面试回答模板

> **"生成测试用例和生成测试代码是两个不同的场景，约束点不同。**
>
> **生成用例时，我主要约束场景维度——要求 AI 必须覆盖正向、异常、边界值、并发、数据一致性、幂等这六个维度，并且每条用例必须有前置条件和预期结果。这样生成的用例才全面，不会漏场景。**
>
> **生成代码时，我主要约束框架和风格——Pytest 框架、YAML 管理数据、assert 断言、session 级 fixture 复用 token。同时确保 URL 和密码不硬编码，从配置文件中读取。这样生成的代码直接能跑，不用二次返工。"**

---

**关键词**：测试用例生成 / 测试代码生成 / AI 约束 / 场景维度 / 代码规范
---

## Q3. 用过哪些大模型？各自的优缺点和使用体验如何？

### 问题描述

简历上写了"AI辅助测试"，面试官大概率会追问：你用过哪些大模型？体验下来各有什么优缺点？选模型的依据是什么？

> 这道题考察的是**你是否有真实的AI使用深度**，而不是"听过名字但没用过"。回答要有对比、有场景、有实际体会。

---

### 一、我常用的模型一览

| 模型/工具 | 使用场景 | 使用频率 |
|:---|:---|:---:|
| **ChatGPT（GPT-4o）** | 日常主力：写用例、分析需求、代码审查 | ⭐⭐⭐⭐⭐ |
| **DeepSeek** | 写测试代码、复杂逻辑推理、代码debug | ⭐⭐⭐⭐ |
| **Cursor（Composer）** | 项目级代码开发、批量生成自动化用例 | ⭐⭐⭐⭐ |
| **GitHub Copilot** | 代码行级补全、编写测试方法 | ⭐⭐⭐ |
| **通义千问** | 国内替代方案、中文需求理解 | ⭐⭐ |

---

### 二、各模型详细体验

#### 1. ChatGPT（GPT-4o）—— 综合能力最强

**使用场景**：
- 分析需求文档，生成测试要点和场景
- 代码审查：让AI扫描代码，找出潜在Bug和风格问题
- 写测试方案、测试报告

**优点**：
- **需求理解能力最强**：给它一段产品需求，能准确提取测试点，生成用例覆盖度很高
- **上下文窗口大**（128K）：可以一次扔进整个模块的代码+文档，做全量分析
- **代码质量稳定**：生成的Pytest代码风格统一，很少出语法错误
- **多轮对话表现好**：不会忘记前面的约定，持续对话中越来越懂项目风格

**缺点**：
- **需要梯子**：国内访问不方便，团队协作时是个障碍
- **付费**：Plus订阅 $20/月，API调用按token计费
- **中文有时会跑偏**：处理纯中文的业务需求时，偶尔对ERP领域术语理解不够精准

**典型用法**：
```
我在做ERP系统的测试，需求是"销售改价功能，改价后要同步更新库存成本，
历史订单的毛利也要重算"。
请帮我从正向、异常、边界、并发、数据一致性、幂等6个维度生成测试用例。
```

---

#### 2. DeepSeek —— 性价比最高，推理能力强

**使用场景**：
- 写接口自动化测试代码（Python + Pytest + Requests）
- 复杂逻辑的代码Debug：扔给DeepSeek一段报错日志，它能快速定位根因
- 代码生成和重构

**优点**：
- **推理能力强**：逻辑题、复杂场景的代码生成质量很高，不输GPT-4o
- **完全免费**：目前没有付费墙，API也很便宜
- **国内直连**：不需要梯子，访问稳定
- **中文理解好**：对中文业务需求的理解比ChatGPT更自然
- **代码质量高**：生成的Python测试代码结构清晰，很少需要二次修改

**缺点**：
- **服务偶尔不稳定**：高峰期可能排队或响应慢
- **上下文不如GPT-4o长**：扔大段代码时需要注意分段
- **生态不如ChatGPT完善**：插件、工具链相对少
- **对话记忆略弱**：长对话中偶尔忘记前面的约定

**典型用法**：
```
这段代码报错 "KeyError: 'token'"，帮我分析一下原因。
[贴代码和报错日志]
```

---

#### 3. Cursor（Composer）—— 项目级开发效率最高

**使用场景**：
- 在项目中批量生成自动化测试用例
- 通过 .cursorrules 约束代码风格，保证一致性
- 多文件联动的代码修改

**优点**：
- **项目级上下文理解**：能读取整个项目结构，理解目录、模块之间的关系
- **.cursorrules 约束**：可以定义项目规则文件，AI自动遵守，生成的代码风格统一
- **Composer模式**：一次对话可以生成/修改多个文件，适合批量产出测试用例
- **IDE内集成**：不需要切窗口，写代码效率极高

**缺点**：
- **付费**：Pro版 $20/月
- **学习成本**：.cursorrules 写得好效果才好，规则写得不好反而干扰
- **偶尔"想太多"**：修改代码时可能会动到不相关的部分
- **需求分析不如ChatGPT**：更偏代码生成，纯文本需求分析不如ChatGPT

**我的 .cursorrules 示例**：
```markdown
# 测试框架约定
- 测试框架：Pytest
- 接口请求：requests 库
- 数据管理：YAML 文件，不硬编码
- 断言：assert 原生语法
- fixture scope 优先用 session

# 命名规范
- 测试文件：test_xxx.py
- 测试类：TestXxx
- 关键字方法：_xxx（下划线开头）
```

**效率提升**：用 Cursor + .cursorrules 后，生成100条接口自动化用例的时间从3天缩短到半天。

---

#### 4. GitHub Copilot —— 行级补全最快

**使用场景**：
- 写测试方法时，Copilot自动补全断言和请求代码
- 写数据驱动代码时，自动补全YAML读取和参数化

**优点**：
- **行级补全极快**：写几个字就能猜出后面的代码，手速提升明显
- **IDE无缝集成**：VS Code / JetBrains 原生支持
- **学习你的代码风格**：项目中写得越多，补全越准

**缺点**：
- **只能补全，不能"对话"**：不像ChatGPT可以讨论需求，只能生成代码片段
- **上下文有限**：只能看到当前文件和少量相邻文件
- **付费**：$10/月（个人版）
- **偶尔补全很离谱**：比如你写 `# 测试登录`，它可能生成一整套不相关的代码

**实际体会**：Copilot 适合"已有明确思路，快速敲代码"的场景。比如你脑子里已经知道要写什么断言，它帮你敲出来。但如果是"不知道测试方案该怎么设计"，Copilot 帮不上忙。

---

#### 5. 通义千问 —— 国内备选

**使用场景**：
- 中文需求文档分析（通义对中文的理解最自然）
- 国内环境下的替代方案
- 敏感业务场景（数据不出境）

**优点**：
- **中文理解最好**：阿里系，对国内业务术语的理解最准确
- **完全免费**：基础功能免费
- **国内合规**：数据不出境，适合敏感业务

**缺点**：
- **代码能力偏弱**：生成的测试代码质量不如 DeepSeek / ChatGPT
- **推理能力一般**：复杂逻辑场景的代码生成容易出错
- **上下文较短**：不适合大段代码分析

---

### 三、我的选择策略

| 场景 | 首选模型 | 原因 |
|:---|:---|:---|
| 需求分析 + 生成测试方案 | ChatGPT（GPT-4o） | 需求理解最强，场景覆盖全面 |
| 写接口自动化代码 | DeepSeek 或 Cursor | 代码质量高，免费/项目级集成 |
| 日常代码补全 | Copilot | 行级补全最快，IDE内无感 |
| 批量生成用例（项目级） | Cursor Composer | 多文件联动，.cursorrules约束 |
| Debug + 定位Bug | DeepSeek | 推理能力强，免费 |
| 中文文档处理 | 通义千问 | 中文理解最好 |
| 代码审查 | ChatGPT | 上下文窗口大，一次分析整个模块 |

---

### 四、面试回答模板（3分钟版）

> **"我日常用AI辅助测试，接触过多个模型，各有侧重。**
>
> **主力是ChatGPT的GPT-4o，我主要用它做需求分析和生成测试方案。给它一段产品需求，它能从正向、异常、边界、并发、数据一致性、幂等六个维度帮我梳理测试点，生成的用例覆盖度很高。代码审查我也用ChatGPT，因为它上下文窗口大，可以一次扔进整个模块的代码做全量分析。**
>
> **写代码我用DeepSeek比较多，它的推理能力很强，生成的Pytest代码质量高，而且完全免费。还有一个好处是国内直连，不需要梯子。Cursor我配了 .cursorrules，定义了Pytest框架、YAML数据驱动、assert断言这些规范，批量生成用例时风格统一，效率提升了50%以上。**
>
> **日常写代码Copilot做行级补全，写几个字就自动补完断言和请求代码。国内场景我也用过通义千问，中文理解最好，但代码能力不如DeepSeek。**
>
> **总的来说，我的策略是：需求分析找ChatGPT，写代码找DeepSeek，批量生成用Cursor，日常补全靠Copilot。每个模型用在它最擅长的场景里，效率最大化。"**

---

### 五、面试官可能追问的点

**追问1**："你说效率提升50%，具体怎么算出来的？"

> "我们项目大概有100条核心接口自动化用例。之前纯手写，从分析接口文档到写完调试通过，差不多要3天。后来用Cursor + .cursorrules，AI生成基础代码框架，我只需要调整业务断言和测试数据，半天就搞定了。算下来效率提升了50%以上。"

**追问2**："AI生成的代码你敢直接上生产环境吗？"

> "不会直接上。AI生成的代码我会做两件事：一是人工审查，重点看断言逻辑和边界值覆盖；二是在测试环境先跑一遍，确认通过后再合并。AI是辅助，质量把关还得人来做。"

**追问3**："你团队其他人用AI吗？你怎么推广的？"

> "我之前带过一个小团队做自动化，我把 .cursorrules 和常用的提示词模板沉淀下来分享给组员。刚开始大家不太习惯，后来发现确实快很多，慢慢就都用起来了。关键是要有一套统一的规范，不然每个人用AI生成的风格不一样，合并代码时很痛苦。"

---

**关键词**：ChatGPT / DeepSeek / Cursor / Copilot / 通义千问 / AI辅助测试 / 模型对比 / 效率提升 / .cursorrules

