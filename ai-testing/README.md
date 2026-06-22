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