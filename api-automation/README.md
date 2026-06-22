# 接口自动化 / 测试框架设计面试题

---

## Q1. 关键字驱动和数据驱动的区别是什么？

### 问题描述

面试官问你关键字驱动和数据驱动有什么区别，你用什么框架、为什么这么选？这个问题考的是你对**自动化框架设计**的理解深度。

### 答案

先说一句话总结，再展开讲区别：

> **数据驱动是"同一套操作逻辑，换不同的数据去跑"；关键字驱动是"把操作步骤也抽象出来，用配置文件控制测什么、怎么测"。**

---

### 一、数据驱动（Data-Driven Testing）

**核心思想**：把**测试数据**从代码中剥离出来，放到外部文件（JSON / YAML / Excel / CSV）里管理。代码只负责执行逻辑，数据变了用例就变了。

**你简历里的实际应用**：

> 简历中写的：*"YAML 管理测试数据，数据与代码完全解耦"*

```python
# data/test_data.yaml
login_cases:
  - case: "正确账号密码登录"
    username: "admin"
    password: "123456"
    expect: "登录成功"

  - case: "密码错误"
    username: "admin"
    password: "wrong"
    expect: "密码错误，请重试"

  - case: "账号不存在"
    username: "not_exist"
    password: "123456"
    expect: "账号不存在"

search_cases:
  - case: "搜索存在的商品"
    keyword: "T恤"
    expect_count: 10

  - case: "搜索不存在的商品"
    keyword: "xxxxxxxxxx"
    expect_count: 0
```

```python
# 测试代码 - 只负责执行逻辑
import pytest
import yaml

class TestLogin:
    @pytest.mark.parametrize("case_data",
        yaml.safe_load(open("data/test_data.yaml"))["login_cases"])
    def test_login(self, case_data):
        resp = api_client.post("/auth/login", json={
            "username": case_data["username"],
            "password": case_data["password"]
        })
        assert case_data["expect"] in resp.text
```

**数据驱动的优点**：

| 优点 | 说明 |
|:---|:---|
| **数据与代码分离** | 改数据不用改代码，测试人员也能维护 |
| **覆盖率高** | 加一条数据就是加一条用例，轻松覆盖各种入参组合 |
| **可读性强** | YAML / Excel 格式直观，业务方也能看懂 |

**数据驱动的缺点**：

| 缺点 | 说明 |
|:---|:---|
| **逻辑固定** | 代码写死了"先登录→再查询"的步骤顺序，换场景还得改代码 |
| **不适合复杂链路** | 如果测试场景涉及"先A操作→再B操作→条件满足时做C→否则做D"，数据驱动就管不住了 |

---

### 二、关键字驱动（Keyword-Driven Testing）

**核心思想**：把**操作步骤**也抽象成关键字，放在配置文件里。代码变成"关键字执行引擎"，不关心业务逻辑，只负责读关键字、按顺序执行。

**你简历里的实际应用**：

> 简历中写的：*"搭建基于关键字驱动的接口自动化框架（Python + Requests）"*

```yaml
# keyword/test_case.yaml
test_sale_flow:  # 销售改价全流程
  - keyword: login
    params:
      username: "admin"
      password: "123456"

  - keyword: query_stock
    params:
      sku: "SKU001"

  - keyword: assert_equal
    params:
      actual: "{response.data.quantity}"
      expected: 100

  - keyword: update_price
    params:
      sku: "SKU001"
      new_price: 99.00

  - keyword: assert_equal
    params:
      actual: "{response.data.amount}"
      expected: 99.00

  - keyword: check_finance_sync  # 校验财务模块是否同步
    params:
      sku: "SKU001"
      expect_price: 99.00
```

```python
# 关键字执行引擎
class KeywordEngine:
    """关键字驱动引擎——不关心业务，只负责执行关键字"""

    def __init__(self):
        self.context = {}  # 上下文，存储中间结果
        self.keywords = {
            "login": self._login,
            "query_stock": self._query_stock,
            "update_price": self._update_price,
            "assert_equal": self._assert_equal,
            "check_finance_sync": self._check_finance_sync,
        }

    def run(self, test_steps):
        for step in test_steps:
            keyword = step["keyword"]
            params = step.get("params", {})
            # 支持参数引用：{response.data.xxx} 引用上一步的结果
            resolved_params = self._resolve_params(params)
            result = self.keywords[keyword](**resolved_params)
            self.context["response"] = result
            self.context[f"{keyword}_result"] = result

    def _login(self, username, password):
        return requests.post("/auth/login", json={"username": username, "password": password})

    def _update_price(self, sku, new_price):
        return requests.post("/sale/price/update", json={"sku": sku, "price": new_price})

    def _assert_equal(self, actual, expected):
        assert actual == expected, f"断言失败: {actual} != {expected}"


# 使用
engine = KeywordEngine()
engine.run(yaml.safe_load(open("keyword/test_case.yaml"))["test_sale_flow"])
```

**关键字驱动的优点**：

| 优点 | 说明 |
|:---|:---|
| **业务人员也能写用例** | 只需要写 YAML，不用写代码 |
| **用例即文档** | YAML 里每一步写清楚了做什么，一眼看懂测试场景 |
| **高度可复用** | 写一个 `login` 关键字，所有用例都能调 |
| **灵活组合** | 加新场景不用写代码，组合已有关键字就行 |

**关键字驱动的缺点**：

| 缺点 | 说明 |
|:---|:---|
| **前期投入大** | 需要先搭关键字引擎，开发成本高 |
| **调试困难** | 出错了要追：是关键字引擎的 Bug 还是关键字实现的问题？ |
| **复杂逻辑受限** | 条件分支、循环嵌套在 YAML 里表达很别扭 |

---

### 三、一张图看懂区别

```
数据驱动:
┌──────────────────────────────────────────────┐
│  代码: login(username, password)              │
│                                              │
│  数据:                                        │
│  ┌──────────┬──────────┬──────────┐           │
│  │ admin    │ 123456   │ 登录成功 │           │
│  │ admin    │ wrong    │ 密码错误 │           │
│  │ not_exist│ 123456   │ 不存在   │           │
│  └──────────┴──────────┴──────────┘           │
└──────────────────────────────────────────────┘
  同一段代码，换不同的数据跑多次

关键字驱动:
┌──────────────────────────────────────────────┐
│  引擎: 读关键字 → 找实现 → 执行 → 存结果      │
│                                              │
│  关键字（YAML）:                               │
│  1. login                                    │
│  2. query_stock                              │
│  3. assert_equal                             │
│  4. update_price                             │
│  5. check_finance_sync                       │
│                                              │
│  每个关键字背后都对应一段 Python 实现           │
└──────────────────────────────────────────────┘
  不写代码，用"关键字组合"来描述测试场景
```

---

### 四、实战中怎么选

**选数据驱动**（互联网公司常用，你简历里也写了）：

```
场景：单接口测试、参数组合覆盖
  如：登录接口测 10 组账号密码
  如：查询接口分页测 5 种 page_size

理由：简单直接，pytest + parametrize 就搞定了
```

**选关键字驱动**（大型项目/复杂业务链路）：

```
场景：多步骤、多模块的业务链路
  如：登录 → 查库存 → 改价 → 校验财务同步 → 出库 → 校验凭证

理由：测试步骤可配置，业务同学也能维护
```

**实际项目中经常混用**：

```
你简历里的框架大概率是"关键字驱动 + 数据驱动 混合"：

框架结构:
├── keywords/           ← 关键字驱动层（每个关键字一个文件）
│   ├── login.py
│   ├── query_stock.py
│   └── update_price.py
├── testcases/          ← 数据驱动层（YAML 管理测试数据）
│   ├── login_success.yaml
│   └── login_fail.yaml
├── engine.py           ← 关键字执行引擎
└── conftest.py         ← pytest 配置
```

---

### 五、面试回答模板

> **"数据驱动和关键字驱动的核心区别是抽象层级不同。数据驱动是把数据从代码中抽出来，同一段逻辑换不同数据跑，适合单接口的参数组合覆盖。关键字驱动是把操作步骤也抽出来，用配置描述测试场景，适合复杂的业务链路。**
>
> **我之前搭建的接口自动化框架是两者混用的。接口层用数据驱动——登陆、查询这些单接口用 YAML 管理多组入参和期望结果，pytest 的 parametrize 直接驱动。业务链路层用关键字驱动——把登录、查库存、改价、校验财务同步每个操作封装成关键字，用 YAML 定义测试步骤。这样单接口覆盖用数据驱动、业务链路用关键字驱动，各自发挥优势。**
>
> **关键字的数量大概是 15-20 个核心关键字，覆盖了进销存链路上 90% 的操作。新增一个业务场景只需要组合已有关键字写 YAML，不需要写代码。"**

---

**关键词**：关键字驱动 / 数据驱动 / 测试框架设计 / YAML 数据管理 / 关键字引擎 / pytest parametrize

---

## Q2. Pytest 框架的特点是什么？

### 问题描述

面试高频题：你用的 Pytest 有哪些特点？和 unittest 比有什么区别？考的是你对常用测试框架的理解深度。

### 答案

Pytest 是 Python 生态中最主流的测试框架，没有之一。它的核心特点一句话总结：

> **简单到不用学就会用，强大到能满足任何复杂测试场景。**

---

### 一、Pytest 的 7 大核心特点

#### 特点 1：自动发现测试用例

Pytest 不需要像 unittest 那样手动声明 TestSuite，它自动按规则找用例：

```
规则：
  - 文件名以 test_ 开头或 _test 结尾 → 识别为测试文件
  - 类名以 Test 开头 → 识别为测试类
  - 函数/方法名以 test_ 开头 → 识别为测试用例
```

```bash
# 直接跑，什么都不用配
pytest tests/

# 按关键字筛选
pytest tests/ -k "login"
pytest tests/ -k "login or order"
pytest tests/ -k "not slow"
```

#### 特点 2：assert 断言简洁自然

```python
# unittest 风格——要记一堆
self.assertEqual(a, 10)
self.assertTrue(a > 5)
self.assertIn("成功", text)

# pytest 风格——就是原生 assert
assert a == 10
assert a > 5
assert "成功" in text
with pytest.raises(ValueError):
    func()
```

报错信息也比 unittest 详细：`assert 5 == 10` 直接告诉你两个值各是多少。

#### 特点 3：fixture 管理前后置

fixture 替代了 unittest 的 `setUp` / `tearDown`，支持 5 种 scope：

```python
@pytest.fixture(scope="function")   # 每个用例执行前后
@pytest.fixture(scope="module")     # 每个模块执行前后
@pytest.fixture(scope="session")    # 整个会话只执行一次

def api_client():
    client = APIClient()
    client.login("admin", "123456")
    yield client                     # 交给测试用例
    client.close()                   # 用完清理
```

fixture 可以嵌套、通过 `conftest.py` 跨文件共享、`autouse=True` 自动执行。

#### 特点 4：参数化测试（数据驱动原生支持）

```python
@pytest.mark.parametrize("username,password,expected", [
    ("admin", "123456", "登录成功"),
    ("admin", "wrong", "密码错误"),
])
def test_login(username, password, expected):
    resp = api_client.post("/auth/login", json={"username": username, "password": password})
    assert expected in resp.text
```

搭配 YAML 文件，加一条数据就是加一条用例。

#### 特点 5：丰富的插件生态

```bash
pytest-rerunfailures   # 失败自动重跑
pytest-xdist           # 分布式并行执行
pytest-html            # HTML 报告
pytest-cov             # 覆盖率统计
pytest-timeout         # 用例超时控制
```

#### 特点 6：标记机制（Mark）

```python
@pytest.mark.smoke           # 冒烟测试
@pytest.mark.slow            # 慢测试
@pytest.mark.skip(reason="未完成")  # 跳过
@pytest.mark.xfail(reason="已知Bug")  # 预期失败
```

```bash
pytest tests/ -m smoke              # 只跑冒烟
pytest tests/ -m "not slow"         # 跳过慢的
```

#### 特点 7：Hook 机制

```python
# conftest.py
def pytest_collection_modifyitems(items):
    items.sort(key=lambda x: x.name)  # 按名称排序
```

---

### 二、Pytest vs unittest 对比

| 对比维度 | Pytest | unittest |
|:---:|:---|:---|
| **用例发现** | 自动发现 | 手动声明 TestSuite |
| **断言** | `assert` 原生语法，报错信息详细 | `self.assertEqual` 等 30+ 方法 |
| **前后置** | fixture，scope + 嵌套 + conftest 共享 | setUp/tearDown，每个类一套 |
| **参数化** | `@pytest.mark.parametrize` 原生支持 | subTest 或自己写循环 |
| **插件** | 1000+ 插件生态 | 几乎没有 |
| **并发** | pytest-xdist 支持 | 不支持 |

---

### 三、面试回答模板

> **"我之前做的接口自动化框架就是用 Pytest 搭的。第一是自动发现用例，按 test_ 前缀就能跑，不用配 TestSuite。第二是 fixture 管理前后置，scope 可以控制到 function/module/session 级别。我写了一个 api_client fixture 自动登录、用完自动 close，所有用例都能用。第三是参数化测试，@pytest.mark.parametrize 搭配 YAML 文件管理测试数据，加一条数据就是加一条用例。第四是插件生态，rerunfailures 做失败重跑、xdist 做并发执行、Allure 出报告，项目里都用到了。**
>
> **和 unittest 比，Pytest 不需要记一堆断言方法，assert 原生语法就够了，报错信息也更详细。现在 Python 的自动化测试都在用 Pytest。"**

---

**关键词**：Pytest / fixture / 参数化测试 / 插件生态 / 自动发现 / 断言 / unittest 对比

---

## Q3. 整个测试套加一个前置条件的操作怎么写？

### 问题描述

比如我要跑整个销售模块的测试套件，跑之前必须先登录获取 token、初始化测试数据。这个"整个套件执行一次"的前置条件怎么实现？

### 答案

你说得对，**配置信息不应该硬编码在 fixture 里**。实际项目中的最佳实践是：**配置数据放文件，执行逻辑放 fixture，两者配合**。

**正确的分层**：

```
config.yaml / .env / pytest.ini       ← 放配置数据（URL/账号/密码/环境变量）
          ↓ 读取
conftest.py 中的 session fixture       ← 放执行逻辑（登录/初始化数据/清理）
          ↓ 驱动
test_xxx.py                            ← 放测试用例
```

下面分别说 Pytest 和 unittest 两种实现方式。

---

### 一、Pytest 实现方式（推荐）

#### 方式 1：conftest.py + session 级别 fixture（从配置文件读取）

```yaml
# config/config.yaml —— 配置数据放这里，不硬编码
env:
  base_url: "https://test-api.example.com"
  username: "test_admin"
  password: "test_pass"

testdata:
  init_url: "/testdata/init"
  cleanup_url: "/testdata/cleanup"
```

```python
# conftest.py —— tests/ 目录下的所有测试文件共享
import pytest
import requests
import yaml
import os

# 方式 A：从 YAML 配置文件读取
def load_config():
    config_path = os.path.join(os.path.dirname(__file__), "..", "config", "config.yaml")
    return yaml.safe_load(open(config_path, encoding="utf-8"))

# 方式 B：从环境变量读取（更安全，适合 CI/CD）
# export TEST_USERNAME="test_admin"
# export TEST_PASSWORD="test_pass"
# export BASE_URL="https://test-api.example.com"

@pytest.fixture(scope="session", autouse=True)
def global_setup():
    """
    scope="session" → 整个测试会话只执行一次
    autouse=True    → 所有用例自动生效，不需要显式声明
    """
    # 读取配置（不硬编码）
    config = load_config()
    base_url = config["env"]["base_url"]
    username = config["env"]["username"]
    password = config["env"]["password"]

    print(f"\n===== [session] 全局前置：登录获取 token ({base_url}) =====")

    # Step 1：登录获取 token
    resp = requests.post(f"{base_url}/auth/login", json={
        "username": username,
        "password": password
    })
    token = resp.json()["token"]
    assert token is not None, "登录失败，无法获取 token"

    # Step 2：把 token 存到全局，供所有用例使用
    pytest.global_token = token

    # Step 3：初始化测试数据
    init_resp = requests.post(f"{base_url}{config['testdata']['init_url']}", headers={
        "Authorization": f"Bearer {token}"
    })
    assert init_resp.status_code == 200, "测试数据初始化失败"

    print("===== [session] 全局前置完成 =====")

    yield

    # 后置清理
    print("\n===== [session] 全局后置：清理测试数据 =====")
    requests.post(f"{base_url}{config['testdata']['cleanup_url']}", headers={
        "Authorization": f"Bearer {token}"
    })
    print("===== [session] 后置清理完成 =====")


@pytest.fixture(scope="session")
def global_token():
    """把 token 暴露给需要显式引用的用例"""
    return pytest.global_token
```

