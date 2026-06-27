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

---

## Q30. Python 包的版本控制

### 问题描述

面试官问你在自动化项目中如何管理 Python 包的版本？这个问题考的是你对**项目依赖管理和环境一致性**的实际经验，尤其是多人协作、多环境部署时的能力。

---

### 答案

#### 一句话核心

> **版本控制的本质是把"某台机器能跑"变成"任何机器、任何时间都能复现"。**

---

### 一、为什么要做包版本控制？

在自动化项目中，最常见的翻车场景：

- 你本地用 `requests 2.28`，CI 服务器装的是 `2.31`，某个 API 行为不一致，用例莫名其妙挂掉
- 新同事 `pip install -r requirements.txt` 装完，跑不起来，因为版本约束写的太松
- `pytest-html` 升级了大版本，报告模板接口变了，生成报告直接报错

---

### 二、核心工具：requirements.txt

最常用的方式，适合中小型项目。

**生成方式：**

```bash
# 导出当前环境所有已安装包（含依赖的依赖，比较重）
pip freeze > requirements.txt

# 只导出项目直接依赖（更推荐，更干净）
pip list --not-required --format=freeze > requirements.txt
```

**requirements.txt 写法规范：**

```text
# ✅ 推荐：锁定完整版本号（生产/CI 环境）
requests==2.31.0
pytest==7.4.3
selenium==4.15.2
PyYAML==6.0.1
allure-pytest==2.13.2

# ⚠️ 允许小版本更新（开发环境可接受）
requests>=2.28.0,<3.0.0

# ❌ 不写版本号（等于随机安装，危险）
requests
```

**安装：**

```bash
pip install -r requirements.txt
```

---

### 三、虚拟环境（必须配套使用）

光有 `requirements.txt` 不够，还要隔离环境，否则不同项目的包互相污染。

```bash
# 创建虚拟环境
python -m venv venv

# 激活（Windows）
venv\Scripts\activate

# 激活（Linux / Mac）
source venv/bin/activate

# 退出
deactivate
```

**结合 `requirements.txt` 的完整工作流：**

```bash
# 第一次搭建项目
python -m venv venv
source venv/bin/activate
pip install requests pytest selenium PyYAML
pip freeze > requirements.txt

# 新同事 / CI 服务器还原环境
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

### 四、结合 ERP 项目的实际应用

> 我在搭建接口 + UI 双自动化框架时，把依赖管理写成了两个文件：

```text
requirements.txt        # 运行依赖（CI 安装这个）
requirements-dev.txt    # 开发依赖（本地开发用，含调试工具）
```

**requirements.txt（运行依赖）：**

```text
requests==2.31.0
pytest==7.4.3
pytest-rerunfailures==12.0
selenium==4.15.2
PyYAML==6.0.1
allure-pytest==2.13.2
```

**requirements-dev.txt（开发依赖）：**

```text
-r requirements.txt
pytest-html==4.1.1
ipython==8.18.1
black==23.11.0
```

Jenkins Pipeline 里只跑 `pip install -r requirements.txt`，不装开发工具，保持 CI 环境干净。

---

### 五、进阶：pip-tools（版本锁定更严格）

适合对环境一致性要求更高的团队：

```bash
pip install pip-tools

# 写 requirements.in（只写直接依赖，不写版本约束）
# requests
# pytest
# selenium

# 自动解析依赖树，生成锁定版本的 requirements.txt
pip-compile requirements.in

# 安装
pip-sync requirements.txt
```

**优势**：`pip-compile` 会把所有间接依赖也锁死，不会因为第三方库升级而意外引入 breaking change。

---

### 六、面试回答框架

```
① 说工具：用 requirements.txt + 虚拟环境管理项目依赖
② 说规范：版本号锁定策略（==精确 vs >=范围）
③ 说场景：分生产依赖/开发依赖两个文件，CI 只装运行依赖
④ 说踩坑：没锁版本导致 CI 和本地不一致的问题，以及怎么修复的
```

---

**关键词**：requirements.txt / pip freeze / virtualenv / venv / pip-tools / 依赖管理 / 环境隔离
---

## Q31. Python 可变类型和不可变类型有哪些？在自动化框架中怎么用？

### 问题描述

面试官问 Python 的可变类型和不可变类型——这是基础题，但要拿高分必须**结合自动化框架的实际场景**来说，不能只背概念。

---

### 答案

#### 一句话核心

> **不可变类型：改了就变成新对象（id变了）；可变类型：改了还是原来那个对象（id不变）。**

---

### 一、概念区分

```python
# 不可变类型
a = 10
print(id(a))   # 1407048...
a = a + 1
print(id(a))   # 1407048... 不一样了！a指向了新对象

# 可变类型
lst = [1, 2, 3]
print(id(lst))  # 22874...
lst.append(4)
print(id(lst))  # 22874... 一样！原地修改
```

---

### 二、类型清单

| 类型 | 可变/不可变 | 举例 |
|------|:------------:|------|
| `int` | ❌ 不可变 | `x = 10; x = x + 1` |
| `float` | ❌ 不可变 | `y = 3.14` |
| `bool` | ❌ 不可变 | `flag = True` |
| `str` | ❌ 不可变 | `s = "hello"; s += "!"` |
| `tuple` | ❌ 不可变 | `t = (1, 2); t[0] = 3  # 报错` |
| `list` | ✅ 可变 | `lst = [1,2]; lst.append(3)` |
| `dict` | ✅ 可变 | `d = {"a":1}; d["b"] = 2` |
| `set` | ✅ 可变 | `s = {1,2}; s.add(3)` |

---

### 三、在自动化框架中的实际应用

**这是面试拿高分的关键——结合你的框架说！**

---

#### 场景1：全局配置用 `dict`（可变），但要注意引用陷阱

```python
# config.yaml 读取后存成 dict
config = {
    "base_url": "https://api.example.com",
    "timeout": 30,
    "headers": {"Content-Type": "application/json"}
}

# 多个测试文件共享同一个 config
# 如果在某个用例里改了 config["headers"]，
# 会影响其他所有用例！
# 解决方式：每个用例用 deepcopy，或者用 fixture 隔离
```

**面试话术：**
> "全局配置我用 `dict` 存，因为可以动态修改。但要注意——如果在用例里改了 `config["headers"]`，会影响其他用例。我的做法是把 config 封装成单例，暴露只读接口，需要改的时候 clone 一份，避免用例之间互相污染。"

---

#### 场景2：`tuple` 做固定配置（不可变，安全）

```python
# 接口返回的状态码枚举，用 tuple 防止误改
ORDER_STATUS = (
    "DRAFT",     # 草稿
    "SUBMITTED", # 已提交
    "AUDITED",   # 已审核
    "CANCELLED"  # 已取消
)
# 如果有人写 ORDER_STATUS[0] = "XXX" → 直接报错，安全！
```

**面试话术：**
> "框架里一些固定枚举值我用 `tuple` 而不是 `list`，因为 `tuple` 不可变，防止测试过程中被误改，更安全地表达'这些值不该变'的语义。"

---

#### 场景3：`list` 收集测试结果（可变，方便追加）

```python
# 收集所有失败的用例名
failed_cases = []

@pytest.fixture
def collect_failures():
    yield
    # teardown：把失败的用例名追加到列表
    # （pytest 有内置的 request.node.rep_* 可以拿到结果）

# 最后统一发通知
send_alert(failed_cases)
```

---

#### 场景4：`str` 不可变 → 拼接用 `join` 而不是 `+`

```python
# ❌ 不推荐（每次 += 都创建新字符串，效率低）
url = base_url
url += "/api"
url += "/login"

# ✅ 推荐
url = "/".join([base_url, "api", "login"])
```

**面试话术：**
> "拼接 URL 或日志消息时，我不会用 `+` 号，因为 `str` 不可变，每次拼接都创建新对象。用 `join` 或 f-string 更高效，这个在框架里处理大量日志时比较明显。"

---

#### 场景5：函数默认参数陷阱（高频坑！）