```python
# test_sale.py —— 销售模块测试文件
import pytest
import requests

def test_create_order(global_token):
    """用全局 token 发送请求"""
    resp = requests.post("https://api.example.com/order/create", json={
        "sku": "SKU001",
        "quantity": 10
    }, headers={"Authorization": f"Bearer {global_token}"})
    assert resp.status_code == 200

def test_query_order(global_token):
    """同一个 session 中 global_token 自动可用"""
    resp = requests.get("https://api.example.com/order/query", params={
        "page": 1
    }, headers={"Authorization": f"Bearer {global_token}"})
    assert resp.status_code == 200
```

**执行效果**：

```bash
$ pytest tests/sale/ -v

===== [session] 全局前置：登录获取 token =====
===== [session] 全局前置完成 =====
test_sale.py::test_create_order PASSED
test_sale.py::test_query_order PASSED
===== [session] 全局后置：清理测试数据 =====
===== [session] 后置清理完成 =====
```

**关键点**：
- `scope="session"` → 整个测试过程只执行一次前置 + 一次后置
- `autouse=True` → 所有测试文件自动生效，不用每个文件 import
- `yield` 前面的代码是前置，后面的代码是后置

#### 方式 2：session 级 fixture + 全局变量（不依赖 autouse）

如果你不想用 `autouse=True`，也可以显式在用例中引用：

```python
# conftest.py
@pytest.fixture(scope="session")
def login_token():
    """非 autouse，只有显式引用的用例才会触发"""
    print("登录获取 token...")
    return get_token()

# test_sale.py
def test_create_order(login_token):  # 显式引用
    resp = requests.post("/order/create", headers={
        "Authorization": f"Bearer {login_token}"
    })
    assert resp.status_code == 200

def test_query_stock():  # 没引用 login_token，不会触发前置
    ...
```

#### 方式 3：按目录分不同的前置条件

```python
# tests/sale/conftest.py —— 只对 tests/sale/ 目录生效
@pytest.fixture(scope="session", autouse=True)
def sale_session_setup():
    """销售模块的 session 前置"""
    print("销售模块前置：初始化销售数据")
    yield
    print("销售模块后置：清理销售数据")

# tests/finance/conftest.py —— 只对 tests/finance/ 目录生效
@pytest.fixture(scope="session", autouse=True)
def finance_session_setup():
    """财务模块的 session 前置"""
    print("财务模块前置：初始化财务数据")
    yield
    print("财务模块后置：清理财务数据")
```

---

### 二、unittest 实现方式（了解即可）

```python
import unittest
import requests

class TestSaleModule(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        """
        整个测试类执行一次（相当于 session 级别）
        注意：必须用 @classmethod 装饰器
        """
        print("整个测试套前置：登录获取 token")
        resp = requests.post("https://api.example.com/auth/login", json={
            "username": "test_admin",
            "password": "test_pass"
        })
        cls.token = resp.json()["token"]

    @classmethod
    def tearDownClass(cls):
        """整个测试类执行完毕后清理"""
        print("整个测试套后置：清理数据")
        requests.post("https://api.example.com/testdata/cleanup", headers={
            "Authorization": f"Bearer {cls.token}"
        })

    def setUp(self):
        """每个用例执行前——可选"""
        print("每个用例前：刷新页面")

    def test_create_order(self):
        resp = requests.post("https://api.example.com/order/create", headers={
            "Authorization": f"Bearer {self.token}"
        }, json={"sku": "SKU001", "quantity": 10})
        self.assertEqual(resp.status_code, 200)

    def test_query_order(self):
        resp = requests.get("https://api.example.com/order/query", headers={
            "Authorization": f"Bearer {self.token}"
        })
        self.assertEqual(resp.status_code, 200)
```

**unittest 的局限**：

```
setUpClass      — 一个测试类执行一次（≈ session 级别）
setUp           — 每个用例执行一次（≈ function 级别）
tearDown        — 每个用例执行后
tearDownClass   — 整个测试类执行完后

局限：
  1. 只能在类级别，不能在模块级别或 session 级别跨文件
  2. 不支持 autouse，每个用例都需要继承同一个 TestCase
  3. 不能嵌套，一套 setUpClass 只能管一个类
```

---

### 三、pytest + unittest 混用（实战中常见的情况）

如果你在项目里看到了 `setUpClass` 和 fixture 混着用的代码，比如：

```python
import pytest

class TestERP(unittest.TestCase):
    """unittest 风格的测试类，但用 Pytest 运行"""

    @classmethod
    def setUpClass(cls):
        cls.token = login()

    def test_create(self):
        ...

    def test_query(self):
        ...
```

`pytest` 完全兼容 `unittest` 风格的用例，可以直接跑。

---

### 五、补充：几种配置文件的写法对比

Pytest 支持多种方式管理配置，你可以根据项目需要选择：

#### 方式 A：YAML 配置文件（推荐，项目中最常用）

```yaml
# config/config.yaml
env:
  base_url: "https://test-api.example.com"
  username: "test_admin"
  password: "test_pass"
```

```python
# conftest.py 中读取
import yaml
config = yaml.safe_load(open("config/config.yaml"))
base_url = config["env"]["base_url"]
```

**优点**：结构化、可读性强、支持嵌套、测试数据也能放里面
**缺点**：敏感信息（密码）明文存储，不适合提交到 Git

#### 方式 B：.env 环境变量文件（适合 CI/CD）

```bash
# .env
BASE_URL=https://test-api.example.com
TEST_USERNAME=test_admin
TEST_PASSWORD=test_pass
```

```python
# conftest.py 中读取
from dotenv import load_dotenv
load_dotenv()
import os

base_url = os.getenv("BASE_URL")
username = os.getenv("TEST_USERNAME")
password = os.getenv("TEST_PASSWORD")
```

**优点**：敏感信息不在代码里、CI/CD 可以直接注入环境变量
**缺点**：不支持嵌套，只有键值对

#### 方式 C：pytest.ini（Pytest 官方配置）

```ini
# pytest.ini
[pytest]
base_url = https://test-api.example.com
test_username = test_admin
test_password = test_pass
```

```python
# conftest.py 中读取
@pytest.fixture(scope="session")
def base_url(pytestconfig):
    return pytestconfig.getini("base_url")
```

**优点**：Pytest 原生支持，不用额外导入
**缺点**：所有值都是字符串，不适合放复杂结构

#### 实战推荐组合

```
项目结构：
tests/
├── conftest.py          ← session fixture（执行逻辑）
├── config/
│   ├── config.yaml      ← 结构化配置（环境/账号/测试数据）
│   └── .env             ← 敏感信息（密码/token）（不提交Git）
└── test_xxx.py          ← 测试用例

conftest.py 中：
  1. 从 config.yaml 读结构化配置（URL、测试数据）
  2. 从 .env 读敏感信息（密码、token）
  3. 用读到的配置执行登录/初始化
  4. yield 前做前置，yield 后做清理
```

---

### 六、更新后的面试回答模板

> **"整个测试套的前置条件我分两层处理。配置数据放文件——URL、账号密码放 config.yaml，敏感信息放 .env 不提交 Git。执行逻辑放 conftest.py——session 级别的 fixture 从配置文件中读取数据，执行登录和初始化。**
>
> **这样改环境只需要改配置文件，不改代码。跑不同环境（dev/staging/prod）就切换不同的配置文件，fixture 不用动。"**

---

**关键词**：session fixture / conftest.py / setUpClass / autouse / 测试前置条件 / 配置管理 / config.yaml / .env

---

## Q4. 调用每个关键字前自动打日志，不想在每个方法里手动加，怎么写？

### 问题描述

关键字驱动框架里有很多关键字方法（login、query_stock、update_price 等），想在调用每个关键字之前自动打印一条日志，比如 ">>> 开始执行关键字: login"，但又不想在每个方法里手动写一行 print，想写成一个全局统一的拦截器。

### 答案

这个需求本质上是 **AOP（面向切面编程）**，核心思路是"在执行目标方法前自动插入日志逻辑"。Python 有几种实现方式，从最简单到最优雅。

---

### 方案一：在引擎的 run 方法里统一打日志（最简单）

不需要动任何关键字方法，只改引擎的 `run` 方法：

```python
class KeywordEngine:
    """关键字驱动引擎——run 方法统一打日志"""

    def __init__(self):
        self.context = {}
        self.keywords = {
            "login": self._login,
            "query_stock": self._query_stock,
            "update_price": self._update_price,
            "check_finance_sync": self._check_finance_sync,
        }

    def run(self, test_steps):
        for step in test_steps:
            keyword = step["keyword"]
            params = step.get("params", {})

            # ★ 全局统一打日志，只加这一行 ★
            print(f"[Keyword] >>> 开始执行关键字: {keyword} | 参数: {params}")

            resolved_params = self._resolve_params(params)
            result = self.keywords[keyword](**resolved_params)
            print(f"[Keyword] <<< 关键字完成: {keyword} | 耗时: xxx")

            self.context["response"] = result
```

**优点**：一行代码解决问题，最简洁
**缺点**：只能打简单的 before/after 日志，拿不到方法内部的详细耗时

---

### 方案二：装饰器 + 自动扫描（不需要每个方法手动加 @）

用类装饰器，在创建类时自动给所有方法加上日志装饰器：

```python
import time
import functools

# Step 1：定义一个日志装饰器
def log_keyword_call(func):
    """装饰器：自动打印关键字调用的日志"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        keyword_name = func.__name__.lstrip("_")  # _login → login
        print(f"[Keyword] >>> 开始执行关键字: {keyword_name}")
        print(f"[Keyword]    参数: {kwargs}")

        start = time.time()
        try:
            result = func(*args, **kwargs)
            elapsed = time.time() - start
            print(f"[Keyword] <<< 关键字完成: {keyword_name} | 耗时: {elapsed:.3f}s | 状态: OK")
            return result
        except Exception as e:
            elapsed = time.time() - start
            print(f"[Keyword] <<< 关键字失败: {keyword_name} | 耗时: {elapsed:.3f}s | 错误: {e}")
            raise
    return wrapper


# Step 2：类装饰器 — 自动给所有私有方法（关键字方法）加上日志装饰器
def auto_log_keywords(cls):
    """类装饰器：遍历类的所有方法，自动给关键字方法加日志"""
    for name, method in list(vars(cls).items()):
        # 只给私有方法（_xxx）加上日志，跳过 public 方法（如 run）
        if name.startswith("_") and callable(method) and not name.startswith("__"):
            setattr(cls, name, log_keyword_call(method))
    return cls


# Step 3：使用类装饰器，不需要在每个方法上加 @log_keyword_call
@auto_log_keywords
class KeywordEngine:
    """关键字驱动引擎——所有关键字方法自动打日志"""

    def __init__(self):
        self.context = {}
        self.keywords = {
            "login": self._login,
            "query_stock": self._query_stock,
            "update_price": self._update_price,
        }

    def run(self, test_steps):
        for step in test_steps:
            keyword = step["keyword"]
            params = step.get("params", {})
            resolved_params = self._resolve_params(params)
            result = self.keywords[keyword](**resolved_params)
            self.context["response"] = result

    # 这些方法不需要加 @log，装饰器自动处理了
    def _login(self, username, password):
        return requests.post("/auth/login", json={"username": username, "password": password})

    def _update_price(self, sku, new_price):
        return requests.post("/sale/price/update", json={"sku": sku, "price": new_price})
```

**执行效果**：

```bash
[Keyword] >>> 开始执行关键字: login
[Keyword]    参数: {'username': 'admin', 'password': '******'}
[Keyword] <<< 关键字完成: login | 耗时: 0.235s | 状态: OK

[Keyword] >>> 开始执行关键字: update_price
[Keyword]    参数: {'sku': 'SKU001', 'new_price': 99.00}
[Keyword] <<< 关键字完成: update_price | 耗时: 0.112s | 状态: OK
```

**优点**：
- 一行 `@auto_log_keywords` 搞定所有关键字方法
- 新增关键字方法不需要加任何代码，自动打日志
- 还能统计耗时、捕获异常

**缺点**：需要理解装饰器和类装饰器

---

### 方案三：__getattr__ 动态代理（最优雅）

利用 Python 的 `__getattr__` 方法，在调用不存在的属性时自动拦截：

```python
import time

class LoggingProxy:
    """日志代理——自动拦截所有关键字调用，打印日志"""

    def __init__(self, target):
        self._target = target
        self._keyword_prefix = "_"  # 关键字方法都以 _ 开头

    def __getattr__(self, name):
        """当访问的属性不存在时触发"""
        original_attr = getattr(self._target, name)

        # 只拦截可调用且以 _ 开头的方法（关键字方法）
        if name.startswith(self._keyword_prefix) and callable(original_attr):
            def logged_method(*args, **kwargs):
                keyword_name = name.lstrip("_")
                print(f"[Keyword] >>> 开始执行关键字: {keyword_name}")
                print(f"[Keyword]    参数: {kwargs if kwargs else args}")

                start = time.time()
                try:
                    result = original_attr(*args, **kwargs)
                    elapsed = time.time() - start
                    print(f"[Keyword] <<< 关键字完成: {keyword_name} | 耗时: {elapsed:.3f}s")
                    return result
                except Exception as e:
                    elapsed = time.time() - start
                    print(f"[Keyword] <<< 关键字失败: {keyword_name} | 耗时: {elapsed:.3f}s | 错误: {e}")
                    raise
            return logged_method

        return original_attr


# 使用——引擎类不需要任何修改
class KeywordEngine:
    def __init__(self):
        self.keywords = {
            "login": self._login,
            "query_stock": self._query_stock,
        }

    def _login(self, username, password):
        return requests.post("/auth/login", json={"username": username, "password": password})

    def _query_stock(self, sku):
        return requests.get("/wms/stock/query", params={"sku": sku})


# 用代理包装引擎——所有关键字调用自动打日志
engine = LoggingProxy(KeywordEngine())
engine._login(username="admin", password="123456")
# 自动输出：
# [Keyword] >>> 开始执行关键字: login
# [Keyword]    参数: {'username': 'admin', 'password': '123456'}
# [Keyword] <<< 关键字完成: login | 耗时: 0.235s
```

**优点**：
- **引擎类零修改**，不需要加任何装饰器
- 用一个包装类就实现了全局拦截
- 想关日志就换回原始引擎，不影响逻辑

---

### 四、方案对比

| 方案 | 代码量 | 侵入性 | 灵活性 | 推荐场景 |
|:---:|:---:|:---:|:---:|:---|
| **① run 方法统一打** | 1 行 | 低 | 低 | 快速实现，项目初期 |
| **② 类装饰器** | 10 行 | 中 | 高 | 框架成熟期，需统计耗时 |
| **③ __getattr__ 代理** | 20 行 | **零** | 最高 | **最推荐**，不改引擎代码 |

### 五、面试回答模板

> **"我不在每个关键字方法里手动加日志，而是用 __getattr__ 实现了一个日志代理。在代理类里重写了 __getattr__，拦截所有以 _ 开头的方法调用，自动打印开始/结束/耗时/结果。**
>
> **这样引擎类本身不用改任何代码，想加日志就用 LoggingProxy 包装一下，想关日志就用回原始引擎。新增的关键字方法也自动生效，不需要加任何装饰器。"**

---

**关键词**：AOP / 日志代理 / 装饰器 / __getattr__ / 面向切面编程 / 关键字引擎日志

---

## Q5. 什么时候适合做 UI 自动化和接口自动化？前提条件是什么？

### 问题描述

面试官问你：你决定做自动化的时候，怎么判断这个项目/模块适不适合？UI 自动化和接口自动化的准入条件分别是什么？

### 答案

自动化不是万能的，做之前要判断"值不值得做"。UI 自动化和接口自动化的准入条件不完全一样。

一句话总结：

> **接口自动化门槛低，核心链路先上；UI 自动化门槛高，稳定页面再上。**

---

### 一、接口自动化的前提条件

| 条件 | 说明 | 你的 ERP 项目 |
|:---|:---|:---|
| **接口已经稳定** | 不再频繁改字段名、返回值结构 | ✅ ERP 接口相对稳定 |
| **有接口文档** | Swagger/YAPI/手写都行，但要有 | ✅ 有 Swagger |
| **有测试环境** | 独立的测试环境，不和生产共用 | ✅ 有 QA 环境 |
| **业务逻辑可验证** | 不是"调了就行"，要能验证返回的正确性 | ✅ 出库金额 vs 凭证金额 |
| **维护成本可接受** | 返工量和节省的时间相比是划算的 | ✅ 核心链路 90 条 |

**什么时候不适合做**：

```text
❌ 接口还在频繁变动（字段改来改去，维护成本高）
❌ 没有接口文档（完全靠猜，写出来的脚本不可靠）
❌ 纯读接口（只查数据，没有业务断言可验证）
❌ 一次性任务（只跑一次，写脚本的时间比手工跑还长）
```

---

### 二、UI 自动化的前提条件

| 条件 | 说明 | 你的 ERP 项目 |
|:---|:---|:---|
| **界面已经稳定** | 不会频繁改版、改元素、改交互流程 | ⚠️ 部分页面会改版 |
| **元素有唯一标识** | 有 id / name / data-testid，不是靠 class 定位 | ⚠️ 部分元素只有 class |
| **业务流程核心且有复用价值** | 不是测一次就不用，要反复回归 | ✅ 采购/销售流程 |
| **有人力维护** | UI 自动化维护成本高，必须有专人管 | ⚠️ 人力有限 |
| **有独立测试环境** | 不和开发共用，避免互相干扰 | ✅ 有 QA 环境 |

**什么时候不适合做**：

```text
❌ 页面频繁改版（今天写的脚本下周全废）
❌ 元素没有稳定的定位方式（全靠 class 和 xpath 硬定位）
❌ 低频率的功能（一个月用一次，自动化回报低）
❌ 一次性测试任务
❌ 验证码无法绕过的场景
❌ 人力不够（UI 自动化维护成本高，没人管的话 3 个月就全废）
```

---

### 三、两者对比

| 维度 | 接口自动化 | UI 自动化 |
|:---|:---|:---|
| **门槛** | 低 | 高 |
| **稳定性** | 高（接口不改就不坏） | 低（一个 class 名变了就挂） |
| **维护成本** | 低（改参数就行） | 高（要改元素定位） |
| **执行速度** | 快（秒级） | 慢（秒到分钟级） |
| **覆盖范围** | 后端逻辑、数据处理 | 端到端用户体验 |
| **什么人能写** | 测试人员 | 需要懂前端基础 |
| **失败时定位难度** | 易（看接口返回） | 难（截图看为什么找不到元素） |

---

### 四、实战建议：你的 ERP 项目怎么选

**接口自动化优先（先做，门槛低，回报高）**：

```text
✅ 核心业务链路（创建订单 → 出库 → 凭证）
   → 接口自动化覆盖，投入产出比最高

✅ 多模块联动场景（改价 → 成本同步 → 财务校验）
   → 接口自动化最适合这种跨模块的数据一致性校验

✅ 回归频率高的接口（每次版本都变的功能）
   → 接口自动化，省手工回归时间

✅ 异常场景（库存不足、权限校验）
   → 接口自动化方便造异常数据
```

**UI 自动化谨慎做（后做，选最稳定的页面）**：

```text
✅ 登录页 → 最稳定，几乎不改
✅ 核心业务操作页面（出库审核页）→ 相对稳定的流程
❌ 经常改版的管理后台首页 → 暂时不做
❌ 报表页面 → 数据量大，加载慢，自动化效率低
```

**你不会 UI 自动化的策略**（如果面试官追问）：

> "目前在接口自动化上投入更多，因为投入产出比高、维护成本低。UI 自动化只做了登录和出库审核这两个最稳定的页面。后续如果人力允许，会逐步扩展到其他核心页面。"

---

### 五、面试回答模板

> **"接口自动化的前提条件比较低——接口稳定、有文档、有独立的测试环境就可以做。我做的第一步就是把核心业务链路的接口自动化了，维护成本低、回归效率高。**
>
> **UI 自动化的准入条件更严格——页面要稳定、元素要有唯一标识、业务流程要有反复回归的价值。UI 自动化的维护成本比接口高很多，所以我只做了登录和出库审核这两个最稳定的页面的 UI 自动化，其余的暂时通过接口和手工覆盖。**
>
> **核心原则是：接口自动化优先、UI 自动化谨慎做。不要把时间花在维护不稳定的 UI 脚本上。"**

---

**关键词**：接口自动化条件 / UI 自动化条件 / 自动化准入 / 维护成本 / 投入产出比

---

## Q6. 为什么选择 Python + Requests + Selenium 作为自动化测试框架？

### 问题描述

面试官问：市面上有这么多语言和工具，你为什么选 Python + Requests + Selenium 这个组合？考的是你**选型有没有自己的思考**，而不是"因为大家都在用"。

### 答案

我选这个组合不是跟风，是有具体理由的：

---

### 一、为什么选 Python

| 理由 | 说明 | 对你的实际价值 |
|:---|:---|:---|
| **学习成本低** | 语法简单，新手上手快 | 你从财务转测试，Python 是最友好的入门语言 |
| **生态丰富** | Pytest + Requests + Selenium + Allure 无缝配合 | 一个语言覆盖接口/UI/报告/CI |
| **测试框架成熟** | Pytest 是 Python 测试的事实标准 | fixture + parametrize 直接解决前后置和数据驱动 |
| **团队友好** | 开发也用 Python 写脚本，沟通无障碍 | 出了问题可以一起看代码 |
| **社区活跃** | 问题 Google 一下就有答案 | 碰上坑不用从头啃文档 |

**对比：为什么没选 Java？**

```text
Java（TestNG + RestAssured + Selenium）：
  ✅ 强类型，适合大型项目
  ❌ 代码量大、配置繁琐、学习曲线陡
  ❌ 不适合中小团队快速迭代

Python：
  ✅ 语法简洁、开发效率高
  ✅ 快速验证想法，写一条用例几分钟
  ✅ 非常适合测试自动化
```

**一句话**：Python 写一条用例 10 行代码，Java 可能要 30 行。

---

### 二、为什么选 Requests

| 理由 | 说明 | 对比 |
|:---|:---|:---|
| **HTTP 库事实标准** | Python 生态里最主流的 HTTP 库 | 没有对手 |
| **API 简洁** | `requests.post(url, json=data)` 一行搞定 | 比 urllib 简单 10 倍 |
| **支持所有 HTTP 方法** | GET/POST/PUT/DELETE/PATCH 开箱即用 | — |
| **Session 支持** | 自动管理 Cookie，不用每次手动传 | 模拟登录状态极方便 |
| **JSON 原生支持** | `resp.json()` 直接解析 | 不用手动 import json |

**实际代码对比**：

```python
# Requests —— 简洁
import requests
resp = requests.post("https://api.example.com/outbound/audit",
                     json={"outbound_id": 1001},
                     headers={"Authorization": f"Bearer {token}"})
data = resp.json()
assert data["status"] == "审核通过"

# 用 urllib（Python 内置库）—— 啰嗦
import urllib.request, json
data = json.dumps({"outbound_id": 1001}).encode()
req = urllib.request.Request("https://api.example.com/outbound/audit",
                              data=data,
                              headers={"Authorization": f"Bearer {token}",
                                       "Content-Type": "application/json"})
resp = urllib.request.urlopen(req)
result = json.loads(resp.read())
assert result["status"] == "审核通过"
```

---

### 三、为什么选 Selenium

| 理由 | 说明 | 对比 |
|:---|:---|:---|
| **浏览器兼容** | 支持 Chrome/Edge/Firefox | Playwright 也支持，但 Selenium 历史更久 |
| **生态成熟** | 资料多、踩过的坑都有解决方案 | Cypress 只支持 JS，Playwright 相对新 |
| **多语言支持** | Python/Java/JS/C# 都能用 | 团队语言切换时不用换工具 |
| **WebDriver 标准** | W3C 标准，浏览器厂商原生支持 | 不会说换就被换 |
| **和 Pytest 无缝集成** | 一个 driver fixture 搞定所有用例 | — |

**对比：为什么没选 Cypress / Playwright？**

```text
Cypress：
  ❌ 只支持 JavaScript，团队用 Python
  ❌ 不支持多 Tab / 多窗口

Playwright：
  ✅ 更快、更现代
  ❌ 相对新，遇到坑的资料不如 Selenium 多
  ❌ 你项目里已经有 Selenium 积累的代码了，换的成本高
```

**一句话**：Selenium 是 UI 自动化的"老中医"，虽然不新但最稳。

---

### 四、这个组合配合 Pytest 的效果

```python
# Pytest 把这些库串起来，形成完整框架
import pytest
import requests
from selenium import webdriver

@pytest.fixture(scope="session")
def api_token():
    """接口自动化：用 requests 登录拿 token"""
    resp = requests.post("https://api.example.com/auth/login", json={
        "username": "admin", "password": "123456"
    })
    return resp.json()["token"]

@pytest.fixture(scope="function")
def driver():
    """UI 自动化：Selenium 管理浏览器"""
    d = webdriver.Chrome()
    yield d
    d.quit()

def test_interface(api_token):
    """接口测试：一行 requests 搞定"""
    resp = requests.get("https://api.example.com/order/list", headers={
        "Authorization": f"Bearer {api_token}"
    })
    assert resp.status_code == 200

def test_ui(driver, api_token):
    """UI 测试：Selenium 操作浏览器"""
    driver.get("https://erp.example.com/outbound")
    # ... 页面操作 ...
    assert "审核通过" in driver.page_source
```

Python + Requests + Selenium + Pytest = 接口/UI/报告/CI 全搞定，一个语言覆盖所有。

---

### 五、面试回答模板

> **"我选 Python + Requests + Selenium 这个组合有三个原因。**
>
> **第一，Python 学习成本低、生态丰富。之前做财务转测试，Python 上手最快。而且 Pytest + Requests + Selenium + Allure 全在一个语言里搞定，不需要接口用 Java、UI 用 JS。**
>
> **第二，Requests 是 Python 里最简洁的 HTTP 库，一行代码发请求、一行解析 JSON。和 urllib 比简洁了不止一个量级。**
>
> **第三，Selenium 生态最成熟，多浏览器支持好，和 Pytest 无缝集成。虽然 Playwright 更快，但 Selenium 的坑基本都有人踩过了，出问题搜一下就有答案。**
>
> **三个库配合 Pytest，接口自动化用 Requests 发请求 + assert 断言，UI 自动化用 Selenium 操作页面 + POM 模式管理元素，全部用 Pytest 的 fixture + parametrize 串联起来。"**

---

**关键词**：技术选型 / Python / Requests / Selenium / Pytest / 自动化框架选型

---

## Q7. Requests 做多接口传参怎么处理？

### 问题描述

接口自动化里经常是一条业务链路涉及多个接口——登录拿 token → 创建订单得 order_id → 用 order_id 审核出库。这些接口之间的数据怎么传递？

### 答案

多接口传参有三种主流方案，从简单到完善。你的项目里应该用的是**方案二 + 方案三的组合**。

---

### 方案一：Pytest fixture 传参（最简单）

```python
import pytest
import requests

@pytest.fixture(scope="session")
def token():
    """登录获取 token，session 级复用"""
    resp = requests.post("https://api.example.com/auth/login", json={
        "username": "admin", "password": "123456"
    })
    return resp.json()["token"]

@pytest.fixture(scope="function")
def order_id(token):
    """创建订单拿 order_id，依赖 token"""
    resp = requests.post("https://api.example.com/order/create",
        json={"sku": "SKU001", "quantity": 100},
        headers={"Authorization": f"Bearer {token}"}
    )
    return resp.json()["id"]

def test_outbound_audit(token, order_id):
    """审核出库，依赖 token 和 order_id"""
    resp = requests.post(f"https://api.example.com/outbound/{order_id}/audit",
        headers={"Authorization": f"Bearer {token}"}
    )
    assert resp.status_code == 200
    assert resp.json()["status"] == "审核通过"
```

**优点**：Pytest 原生，不需要额外代码
**缺点**：fixture 多了嵌套深，读起来费劲

---

### 方案二：上下文对象传参（最实用）

用一个 context 字典在多个接口之间传递数据：

```python
class APIClient:
    """带上下文的 API 客户端"""

    def __init__(self, base_url):
        self.base_url = base_url
        self.context = {}  # 上下文字典，存中间结果
        self.session = requests.Session()  # Session 自动管理 Cookie

    def login(self, username, password):
        resp = self.session.post(f"{self.base_url}/auth/login", json={
            "username": username, "password": password
        })
        self.context["token"] = resp.json()["token"]
        self.context["user_id"] = resp.json()["user_id"]
        self.session.headers.update({
            "Authorization": f"Bearer {self.context['token']}"
        })
        return resp

    def create_order(self, sku, quantity):
        resp = self.session.post(f"{self.base_url}/order/create", json={
            "sku": sku, "quantity": quantity
        })
        # ★ 存到上下文，后续接口自动可用
        self.context["order_id"] = resp.json()["id"]
        self.context["order_amount"] = resp.json()["amount"]
        return resp

    def audit_outbound(self):
        # ★ 从上下文取 order_id，不用手动传
        order_id = self.context["order_id"]
        resp = self.session.post(f"{self.base_url}/outbound/{order_id}/audit")
        return resp

    def verify_voucher(self):
        # ★ 关联校验：订单金额 vs 凭证金额
        order_amount = self.context["order_amount"]
        resp = self.session.get(f"{self.base_url}/finance/voucher/query", params={
            "order_id": self.context["order_id"]
        })
        voucher_amount = resp.json()["amount"]
        assert voucher_amount == order_amount, \
            f"对账不平！订单:{order_amount} 凭证:{voucher_amount}"
        return resp


# Pytest fixture —— session 级复用
@pytest.fixture(scope="session")
def api():
    client = APIClient("https://test-api.example.com")
    client.login("admin", "123456")
    yield client

def test_full_flow(api):
    """业务链路测试：数据自动在 context 里流转"""
    api.create_order("SKU001", 100)
    api.audit_outbound()
    api.verify_voucher()
    # 所有传参都是自动的，测试用例不用手动传 token 和 order_id
```

**优点**：
- 链路上的数据自动流转，测试用例极其简洁
- `requests.Session` 自动管理 Cookie
- `self.context` 里可以存任何中间结果

**缺点**：要写一个 APIClient 包装类

---

### 方案三：关键字引擎传参（你的框架）

这就是你之前说的关键字驱动框架，通过 YAML 配置 + 引擎上下文传参：

```yaml
# testcase.yaml
steps:
  - keyword: login
    params:
      username: admin
      password: "123456"
    save_as: login_result

  - keyword: create_order
    params:
      sku: SKU001
      quantity: 100
    save_as: order_result

  - keyword: audit_outbound
    params:
      order_id: "{order_result.data.id}"  # ★ 用 {xxx} 引用上一步的结果
```

```python
class KeywordEngine:
    def __init__(self):
        self.context = {}  # 全局上下文

    def _resolve_params(self, params):
        """解析参数中的 {xxx} 引用"""
        resolved = {}
        for key, value in params.items():
            if isinstance(value, str) and value.startswith("{") and value.endswith("}"):
                # "{order_result.data.id}" → 从 context 取值
                path = value[1:-1].split(".")
                val = self.context
                for p in path:
                    val = val[p]
                resolved[key] = val
            else:
                resolved[key] = value
        return resolved

    def run(self, test_steps):
        for step in test_steps:
            keyword = step["keyword"]
            params = self._resolve_params(step.get("params", {}))

            result = self.keywords[keyword](**params)

            # 存到上下文，后面步骤可以引用
            if "save_as" in step:
                self.context[step["save_as"]] = result
```

---

### 三、方案对比

| 方案 | 传参方式 | 适用场景 | 你在用？ |
|:---|:---|:---|:---:|
| **fixture** | Pytest 依赖注入 | 简单链路，2-3个接口 | 是 |
| **上下文对象** | `self.context` 字典 | 中等复杂度，10 个接口内 | 是 |
| **关键字引擎** | YAML 配置 `{xxx}` 引用 | 复杂链路，业务流程多变 | 是 |

你的框架大概率是**上下文对象做基础 + 关键字引擎做封装**的组合。

---

### 四、面试回答模板

> **"多接口传参我用两种方式。基础层用上下文对象——写一个 APIClient 类，里面维护一个 context 字典。比如登录后把 token 存到 context，后续接口自动从 context 取 token 拼到 header 里。创建订单后把 order_id 存到 context，审核出库直接从 context 取，不需要手动传参。**
>
> **业务链路层用关键字引擎——YAML 里定义步骤，每一步存一个 save_as，后面步骤用 {login_result.token} 引用。比如创建订单步骤依赖 token，YAML 里直接写 {login_result.token}，引擎自动解析 context 里的值传过去。"**