```python
# ❌ 危险！list 是可变对象，默认参数会被共享
def add_case(case, cases=[]):
    cases.append(case)
    return cases

print(add_case("登录"))   # ["登录"]
print(add_case("下单"))   # ["登录", "下单"]  ← 脏数据！

# ✅ 正确写法
def add_case(case, cases=None):
    if cases is None:
        cases = []
    cases.append(case)
    return cases
```

**面试话术：**
> "这是 Python 的经典坑——默认参数如果指向可变对象（`list`/`dict`），多次调用会共享同一个对象。我在写 fixture 或工具函数时特别注意这点，默认参数一律用 `None`，函数体内再初始化。"

---

### 四、面试回答框架

```
① 先说概念：不可变（int/str/tuple）改了变新对象；
              可变（list/dict/set）改了还是原对象
② 再说坑：默认参数用 list/dict 会翻车
③ 再说框架应用：config用dict、枚举用tuple、
                  URL拼接用join、fixture里注意引用共享
```

---

**关键词**：可变类型 / 不可变类型 / id() / 默认参数陷阱 / deepcopy / 框架配置管理

---

## Q32. 列表（list）的增删改查怎么用？在自动化框架中有哪些场景？

### 问题描述

面试官问 list 的增删改查——基础题，但要拿高分必须**结合自动化框架的实际使用场景**。

---

### 答案

#### 一句话核心

> **list 是有序可变的容器，增删改查都有多种方法，选对方法比记住方法更重要。**

---

### 一、增删改查方法速查

#### 增（添加元素）

```python
lst = [1, 2, 3]

lst.append(4)           # [1, 2, 3, 4]          尾部追加一个
lst.extend([5, 6])     # [1, 2, 3, 4, 5, 6]   尾部追加多个（拆开）
lst.insert(0, 0)       # [0, 1, 2, 3, 4, 5, 6] 指定位置插入
```

| 方法 | 区别 |
|------|------|
| `append(x)` | 把 x 当**一个元素**加进去 |
| `extend([x, y])` | 把 x、y **分别**加进去（等价于 `append(x); append(y)`） |
| `insert(i, x)` | 插入到索引 i 的位置 |

⚠️ **常见误区：**

```python
lst = [1, 2, 3]
lst.append([4, 5])
# 结果：[1, 2, 3, [4, 5]]  ← 嵌套了！不是 [1,2,3,4,5]
# 想展开加要用 extend
```

---

#### 删（删除元素）

```python
lst = [1, 2, 3, 2, 4]

lst.remove(2)      # [1, 3, 2, 4]        删除第一个值为2的元素
val = lst.pop()    # val=4, lst=[1,3,2]   删除并返回尾部元素
val = lst.pop(0)  # val=1, lst=[3,2]      删除并返回指定位置元素
lst.clear()         # []                       清空全部
del lst[0]         # 按索引删，不返回值       lst = [3, 2]
del lst[:]          # 删切片（等价于 clear()）  lst = []
```

| 方法 | 删什么 | 返回值 | 元素/索引不存在时 |
|------|---------|--------|---------------------|
| `remove(x)` | 按**值**删第一个 | ❌ 不返回 | 报 `ValueError` |
| `pop(i)` | 按**索引**删 | ✅ 返回被删元素 | 报 `IndexError` |
| `pop()` | 删尾部 | ✅ 返回被删元素 | 空列表报 `IndexError` |
| `clear()` | 删全部 | ❌ 不返回 | 无（空列表也不报错） |
| `del lst[i]` | 按**索引**删 | ❌ 不返回 | 报 `IndexError` |

---

#### ⚡ 重点：`del` 语句 vs `remove`/`pop`/`clear` 方法

**核心区别一句话：**

> **`del` 是 Python 的通用语句，不是 list 的方法；`remove`/`pop`/`clear` 是 list 自带的方法。**

```python
# del 的用法（不止能删 list 元素）
lst = [1, 2, 3]
del lst[0]        # 删 list 中某个索引的元素
del lst[:]        # 删切片（等价于 clear()）
del lst            # 删整个变量（后面再访问 lst 会报 NameError）

# remove/pop/clear 只能操作 list 本身
lst.remove(1)     # ✅ 是 list 的方法
del lst[0]        # ✅ 是 Python 语句，不是方法
```

**面试回答框架：**

```
"del 和 remove/pop 的区别主要在两点：

第一，del 是 Python 语句，不是 list 的方法，
     还能删变量、删切片；
     remove/pop/clear 是 list 专属方法。

第二，pop 有返回值（被删的元素），
     del 和 remove 没有返回值。
     如果我删完还要用这个值，就用 pop；
     只是想删掉，用 del 或 remove 都行。

实际在框架里，我常用 pop 来处理
待执行用例列表——pop(0) 取出来执行，
天然实现了队列的效果。"
```

---

#### 改（修改元素）

```python
lst = [1, 2, 3]
lst[0] = 10              # [10, 2, 3]       直接赋值改
lst[0:2] = [10, 20]    # [10, 20, 3]     切片改多个
```

---

#### 查（查找元素）

```python
lst = [1, 2, 3, 2, 4]

lst[0]           # 1              按索引取（越界报 IndexError）
lst.index(2)     # 1              第一个值为2的索引
lst.count(2)     # 2              值为2的元素个数
2 in lst         # True            是否包含（推荐用这个判断）
```

---

### 二、在自动化框架中的实际场景

#### 场景1：收集失败用例名（增 + 查）

```python
failed_cases = []

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    # 每个用例执行完，把失败的加进列表
    if call.excinfo is not None:
        failed_cases.append(item.nodeid)

# 全部跑完后
if failed_cases:
    send_notification(f"失败用例：{failed_cases}")
```

---

#### 场景2：动态组装 URL 参数（改 + 查）

```python
# 把参数列表拼成 query string
params = []
for k, v in {"page": 1, "size": 20}.items():
    params.append(f"{k}={v}")
query_string = "&".join(params)   # page=1&size=20
```

---

#### 场景3：用例执行顺序控制（删 + 查）

```python
# 有些用例有依赖，前面失败了后面就不跑
pending_cases = ["test_login", "test_create_order", "test_cancel_order"]

for case in pending_cases:
    result = run_case(case)
    if result == "FAILED":
        # 后续用例从列表里移除，不跑了
        pending_cases.remove(case)
        break
```

---

#### 场景4：fixture 里用 list 收集请求响应（增）

```python
@pytest.fixture
def api_recorder():
    records = []   # 用 list 记录所有请求
    yield records
    # teardown：输出所有记录
    for r in records:
        print(f"{r['method']} {r['url']} -> {r['status']}")
```

---

### 三、list 和 tuple 怎么选？

| 需求 | 用 list | 用 tuple |
|------|---------|----------|
| 数据会变 | ✅ | ❌ |
| 数据固定不变（如枚举值） | ❌ 不安全 | ✅ 防误改 |
| 需要 append/remove | ✅ | ❌ 不支持 |
| 作为 dict 的 key | ❌ 不可哈希 | ✅ 可以 |

```python
# ✅ tuple 可以当 dict 的 key
cache = {}
key = ("login", "POST", "/api/login")
cache[key] = "response_data"

# ❌ list 不行
key = ["login", "POST", "/api/login"]
cache[key] = "..."   # 报 TypeError：unhashable type
```

---

### 四、面试回答框架

```
① 先说基本操作：append/extend/remove/pop/index/count
② 说一个常见误区：append([x,y])会嵌套，要extend
③ 说框架应用：收集失败用例、组装参数、fixture记录
④ 说 list vs tuple 的选择依据
```

---

**关键词**：list / append / extend / remove / pop / index / 可变类型 / 用例收集

---

## Q33. lambda 匿名函数是什么？什么场景适合用？

### 问题描述

面试官问 lambda 匿名函数——不只考语法，更在问你对**函数式编程思维**的理解，以及在框架里**何时该用 lambda、何时该用 def**。

---

### 答案

#### 一句话核心

> **lambda 是"只用一次、很简单"的函数的快捷写法，不需要正式 def 定义。**

---

### 一、基本语法

```python
# 普通函数
def add(x, y):
    return x + y

# lambda 等价写法
add = lambda x, y: x + y
add(1, 2)   # 3
```

**限制：lambda 函数体只能是「一个表达式」，不能写多行、不能有 if/for 语句体（只能用三元表达式）。**

```python
# ✅ 可以：三元表达式算一行
f = lambda x: x * 2 if x > 0 else 0

# ❌ 不可以：多行逻辑
f = lambda x:
    y = x * 2    # 语法错误！
    return y
```

---

### 二、lambda vs def 怎么选？

| | `def` | `lambda` |
|---|---|---|
| 有没有名字 | ✅ 有 | ❌ 匿名（可赋值但不推荐） |
| 函数体 | 多行，任意复杂 | 只能一行表达式 |
| 适合场景 | 复用逻辑、复杂逻辑 | 一次性、简单转换 |
| 可读性 | 复杂逻辑更清晰 | 简单逻辑更简洁 |

**一句话决策：逻辑只用一次 + 只有一行 → lambda；否则 → def。**

---

### 三、在自动化框架中的适用场景

#### 场景1：排序的 key 函数（最常用）

```python
# 按优先级排序用例（P0 → P1 → P2）
cases = [
    {"name": "test_login", "priority": "P2"},
    {"name": "test_order", "priority": "P0"},
    {"name": "test_cancel", "priority": "P1"},
]

priority_order = {"P0": 0, "P1": 1, "P2": 2}
cases.sort(key=lambda c: priority_order[c["priority"]])
```

**为什么用 lambda 而不写 def：** key 函数只用这一次，逻辑只有一行，写 def 反而多了一个只在 sort 里用的函数名，增加阅读负担。

---

#### 场景2：配合 map / filter 做数据转换

```python
# 接口返回的 ID 是字符串，转成 int
response = [{"id": "1"}, {"id": "2"}, {"id": "3"}]
ids = list(map(lambda d: int(d["id"]), response))

# 过滤掉已取消的订单
orders = [{"id": 1, "status": "CANCELED"}, {"id": 2, "status": "AUDITED"}]
valid = list(filter(lambda o: o["status"] != "CANCELED", orders))
```

---

#### 场景3：pytest 参数化里的延迟执行

```python
import pytest
import requests

# ❌ 不用 lambda：请求在参数收集阶段就发了（太早！）
test_data = [
    ("登录", requests.post(url+"/login", json={...})),  # 立刻执行
]

# ✅ 用 lambda：把请求包成函数，用例执行时才调用
test_data = [
    ("登录", lambda: requests.post(url+"/login", json={...})),
]

@pytest.mark.parametrize("name, func", test_data)
def test_api(name, func):
    result = func()   # 这里才真正发请求
    assert result.status_code == 200
```

---

#### 场景4：动态生成断言函数

```python
# 根据 YAML 里的 expected 动态生成断言
def make_assertion(expected):
    return lambda resp: resp.json()["code"] == expected["code"]

assertion = make_assertion({"code": 0})
assertion(response)   # True
```

---

### 四、什么场景不适合用 lambda？

```python
# ❌ 逻辑超过一行 → 用 def
# lambda 不能写多行，强行写复杂逻辑可读性极差

# ❌ 需要复用 → 用 def
# 如果同一个逻辑在 3 个地方用到，lambda 写 3 遍维护成本高

# ❌ 需要异常处理 → 用 def
# lambda 里不能写 try/except
```

---

### 五、面试回答框架

```
① 先说是什么：lambda 是匿名函数，只有一行表达式
② 再说适用场景：一次性、简单逻辑，
                 典型是排序 key、map/filter、延迟执行
③ 再说不适用：多行逻辑、需要复用、需要异常处理 → 用 def
④ 结合框架：我主要用在排序用例优先级、
                 接口返回数据转换这两个场景
```

---

**关键词**：lambda / 匿名函数 / 函数式编程 / map / filter / 排序key函数 / 延迟执行

---

## Q34. 怎么定义一个类？在自动化框架中有哪些应用？

### 问题描述

面试官问 Python 类的定义——不只考语法，更在问你对**面向对象设计**的理解，以及在自动化框架中**如何用类组织代码**。

---

### 答案

#### 一句话核心

> **类是数据和操作数据的方法的封装；在自动化框架里，Page Object、关键字库、配置管理全是用类实现的。**

---

### 一、基本语法

```python
class User:
    # 类属性（所有实例共享）
    role = "tester"

    # 构造方法：创建实例时自动调用
    def __init__(self, name, age):
        self.name = name   # 实例属性
        self.age = age

    # 实例方法
    def say_hi(self):
        return f"Hi, I'm {self.name}"

# 使用
u = User("龙哥", 30)   # 调用 __init__
u.say_hi()              # "Hi, I'm 龙哥"
User.role                # "tester"（类属性，通过类访问）
u.role                   # "tester"（实例也可以访问）
```

**关键点：**

| 概念 | 说明 |
|------|------|
| `class` | 定义类的关键字 |
| `__init__` | 构造方法，创建实例时自动调用 |
| `self` | 指向实例本身，定义和调用时第一个参数必须是它 |
| 实例属性 | `self.xxx`，每个实例各自持有 |
| 类属性 | 直接在类体里定义，所有实例共享 |

---

### 二、在自动化框架中的三大应用场景

#### 场景1：POM 页面类（最核心！）

```python
from selenium.webdriver.remote.webdriver import WebDriver
from selenium.webdriver.common.by import By

class LoginPage:
    # 元素定位器（类属性，所有实例共享）
    USERNAME_INPUT = (By.ID, "username")
    PASSWORD_INPUT = (By.ID, "password")
    LOGIN_BTN = (By.ID, "btn-login")

    def __init__(self, driver: WebDriver):
        self.driver = driver

    def login(self, username: str, password: str):
        self.driver.find_element(*self.USERNAME_INPUT).send_keys(username)
        self.driver.find_element(*self.PASSWORD_INPUT).send_keys(password)
        self.driver.find_element(*self.LOGIN_BTN).click()

    def get_error_msg(self) -> str:
        return self.driver.find_element(By.CLASS_NAME, "error").text

# 用例里用
def test_login(driver):
    login_page = LoginPage(driver)
    login_page.login("admin", "123456")
    # 断言...
```

**为什么用类：** 把「元素定位」和「操作逻辑」封装在一起，用例代码变简洁，元素变了只改 Page 类。

---

#### 场景2：关键字驱动的关键字库类

```python
class KeywordLibrary:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()

    def login(self, username: str, password: str):
        resp = self.session.post(
            f"{self.base_url}/login",
            json={"username": username, "password": password}
        )
        return resp

    def create_order(self, product_id: int, quantity: int):
        resp = self.session.post(
            f"{self.base_url}/order",
            json={"product_id": product_id, "quantity": quantity}
        )
        return resp

# YAML 里写：action: login, data: {username: admin, password: "123456"}
# 框架里用 getattr 动态调用
lib = KeywordLibrary("https://api.example.com")
action_name = "login"                     # 从 YAML 读取
func = getattr(lib, action_name)          # 拿到 lib.login 方法
func(**{"username": "admin", "password": "123456"})
```

---

#### 场景3：单例模式管理配置（全局唯一）

```python
class Config:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.data = {}
        return cls._instance

    def load(self, path: str):
        with open(path, "r", encoding="utf-8") as f:
            self.data = yaml.safe_load(f)

    def get(self, key: str):
        return self.data.get(key)

# 不管在哪调用，拿到的都是同一个对象
c1 = Config()
c1.load("config.yaml")
c2 = Config()
c2.get("base_url")   # 能拿到！c1 和 c2 是同一个实例
```

---

### 三、常见面试追问

**Q：self 是什么？为什么每个方法第一个参数都是它？**

```
self 是指向实例本身的引用，等价于 Java 的 this。
Python 规定实例方法的第一个参数必须是 self，
这样方法内部才能访问实例的属性（self.name 等）。
调用时不需要传 self，Python 自动把实例传进去。
```

**Q：类属性和实例属性有什么区别？**

```python
class TestCase:
    timeout = 30   # 类属性：所有实例共享，相当于"默认配置"

    def __init__(self, name):
        self.name = name   # 实例属性：每个实例各自持有

tc1 = TestCase("登录")
tc2 = TestCase("下单")
tc1.timeout = 60        # 只改 tc1 的，不影响 tc2
TestCase.timeout = 45   # 改类属性，所有实例的默认值都变
```