---

**关键词**：多接口传参 / 上下文对象 / 关键字引擎 / Requests Session / context 字典

---

## Q8. 测试中遇到重定向问题怎么处理？

### 问题描述

接口自动化测试时，遇到接口返回 301/302 重定向，有时候需要自动跟随、有时候需要禁止跟随、有时候需要验证重定向到了正确的地方。用 Requests 怎么处理？

### 答案

Requests 默认**自动跟随重定向**，大部分场景下这是好事。但测试中需要"看中间过程"或者"拦截重定向"时，就要手动控制。

---

### 一、默认行为：自动跟随重定向

```python
import requests

# 访问 http，服务端重定向到 https
resp = requests.get("http://api.example.com/health")

print(resp.status_code)      # 200（最终结果）
print(resp.url)               # https://api.example.com/health（最终地址）

# Requests 自动完成了 http → https 的跟随，你拿到的就是最终结果
```

**大多数场景下这就是你要的**，不需要额外处理。

---

### 二、禁止跟随重定向

测试中需要验证"服务端是否正确返回了 302"：

```python
import requests

resp = requests.get("http://old.example.com", allow_redirects=False)

assert resp.status_code == 302, "期望重定向但实际返回了其他状态码"
assert resp.headers["Location"] == "https://new.example.com", "重定向地址不对"
```

**场景**：测试旧域名迁移，验证 302 指向了正确的地址。

---

### 三、查看重定向历史

需要验证重定向路径时：

```python
import requests

resp = requests.get("http://a.example.com/page")

# 查看重定向历史
for i, r in enumerate(resp.history):
    print(f"第 {i+1} 跳: {r.status_code} → {r.headers['Location']}")

print(f"最终: {resp.status_code} {resp.url}")

# 输出示例：
# 第 1 跳: 301 → https://a.example.com/page
# 第 2 跳: 302 → https://b.example.com/new-page
# 最终: 200 https://b.example.com/new-page
```

**场景**：验证多重跳转链路是否正确。

---

### 四、保持 Session 跨重定向

重定向通常会丢失 Cookie 或自定义 Header。用 `requests.Session` 保持状态：

```python
import requests

session = requests.Session()

# 登录（可能涉及 OAuth 重定向）
login_resp = session.post("https://api.example.com/auth/login", json={
    "username": "admin", "password": "123456"
})

# 访问需要登录的页面（自动带 Cookie）
resp = session.get("https://api.example.com/admin/dashboard")
assert resp.status_code == 200
# OAuth 重定向：login → 302 → 授权页 → 302 → dashboard
# Session 自动处理多跳，携带 Cookie
```

---

### 五、测试中的常见重定向场景

| 场景 | 处理方式 | 代码 |
|:---|:---|:---|
| **HTTP → HTTPS** | 默认跟随 | 不用管 |
| **旧域名迁移** | 禁止跟随，验证 302 指向 | `allow_redirects=False` |
| **登录后重定向** | Session 保持状态 | `session.post()` |
| **支付回调重定向** | 禁止跟随，拿到回调 URL 做二次请求 | `allow_redirects=False` |
| **认证失败重定向** | 验证重定向到了登录页 | 检查 `resp.history[0]` |

---

### 六、UI 自动化中的重定向

Selenium 中重定向是自动的，但要验证"页面跳到了正确的地方"：

```python
from selenium import webdriver

driver = webdriver.Chrome()
driver.get("http://old.example.com")

# 验证自动跳转到了新域名
assert "new.example.com" in driver.current_url

# 验证标题
assert driver.title == "ERP管理系统"
```

---

### 七、面试回答模板

> **"测试中遇到重定向，大部分场景不用管——Requests 默认自动跟随，Session 自动管理 Cookie，登录后的多跳重定向都自动处理了。**
>
> **需要验证重定向行为时，把 allow_redirects 设为 False，拿到 302 响应检查 Location 头是否指向了正确地址。比如做过一次旧域名迁移，我就是写了一条用例禁止跟随重定向，验证返回了 302 且 Location 指向了新地址。**
>
> **如果涉及 OAuth 登录这种多跳重定向，Session 自动帮我把 Cookie 跨跳带到最终页面，不需要手动处理。"**

---

**关键词**：重定向 / allow_redirects / Session / 302 / 301 / OAuth

---

## Q9. Python 常用的数据类型有哪些？（结合测试场景）

### 问题描述

面试官问你 Python 常用的数据类型，不是在考你背八股，而是看你**在日常测试工作里真正用过哪些**。你要结合测试场景说出来。

### 答案

Python 有 6 种最常用的数据类型，我在接口自动化里天天用到：

---

### 一、6 种常用类型

| 类型 | 写法 | 可变 | 有序 | 可重复 | 测试中什么时候用 |
|:---|:---|:---:|:---:|:---:|:---|
| **str** | `"hello"` | ❌ | — | — | URL、响应体、断言文本 |
| **int/float** | `200` / `99.99` | ❌ | — | — | 状态码、金额、数量 |
| **list** | `[1, 2, 3]` | ✅ | ✅ | ✅ | 用例列表、多组参数 |
| **dict** | `{"key": "val"}` | ✅ | ❌ | key唯一 | 接口入参、响应解析、上下文 |
| **tuple** | `(1, 2, 3)` | ❌ | ✅ | ✅ | 固定配置、parametrize 参数 |
| **set** | `{1, 2, 3}` | ✅ | ❌ | ❌ | 去重（重复订单ID校验） |

### 二、结合测试场景说说怎么用

#### str（字符串）—— 最常用

```python
# URL 拼接
url = f"{base_url}/order/create"

# 断言：检查响应体中是否包含期望文本
assert "登录成功" in resp.text
assert "审核通过" in resp.text

# 提取子串
token = resp.json()["token"]     # 从 JSON 里拿字符串
order_id = re.findall(r"ORD\d+", resp.text)[0]  # 正则提取订单号
```

#### int / float（数字）—— 状态码和金额

```python
# 状态码断言
assert resp.status_code == 200

# 金额精确比对（财务数据必须精确）
assert resp.json()["amount"] == 100.00
assert abs(outbound_amount - voucher_amount) < 0.01  # 浮点数用 abs 比较
```

#### list（列表）—— 用例批量管理

```python
# 从 YAML 读取多组用例数据，列表存储
cases = yaml.safe_load(open("data/cases.yaml"))

# 遍历用例
for case in cases:
    resp = requests.post(case["url"], json=case["body"])
    assert resp.status_code == case["expect"]

# 查接口返回的订单列表
orders = resp.json()["items"]        # items 就是 list
assert len(orders) == 10             # 验证列表长度
assert orders[0]["status"] == "PAID"  # 取第一个
```

#### dict（字典）—— 接口自动化的核心

```python
# 接口入参就是 dict
resp = requests.post("/order/create", json={
    "sku": "SKU001",
    "quantity": 100,
    "warehouse_id": "WH001"
})

# 解析响应
data = resp.json()            # .json() 返回的就是 dict
order_id = data["id"]         # 取字段
amount = data["amount"]

# 上下文传参：context 就是 dict
self.context["token"] = token
self.context["order_id"] = order_id

# 遍历字典
for key, value in data.items():
    print(f"{key} = {value}")
```

#### tuple（元组）—— parametrize 参数

```python
# pytest parametrize 的元组用法
@pytest.mark.parametrize("username,password,expected", [
    ("admin", "123456", "登录成功"),      # 每个就是一个 tuple
    ("admin", "wrong", "密码错误"),
])

# 元组和列表的区别：元组不可变
config = ("https://test-api.com", "admin", "123456")  # 配置不会变，用 tuple
cases = [("case1", 200), ("case2", 401)]               # 用例会增减，用 list
```

#### set（集合）—— 去重校验

```python
# 并发测试：验证没有重复订单ID
order_ids = []
with ThreadPoolExecutor(max_workers=20) as executor:
    futures = [executor.submit(create_order, i) for i in range(50)]
    for f in futures:
        order_ids.append(f.result().json()["id"])

# set 去重验证
assert len(order_ids) == len(set(order_ids)), \
    f"存在重复订单！总数:{len(order_ids)} 去重后:{len(set(order_ids))}"
```

### 三、可变 vs 不可变（面试加分点）

```python
# 不可变类型：str、int、float、tuple
# 改了就是新对象
a = "hello"
b = a
a = "world"
print(b)  # "hello"（没被影响）

# 可变类型：list、dict、set
# 多个变量指向同一个对象
a = [1, 2, 3]
b = a
a.append(4)
print(b)  # [1, 2, 3, 4]（b 也被改了！）
```

**测试中的坑**：

```python
# ❌ 默认参数用可变类型有坑
def run_tests(cases=[]):     # 默认值只初始化一次
    cases.append("new")
    return cases

print(run_tests())  # ['new']
print(run_tests())  # ['new', 'new'] ← 第二次调用还带着上次的数据！

# ✅ 正确写法
def run_tests(cases=None):
    if cases is None:
        cases = []
    cases.append("new")
    return cases
```

### 四、面试回答模板

> **"我测试中用得最多的是五种：str 做断言文本匹配、int 做状态码校验、dict 做接口入参和响应解析、list 做用例批量管理、set 做重复数据校验。**
>
> **dict 是接口自动化的核心——入参是 dict、响应 .json() 返回 dict、上下文传参也是 dict，基本所有数据流转都在 dict 里。**
>
> **另外面试官如果追问可变和不可变，可以说 str/int/tuple 不可变，list/dict/set 可变，用可变类型做默认参数有坑。"**

---

**关键词**：Python 数据类型 / dict / list / tuple / set / 可变不可变

---

## Q10. `is` 和 `==` 有什么区别？

### 答案

一句话：**`is` 比身份（是不是同一个对象），`==` 比值（内容是不是一样的）。**

```python
a = [1, 2, 3]
b = [1, 2, 3]

print(a == b)   # True  — 值一样
print(a is b)   # False — 但它们是两个不同的对象（内存地址不同）
```

**测试中最常见的使用场景**：

```python
# ✅ 判断 None 必须用 is
if result is None:
    print("接口返回为空")

# ✅ 判断布尔
if flag is True:
    ...

# ✅ 判断状态码用 ==
assert resp.status_code == 200
assert data["status"] == "审核通过"
```

**记住两条规则就够了**：

```text
None / True / False  → 用 is
数字 / 字符串 / 其他值  → 用 ==
```

**面试一句话**：`is` 看是不是同一个人，`==` 看长得像不像。

---

**关键词**：is vs == / Python 身份判断 / None 判断

---

## Q11. 生成器和迭代器是什么？在测试工作中怎么用？

### 问题描述

Python 面试高频题：生成器和迭代器有什么区别？你在测试里用过吗？

### 答案

一句话：**迭代器是"可以一个一个取"的对象，生成器是"用 `yield` 写的迭代器"——更懒、更省内存。**

---

### 一、迭代器（Iterator）

迭代器就是能被 `for` 循环遍历的东西，一次只给你一个值：

```python
# list 是迭代器，但它是"先全部装好再一个个给"
nums = [1, 2, 3, 4, 5]  # 5 个数全在内存里
for n in nums:
    print(n)

# 真正的迭代器——用 iter()
it = iter([1, 2, 3])
print(next(it))  # 1
print(next(it))  # 2
print(next(it))  # 3
print(next(it))  # StopIteration（没了）
```

---

### 二、生成器（Generator）

生成器是迭代器的懒人版——**不提前产出数据，用到才生成**：

```python
# 生成器函数（用 yield）
def generate_numbers(n):
    for i in range(n):
        yield i  # 每次只产出 1 个，不占内存

gen = generate_numbers(1000000)
print(next(gen))  # 0
print(next(gen))  # 1
# 虽然能产出 100 万个，但内存里只有当前这一个

# 生成器表达式（列表推导式的懒人版）
squares = (x**2 for x in range(1000000))  # 不占内存
squares_list = [x**2 for x in range(1000000)]  # 100 万个全在内存里
```

**和列表对比**：

```python
# 列表：100 万个数据，内存直接炸
big_list = [i for i in range(1000000)]  # ~40MB

# 生成器：还是那点内存
big_gen = (i for i in range(1000000))   # ~200 bytes
```

---

### 三、测试中的实际应用（结合你的项目）

#### 应用 1：读取大日志文件

```python
# ❌ 列表方式——百 MB 日志文件一口气读，内存爆炸
with open("/var/log/app/order.log") as f:
    lines = f.readlines()  # 全部加载到内存
    for line in lines:
        if "ERROR" in line:
            print(line)

# ✅ 生成器方式——文件对象本身就是迭代器，逐行读
with open("/var/log/app/order.log") as f:
    for line in f:  # 每次只读一行
        if "ERROR" in line:
            print(line)
```

> 线上查日志时，出库日志动不动几个 GB，用生成器逐行读不会撑爆内存。

#### 应用 2：批量用例生成（数据驱动）

```python
# 生成器函数——按需产出测试数据，不提前生成所有
def load_cases_from_yaml(filepath):
    """逐条读取 YAML 里的用例，不是一次性全加载"""
    data = yaml.safe_load(open(filepath))
    for case in data["cases"]:
        yield case  # 用一条产一条

# pytest parametrize 配合生成器
@pytest.mark.parametrize("case", load_cases_from_yaml("data/login.yaml"))
def test_login(case, api_client):
    resp = api_client.post("/auth/login", json=case["body"])
    assert case["expect"]["status"] == resp.status_code
```

#### 应用 3：批量接口调用，避免内存爆炸

```python
def generate_users(count):
    """按需生成用户数据——用了才产生，不是一次性生成 10000 个"""
    for i in range(count):
        yield {
            "username": f"test_user_{i}",
            "password": "123456"
        }

def batch_create_users(count):
    for user in generate_users(count):  # 用一条处理一条
        resp = requests.post("/user/create", json=user)
        assert resp.status_code == 200
```

#### 应用 4：数据库查询结果逐条处理

```python
# 生成器包装数据库查询——不把结果集全拉进内存
def query_orders_stream(db, date):
    """逐条返回订单，不一次性加载"""
    cursor = db.cursor()
    cursor.execute("SELECT id, amount FROM sale_order WHERE created_at = %s", date)
    while True:
        row = cursor.fetchone()
        if row is None:
            break
        yield {"id": row[0], "amount": row[1]}

# 处理 100 万条订单，内存不会炸
for order in query_orders_stream(db, "2026-06-21"):
    if order["amount"] > 10000:
        print(f"大额订单: {order['id']}")
```

---

### 四、面试回答模板

> **"迭代器就是可以逐个遍历的对象，生成器是用 yield 写出来的懒加载迭代器。测试中用生成器最多的是处理大数据——查线上日志、批量读取 YAML 用例、数据库查询结果集，全用生成器避免一次性加载撑爆内存。**
>
> **比如之前查出库日志，日志文件 5 个 G，用 `for line in file` 就是生成器方式逐行读，搜 ERROR 只占几 MB 内存。如果 `readlines()` 全部加载，内存直接炸。**
>
> **生成器表达式和列表推导式是同一个写法——一个用小括号一个用方括号，区别就是前者不占内存、后者全加载。"**

---

**关键词**：生成器 / 迭代器 / yield / 懒加载 / 大数据处理 / 内存优化

---

## Q12. 面向对象里哪两个方法最重要？（`__init__` / `__str__`）

### 问题描述

Python 的魔术方法（魔法方法）很多，面试官问你最重要的两个，一般是考你对面向对象的理解深度。

### 答案

最常用也最重要的两个：**`__init__`（构造）和 `__str__`（字符串表示）**。但如果结合测试框架，`__call__` 和 `__getattr__` 也极其重要。

---

### 一、`__init__` —— 对象初始化

**测试中的使用频率：⭐×100，每个类都要写。**

```python
# POM 的 Page 类 —— 初始化时传 driver
class OutboundPage:
    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

# API 客户端 —— 初始化时设 base_url
class APIClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.context = {}
        self.session = requests.Session()

# 关键字引擎 —— 初始化注册关键字
class KeywordEngine:
    def __init__(self):
        self.context = {}
        self.keywords = {
            "login": self._login,
            "query_stock": self._query_stock,
        }
```

---

### 二、`__str__` —— 打印时显示什么

```python
class Order:
    def __init__(self, order_id, amount, status):
        self.order_id = order_id
        self.amount = amount
        self.status = status

    # 不写 __str__ 时 print 出来是 <__main__.Order object at 0x7f...>
    def __str__(self):
        return f"订单 {self.order_id} | 金额: {self.amount} | 状态: {self.status}"

order = Order("ORD001", 100.00, "PAID")
print(order)  # 订单 ORD001 | 金额: 100.0 | 状态: PAID
```