**Q：__init__ 和 __new__ 有什么区别？**

| | `__new__` | `__init__` |
|---|---|---|
| 调用时机 | 创建实例**之前** | 创建实例**之后** |
| 作用 | 控制实例创建（单例模式用它） | 初始化实例属性 |
| 返回值 | 必须返回实例 | 不返回值（返回 None） |

---

### 四、面试回答框架

```
① 先说基本语法：class 定义类，__init__ 初始化，
              self 指向实例本身
② 再说框架应用：
    - POM 页面类封装元素和操作
    - 关键字库类配合 getattr 实现关键字驱动
    - 单例 Config 管理全局配置
③ 再说设计好处：封装、复用、维护成本低
```

---

**关键词**：class / __init__ / self / 类属性 / 实例属性 / POM / 关键字驱动 / 单例模式

---

## Q35. 实例属性和类属性有什么区别？

### 问题描述

面试官单独问实例属性和类属性的区别——这是 Q34 的追问，考的是你对**Python 对象模型**的理解深度。

---

### 答案

#### 一句话核心

> **类属性：所有实例共享同一份；实例属性：每个实例各自持有一份。**

---

### 一、用代码说话

```python
class TestCase:
    timeout = 30   # ← 类属性（写在类体里，不在 __init__ 里）

    def __init__(self, name):
        self.name = name   # ← 实例属性（在 __init__ 里，带 self.）

# 创建两个实例
tc1 = TestCase("登录")
tc2 = TestCase("下单")

# 读：实例可以读类属性
tc1.timeout    # 30
tc2.timeout    # 30

# 改实例属性：只影响自己
tc1.timeout = 60
tc1.timeout    # 60
tc2.timeout    # 30  ← 没变！

# 改类属性：通过类去改，所有实例都会变
TestCase.timeout = 45
tc1.timeout    # 60  ← tc1 已经自己有一份了，不受影响
tc2.timeout    # 45  ← tc2 没有自己的，读到的是类属性（变了）
```

---

### 二、内存视角（面试加分）

```
类属性 timeout = 30：
    TestCase 类对象
        └── timeout: 30          ← 只有一份，存在类对象上

实例属性 self.name：
    tc1 实例 ─── name: "登录"   ← 各自存一份
    tc2 实例 ─── name: "下单"   ← 各自存一份
```

**读属性时的查找顺序：**
实例 → 类 → 父类（找不到报 `AttributeError`）

```python
tc1.timeout
# 第一步：看 tc1 有没有自己的 timeout → 有（刚才设了60）→ 返回 60
# 如果 tc1 没有 → 去 TestCase 类里找 → 找到 30
```

---

### 三、在自动化框架中的应用

#### 场景1：POM 页面类的元素定位器（用类属性）

```python
class LoginPage:
    USERNAME_INPUT = (By.ID, "username")   # 类属性：所有实例共享
    PASSWORD_INPUT = (By.ID, "password")

    def __init__(self, driver):
        self.driver = driver                      # 实例属性：各自持有

# 元素定位器用类属性：
# ① 不需要创建实例就能访问：LoginPage.USERNAME_INPUT
# ② 所有实例共享同一份，不浪费内存
# ③ 语义清晰：这是"这个类的属性"，不是"某个实例的状态"
```

#### 场景2：框架默认配置（用类属性）

```python
class Config:
    BASE_URL = "https://api.example.com"   # 类属性：默认配置
    TIMEOUT = 30

    def __init__(self, env="test"):
        self.env = env             # 实例属性：每个环境各自一份
        self.token = None          # 实例属性：每个实例各自持有
```

#### 场景3：陷阱——用可变对象做类属性

```python
# ❌ 危险！类属性是可变对象，所有实例会共享修改
class TestCase:
    tags = []   # 类属性，所有实例共享同一个 list

tc1 = TestCase()
tc2 = TestCase()
tc1.tags.append("smoke")
tc2.tags          # ["smoke"]  ← 被 tc1 污染了！

# ✅ 正确：实例属性各自持有
class TestCase:
    def __init__(self):
        self.tags = []   # 每个实例各自一份
```

---

### 四、面试回答框架

```
① 先说定义位置：
   类属性写在类体里（不在 __init__）；
   实例属性在 __init__ 里用 self.xxx 定义

② 再说共享方式：
   类属性所有实例共享同一份；
   实例属性每个实例各自持有一份

③ 再说查找顺序：
   读属性时先找实例 → 再找类 → 再找父类

④ 再说框架应用：
   元素定位器、默认配置用类属性；
   每个实例的状态数据用实例属性
```

---

### 五、常见追问

**Q：实例对象能改类属性吗？**

```
能"改"，但实际上改的是「给实例加了一个同名实例属性」，
类属性本身没变。
```

```python
class TestCase:
    timeout = 30

tc1 = TestCase()
tc1.timeout = 60   # 给 tc1 加了个实例属性 timeout=60

tc1.timeout        # 60  ← 读到的是实例属性
TestCase.timeout  # 30  ← 类属性本身没变！
tc2 = TestCase()
tc2.timeout        # 30  ← 其他实例不受影响
```

**要真正改类属性，必须通过类去改：**

```python
TestCase.timeout = 45   # ✅ 真正改了类属性
tc2.timeout          # 45  ← 现在能读到了
tc1.timeout          # 60  ← tc1 有自己的实例属性，不受影响
```

**删除实例属性后，又能读到类属性：**

```python
del tc1.timeout     # 删掉 tc1 的实例属性
tc1.timeout        # 45  ← 现在找不到实例属性，去类里找
```

**面试回答：**

```
实例对象"改"类属性，
实际上是在实例上创建了一个同名实例属性，
类属性本身没变。

要真正改类属性，
必须通过类名去改：ClassName.attr = xxx。
```

---

**关键词**：实例属性 / 类属性 / self / __init__ / 属性查找顺序 / 可变类属性陷阱 / 实例改类属性

---

## Q36. 你对 pytest 框架了解到什么程度？

### 问题描述

面试官问对 pytest 的了解程度——不只考用了哪些功能，更在考你对 **pytest 运行原理**和**实际工程化经验**的掌握深度。

---

### 答案

#### 一句话核心

> **会用 fixture/parametrize 是基础；理解 scope 生命周期、conftest 自动发现、钩子函数才是进阶。**

---

### 一、回答框架：分三层展示深度

```
我对 pytest 的了解可以分三个层次：

第一层：会用（基础）
  会写用例、用 fixture 做前后置、
  @pytest.mark.parametrize 做参数化、
  用 pytest.ini 管理配置和 markers。

第二层：理解原理（进阶）
  知道 fixture 的 scope 生命周期、
  conftest.py 的自动发现机制、
  pytest 的插件系统（pytest-rerunfailures、
  allure-pytest 都是插件）。

第三层：能扩展（高级）
  会用 pytest_runtest_makereport 钩子
  收集测试结果、能写 conftest 共享 fixture、
  能把 pytest 和 Jenkins + Allure 集成。

目前我在第二层到第三层之间，
框架搭建和 CI 集成都有实际经验。
```

---

### 二、结合 ERP 项目的真实案例

#### 案例1：fixture scope 管理登录态（session 级只登一次）

```python
@pytest.fixture(scope="session")
def login_token():
    token = login("admin", "123456")
    yield token
    logout(token)   # teardown：全部用例跑完才登出

def test_xxx(login_token):
    token = login_token   # 所有用例共享同一个 token
```

**面试加分：** 说清楚 `scope="session"` 的含义，以及为什么登录不用默认的 `function`（每次都登太慢）。

---

#### 案例2：conftest.py 自动发现机制

```
testcases/
├── conftest.py          # 根目录：放全局 fixture
├── test_login.py
├── order/
│   ├── conftest.py    # 子目录：只给 order 模块用的 fixture
│   └── test_order.py
└── inventory/
    ├── conftest.py    # 子目录：只给 inventory 模块用的 fixture
    └── test_inventory.py
```