**测试中的实用场景**：日志输出、调试打印、测试报告里的对象表示。

---

### 三、额外加分：`__call__` —— 让实例像函数一样调用

这就是你之前问的"装饰器"在底层用的：

```python
class RetryCaller:
    """重试执行器——实例可以像函数一样调用"""
    def __init__(self, max_retries=3):
        self.max_retries = max_retries

    def __call__(self, func, *args, **kwargs):
        for attempt in range(self.max_retries):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                print(f"重试 {attempt+1}...")

retry = RetryCaller(3)
# 实例直接当函数用
retry(lambda: requests.get("https://api.example.com/health"))
```

---

### 四、额外加分：`__getattr__` —— 动态属性拦截

这就是你在 Q4 写的关键字日志代理的核心：

```python
class LoggingProxy:
    def __getattr__(self, name):
        """调用不存在的属性时触发"""
        print(f">>> 调用: {name}")
        return getattr(self._target, name)
```

---

### 五、测试框架中常用的魔术方法速查

| 方法 | 触发时机 | 测试中哪里用 |
|:---|:---|:---|
| `__init__` | 创建对象 | Page 类、APIClient、引擎初始化 |
| `__str__` | `print(obj)` / `str(obj)` | 日志输出、调试打印 |
| `__call__` | `obj()` 当函数调 | 装饰器、重试器 |
| `__getattr__` | 访问不存在的属性 | 日志代理 |
| `__len__` | `len(obj)` | 自定义集合类 |
| `__eq__` | `obj1 == obj2` | 自定义对比逻辑 |

### 六、面试回答模板

> **"用得最多的是 `__init__`，每个类都要写，比如 Page 类初始化传 driver、API 客户端初始化设 base_url。第二个是 `__str__`，调试打印时让对象以可读方式展示，比如打印订单显示 'ORD001 | 100.00 | PAID' 而不是内存地址。**
>
> **另外我在测试框架里还用到了 `__call__` 和 `__getattr__`——`__call__` 做重试装饰器，`__getattr__` 做关键字调用日志代理，不侵入原有代码就能加日志。"**

---

**关键词**：魔术方法 / __init__ / __str__ / __call__ / __getattr__ / 面向对象

---

## Q13. `@classmethod` 和 `@staticmethod` 有什么区别？

### 答案

一句话：**classmethod 能拿到类本身（cls），staticmethod 什么都拿不到，就是个普通函数。**

```python
class APIClient:
    base_url = "https://test-api.example.com"

    # ✅ classmethod：第一个参数是 cls（类本身），能访问类属性
    @classmethod
    def get_base_url(cls):
        return cls.base_url

    # ✅ staticmethod：没 cls 也没 self，和独立函数一样
    @staticmethod
    def build_url(path):
        return f"https://test-api.example.com{path}"

    # 普通方法：第一个参数是 self（实例）
    def login(self, username, password):
        return requests.post(f"{self.base_url}/auth/login", json={...})


# 使用
print(APIClient.get_base_url())    # classmethod：不用创建实例
print(APIClient.build_url("/health"))  # staticmethod：不用创建实例

client = APIClient()
client.login("admin", "123456")    # 普通方法：需要实例
```

**核心区别**：

| | `@classmethod` | `@staticmethod` | 普通方法 |
|:---|:---:|:---:|:---:|
| 第一个参数 | `cls`（类） | 无 | `self`（实例） |
| 能访问类属性 | ✅ | ❌ | ✅（通过self） |
| 能访问实例属性 | ❌ | ❌ | ✅ |
| 不需要实例就能调 | ✅ | ✅ | ❌ |
| 什么时候用 | 工厂方法、类级别操作 | 工具函数放类里归类 | 常规业务方法 |

**测试中什么时候用**：

```python
# classmethod：YAML 数据加载器
class TestDataLoader:
    @classmethod
    def from_yaml(cls, filepath):
        """工厂方法——根据 YAML 文件创建实例"""
        data = yaml.safe_load(open(filepath))
        return cls(data)  # cls 就是 TestDataLoader

    @classmethod
    def from_env(cls, env_name):
        """根据环境名加载不同配置"""
        return cls(f"config/{env_name}.yaml")

# staticmethod：纯工具函数
class ResponseValidator:
    @staticmethod
    def is_success(status_code):
        return 200 <= status_code < 300

    @staticmethod
    def extract_order_id(text):
        return re.findall(r"ORD\d+", text)[0]
```

**一句话记**：有 `cls` 参数的是 classmethod，没参数的是 staticmethod，有 `self` 的是普通方法。

---

**关键词**：@classmethod / @staticmethod / cls / 工厂方法

---

## Q14. 你的测试框架里用了哪些设计模式？（结合项目说）

### 问题描述

面试官问你：你的测试框架里用了什么设计模式？考的是你**有没有架构意识**，不是会不会写单个函数。

### 答案

代码设计模式不是背概念，而是在框架里真正解决了问题。我的接口和 UI 自动化框架里，最核心的 5 个设计模式：

---

### 模式 1：POM（Page Object Model）—— UI 自动化的核心

**你项目里怎么用的**：

```
一个页面一个类 → OutboundPage、LoginPage、SaleOrderPage
测试用例只调方法 → 不碰 xpath
元素定位集中在 page 类 → 改一处全局生效
```

（详见 `ui-automation/README.md` Q1）

---

### 模式 2：装饰器模式 —— 给关键字自动加日志

**你项目里怎么用的**：

```python
# 之前 Q4 写的日志装饰器
@log_keyword_call
def _audit_outbound(self, order_id):
    ...  # 自动打印 ">>> 开始执行关键字: audit_outbound"

# 类装饰器 —— 一行搞定所有关键字
@auto_log_keywords
class KeywordEngine:
    # 所有 _xxx 方法自动加日志，不用每个方法写一遍
```

**解决的问题**：不在每个关键字里手动写 `print`，全局统一打日志。

---

### 模式 3：单例模式 —— WebDriver 全局只有一个

**你项目里怎么用的**：

```python
import pytest
from selenium import webdriver

_driver = None

def get_driver():
    """全局只有一个 driver 实例"""
    global _driver
    if _driver is None:
        _driver = webdriver.Chrome()
    return _driver

# 或者用 Pytest fixture scope="session"
@pytest.fixture(scope="session")
def driver():
    d = webdriver.Chrome()
    yield d
    d.quit()
```

**解决的问题**：测试套件只启动一次浏览器，所有用例共享，不反复开关。

---

### 模式 4：代理模式 —— 关键字日志代理

**你项目里怎么用的**：

```python
# 之前 Q4 写的日志代理
class LoggingProxy:
    def __init__(self, target):
        self._target = target

    def __getattr__(self, name):
        original = getattr(self._target, name)
        def wrapper(*args, **kwargs):
            print(f">>> 执行关键字: {name}")
            return original(*args, **kwargs)
        return wrapper

# 使用：引擎类零修改，包装一下就自动打日志
engine = LoggingProxy(KeywordEngine())
```

**解决的问题**：不侵入引擎代码就能加日志。

---

### 模式 5：工厂模式 —— 根据环境加载不同配置

**你项目里怎么用的**：

```python
class Config:
    @classmethod
    def from_env(cls, env_name):
        """工厂方法：根据环境名加载不同配置"""
        config = yaml.safe_load(open(f"config/{env_name}.yaml"))
        return cls(config["env"]["base_url"], config["env"]["username"])

# CI 里：Config.from_env("staging")
# 本地：Config.from_env("dev")
# 生产：Config.from_env("prod")
```

**解决的问题**：切环境改一行就行，不用改代码。

---

### 五模式速查

| 模式 | 你在哪用了 | 一句话 |
|:---|:---|:---|
| **POM** | UI 自动化 | 一个页面一个类，用例不碰 xpath |
| **装饰器** | 关键字自动日志 | `@auto_log_keywords` 一行搞定 |
| **单例** | WebDriver | 全测试只启动一次浏览器 |
| **代理** | 日志代理 | `LoggingProxy` 包装引擎，零侵入 |
| **工厂** | 环境配置加载 | `Config.from_env("staging")` |

### 面试回答模板

> **"我的测试框架里用了几个设计模式。UI 自动化用的是 POM——一个页面一个类，元素定位集中管理。接口自动化关键字引擎用了装饰器——@auto_log_keywords 自动给所有关键字方法加日志。WebDriver 用的单例——整个测试会话只启动一次浏览器。**
>
> **另外还有代理模式做日志拦截、工厂模式做多环境配置切换。这些都不是为了用模式而用，是真的解决了问题——日志不用每个方法写、切环境改一行、driver 不反复开关。"**

---

**关键词**：设计模式 / POM / 装饰器 / 单例 / 代理 / 工厂模式 / 测试架构

---

## Q15. 怎么用 Python 实现单例模式？（结合测试场景）

### 问题描述

单例模式的核心是：**整个程序运行期间，一个类只有一个实例。** 测试中最典型的场景是 WebDriver——不该每次打开新浏览器。Python 有 4 种实现方式。

### 方式一：`__new__` 控制（最经典）

```python
class WebDriverSingleton:
    _instance = None  # 类变量，存唯一实例

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.driver = webdriver.Chrome()  # 只启动一次
        return cls._instance

# 验证
d1 = WebDriverSingleton()
d2 = WebDriverSingleton()
print(d1 is d2)        # True —— 同一个对象
print(d1.driver is d2.driver)  # True —— 同一个浏览器
```

**`__new__` vs `__init__`**：
- `__new__`：创建对象时最先调用，控制**要不要真的创建新对象**
- `__init__`：对象创建之后调用，初始化属性
- 单例模式在 `__new__` 里拦截，不让重复创建

---

### 方式二：模块级别（最简单，最推荐）

Python 模块本身就是单例——一个文件只被导入一次：

```python
# singleton_driver.py —— 模块方式
from selenium import webdriver

driver = None

def get_driver():
    global driver
    if driver is None:
        driver = webdriver.Chrome()
    return driver

# 其他地方用
from singleton_driver import get_driver
driver = get_driver()  # 全局永远只创建一次
```

**优点**：最简单的单例，不需要任何 `__new__` 的魔法。

---

### 方式三：装饰器（优雅）

```python
def singleton(cls):
    """类装饰器：把类变成单例"""
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class Config:
    def __init__(self, env="dev"):
        self.env = env
        self.url = f"https://{env}-api.example.com"

# 验证
c1 = Config("staging")
c2 = Config("prod")
print(c1 is c2)           # True —— 同一个
print(c1.env)             # "staging" —— 第一次创建的参数生效
```

**测试场景**：全局配置只加载一次，不会因为重复导入而覆盖。

---

### 方式四：元类（最"高级"）

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class APIClient(metaclass=SingletonMeta):
    def __init__(self, base_url):
        self.base_url = base_url
        self.token = login(base_url)

# 验证
c1 = APIClient("https://test.example.com")
c2 = APIClient("https://prod.example.com")  # 第二次创建无效
print(c1 is c2)          # True
print(c1.base_url)       # "https://test.example.com"
```

---

### 测试中的实用场景

```python
# 场景 1：WebDriver 全局只有一个
driver1 = WebDriverSingleton().driver
driver2 = WebDriverSingleton().driver
# 全测试套件只启动一次 Chrome，不反复开关

# 场景 2：配置只加载一次
@singleton
class Config:
    def __init__(self):
        self.config = yaml.safe_load(open("config/config.yaml"))

config1 = Config()
config2 = Config()
# 同一个实例，不会重复读文件

# 场景 3：API 客户端复用
client1 = APIClient("https://test-api.example.com")
client2 = APIClient("https://test-api.example.com")
# 同一个 client，token 复用，不会重复登录
```

---

### 版别速查

| 方式 | 难度 | 推荐度 | 什么时候用 |
|:---|:---:|:---:|:---|
| **模块级别** | ⭐ | ⭐⭐⭐⭐⭐ | 最推荐，简单可靠 |
| **`__new__`** | ⭐⭐ | ⭐⭐⭐⭐ | 面试常考，经典写法 |
| **装饰器** | ⭐⭐ | ⭐⭐⭐⭐ | 复用性强，装饰任意类 |
| **元类** | ⭐⭐⭐ | ⭐⭐ | 框架级需求，平时少用 |

### 面试回答模板

> **"Python 实现单例最简单的是模块级别——一个 .py 文件只导入一次，全局变量自然就是单例。面试常考的是 `__new__` 方式——在类变量 `_instance` 存唯一实例，`__new__` 里判断已经创建了就返回旧的。**
>
> **测试框架里我用得最多的是 WebDriver 的单例——整个测试会话只启动一次浏览器，所有用例共享同一个 driver。装饰器方式也很实用——给 Config 类加 @singleton，配置文件只加载一次。"**

---

**关键词**：单例模式 / __new__ / 装饰器 / 元类 / WebDriver 单例 / 模块单例

---

## Q16. 说说 fixture 的作用域有哪些？结合 ERP 项目举例

### 问题描述

Pytest 的 fixture 有 5 个作用域（scope）。面试官问你：你知道 fixture 的 scope 是干什么的吗？你在 ERP 项目里分别怎么用的？

### 答案

fixture 的 scope 决定了**这个 fixture 在什么范围内复用**。它是 fixture 最强大的地方——同一个 API token 可以让整个测试会话复用，不用每条用例都登录一次。

---

### 一、5 种 scope 一览

| scope | 执行频率 | 什么时候用 | ERP 中的例子 |
|:---|:---|:---|:---|
| **function** | 每条用例前后 | 默认 | 清空上下文、重置测试数据 |
| **class** | 每个测试类前后 | 同一类用例共享 | 一组销售用例共享预置订单 |
| **module** | 每个文件前后 | 模块级初始化 | 销售模块统一初始化商品 |
| **package** | 每个目录前后 | 7.0+新增 | 财务目录统一初始化 |
| **session** | 整个测试会话一次 | 全局复用 | **登录 token、driver、config** |

---

### 二、结合 ERP 项目逐个讲

#### session（整个测试跑一次）—— 用得最多

```python
# conftest.py —— tests/ 根目录，全局生效
import pytest
import requests

@pytest.fixture(scope="session")
def api_token():
    """整个测试会话只登录一次，所有用例复用同一个 token"""
    resp = requests.post("https://api.example.com/auth/login", json={
        "username": "admin", "password": "123456"
    })
    token = resp.json()["token"]
    print("\n[全局] token 已获取")
    return token

@pytest.fixture(scope="session")
def api_client(api_token):
    """APIClient 也只创建一次，所有用例共用"""
    client = APIClient("https://api.example.com")
    client.token = api_token
    return client
```

**ERP 场景**：90 条回归用例跑完，只登录一次。如果每条用例都登录就是 90 次登录，session 级直接把登录次数砍到 1。

#### module（每个文件跑一次）—— 模块级初始化

```python
# tests/sale/conftest.py —— 只对 tests/sale/ 目录生效
@pytest.fixture(scope="module")
def init_sale_test_data(api_client):
    """销售模块的所有用例跑之前，初始化一批商品数据"""
    api_client.post("/testdata/sale/init", json={
        "skus": ["SKU001", "SKU002", "SKU003"]
    })
    print("\n[销售模块] 测试数据已初始化")
    yield
    api_client.post("/testdata/sale/cleanup")
    print("\n[销售模块] 测试数据已清理")
```

**ERP 场景**：销售模块的所有用例共用同一批商品数据，不需要在每个测试文件的 fixture 里重复创建。

#### class（每个测试类跑一次）—— 共享前置

```python
class TestSaleOrder:
    """一整个测试类的用例共用一个预置订单"""

    @pytest.fixture(scope="class")
    def prepared_order(self, api_client):
        """创建一条订单，该类所有用例共用"""
        resp = api_client.post("/sale/order/create", json={
            "sku": "SKU001", "quantity": 100
        })
        order_id = resp.json()["id"]
        print(f"\n[TestSaleOrder] 预置订单: {order_id}")
        return order_id

    def test_audit_order(self, prepared_order):
        """用同一个订单测审核"""
        api_client.post(f"/sale/order/{prepared_order}/audit")

    def test_cancel_order(self, prepared_order):
        """用同一个订单测取消"""
        api_client.post(f"/sale/order/{prepared_order}/cancel")
```

**ERP 场景**：一个测试类里的所有用例操作同一个订单，不需要每条用例都创建新订单。

#### function（每条用例一次）—— 默认，每条用例隔离

```python
@pytest.fixture(scope="function")  # function 是默认值，可以不写
def clean_context():
    """每条用例跑完清空上下文，避免互相污染"""
    yield
    context.clear()
    print("[每条用例] 上下文已清空")