```python
# testcases/conftest.py（全局）
@pytest.fixture
def base_url():
    return "https://api.example.com"

# testcases/order/conftest.py（局部）
@pytest.fixture
def order_data():
    return {"product_id": 1, "quantity": 2}
```

**面试加分：** conftest 不需要 import，pytest 自动发现并注入，这个机制说清楚就是进阶水平。

---

#### 案例3：pytest.ini 管理 markers 和路径

```ini
[pytest]
markers =
    smoke: 冒烟用例
    order: 订单模块
    inventory: 库存模块
    @pytest.mark.run(order=1): 控制执行顺序（需安装 pytest-order）

testpaths = testcases
python_files = test_*.py
python_classes = Test*
python_functions = test_*

addopts =
    --reruns 1
    --reruns-delay 2
    --alluredir=allure-results
```

---

#### 案例4：钩子函数收集失败用例

```python
# conftest.py
import pytest

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    if rep.when == "call" and rep.failed:
        # 把失败用例名写进文件，Jenkins 邮件里附上
        with open("failed_cases.txt", "a") as f:
            f.write(item.nodeid + "\n")
```

---

### 三、面试打分自测（给自己定位）

| 能力 | 掌握 | 没掌握 |
|------|------|--------|
| 写用例、assert | ✅ | |
| fixture 基础用法 | ✅ | |
| parametrize 参数化 | ✅ | |
| fixture scope（session/module） | ✅ | |
| conftest 自动发现 | ✅ | |
| pytest.ini 配置 | ✅ | |
| 钩子函数（makereport 等） | ✅ | |
| 写自定义 plugin | | ❌ | |
| pytest-xdist 分布式执行 | | ❌ | |

**面试话术：**

```
满分10分我给自己打7-8分。
基础用法和工程化集成我都熟，
钩子函数也有实战经验。
唯一还没在生产环境用过的
是 pytest-xdist 分布式执行，
原理我了解，还没机会实践。
```

---

### 四、高频追问

**Q：fixture 的 scope 有哪些？**

```
function：每个用例都执行一次（默认）
class：每个测试类执行一次
module：每个模块（.py文件）执行一次
session：整个测试会话只执行一次
```

**Q：fixture 的 autouse=True 有什么用？**

```python
@pytest.fixture(autouse=True)
def setup_env():
    # 每个用例自动执行，不用显式传参
    print("开始执行用例...")

# 所有用例都会自动跑 setup_env，不用写 def test_xxx(setup_env):
```

---

**关键词**：pytest / fixture / scope / conftest / parametrize / 钩子函数 / makereport / pytest.ini / 自测打分

---

## Q37. pytest 的 fixture 是怎么管理的？结合项目说一下。

### 问题描述

面试官追问 fixture 的具体管理方式——不只考你会用 `@pytest.fixture`，更在考你是否有**工程化的 fixture 管理经验**，以及怎么保证用例之间不互相污染。

---

### 答案

#### 一句话核心

> **fixture 管理 = 按 scope 分层 + 集中到 conftest.py + yield 做数据清理。**

---

### 一、结合 ERP 项目的真实做法

#### 做法1：按 scope 分层，避免重复执行

```python
# scope=session：整个测试会话只执行一次（登录态）
@pytest.fixture(scope="session")
def login_token():
    token = login("admin", "123456")
    yield token
    logout(token)   # 全部用例跑完才登出

# scope=module：每个 .py 文件执行一次（模块级数据准备）
@pytest.fixture(scope="module")
def setup_module_data():
    # 给当前模块准备测试数据
    yield
    # teardown：清理模块级数据
```

**为什么这么做：** 登录每个用例都跑太慢，`scope="session"` 让整个会话只登一次，90条接口用例原来要登90次，现在只登1次，时间省很多。

---

#### 做法2：conftest.py 按目录分层管理

```
testcases/
├── conftest.py              # 全局 fixture（所有模块都能用）
├── test_login.py
├── order/
│   ├── conftest.py         # 订单模块专属 fixture
│   └── test_order.py
└── inventory/
    ├── conftest.py         # 库存模块专属 fixture
    └── test_inventory.py
```

```python
# testcases/conftest.py（全局）
@pytest.fixture(scope="session")
def login_token():
    ...

@pytest.fixture
def base_url():
    return "https://api.example.com"

# testcases/order/conftest.py（订单模块专属）
@pytest.fixture
def order_test_data():
    return {"product_id": 1, "quantity": 2, "warehouse_id": 10}
```

**关键点：** conftest.py 不需要 import，pytest 自动发现并注入，子目录的 conftest 会覆盖父目录的同名 fixture（如果有需要）。

---

#### 做法3：用 yield 做数据清理，保证用例不互相污染

```python
@pytest.fixture
def create_test_order(login_token, base_url):
    # setup：创建一条测试订单
    resp = requests.post(
        f"{base_url}/order",
        json={"product_id": 1, "quantity": 1},
        headers={"Authorization": f"Bearer {login_token}"}
    )
    order_id = resp.json()["data"]["order_id"]
    
    yield order_id
    
    # teardown：自动删除测试订单（不管用例成功还是失败）
    requests.delete(
        f"{base_url}/order/{order_id}",
        headers={"Authorization": f"Bearer {login_token}"}
    )

def test_cancel_order(create_test_order):
    order_id = create_test_order
    # 执行取消操作...
    # 用例结束，fixture 自动清理这条订单
```

**为什么这么做：** 如果用例失败没清理，数据库里会留下脏数据，下次跑用例可能冲突。`yield` 的 teardown 不管用例成败都会执行，保证干净。

---

### 二、面试回答话术（直接背）

```
fixture 的管理我主要做了三件事：

第一，按 scope 分层。
登录态我用 scope=session，
整个测试会话只登一次，
token 通过 fixture 传给所有用例，
不用每个用例都登录。

第二，集中到 conftest.py 管理。
全局用的 fixture（比如 base_url、login_token）
放在项目根目录的 conftest.py，
各模块专属的放在对应子目录的 conftest.py，
pytest 会自动发现，用例里直接传参就用，
不用每个文件都 import。

第三，用 fixture 的 yield 机制做数据准备和清理。
比如创建订单的用例，我在 yield 之前
创建测试数据，yield 之后自动删除，
保证用例之间不互相污染，
不管用例成功还是失败，数据都会被清理。
```

---

### 三、高频追问

**Q：conftest.py 多个目录都有，pytest 怎么找 fixture？**

```
pytest 从当前用例文件所在目录开始，
往上层目录逐级找 conftest.py，
找到第一个匹配的 fixture 就用那个。

所以子目录的 conftest 可以覆盖父目录的同名 fixture，
这个设计很灵活。
```

**Q：fixture 之间能互相依赖吗？**

```python
# ✅ 可以！fixture 可以调用其他 fixture
@pytest.fixture
def login_token():
    ...

@pytest.fixture
def auth_headers(login_token):      # ← 依赖 login_token
    return {"Authorization": f"Bearer {login_token}"}

def test_api(auth_headers):         # ← 只用 auth_headers 就行
    ...
```

---

### 四、面试加分细节

| 说法 | 为什么加分 |
|------|-------------|
| "scope=session 让登录只跑一次" | 说明你关注执行效率 |
| "conftest 按目录分层" | 说明你有工程化组织能力 |
| "yield teardown 保证数据干净" | 说明你考虑到了用例隔离 |
| "fixture 之间可以依赖注入" | 说明你理解 pytest 的设计思想 |

---

**关键词**：fixture / scope / conftest.py / yield / teardown / 数据隔离 / 依赖注入

---

## Q38. fixture 写在 test 文件里，其他文件能 import 过来用吗？

### 问题描述

面试官追问 fixture 的引用方式——技术上能不能 import，以及为什么 pytest 不推荐这么做。

---

### 答案

#### 一句话核心

> **技术上可以 import，但强烈不推荐——会重复收集用例、绕过依赖注入机制。正确做法是放 conftest.py。**

---

### 一、为什么不推荐 import fixture？

```python
# test_order.py
import pytest

@pytest.fixture
def order_data():
    return {"product_id": 1, "quantity": 2}

# test_inventory.py（另一个文件）
from test_order import order_data   # ❌ 可以跑，但很糟糕
```

**三个问题：**

**① 用例重复收集**

```
pytest 扫描目录时把 test_order.py 里的用例收集一次，
你 from test_order import ... 后，
pytest 可能又把 test_order.py 收集一次，
导致用例重复执行。
```

**② 绕过依赖注入机制**

```
fixture 的设计思想是"声明依赖、自动注入"——
用例只声明需要什么，pytest 负责找。

手动 import 等于绕过了这个机制，
代码耦合变高，维护成本增加。
```

**③ conftest.py 就是专门解决这个问题的**

```
把 fixture 放 conftest.py，
所有测试文件自动可用，不用 import，
这是 pytest 官方推荐的做法。
```

---

### 二、正确的做法

#### 场景1：fixture 只给当前文件用 → 直接写在 test 文件里

```python
# test_order.py

@pytest.fixture
def order_test_data():
    return {"product_id": 1, "quantity": 2}

def test_create_order(order_test_data):
    # 只有这个文件里的用例能用
    ...
```

#### 场景2：fixture 需要跨文件共享 → 放 conftest.py

```python
# testcases/conftest.py

@pytest.fixture
def login_token():
    ...

@pytest.fixture
def base_url():
    return "https://api.example.com"

# test_order.py 和 test_inventory.py 都能直接用，不用 import
def test_xxx(login_token, base_url):
    ...
```

#### 场景3：fixture 只给某个模块用 → 放子目录的 conftest.py

```
testcases/
├── conftest.py              # 全局
├── order/
│   ├── conftest.py         # 只给 order 模块用
│   └── test_order.py
└── inventory/
    ├── conftest.py         # 只给 inventory 模块用
    └── test_inventory.py
```

---

### 三、面试回答话术（直接背）

```
技术上可以 import 另一个 test 文件里的 fixture，
但我不这么做，有两个问题：

一是 import 另一个 test 文件，
pytest 可能收集到重复的用例，导致重复执行；

二是绕过了 pytest 的依赖注入机制，
fixture 的意义就是"声明依赖、自动注入"，
手动 import 等于把这个优势丢了。

正确的做法是把公共 fixture
放到 conftest.py 里，
pytest 会自动发现并注入，
不用任何 import，这是官方推荐的做法。

如果 fixture 只给当前文件用，
直接写在 test 文件里也没问题。
```

---

### 四、高频追问

**Q：conftest.py 一定要放在项目根目录吗？**

```
不一定。conftest.py 可以放在任何目录，
pytest 会自动发现所有 conftest.py，
并按目录层级注入 fixture。

子目录的 conftest 可以覆盖父目录的同名 fixture，
这个设计很灵活。
```

**Q：fixture 同名了怎么办？**

```
子目录 conftest 里的 fixture
会覆盖父目录 conftest 里的同名 fixture。

比如：
testcases/conftest.py 里有 base_url
testcases/order/conftest.py 里也有 base_url
→ order 模块下的用例用的是 order/conftest 里的版本。
```

---

**关键词**：fixture / conftest.py / import 陷阱 / 用例重复收集 / 依赖注入 / 目录分层

---

## Q39. UI 用例只有 100 条，还需要并发执行吗？

### 问题描述

面试官问 UI 用例数量不多的情况下是否需要并发——考的是你对**执行时间量化**和**CI 效率**的理解，以及是否了解 pytest-xdist。

---

### 答案

#### 一句话核心

> **并发值不值得做，看的是总执行时间，不是用例条数。UI 用例慢，100 条也需要并发。**

---

### 一、量化分析：串行 vs 并发

```python
# UI 用例执行时间估算
UI单条执行时间 ≈ 10-20秒（含页面加载、元素等待）
100条串行 = 100 × 15秒 ≈ 25分钟

# 开4个worker并发
100条并发 ≈ 25分钟 / 4 ≈ 6分钟

# 接口用例执行时间估算
接口单条执行时间 ≈ 1-2秒
90条串行 = 90 × 1.5秒 ≈ 2分钟

# 结论：UI用例并发收益大，接口用例并发收益小
```

**面试回答话术：**

```
100条UI用例确实不算多，
但UI用例执行时间长，每条大概10-20秒，
100条串行跑下来要20-30分钟，
在CI里这个时间偏长了。

用 pytest-xdist 开4个worker 并发，
可以压到5-8分钟，
这个收益是明显的。

接口用例更快一些，单条1-2秒，
90条串行大概2-3分钟，
并发的收益相对小一点，
但CI里能快几十秒也算有价值。

所以我实际是：
UI用例必开并发，
接口用例可选，看 pipeline 整体时间够不够。
```

---

### 二、pytest-xdist 基本用法

```bash
# 安装
pip install pytest-xdist

# 用4个worker并发执行
pytest -n 4

# 自动检测CPU核心数
pytest -n auto
```

```python
# pytest.ini 里配置
[pytest]
addopts = -n 4
```

---

### 三、并发执行的坑（面试加分）

**坑1：用例之间共享数据会互相踩踏**

```python
# ❌ 并发时多个worker用同一个测试账号，会冲突
def test_login():
    # worker1 在改密码，worker2 在登录 → 失败

# ✅ 解决方式1：只读用例并发，写操作串行
@pytest.mark.run_in_order
def test_create_order():
    ...

# ✅ 解决方式2：给每个worker分配不同的测试数据
def test_login(test_user):
    # test_user 按worker ID分配不同账号
    ...
```

**坑2：用例执行顺序不确定**

```python
# ❌ 并发时用例执行顺序不保证
# 如果用例A依赖用例B的执行结果 → 失败

# ✅ 解决：用例之间完全解耦，不依赖执行顺序
# 每条用例自己准备数据、自己清理
```

**坑3：测试报告聚合**

```python
# 多个worker各自生成报告，需要聚合
# allure 支持自动聚合，没问题
# 自己写的报告要注意这个结果合并
```

---

### 四、如果没用过 xdist，怎么回答？

```
并发执行我了解原理，但生产环境还没部署过。

我的理解是 pytest-xdist 通过
多进程并行执行用例，适合UI用例
执行时间长的场景。

目前我们UI用例100条，
串行跑大概20多分钟，
确实有优化空间，
如果有机会我打算引入 xdist 做这个优化。

我也了解到并发有个坑——
用例之间如果有共享数据会互相踩踏，
所以并发一般只跑只读用例，
有写操作的用例要串行跑。
```

---

### 五、面试加分细节

| 说法 | 为什么加分 |
|------|-------------|
| "100条UI用例串行约25分钟" | 有量化意识 |
| "UI必开并发，接口可选" | 区分场景，不盲目 |
| "并发只跑只读用例" | 知道坑在哪里 |
| "xdist 的 -n auto" | 有实际了解 |

---

**关键词**：pytest-xdist / 并发执行 / worker / 用例隔离 / UI自动化效率 / CI优化

---

## Q40. pytest-xdist 有几种并发策略？结合实际说一下。

### 问题描述

面试官追问 pytest-xdist 的并发策略——不只考会用 `-n 4`，更在考你是否**根据项目结构选择合适的分发策略**。

---

### 答案

#### 一句话核心

> **默认 loadfile 按文件分发；loadscope 按类/模块分发；实际选哪个看用例结构。**

---

### 一、三种并发策略

#### 策略1：`loadfile`（默认）

```bash
pytest -n 4 --dist=loadfile
```

```
分发逻辑：把不同的 .py 文件分给不同的 worker。
同一文件的用例给同一个 worker 串行执行。

适合：每个文件用例量均匀。
问题：某个文件特别大，worker 负载不均。
```

---

#### 策略2：`loadscope`

```bash
pytest -n 4 --dist=loadscope
```

```
分发逻辑：把同一个类/模块分给同一个 worker。
同一类里的用例串行，不同类可以并行。

适合：POM 结构（同一页面类的用例放一个类里），
     保证 fixture scope=class 不会冲突。
问题：同一类的用例必须串行，并发度下降。
```

---

#### 策略3：手动分组（不常用）

```bash
# 用 pytest-collect-only 先收集，再手动分组
# 或者通过 pytest.ini 的 markers 分组执行
pytest -n 2 -m "order"      # worker1 跑订单模块
pytest -n 2 -m "inventory"  # worker2 跑库存模块
```