```

**ERP 场景**：并发测试时，每条用例的 context 独立，避免 order_id 被上一条用例污染。

---

### 三、scope 递进关系

```
session（整个测试） → 1 次
  └── package（每个包）→ 1-N 次
       └── module（每个文件）→ 1-N 次
            └── class（每个类）→ 1-N 次
                 └── function（每条用例）→ N 次
```

**实际效果**（ERP 项目跑回归时）：

```
pytest tests/ -v

[session] 登录获取 token        ← 只执行 1 次
[module]  销售模块初始化商品     ← 销售模块的文件执行 1 次
[class]   预置订单 ORD001       ← TestSaleOrder 类执行 1 次
  test_audit_order PASSED      ← function 级 fixture 执行
  test_cancel_order PASSED     ← function 级 fixture 执行
[module]  采购模块初始化商品     ← 采购模块的文件执行 1 次
  test_purchase PASSED
[session] 清理                  ← 整个测试结束，只执行 1 次
```

---

### 四、面试回答模板

> **"fixture 的 scope 我主要是根据复用需求来选的。用得最多的是 session——全局登录只做一次，所有用例复用同一个 token，接口回归 90 条用例就跑一次登录。**
>
> **module 用于模块级数据初始化——销售模块所有用例共用一批测试商品，跑完统一清理。class 用于一个测试类里多条用例操作同一个对象，比如创建一条订单、审核、取消全用同一条。function 是默认值，用在上一条和下一条用例的数据完全独立、不能互串的场景。**
>
> **scope 用得对的话，把不必要的重复操作砍掉，测试效率能翻好几倍。"**

---

**关键词**：fixture scope / session / module / class / function / 测试效率

---

## Q17. YAML 测试用例的字段怎么设计？结合 ERP 项目

### 问题描述

你用 YAML 管理测试数据，那你 YAML 文件里一般有哪些字段？为什么这样设计？

### 答案

YAML 用例的字段不是随便定的，核心原则是：**入参、断言、描述三位一体**，读一条 YAML 就能看懂一条用例。

---

### 一、三条铁律

```text
1. 每条用例必须有描述——半年后回来还能看懂
2. 入参和断言必须分开放——body 和 expect 搞混了很难维护
3. 关联校验字段独立——单接口断言和跨模块对账分开
```

**错误示例**：

```yaml
# ❌ 没有描述、入参和断言揉在一起
cases:
  - login: admin
    password: "123456"
    msg: 登录成功
    code: 200
```

半年后你再看：msg 是入参还是预期的结果？code 是传的还是返回的？完全搞不清。

**正确示例**：

```yaml
# ✅ 描述 + body + expect 三段式
cases:
  - desc: "管理员正常登录"
    body:
      username: "admin"
      password: "123456"
    expect:
      status: 200
      msg: "登录成功"
```

---

### 二、ERP 项目里 YAML 的实战结构

#### 结构 1：单接口多条件用例

```yaml
# data/login.yaml —— 登录接口的多组入参覆盖
cases:
  - desc: "管理员登录"
    body:
      username: "admin"
      password: "123456"
    expect:
      status: 200
      msg: "登录成功"

  - desc: "密码错误"
    body:
      username: "admin"
      password: "wrong"
    expect:
      status: 401
      msg: "密码错误"

  - desc: "用户名为空"
    body:
      username: ""
      password: "123456"
    expect:
      status: 400
      msg: "用户名不能为空"

  - desc: "SQL注入尝试"
    body:
      username: "admin' OR '1'='1"
      password: "123456"
    expect:
      status: 401
      msg: "用户名或密码错误"
```

```python
# pytest 驱动
@pytest.mark.parametrize("case", yaml.safe_load(open("data/login.yaml"))["cases"])
def test_login(case, api_client):
    resp = api_client.post("/auth/login", json=case["body"])
    assert case["expect"]["status"] == resp.status_code
    assert case["expect"]["msg"] in resp.text
```

#### 结构 2：业务链路（多接口联动）

```yaml
# data/sale_flow.yaml —— 销售改价全链路
scenarios:
  - desc: "改价后库存成本同步校验"
    steps:
      - keyword: login
        body:
          username: "admin"
          password: "123456"
        save_as: login_result

      - keyword: query_stock
        body:
          sku: "SKU001"
        save_as: stock_before

      - keyword: update_price
        body:
          sku: "SKU001"
          new_price: 99.00
        save_as: price_result

      - keyword: query_stock
        body:
          sku: "SKU001"
        save_as: stock_after

      - keyword: reconcile  # 对账：改价前后成本必须一致
        expect:
          stock_before_cost: "{stock_before.data.cost}"
          stock_after_cost: "{stock_after.data.cost}"
          price_changed: true
```

#### 结构 3：对账校验用例

```yaml
# data/reconcile.yaml —— 数据一致性对账
reconcile_cases:
  - desc: "出库单金额 = 凭证金额"
    rule: "outbound_vs_voucher"
    query:
      date: "2026-06-21"
    expect:
      match: true
      tolerance: 0.01  # 允许 1 分钱的浮点误差

  - desc: "调拨后两仓库存总和不变"
    rule: "transfer_balance"
    query:
      transfer_id: 1001
    expect:
      before_total: "{before.total}"
      after_total: "{after.total}"
      match: true
```

---

### 三、字段规范速查

| 字段 | 作用 | 必填 | 示例 |
|:---|:---|:---:|:---|
| `desc` | 用例描述，人读 | ✅ | `"管理员登录"` |
| `body` | 接口入参 | ✅ | `username: "admin"` |
| `expect` | 期望结果（单接口断言） | ✅ | `status: 200` |
| `expect_msg` | 期望的响应文本 | ❌ | `"登录成功"` |
| `save_as` | 链路中存上下文 | ❌ | `save_as: login_result` |
| `rule` | 对账规则名 | ❌ | `"outbound_vs_voucher"` |
| `tolerance` | 浮点误差容忍 | ❌ | `0.01` |
| `skip` | 是否跳过此条 | ❌ | `skip: true` |

---

### 四、接口 vs 链路的两套结构

```yaml
# 单接口用例结构（数据驱动）
cases:
  - desc: "xxx"
    body: {...}       # 接口入参
    expect: {...}     # 期望返回

# 业务链路结构（关键字驱动）
scenarios:
  - desc: "xxx"
    steps:
      - keyword: xxx
        body: {...}
        save_as: xxx  # 上下文存起来
        expect: {...} # 每步也能断言
```

### 五、面试回答模板

> **"我设计 YAML 用例有三个核心字段：desc（人读的描述）、body（接口入参）、expect（断言条件）。body 和 expect 严格分开——入参只管传什么，断言只管结果对不对，防止搞混。**
>
> **单接口用例就是标准的三段式——desc + body + expect，pytest parametrize 直接驱动。业务链路还会额外加 save_as 和 keyword，每一步存一个上下文变量，后面步骤用 {xxx} 引用。**
>
> **ERP 里还有专门的对账用例——rule 字段写对账规则名，tolerance 处理一分钱的浮点误差。"**

---

**关键词**：YAML / 用例字段 / desc+body+expect / 三段式 / 对账 / 数据驱动

---

## Q18. YAML 导入 Pytest 卡顿怎么处理？

### 问题描述

你的 YAML 文件里几百条用例，每次 pytest 收集阶段都要读一遍 YAML，几十个文件每个都打开一遍，收集就卡了半分钟。怎么优化？

### 答案

卡顿的原因：**pytest 收集阶段会执行所有 `parametrize` 的装饰器参数，每次 `yaml.safe_load` 都在重复读文件。**

---

### 方案一：session 级 fixture 缓存（最有效）

```python
# ❌ 卡顿写法——每个 parametrize 都读一次文件
@pytest.mark.parametrize("case", yaml.safe_load(open("data/login.yaml"))["cases"])
def test_login(case, api_client):
    ...

@pytest.mark.parametrize("case", yaml.safe_load(open("data/order.yaml"))["cases"])
def test_order(case, api_client):
    ...
# 10 个文件 × 500 条用例 → 收集阶段卡 30 秒

# ✅ 优化写法——conftest.py 里全局只加载一次
# conftest.py
@pytest.fixture(scope="session")
def test_data():
    """session 级缓存：所有 YAML 只读一次"""
    data = {}
    data["login"] = yaml.safe_load(open("data/login.yaml"))
    data["order"] = yaml.safe_load(open("data/order.yaml"))
    data["reconcile"] = yaml.safe_load(open("data/reconcile.yaml"))
    return data  # 整个测试会话只读一次，不在收集阶段读

# test_login.py
def test_login(test_data):
    for case in test_data["login"]["cases"]:
        resp = api_client.post("/auth/login", json=case["body"])
        assert case["expect"]["status"] == resp.status_code
```

**但这样就不能用 parametrize 了**，每条用例通过 `for` 循环跑。有得有失。

---

### 方案二：用 `ids` 参数避免 Pytest 生成名称时卡

parametrize 卡很多是因为 Pytest 需要给大量用例生成名称：

```python
# ✅ 手动指定 ids —— 减少 Pytest 的计算量
cases = yaml.safe_load(open("data/login.yaml"))["cases"]

@pytest.mark.parametrize("case", cases,
    ids=[c["desc"] for c in cases]  # 用 desc 做用例名，不用再解析
)
def test_login(case, api_client):
    ...
```

---

### 方案三：多个小文件替代一个大文件

```yaml
# ❌ 一个文件几百条，parametrize 时全加载
# data/all_cases.yaml — 500 条

# ✅ 按模块拆成小文件
# data/login_success.yaml     — 10 条
# data/login_fail.yaml        — 8 条
# data/order_create.yaml      — 15 条
```

小文件被 parametrize 时只加载当前文件的数据，减少无效加载。

---

### 方案四：用 fixture 参数化替代 parametrize

```python
# conftest.py
def pytest_generate_tests(metafunc):
    """动态生成参数——按需加载 YAML"""
    if "login_case" in metafunc.fixturenames:
        # 只在需要时才加载
        cases = yaml.safe_load(open("data/login.yaml"))["cases"]
        metafunc.parametrize("login_case", cases, ids=[c["desc"] for c in cases])

def test_login(login_case, api_client):
    resp = api_client.post("/auth/login", json=login_case["body"])
    assert login_case["expect"]["status"] == resp.status_code
```

**优点**：按需加载，不会在收集阶段读取所有文件。

---

### 方案五：`@pytest.fixture` + 直接循环（最快）

卡顿最严重的场景下，这是终极方案：

```python
# conftest.py
@pytest.fixture(scope="session")
def login_cases():
    return yaml.safe_load(open("data/login.yaml"))["cases"]

# test_login.py —— 弃用 parametrize，改用 for 循环
def test_login_suite(login_cases, api_client):
    """批量跑——收集阶段零卡顿"""
    failed = []
    for case in login_cases:
        resp = api_client.post("/auth/login", json=case["body"])
        if resp.status_code != case["expect"]["status"]:
            failed.append(f"{case['desc']}: 期望{case['expect']['status']} 实际{resp.status_code}")
    assert not failed, "\n".join(failed)
```

**优点**：收集阶段零开销，Pytest 瞬间开始执行
**缺点**：一个用例挂了后续还会接着跑（parametrize 挂了就停了），日志不好区分

---

### 方案速查

| 方案 | 收集速度 | 用例独立性 | 报告清晰度 | 推荐场景 |
|:---|:---:|:---:|:---:|:---|
| session fixture 缓存 | ⭐⭐⭐⭐⭐ | ❌ 一个函数全跑 | ⭐⭐ | 用例多、不在乎独立报告 |
| 大文件拆小 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 最推荐，两者兼顾 |
| `pytest_generate_tests` | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 高级用法 |
| fixture + for 循环 | ⭐⭐⭐⭐⭐ | ❌ | ⭐⭐ | 回归跑得快就行 |

### 面试回答模板

> **"YAML 用例多了 parametrize 会卡在收集阶段，我的优化手段有三层。第一层，一个 500 条的大 YAML 拆成 10 个小文件——login_success、login_fail 等，加载只加载当前文件的数据。第二层，conftest.py 里用 session 级 fixture 缓存 YAML 对象，整个测试会话只读一次。第三层，如果有极慢的场景直接用 fixture + for 循环替代 parametrize，收集阶段零开销。**
>
> **最推荐的是拆小文件——既能保持 parametrize 的独立用例报告，又不会因为全量加载卡收集。"**

---

**关键词**：YAML 卡顿 / parametrize 性能 / session 缓存 / 拆小文件 / pytest_generate_tests

---

## Q19. 接口自动化里怎么区分单接口用例和业务用例？

### 问题描述

接口自动化里有两类用例：一类是测单个接口的多种情况，一类是测多个接口串联的业务链路。面试官问你怎么区分管理，考的是**框架组织能力**。

### 答案

单接口用例和业务用例从组织到执行都不一样，混在一起很难维护。我的做法是**目录分离 + YAML 结构分离 + 运行策略分离**。

---

### 一、目录分离

```
automation/
├── tests/
│   ├── api/              ← 单接口用例（数据驱动）
│   │   ├── test_login.py
│   │   ├── test_order_create.py
│   │   ├── test_price_update.py
│   │   └── test_outbound_audit.py
│   │
│   └── biz/              ← 业务链路用例（关键字驱动）
│       ├── test_sale_flow.py       # 销售全链路
│       ├── test_purchase_flow.py   # 采购全链路
│       ├── test_cancel_rollback.py # 作废回滚链路
│       └── test_reconcile.py       # 对账链路
│
├── data/
│   ├── api/              ← 单接口 YAML
│   │   ├── login.yaml
│   │   └── order_create.yaml
│   │
│   └── biz/              ← 业务链路 YAML
│       ├── sale_flow.yaml
│       └── cancel_rollback.yaml
│
└── conftest.py
```

---

### 二、YAML 结构分离

**单接口 YAML**：入参多条件覆盖

```yaml
# data/api/login.yaml
cases:
  - desc: "正确密码"
    body: {username: "admin", password: "123456"}
    expect: {status: 200, msg: "登录成功"}

  - desc: "错误密码"
    body: {username: "admin", password: "wrong"}
    expect: {status: 401, msg: "密码错误"}
```

**业务链路 YAML**：步骤串联 + 上下文传递

```yaml
# data/biz/sale_flow.yaml
scenarios:
  - desc: "销售改价 → 成本同步校验"
    steps:
      - keyword: login
        body: {username: "admin", password: "123456"}
        save_as: login_result

      - keyword: create_order
        body: {sku: "SKU001", quantity: 100}
        save_as: order_result

      - keyword: audit_outbound
        body: {order_id: "{order_result.data.id}"}

      - keyword: verify_voucher
        body: {order_id: "{order_result.data.id}"}
        expect: {amount_match: true}
```

---

### 三、代码写法分离

**单接口用例**：Pytest parametrize 数据驱动

```python
# tests/api/test_login.py
import pytest
import yaml

@pytest.mark.parametrize("case",
    yaml.safe_load(open("data/api/login.yaml"))["cases"],
    ids=lambda c: c["desc"]
)
def test_login(case, api_client):
    resp = api_client.post("/auth/login", json=case["body"])
    assert case["expect"]["status"] == resp.status_code
    assert case["expect"]["msg"] in resp.text
```

**业务链路用例**：关键字引擎驱动

```python
# tests/biz/test_sale_flow.py
import yaml

def test_sale_price_flow(api_client):
    """销售改价全链路——用引擎跑"""
    scenarios = yaml.safe_load(open("data/biz/sale_flow.yaml"))["scenarios"]

    engine = KeywordEngine(api_client)
    for scenario in scenarios:
        print(f"\n>> 执行场景: {scenario['desc']}")
        engine.run(scenario["steps"])
```

---

### 四、运行策略分离

```bash
# 开发自测/PR 提交：只跑单接口（快，5 秒）
pytest tests/api/ -v

# 版本上线前：跑业务链路（核心流程验证，2 分钟）
pytest tests/biz/ -v

# 全量回归：两个都跑
pytest tests/api/ tests/biz/ -v