---

### 二、结合你的项目结构选择

```python
# 你们的用例结构
testcases/
├── test_login.py              # 少量用例（5条）
├── order/
│   └── test_order.py        # 订单模块（30条）
└── inventory/
    └── test_inventory.py    # 库存模块（20条）

# 分析：
# - 文件数量少，但每个文件用例量不均匀
# - test_order.py 有30条，单独给一个 worker 会忙死
# - 其他文件少，worker 空闲
```

**优化方案：把大文件拆小**

```python
# 拆之前（分发不均）
test_order.py     # 30条 → worker1 忙，worker2/3/4 闲

# 拆之后（分发均匀）
test_order_create.py    # 10条
test_order_cancel.py    # 10条
test_order_query.py     # 10条
# → 4个文件分给4个worker，负载均匀
```

---

### 三、面试回答话术（直接背）

```
pytest-xdist 的并发策略我了解两种：

一种是默认的 loadfile，按文件分发，
不同文件给不同 worker，
同一文件的用例串行执行。
这个适合我们现在的项目结构，
每个模块一个文件，分发比较均匀。

另一种是 loadscope，按类/模块分发，
同一类的用例给同一个 worker。
这个适合 POM 结构——
同一页面类的用例放一个类里，
保证 fixture 的 scope=class 不会冲突。

实际我用默认的 loadfile 就够了，
如果某个文件用例特别多，
我会拆成多个文件让分发更均匀。
```

---

### 四、高频追问

**Q：并发时 fixture 的 scope=session 会有问题吗？**

```
会有问题。
scope=session 的 fixture 在每个 worker 里
各自执行一次，不是全局只执行一次。

比如登录 fixture scope=session，
4个worker 会登4次，不是只登1次。

这个是正常的，每个 worker 是独立进程。
```

**Q：并发时怎么保证测试数据不冲突？**

```
两个做法：

一是只读用例并发，写操作串行，
通过 pytest 的 --forked 或者
把写用例单独放一个文件用 -m 标记串行跑。

二是给每个 worker 分配不同的测试数据，
比如 worker1 用 test_user_1，
worker2 用 test_user_2，
通过 pytest 的 worker ID 环境变量区分：
os.environ.get("PYTEST_XDIST_WORKER")
```

---

### 五、面试加分细节

| 说法 | 为什么加分 |
|------|-------------|
| "默认 loadfile，大文件要拆小" | 有实际优化经验 |
| "loadscope 保证 fixture 不冲突" | 理解并发的坑 |
| "scope=session 每个 worker 各执行一次" | 理解 xdist 原理 |
| "用 markers 把写操作单独串行跑" | 有工程化思维 |

---

**关键词**：pytest-xdist / loadfile / loadscope / worker负载均衡 / 用例拆分 / 并发数据隔离

---

## Q41. 接口测试的流程是什么？从需求到上线的完整步骤？

### 问题描述

面试官问"接口测试流程"，不是想听"看文档→写用例→执行"这种三句话概括——他要看你是不是有系统化的方法论，是从需求分析到持续集成的全闭环思维，还是只会调Postman发请求。

> 这是接口测试岗的**必考题**，答得好直接拉高整场面试的段位定位。

---

### 一、回答总框架：六阶段模型

```
需求分析 → 用例设计 → 环境准备 → 自动化实现 → 执行跟踪 → 回归集成
```

---

### 二、面试回答脚本（约3分钟，完整版）

#### 第一阶段：需求分析 & 文档理解

> "拿到需求后，我首先看接口文档——我们用的是Swagger或者内部Wiki格式的API文档。我关注四个东西：接口的URL和请求方式、入参和出参的字段定义、状态码的含义、以及接口之间的依赖关系（比如这个接口的请求参数需要上一个接口的返回值）。
>
> 如果文档不全或者有歧义，我会在这个阶段直接找开发确认，不等到执行时再发现问题。"

**这个阶段的关键**：面试官想听的不是你"看了文档"，而是你知道文档可能存在什么问题，以及你怎么处理——提前沟通、不等到执行时才发现。

#### 第二阶段：用例设计（四维度覆盖）

> "用例设计我按四个维度覆盖：
>
> **一是正常场景**：正常参数请求，验证返回值的字段、类型、数据是否正确；
>
> **二是异常场景**：必填参数为空、参数类型错误（字符串传数字位置）、参数超长、非法值（负数/特殊字符）——验证接口有没有做参数校验，错误提示是否清晰；
>
> **三是边界值**：金额的小数点精度、数量的最大/最小值、时间字段的边界（跨天/跨月/跨年）；
>
> **四是业务逻辑**：比如'只有状态为审核通过才能提交'——验证状态机流转是否正确，以及数据一致性——接口返回成功了，数据库里的数据是否真的变了。"

**这个阶段的关键**：不要只说"正常和异常"，要说清楚异常包含哪些类型（参数空、类型错、超长、非法值），以及业务逻辑层面的数据一致性——这是中级和初级的区分线。

#### 第三阶段：测试环境准备

> "环境准备我做三件事：
>
> 第一，确认测试环境的服务是否正常——调一个公共查询接口看通不通；
>
> 第二，准备前置数据——有些接口依赖特定数据（比如测'审核接口'要先有一笔待审核的单据），我用数据库直接插数据或者调前置接口来准备；
>
> 第三，处理鉴权——大多数接口需要token，我先请求登录接口拿token，用fixture管理，避免每条用例都重复登录。"

**这个阶段的关键**：前置数据管理和鉴权处理是接口测试的核心痛点，能把这两点说出来说明你不是纸上谈兵。

#### 第四阶段：用例编写 & 自动化实现

> "我用Pytest + Requests写接口自动化用例，按分层架构组织代码：
>
> - `base`层封装HTTP请求方法，做统一的日志记录和响应处理
> - `api`层每个接口一个方法，不直接在用例里写requests.get/post
> - `testcases`层写断言逻辑，调`api`层的方法
> - `data`层用YAML管理测试数据，不硬编码
>
> 断言不只断状态码200，还断返回的业务code、关键字段的值、以及对数据库的二次校验。"

**这个阶段的关键**：说分层架构，而不是"我用Pytest写用例"——分层体现的是工程化思维和可维护性设计。

#### 第五阶段：执行 & 缺陷跟踪

> "用例写完在本地先跑通，然后提交到Jenkins，配置了定时触发和代码push触发。执行完生成Allure报告，失败用例我按两个方向判断：接口本身的bug，还是测试数据/环境问题。确认是bug后在Jira上提单，写清楚：请求参数、实际返回、期望返回、复现步骤。"

**这个阶段的关键**：区分"环境问题"和"代码Bug"——这个判断能力说明你真的在线上遇到过失败，不是只会跑用例。

#### 第六阶段：回归 & 持续集成

> "每次版本发布前跑一遍全量回归，确认新功能没有破坏老接口。如果有接口改动，我会同步更新自动化用例。这一步通过Jenkins + 邮件通知，跑失败了开发和测试都能第一时间收到报告。"

---

### 三、收尾金句（一句话提升段位）

> "整个流程下来，我最看重的是两个点：一是用例设计的异常覆盖——很多bug藏在边界值和异常参数里，正常场景跑通不代表接口质量好；二是断言深度——只断状态码200是不够的，数据一致性才是业务测试的核心。"

---

### 四、面试官可能追问 & 应答

| 追问 | 核心应答要点 |
|:---|:---|
| "你怎么处理接口依赖？" | session级别的fixture管理token；有依赖关系的接口用fixture级联，或者conftest里共享data |
| "接口测试和抓包有什么关系？" | 抓包（Fiddler/Charles）是辅助手段——接口文档不全时用抓包补充；排查接口问题时看实际收发包 |
| "你怎么验证数据一致性？" | 接口返回成功后，直接连测试库查对应数据，用pytest的assert语句断言数据库字段 |
| "接口测试覆盖率怎么算？" | 按核心业务链路100%覆盖，次要功能50-70%，测试价值低的内部工具类接口不做自动化 |
| "接口文档不存在或者不准怎么办？" | 抓包+和开发对齐，同时把这个作为测试风险记录在测试计划里 |
| "前置数据管理遇到过什么问题？" | 并发执行时数据冲突、前一天测试把数据清了没重建——后来加了fixture级前置健康检查和动态数据生成解决 |