# 按标记区分
pytest tests/ -m "smoke"         # P0 冒烟
pytest tests/ -m "api"           # 只跑单接口
pytest tests/ -m "business"      # 只跑业务链路
```

---

### 五、对比速查

| | 单接口用例 | 业务链路用例 |
|:---|:---|:---|
| **测什么** | 一个接口的多种输入 | 多个接口的业务流程 |
| **数据管理** | `data/api/xxx.yaml` | `data/biz/xxx.yaml` |
| **驱动方式** | parametrize 数据驱动 | 关键字引擎 |
| **YAML 结构** | `desc + body + expect` | `steps → keyword + body + save_as` |
| **运行速度** | 快（5-30 秒） | 慢（1-3 分钟） |
| **什么时候跑** | 开发自测、PR 提交 | 版本上线前 |
| **谁维护** | 测试 | 测试 + 业务可参与 |
| **ERP 例子** | 登录 15 组参数 | 销售全链路 4 接口串联 |

### 六、面试回答模板

> **"单接口和业务用例我严格分开管理。目录上 tests/api 放单接口、tests/biz 放业务链路。数据上 data/api 和 data/biz 两个目录，YAML 结构也不一样——单接口是 desc+body+expect 三段式，业务链路是 steps 串联表加 save_as 上下文传递。**
>
> **代码层面单接口用 parametrize 数据驱动，业务链路用关键字引擎驱动。运行层面 PR 提交只跑单接口（快），版本上线前跑业务链路（核心流程验证）。分清楚之后，团队能并行工作——测试写单接口用例，业务写链路用的 YAML。"**

---

**关键词**：单接口用例 / 业务链路 / 目录分离 / 数据驱动 / 关键字驱动 / 运行策略

---

## Q20. 接口自动化的断言方法有哪些？结合 ERP 项目举例

### 问题描述

接口返回的东西怎么判断对不对？你不能只 `assert resp.status_code == 200` 就下班了。面试官想看你的**断言维度够不够全面**。

### 答案

ERP 接口的断言分 5 个维度，从基础到深入：

> **状态码 → 响应字段 → 数据一致性（对账）→ 业务规则 → 性能/结构**

---

### 一、状态码断言（最基础）

```python
# 基础——200 不一定是成功，500 一定有错
assert resp.status_code == 200

# 异常场景——必须返回对应的状态码
assert resp.status_code == 400   # 参数错误
assert resp.status_code == 401   # 未登录
assert resp.status_code == 403   # 无权限
assert resp.status_code == 422   # 业务校验失败
```

---

### 二、响应字段断言

```python
import json

# 响应整体解析
data = resp.json()

# 单字段精确断言
assert data["order_status"] == "PAID"
assert data["amount"] == 100.00
assert data["warehouse_id"] == "WH001"

# 存在性断言
assert "token" in data, "登录后必须有 token"
assert data.get("voucher"), "出库后必须生成凭证"

# 空值断言
assert data["error_msg"] is not None, "报错时必须返回错误信息"
assert len(data["items"]) > 0, "列表不能为空"

# 类型断言
assert isinstance(data["amount"], (int, float)), "金额必须是数字"
assert isinstance(data["items"], list), "列表必须是数组"
```

---

### 三、多接口数据一致性断言（对账）—— ERP 专有

这是 ERP 测试中最重要也最独特的断言——**跨接口、跨模块的数据对得齐**：

```python
def test_outbound_voucher_consistency(api_client):
    """出库后凭证金额是否准确"""

    # 1. 查出库单金额
    outbound = api_client.get("/wms/outbound/1001").json()
    outbound_amount = outbound["total_amount"]

    # 2. 查关联的凭证金额
    voucher = api_client.get("/finance/voucher/query", params={
        "biz_type": "outbound",
        "biz_id": 1001
    }).json()

    # 3. 一致性断言
    assert len(voucher["items"]) == 1, "一笔出库应生成一条凭证"
    assert abs(outbound_amount - voucher["items"][0]["amount"]) < 0.01, \
        f"对账不平！出库={outbound_amount} 凭证={voucher['items'][0]['amount']}"
```

```python
def test_cancel_restore_stock(api_client):
    """作废销售单后库存是否恢复"""

    # 1. 操作前库存
    before = api_client.get("/wms/stock/query", params={"sku": "SKU001"}).json()
    stock_before = before["quantity"]

    # 2. 创建销售单 → 审核（扣库存） → 作废
    order = api_client.post("/sale/order/create", json={"sku": "SKU001", "quantity": 10}).json()
    api_client.post(f"/sale/order/{order['id']}/audit")
    api_client.post(f"/sale/order/{order['id']}/cancel")

    # 3. 操作后库存
    after = api_client.get("/wms/stock/query", params={"sku": "SKU001"}).json()

    # 4. 回滚断言
    assert after["quantity"] == stock_before, \
        f"作废后库存未恢复！操作前={stock_before} 操作后={after['quantity']}"
```

```python
def test_transfer_balance(api_client):
    """调拨后两仓库存总和不变"""

    # 操作前两仓总和
    wh_a = api_client.get("/wms/stock/query", params={"warehouse": "A"}).json()
    wh_b = api_client.get("/wms/stock/query", params={"warehouse": "B"}).json()
    total_before = sum(item["quantity"] for item in wh_a["items"] + wh_b["items"])

    # 调拨操作
    api_client.post("/wms/transfer/audit", json={"from": "A", "to": "B", "sku": "SKU001", "quantity": 50})

    # 操作后两仓总和
    wh_a2 = api_client.get("/wms/stock/query", params={"warehouse": "A"}).json()
    wh_b2 = api_client.get("/wms/stock/query", params={"warehouse": "B"}).json()
    total_after = sum(item["quantity"] for item in wh_a2["items"] + wh_b2["items"])

    assert total_after == total_before, \
        f"调拨后库存不守恒！操作前={total_before} 操作后={total_after}"
```

---

### 四、业务规则断言

```python
# 库存不能为负
assert data["quantity"] >= 0, f"库存不能为负: {data['quantity']}"

# 审核后状态必须变更
assert data["status"] != "PENDING", "审核后状态不应仍为待审核"

# 单据号必须唯一
order_ids = [r.json()["order_no"] for r in results]
assert len(order_ids) == len(set(order_ids)), "存在重复订单号"

# 金额精度——财务必须精确到分
assert round(data["amount"], 2) == data["amount"], "金额精度不足"

# 分页参数
assert len(data["items"]) <= page_size, "返回条数超过分页限制"
assert data["total"] == 50, "总数不对"
```

---

### 五、进阶断言

```python
# 时间断言——凭证生成时间应在出库审核之后
from datetime import datetime
assert datetime.fromisoformat(voucher["created_at"]) > \
       datetime.fromisoformat(outbound["audited_at"]), \
       "凭证生成时间应在出库审核之后"

# 部分匹配（字段太多，只关心核心）
def test_order_basic(api_client):
    resp = api_client.get("/sale/order/1001").json()
    # 只断言核心字段，忽略时间戳、版本号
    assert resp["status"] == "PAID"
    assert resp["amount"] == 100.00
    assert resp["sku"] == "SKU001"

# 模糊匹配
assert "审核通过" in resp.text
assert "操作成功" in resp.text or resp.status_code == 200
```

---

### 六、断言方法速查

| 层级 | 断言什么 | ERP 场景 |
|:---|:---|:---|
| 状态码 | `== 200` / `== 401` | 登录失败必须 401 |
| 字段值 | `data["status"] == "PAID"` | 付款后状态变换 |
| 存在性 | `"token" in data` | 登录必须有 token |
| **对账** | `abs(a-b) < 0.01` | **出库 vs 凭证金额** |
| 守恒 | `total_after == total_before` | 调拨后库存不丢失 |
| 业务规则 | `quantity >= 0` | 库存不能为负 |
| 时间 | 凭证时间 > 审核时间 | 时序逻辑 |
| 模糊匹配 | `"操作成功" in resp.text` | 不带数据断言的场景 |

### 七、面试回答模板

> **"ERP 的接口断言分五个维度。基础的是状态码和字段断言——200、status 对不对。中间是存在性和空值——token 有没有、错误信息给没给。**
>
> **最核心的是多接口数据一致性断言，也就是对账。比如出库审核后，我不仅断言出库接口返回 200，还跨接口查出库单金额和凭证金额，用 `abs()` 校验误差小于 0.01，保证财务数据的一致性。还有调拨后的库存守恒断言，A 仓扣了 B 仓必须加回来。**
>
> **这些对账断言才是 ERP 测试的核心价值——不是看接口通没通，是看数据对没对齐。**"

---

**关键词**：接口断言 / 对账断言 / 数据一致性 / 跨接口校验 / abs / assert

---

## Q21. 接口之间强依赖怎么处理？（比如出库审核依赖订单创建）

### 问题描述

接口不是孤立的——出库审核需要 order_id，凭证查询需要 outbound_id。一个接口挂了，后面全挂。这种强依赖在自动化里怎么处理？

### 答案

ERP 场景里接口强依赖是常态。处理方式分**正常链路**和**异常兜底**两层。

---

### 一、正常链路：fixture 依赖链（最常用）

Pytest 自动检测 fixture 依赖关系，按依赖顺序执行：

```python
# conftest.py —— 依赖链：token → order → outbound

@pytest.fixture(scope="function")
def token(api_client):
    """第 1 步：登录拿 token"""
    resp = api_client.post("/auth/login", json={"username":"admin","password":"123456"})
    assert resp.status_code == 200, "❌ 登录失败，终止后续"
    return resp.json()["token"]

@pytest.fixture(scope="function")
def order_id(api_client, token):  # 依赖 token
    """第 2 步：创建订单，依赖 token"""
    resp = api_client.post("/order/create",
        json={"sku": "SKU001", "quantity": 100},
        headers={"Authorization": f"Bearer {token}"}
    )
    assert resp.status_code == 200, "❌ 创建订单失败，终止后续"
    return resp.json()["id"]

@pytest.fixture(scope="function")
def outbound_id(api_client, token, order_id):  # 依赖 token + order_id
    """第 3 步：生成出库单"""
    resp = api_client.post("/wms/outbound/create",
        json={"order_id": order_id},
        headers={"Authorization": f"Bearer {token}"}
    )
    assert resp.status_code == 200, "❌ 创建出库单失败"
    return resp.json()["id"]
```

**效果**：token 失败 → order 和 outbound 直接跳过，不会连锁挂一堆。

---

### 二、异常兜底：依赖失败时优雅跳过

```python
import pytest

@pytest.fixture(scope="function")
def order_id_safe(api_client, token):
    """安全版 fixture：依赖失败时跳过而非报错"""
    resp = api_client.post("/order/create", json={"sku": "SKU001", "quantity": 100})
    if resp.status_code != 200:
        pytest.skip(f"前置接口不可用，跳过后续测试: {resp.text}")
    return resp.json()["id"]

def test_audit_outbound(order_id_safe):
    # 如果 order_id_safe 跳过了，这里不会执行
    ...
```

---

### 三、关键字引擎中的强依赖：前一步失败后面不跑

```python
class KeywordEngine:
    def run(self, test_steps):
        for step in test_steps:
            keyword = step["keyword"]
            params = self._resolve_params(step.get("params", {}))
            try:
                result = self.keywords[keyword](**params)
                if "save_as" in step:
                    self.context[step["save_as"]] = result
            except Exception as e:
                print(f"❌ 步骤 [{keyword}] 失败: {e}")
                break  # ← 关键：前一步失败，后面全部跳过
```

```yaml
# 链路：登录 → 创建订单 → 出库审核 → 凭证校验
# 如果创建订单失败，出库审核和凭证校验直接跳过
```

---

### 四、弱化依赖：用预置数据替代实时创建

最彻底的方案——不让自动化用例实时创建依赖数据：

```python
# conftest.py —— 预置好测试数据
@pytest.fixture(scope="module")
def prepared_order(api_client):
    """模块级预置：提前创建一批订单，所有用例直接用"""
    orders = []
    for i in range(10):
        resp = api_client.post("/order/create", json={"sku": f"SKU{i:03d}", "quantity": 100})
        orders.append(resp.json()["id"])
    return orders

def test_audit_first_order(prepared_order):
    """直接用预置订单，不依赖创建接口"""
    order = prepared_order[0]  # 已经创建好了
    resp = api_client.post(f"/wms/outbound/{order}/audit")
    assert resp.status_code == 200
```

**优点**：即使创建订单接口挂了，出库审核测试不受影响
**缺点**：依赖数据预置的环境，切环境时需要重新预置

---

### 五、Mock 替代依赖（极端场景）

```python
# 当依赖的接口确实不可用时，mock 掉它
from unittest.mock import patch

@patch("requests.Session.get")
def test_audit_with_mocked_order(mock_get):
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"id": 1001, "status": "CREATED"}
    # 现在出库审核可以直接跑，不依赖订单创建接口
```

**注意**：Mock 只在特殊场景用，正常回归不用 Mock。

---

### 六、方案速查

| 方案 | 适用场景 | 依赖失败时 |
|:---|:---|:---|
| **fixture 依赖链** | 正常回归 | 后续跳过，不连锁报错 |
| **pytest.skip** | 前置可选的场景 | 优雅跳过 |
| **关键字引擎 break** | 业务链路 | 失败后不跑后续步骤 |
| **预置数据** | 高频回归，弱化依赖 | 不受前置接口影响 |
| **Mock** | 依赖方不可用/e临时 | 完全隔离 |

### 七、面试回答模板

> **"接口强依赖在 ERP 里是常态——出库依赖订单、凭证依赖出库。我的处理是分层级的。正常回归用 fixture 依赖链——Pytest 自动按依赖顺序执行，前置失败后续自动跳过。敏感链路用关键字引擎的 break 策略——上一步失败下一步不跑。**
>
> **高频回归用预置数据——每天跑的核心用例提前创建好 10 条订单，不依赖创建接口的可用性。Mock 只在内部接口死了需要隔离第三方的时候才用，正常回归不 Mock。"**

---

**关键词**：接口依赖 / fixture 依赖链 / pytest.skip / 预置数据 / 关键字引擎 break

---

## Q22. 接口自动化怎么清理脏数据？结合 ERP 项目

### 问题描述

每天跑几百条接口自动化，ERP 里创建了一堆测试订单、出库单、凭证。一天不清就几千条脏数据，影响性能也可能干扰业务统计。怎么清理？

### 答案

脏数据清理不是可有可无，是**长期稳定运行的保障**。我分三层处理：用例级清理、模块级清理、定时兜底清理。

---

### 一、用例级清理：fixture yield（最干净）

**核心思想**：谁创建的谁删掉，用例跑完自清洁。

```python
# conftest.py
@pytest.fixture(scope="function")
def test_order(api_client):
    """创建测试订单 → 用例用 → 跑完自动删除"""
    # Step 1: 创建
    resp = api_client.post("/sale/order/create", json={
        "sku": "TEST_SKU_AUTO",
        "quantity": 100,
        "remark": "自动化测试"    # ★ 打标记，方便批量清理兜底
    })
    order_id = resp.json()["id"]
    print(f"创建测试订单: {order_id}")

    # Step 2: 把 order_id 交给用例
    yield order_id

    # Step 3: 用例跑完，自动清理（不管用例成功还是失败都会执行）
    try:
        api_client.post(f"/sale/order/{order_id}/cancel")
        print(f"清理测试订单: {order_id}")
    except Exception as e:
        print(f"清理失败（可能已被业务逻辑删除）: {e}")


def test_audit_order(test_order):
    """用例直接用 test_order，跑完自动清理"""
    resp = api_client.post(f"/sale/order/{test_order}/audit")
    assert resp.status_code == 200
```

**关键原则**：每个 fixture 创建的归自己清，不依赖全局清理。

---

### 二、模块级清理：session fixture 统一清

```python
# tests/sale/conftest.py —— 销售模块统一前后置

@pytest.fixture(scope="module")
def sale_module_cleanup(api_client):
    """销售模块所有用例跑完后，统一清理标记为自动化的测试数据"""
    yield
    print("\n[销售模块] 清理测试数据...")

    # 通过标记批量删
    resp = api_client.post("/testdata/cleanup", json={
        "module": "sale",
        "remark": "自动化测试",
        "created_before_hours": 24  # 只清 24 小时前的，防止误删正在跑的
    })
    print(f"清理完成: {resp.json()['deleted_count']} 条")
```

---

### 三、定时兜底清理：SQL + 定时任务

万一 fixture 清理失败（脚本中途被杀、网络中断），需要兜底：

```python
# cleanup_script.py —— Jenkins 每晚定时执行
import requests

def nightly_cleanup():
    """每晚清理标记为自动化的测试数据"""
    # 通过 API 清理
    resp = requests.post("https://test-api.example.com/testdata/cleanup", json={
        "remark_like": "自动化测试",
        "created_before_hours": 24
    })
    print(f"兜底清理: {resp.json()['deleted_count']} 条")

    # 或者直接 SQL（如果有权限）
    # DELETE FROM sale_order WHERE remark='自动化测试' AND created_at < NOW() - INTERVAL 24 HOUR;
    # DELETE FROM wms_outbound WHERE remark='自动化测试' AND created_at < NOW() - INTERVAL 24 HOUR;
    # DELETE FROM finance_voucher WHERE remark='自动化测试' AND created_at < NOW() - INTERVAL 24 HOUR;
```

```groovy
// Jenkinsfile —— 每晚 2 点执行
pipeline {
    triggers { cron('0 2 * * *') }
    stages {
        stage('Cleanup') {
            steps {
                sh 'python cleanup_script.py'
            }
        }
    }
}
```

---

### 四、ERP 数据的特殊处理

**删除顺序**：因为有主外键约束，必须按顺序删：

```text
凭证（finance_voucher） → 出库单（wms_outbound） → 订单（sale_order）
后创建的先删，前创建的后删
```

**加密数据不删**：财务部门的测试凭证可能和真实凭证混在一起，需要走专门的作废流程而非直接删。

```python
# 财务数据不做物理删除，走作废冲红流程
resp = api_client.post(f"/finance/voucher/{voucher_id}/reversal", json={
    "reason": "自动化测试清理"
})
```

**不等可删的场景**：

```python
# 有些单据不能删（已完成、已归档），那就跳过
if order["status"] in ["ARCHIVED", "DONE"]:
    print(f"跳过 {order_id}（已归档）")
    continue
```

---

### 五、三层清理体系总结

```
第一层（用例级） fi             xture yield → 每条用例跑完自清洁
   ↓ 万一失败
第二层（模块级）                 → session fixture → 统一清理标记为自动化的
   ↓ 还失败（脚本挂了）
第三层（兜底）                   → Jenkins 定时任务 → 每晚清 24 小时前的脏数据
```

### 六、面试回答模板

> **"脏数据清理我分三层。第一层是 fixture yield——每条用例创建的数据用完后自动删掉，不管用例成功还是失败都会清理。第二层是模块级的 session fixture 统一清——销售模块跑完后批量删掉标记为自动化的数据。第三层是兜底——Jenkins 每晚定时跑清理脚本，删掉 24 小时前还未清理的测试数据。**
>
> **ERP 数据清理要额外注意——财务数据不做物理删除走作废冲红流程，删除顺序要按主外键约束从后往前删。"**

---

**关键词**：脏数据清理 / fixture yield / 兜底清理 / 定时任务 / 财务作废

---

## Q23. 环境不稳定导致自动化失败，怎么设置失败自动重跑？

### 问题描述

环境抖动（网络延迟、数据库连接超时、第三方接口波动）导致自动化用例偶发失败，不是 Bug 但影响通过率。怎么让失败的用例自动重跑几次？

### 答案

用 Pytest 的 **pytest-rerunfailures** 插件，一行命令搞定。

---

### 一、命令行方式

```bash
# 安装
pip install pytest-rerunfailures

# 失败后重跑 2 次，每次间隔 5 秒
pytest --reruns 2 --reruns-delay 5

# 只在本次运行有效
pytest tests/ --reruns 3 --reruns-delay 3 -v
```

---

### 二、配置到 pytest.ini（项目级生效）

```ini
# pytest.ini
[pytest]
addopts =
    --reruns=2
    --reruns-delay=5
    --only-rerun=AssertionError   # 只重跑断言失败的，网络错误不重跑
```

---

### 三、只对特定用例重跑

```python
import pytest

# 全局都不重跑，就这一条重跑
@pytest.mark.flaky(reruns=3, reruns_delay=2)
def test_outbound_audit():
    ...
```

---

### 四、结合 Jenkins（CI 中常用）

```groovy
// Jenkinsfile
stage('Run Tests') {
    steps {
        sh '''
            pytest tests/ \\
                --reruns 2 \\
                --reruns-delay 5 \\
                --only-rerun "Timeout\|ConnectionError\|AssertionError" \\
                --junitxml=report.xml
        '''
    }
}
```

---

### 五、Flaky 检测：区分真失败和环境抖动

```python
# conftest.py —— 自动标记 rerun 后通过的用例为 Flaky
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    if hasattr(report, "rerun") and report.outcome == "passed":
        print(f"⚠️ Flaky 用例: {item.nodeid}（首次失败，重跑后通过）")
```

**策略**：

```text
首次失败 + 重跑通过 → Flaky，构建 UNSTABLE（不是 FAILURE）
首次失败 + 重跑也失败 → 真 Bug，构建 FAILURE
持续 Flaky 3 天 → 必须修复，阻塞发版
```

---

### 六、面试回答模板

> **"环境不稳定导致的偶发失败，我们用 pytest-rerunfailures 插件解决。CI 里加了 `--reruns 2 --reruns-delay 5`，失败后自动重跑 2 次。同时在 conftest.py 里加了 Flaky 检测——重跑后通过的标记为 Flaky 而非失败，定期清理。**
>
> **重跑不是万能的——连续 3 天都是 Flaky 的就必须修，不能让它变成常态。另外 `--only-rerun` 只重跑特定异常，避免环境真的挂了一直重跑浪费时间。"**

---

**关键词**：失败重跑 / pytest-rerunfailures / Flaky / 环境不稳定

---

## Q24. Pytest 常用的装饰器有哪些？结合 ERP 项目举例

### 问题描述

面试官问你 Pytest 用过哪些装饰器，不是让你列名称，而是看你在**实际项目中每个装饰器解决什么问题**。

### 答案

我做 ERP 接口自动化时最常用的 7 个 Pytest 装饰器：

---

### ① `@pytest.mark.parametrize` —— 数据驱动（天天用）

```python
import yaml
cases = yaml.safe_load(open("data/login.yaml"))["cases"]

@pytest.mark.parametrize("case", cases, ids=lambda c: c["desc"])
def test_login(case, api_client):
    resp = api_client.post("/auth/login", json=case["body"])
    assert case["expect"]["status"] == resp.status_code
```

**ERP 场景**：登录 15 组参数、改价 10 组金额，一条 parametrize 全搞定。

---

### ② `@pytest.mark.smoke` / `@pytest.mark.p0` —— 标记分级

```python
@pytest.mark.smoke      # 冒烟用例，每次提交必须跑
def test_login_success(api_client):
    ...

@pytest.mark.p1         # P1 核心用例，上线前跑
def test_outbound_audit(api_client):
    ...

@pytest.mark.slow       # 慢用例，平时跳过
def test_full_report(api_client):
    ...
```

```bash
pytest -m "smoke"           # 只跑冒烟（10条，5秒）
pytest -m "p0 or p1"        # 跑 P0+P1（50条）
pytest -m "not slow"        # 跳过慢的
pytest -m "sale and smoke"  # 销售模块的冒烟
```

---

### ③ `@pytest.mark.skip` / `@pytest.mark.skipif` —— 条件跳过

```python
@pytest.mark.skip(reason="出库审核接口下个迭代才改完")
def test_new_audit_rule():
    ...

# 第三方接口不稳定时自动跳过
import os
@pytest.mark.skipif(
    os.getenv("THIRD_API_ENABLED") != "true",
    reason="第三方接口不可用"
)
def test_payment_callback():
    ...
```

---

### ④ `@pytest.mark.xfail` —— 已知 Bug 预期失败

```python
@pytest.mark.xfail(
    reason="Bug #1234: 改价后库存成本偶发滞后更新，开发下迭代修"
)
def test_price_update_cost_sync():
    """失败了算通过（已知Bug），突然过了反而告警（Bug修了？）"""
    ...
```

**ERP 场景**：线上有个偶发的凭证生成延迟 Bug，还没排到修复。用例标记 xfail，Regression 不会被这条用例拖成红色。

---

### ⑤ `@pytest.mark.flaky` —— 环境波动重跑

```python
@pytest.mark.flaky(reruns=3, reruns_delay=2)
def test_third_party_call():
    """第三方接口不稳定，允许重跑 3 次"""
    ...
```

---

### ⑥ `@pytest.mark.timeout` —— 防止卡死

```python
@pytest.mark.timeout(30)
def test_batch_export():
    """报表导出超过 30 秒直接失败，不卡 Pipeline"""
    ...
```

---

### ⑦ `@pytest.mark.run` —— 控制执行顺序

```python
# 需要按顺序跑的链路（一般不推荐，但偶有场景）
@pytest.mark.run(order=1)
def test_create_order():
    ...

@pytest.mark.run(order=2)
def test_audit_order():
    ...
```

---

### 速查表

| 装饰器 | 一句话 | ERP 场景 |
|:---|:---|:---|
| `@parametrize` | 数据驱动，加数据=加用例 | 登录 15 组参数 |
| `@mark.smoke` | 标记分类，按需跑 | PR 提交跑 10 条冒烟 |
| `@mark.skip` | 条件跳过 | 第三方挂了别跑 |
| `@mark.xfail` | 已知 Bug 预期失败 | 凭证差一分钱，下迭代修 |
| `@mark.flaky` | 失败重跑 | 支付接口偶发超时 |
| `@mark.timeout` | 超时杀死 | 报表导出别卡 5 分钟 |
| `@mark.run` | 控制顺序 | 先创建再审核的链路 |

### 面试回答模板

> **"Pytest 装饰器我用得最多的是 parametrize——从 YAML 读数据直接驱动用例，加一条数据就是加一条用例。mark.smoke 做分级——PR 提交只跑冒烟 10 条，上线前跑 P0+P1 全量。mark.xfail 管理已知 Bug——线上有个偶发的凭证延迟，用例写了但标记 xfail，不拖回归通过率。**
>
> **还有 timeout 防止卡 Pipeline、skipif 条件跳过不稳定接口、flaky 让环境波动自动重跑。这些装饰器都不是为用而用，每个都解决了实际问题。"**

---

**关键词**：Pytest 装饰器 / parametrize / mark / xfail / skip / smoke / timeout

---

## Q25. GET、POST、PUT 有什么区别？（结合 ERP 举例）

### 答案

| | GET | POST | PUT |
|:---|:---|:---|:---|
| **做什么** | 读（查询） | 创建 | 更新/替换 |
| **参数在哪** | URL 参数 | Body | Body |
| **幂等** | ✅ 查 10 次结果一样 | ❌ 每调一次创建一条 | ✅ 改 10 次结果一样 |
| **缓存** | ✅ 可缓存 | ❌ 不缓存 | ❌ 不缓存 |

**ERP 场景速记**：

```python
# GET —— 查库存
requests.get("/wms/stock/query", params={"sku": "SKU001"})

# POST —— 创建出库单
requests.post("/wms/outbound/create", json={"sku": "SKU001", "quantity": 100})

# PUT —— 修改价格（完整替换）
requests.put("/sale/price/update", json={"sku": "SKU001", "price": 99.00})
```

**面试加分：幂等性**

```python
# GET：幂等，查多少次都一样
for i in range(10):
    resp = requests.get("/wms/stock/query", params={"sku": "SKU001"})
    # 10 次结果相同，不会产生新数据

# POST：非幂等，每调一次创建一条新单据
for i in range(3):
    requests.post("/wms/outbound/create", json={"sku": "SKU001", "quantity": 10})
    # 创建了 3 条出库单！如果不做防重，就是 Bug

# PUT：幂等，改多少次都一样
for i in range(3):
    requests.put("/sale/price/update", json={"sku": "SKU001", "price": 99.00})
    # 3 次都是 99.00，不会越改越多
```

**一句话**：GET 是看，POST 是新建，PUT 是改——测试 POST 时最需要关注幂等，重复请求别重复建单。

---

**关键词**：GET / POST / PUT / HTTP 方法 / 幂等

---

## Q26. Cookie、Session、Token 有什么区别？

### 答案

| | Cookie | Session | Token |
|:---|:---|:---|:---|
| **存在哪** | 浏览器 | 服务器 | 客户端 |
| **存什么** | session_id 等小数据 | 用户信息 | 加密签名的身份信息 |
| **跨域** | ❌ | ❌ | ✅ 天然支持 |
| **移动端** | ❌ | ❌ | ✅ App 专用 |
| **谁来验证** | 服务器查 Session | 服务器查内存/DB | 客户端自包含，JWT 解密即可 |

**你项目里怎么用的**：

```python
# Session 方式（Requests 自动管理 Cookie）
session = requests.Session()
session.post("/auth/login", json={"username":"admin", "password":"123456"})
session.get("/order/list")  # 自动带 Cookie

# Token 方式（手动拼 Header）
resp = requests.post("/auth/login", json={"username":"admin", "password":"123456"})
token = resp.json()["token"]
requests.get("/order/list", headers={"Authorization": f"Bearer {token}"})
```

**一句话**：Cookie 是浏览器自动带的，Session 存服务器，Token 是客户端手动传的。现在移动端+微服务时代，Token（JWT）是主流。

---

**关键词**：Cookie / Session / Token / JWT / 认证方式

---

## Q27. 常见 HTTP 状态码有哪些？

### 答案

| 范围 | 含义 | 常见举例 |
|:---|:---|:---|
| **2xx** | 成功 | 200 OK / 201 Created |
| **3xx** | 重定向 | 301 永久 / 302 临时 / 304 缓存 |
| **4xx** | 客户端错误 | 400 参数错 / 401 未登录 / 403 无权限 / 404 不存在 / 422 校验失败 |
| **5xx** | 服务端错误 | 500 内部错 / 502 网关坏 / 503 服务挂了 / 504 超时 |

**测试中必关注**：

```python
# 401 —— 没登录就别想查数据
resp = requests.get("/order/list")  # 没带 token
assert resp.status_code == 401

# 403 —— 普通用户不能审核
resp = api_client.post("/outbound/audit")  # guest 身份
assert resp.status_code == 403

# 422 —— 库存不足时返回的校验错误
resp = api_client.post("/outbound/audit", json={"order_id": 1001})
assert resp.status_code == 422
assert "库存不足" in resp.text
```

**面试一句**：4xx 是客户端的锅，5xx 是服务端的锅——排查方向就分清了。

---

**关键词**：HTTP 状态码 / 200 / 401 / 403 / 500 / 502

---

## Q28. Python 装饰器的底层原理是什么？

### 答案

装饰器本质是**一个函数，接收另一个函数，并返回一个新函数**。

```python
# 没有语法糖时的写法
def log(func):
    def wrapper(*args, **kwargs):
        print(f">>> 调用 {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

def audit_outbound(order_id):
    pass

audit_outbound = log(audit_outbound)  # 手动包装

# @语法糖就是上面这行的简写
@log
def audit_outbound(order_id):
    pass
```

**带参数的装饰器**——多包一层：

```python
def retry(max_retries=3):           # 外层：接收参数
    def decorator(func):             # 中层：接收函数
        def wrapper(*args, **kwargs): # 内层：实际执行
            for i in range(max_retries):
                try: return func(*args, **kwargs)
                except: pass
        return wrapper
    return decorator

@retry(max_retries=3)  # 等价于 retry(max_retries=3)(func)
def call_api():
    ...
```

**你在测试框架里的应用**：`@pytest.mark.parametrize`、`@pytest.fixture`、`@auto_log_keywords` 背后全是装饰器原理。

---

**关键词**：装饰器 / @语法糖 / 闭包 / 函数作为参数

---

## Q29. Allure 报告怎么配置？结合 Pytest

### 答案

**命令式三步**：

```bash
# 1. 安装
pip install allure-pytest

# 2. 跑测试（自动生成 allure-results 目录）
pytest tests/ --alluredir=allure-results

# 3. 打开报告
allure serve allure-results
```

**代码中添加注解**：

```python
import allure

@allure.feature("销售模块")
@allure.story("出库审核")
@allure.title("正常出库审核")
@allure.severity(allure.severity_level.CRITICAL)
def test_outbound_audit():
    with allure.step("创建出库单"):
        ...
    with allure.step("提交审核"):
        ...
    with allure.step("验证凭证生成"):
        ...
```

**Jenkins 集成**：

```groovy
stage('Allure Report') {
    steps {
        sh 'pytest tests/ --alluredir=allure-results'
    }
    post {
        always {
            allure includeProperties: false,
                   results: [[path: 'allure-results']]
        }
    }
}
```

**常用注解速查**：

| 注解 | 含义 |
|:---|:---|
| `@allure.feature` | 大模块 |
| `@allure.story` | 子功能 |
| `@allure.step` | 测试步骤 |
| `@allure.title` | 用例标题 |
| `@allure.severity` | 严重程度（BLOCKER→TRIVIAL） |

---

**关键词**：Allure / 测试报告 / Jenkins集成 / allure-pytest