---

### 五、面试加分 & 减分对照

| ⭐ 加分说法 | 为什么加分 |
|:---|:---|
| "断言不只断状态码200，还断业务code和数据库数据" | 有深度 |
| "前置数据用fixture管理，不硬编码" | 有工程化思维 |
| "失败先判断是环境问题还是代码bug" | 有排查经验 |
| "数据一致性校验——接口返回成功不代表数据落库正确" | 知道业务测试的核心 |
| "Jenkins + Allure，定时触发+push触发" | 有CI经验 |

| ❌ 减分说法 | 为什么减分 |
|:---|:---|
| "就是看文档，发请求，看返回码" | 太笼统，像没做过 |
| "异常场景就是传个空参数" | 异常覆盖太浅 |
| 不提数据一致性和数据库校验 | 测试深度不够 |
| 不提分层架构和代码组织 | 没有工程化经验 |

---

### 六、段位自检

| 段位 | 特征 |
|:---|:---|
| 初级 | 只说"看文档、发请求、看状态码200" |
| 中级 | 能说出正常/异常/边界值的覆盖，知道用Pytest写自动化 |
| **你应该达到的** | **说出数据一致性校验、前置数据管理、CI集成、分层架构设计的完整闭环** |

---

**关键词**：接口测试流程 / 六阶段模型 / 用例四维度 / 数据一致性 / 分层架构 / 前置数据管理 / Jenkins持续集成


---

## Q42. 接口请求的Token是否加密？Base64和加密有什么区别？

### 问题描述

面试官问"你们的Token加密了吗"，乍看是技术细节题，其实是**分层理解题**——要区分传输加密、编码签名、数据加密三个不同层面的概念。80%的人会混淆Base64编码和加密。

> 这道题被问到的概率很高，因为Token是接口测试最基础的组件，答不透说明你对底层机制理解不够。

---

### 一、核心结论（先说结论）

| 层面 | Token是否加密 | 说明 |
|:---|:---:|:---|
| Token本身 | ❌ 通常不加密 | Token就是一个凭证字符串，内容是明文或Base64编码 |
| 传输过程 | ✅ HTTPS/TLS加密 | 整个HTTP通信被TLS加密，抓包看到的是密文 |
| JWT Signature | ✅ 有签名 | 签名用来防篡改，不是用来隐藏内容的 |
| 敏感数据 | ❌ 不应存在Token里 | Payload用Base64编码，任何人都能解码，不能放密码/手机号 |

---

### 二、面试回答脚本（约1.5分钟）

> "Token本身通常是不加密的——它就是一个凭证字符串。真正的安全分两层来理解：
>
> **第一层是传输加密**：Token在HTTP请求里一般放在Header中，通过HTTPS/TLS对整个通信链路加密。别人抓包看到的是加密后的密文，解不开Token内容。所以不是Token自己加密了，是传输管道加密了。
>
> **第二层是Token的编码和签名**：以JWT为例，它由三段组成——Header、Payload、Signature。Payload部分是Base64编码，**编码不是加密**，任何人都能解码看到里面的内容。所以JWT的Payload里绝不能放敏感信息（密码、手机号等），只能放用户ID、角色、过期时间这些。Signature是HMAC或RSA签名，用来防篡改，不是用来隐藏内容的。
>
> **那什么情况下Token需要额外加密？** 如果业务要求Payload里必须存敏感数据（比如一些合规场景），会用JWE规范对Payload做真正的AES加密，但这是少数情况。大多数系统的做法是把敏感数据放服务端Session或数据库里，Token里只存一个ID做索引。"

---

### 三、Base64编码 vs 加密（核心区分）

| 对比维度 | Base64编码（Encode） | 加密（Encrypt） |
|:---|:---|:---|
| 英文关键词 | `encode` / `decode` | `encrypt` / `decrypt` |
| 是否需要密钥 | ❌ 不需要，公开算法 | ✅ 需要密钥 |
| 能否还原 | ✅ 任何人都能解码 | ❌ 没有密钥解不开 |
| 目的 | 解决二进制在文本协议中的传输问题 | 保护数据机密性 |
| 举例 | JWT的Payload部分 | HTTPS的TLS加密 |

**一个例子说清楚**：
```
原始内容：{"userId": "12345"}
↓ Base64编码
eyJ1c2VySWQiOiAiMTIzNDUifQ==  ← 复制到 jwt.io 直接解码，不需要密码

↓ AES加密（需要密钥）
a8f5b3c1d2e4...  ← 没有密钥完全解不开
```

---

### 四、JWT三段的含义

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiIxMjM0NSJ9.abc123def456...

  Header          Payload          Signature
  (算法类型)       (数据-Base64)     (签名-HMAC SHA256)
```

| 部分 | 格式 | 是否加密 | 作用 |
|:---|:---|:---:|:---|
| Header | Base64编码 | ❌ | 声明签名算法（HS256/RS256） |
| Payload | Base64编码 | ❌ | 存放Claims（用户ID、过期时间、角色） |
| Signature | HMAC/RSA签名 | 签名≠加密 | 防篡改——改了Payload算不出相同签名 |

> ⚠️ **关键坑**：Payload是Base64编码，任何人拿到JWT都能在 jwt.io 上解码看到Payload内容。所以**密码、身份证号、银行卡号绝不能放在JWT的Payload里**！

---

### 五、Token安全的三个支柱

| 措施 | 具体做法 | 防止什么 |
|:---|:---|:---|
| **HTTPS传输** | 全站HTTPS，强制TLS 1.2+ | 中间人窃听Token |
| **Payload不含敏感数据** | Token里只存userId+exp，不存密码/手机号 | Token泄露后信息暴露最小化 |
| **短期有效 + 刷新机制** | Access Token 30分钟，Refresh Token 7天 | 泄露后攻击窗口有限 |

---

### 六、面试官可能的追问

| 追问 | 应答要点 |
|:---|:---|
| "你说JWT的Payload能解码，那我怎么保证Token安全？" | 安全靠三件事：①传输层用HTTPS防中间人窃听 ②Payload不用Base64存敏感信息 ③Token设置合理的过期时间，比如30分钟 |
| "Access Token和Refresh Token有什么区别？" | Access Token短期有效（30分钟），放请求里访问接口；Refresh Token长期有效（7天），只用来换新Access Token |
| "Token在客户端怎么存？有什么安全问题？" | Web端存HttpOnly Cookie最安全，localStorage有XSS风险；移动端存Keychain/KeyStore等系统安全存储 |
| "你们的Token用的是什么形式？有过期时间吗？" | JWT格式的Bearer Token，有过期时间。自动化测试里用fixture管理，定时刷新。之前遇到过一个坑——并发跑时多条用例同时刷新Token冲突，改成session级别fixture解决 |
| "JWE是什么？和JWT什么关系？" | JWE（JSON Web Encryption）是JWT的加密版——Payload真正用AES加密，解不开看不到内容。用于合规要求较高的场景，性能开销比JWT大 |

---

### 七、加分 & 减分对照

| ⭐ 加分说法 | 为什么加分 |
|:---|:---|
| "Base64是编码不是加密，任何人都能解码" | 概念清晰，大多数人会混淆 |
| "Payload不能放敏感信息，加密靠HTTPS传输层" | 理解分层安全模型 |
| "JWT的Signature是签名，防篡改不是加密" | 区分签名和加密 |
| "遇到过并发刷新Token的冲突问题" | 有真实经验 |

| ❌ 减分说法 | 为什么减分 |
|:---|:---|
| "Token是加密的" / "Base64就是加密" | 概念混淆，基础不扎实 |
| "Token放在URL参数里传" | 有安全风险——URL会被浏览器历史和服务端日志记录 |
| 完全回避这个问题 | 不敢答=完全不懂 |

---

**关键词**：Token加密 / Base64 / JWT / HTTPS / 传输层安全 / Payload / 签名验证 / Access Token / Refresh Token

