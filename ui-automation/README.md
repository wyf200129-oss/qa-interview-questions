# UI 自动化面试题

---

## Q1. 结合 ERP 业务说一下 POM 设计模式

### 问题描述

面试官问：你做的 UI 自动化用的是什么设计模式？POM（Page Object Model）是什么？结合你的 ERP 业务说说你怎么做的。

### 答案

POM 是 UI 自动化最主流的设计模式，核心思想一句话：

> **一个页面一个类，把页面元素和操作封装起来，测试用例只调方法不碰元素定位。**

---

### 一、为什么要用 POM（不用 POM 的痛）

假设你直接写 Selenium，不看模式：

```python
# ❌ 没有 POM —— 每次操作都要重复写元素定位
def test_create_outbound():
    driver = webdriver.Chrome()
    driver.get("http://erp.example.com/wms/outbound")

    # 每次操作都直接写 xpath
    driver.find_element(By.XPATH, "//button[text()='新建出库单']").click()
    driver.find_element(By.ID, "sku_select").click()
    driver.find_element(By.XPATH, "//li[contains(text(), 'SKU001')]").click()
    driver.find_element(By.NAME, "quantity").send_keys("100")
    driver.find_element(By.XPATH, "//button[text()='提交审核']").click()

# 问题：
# 1. 元素定位散落在各个测试用例里，改一个 class 要改 N 个地方
# 2. 测试用例又长又难读，不知道在测什么业务
# 3. 新人接手根本看不懂
```

**用了 POM 之后**：

```python
# ✅ 用了 POM —— 测试用例只关心业务，不关心中间细节
def test_create_outbound():
    page = OutboundPage(driver)
    page.create_outbound(sku="SKU001", quantity=100, warehouse="主仓")
    page.submit_audit()
    assert page.get_audit_status() == "审核通过"
```

---

### 二、POM 怎么搭（结合 ERP 业务举例）

**三层结构**：

```
ui-automation/
├── pages/                    ← 页面对象层（POM 核心）
│   ├── base_page.py          ← 基类（通用操作）
│   ├── login_page.py         ← 登录页
│   ├── outbound_page.py      ← 出库管理页
│   ├── sale_order_page.py    ← 销售单页
│   └── purchase_page.py      ← 采购管理页
├── testcases/                ← 测试用例层（只调 page 方法）
│   ├── test_outbound.py
│   └── test_sale.py
└── conftest.py               ← fixture（driver 管理）
```

#### Layer 1：BasePage（基类，封装通用操作）

```python
# pages/base_page.py
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By

class BasePage:
    """所有页面的基类：封装通用的等待和操作方法"""

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

    def click(self, locator):
        """显式等待元素可点击后点击"""
        element = self.wait.until(EC.element_to_be_clickable(locator))
        element.click()

    def input(self, locator, text):
        """显式等待元素可见后清空并输入"""
        element = self.wait.until(EC.visibility_of_element_located(locator))
        element.clear()
        element.send_keys(text)

    def get_text(self, locator):
        """获取元素文本"""
        element = self.wait.until(EC.visibility_of_element_located(locator))
        return element.text

    def wait_toast(self, text):
        """等待 Toast 提示出现"""
        self.wait.until(
            EC.visibility_of_element_located((By.XPATH, f"//*[contains(text(),'{text}')]"))
        )
```

#### Layer 2：PageObject（ERP 业务页面）

**以出库管理页为例**：

```python
# pages/outbound_page.py
from selenium.webdriver.common.by import By
from pages.base_page import BasePage

class OutboundPage(BasePage):
    """出库管理页面"""

    # ── 页面元素定位（集中管理，改一个地方就行） ──
    BTN_NEW_OUTBOUND = (By.XPATH, "//button[text()='新建出库单']")
    BTN_SUBMIT_AUDIT = (By.XPATH, "//button[text()='提交审核']")
    BTN_CANCEL = (By.XPATH, "//button[text()='作废']")

    INPUT_SKU = (By.ID, "sku_select")
    INPUT_QUANTITY = (By.NAME, "quantity")
    INPUT_WAREHOUSE = (By.ID, "warehouse_select")

    LABEL_AUDIT_STATUS = (By.CLASS_NAME, "audit-status")
    LABEL_STOCK_REMAINING = (By.CLASS_NAME, "stock-remaining")
    TOAST_SUCCESS = "操作成功"

    # ── 页面操作（测试用例调这些方法） ──
    def click_new_outbound(self):
        """点击「新建出库单」按钮"""
        self.click(self.BTN_NEW_OUTBOUND)

    def select_sku(self, sku_code):
        """选择商品"""
        self.click(self.INPUT_SKU)
        self.click((By.XPATH, f"//li[contains(text(), '{sku_code}')]"))

    def input_quantity(self, quantity):
        """输入出库数量"""
        self.input(self.INPUT_QUANTITY, str(quantity))

    def select_warehouse(self, warehouse):
        """选择仓库"""
        self.click(self.INPUT_WAREHOUSE)
        self.click((By.XPATH, f"//li[contains(text(), '{warehouse}')]"))

    def submit_audit(self):
        """提交审核"""
        self.click(self.BTN_SUBMIT_AUDIT)
        self.wait_toast(self.TOAST_SUCCESS)

    def cancel_outbound(self):
        """作废单据"""
        self.click(self.BTN_CANCEL)

    def get_audit_status(self):
        """获取审核状态"""
        return self.get_text(self.LABEL_AUDIT_STATUS)

    def get_stock_remaining(self):
        """获取剩余库存"""
        return self.get_text(self.LABEL_STOCK_REMAINING)

    # ── 复合操作（一次调多个方法，封装业务流程） ──
    def create_outbound(self, sku, quantity, warehouse="主仓"):
        """封装「创建出库单」全流程"""
        self.click_new_outbound()
        self.select_sku(sku)
        self.input_quantity(quantity)
        self.select_warehouse(warehouse)

    def full_flow_create_and_audit(self, sku, quantity):
        """封装「创建 + 审核」完整流程"""
        self.create_outbound(sku, quantity)
        self.submit_audit()
```

#### Layer 3：测试用例（只调 Page 方法，不碰元素定位）

```python
# testcases/test_outbound.py
import pytest
from pages.outbound_page import OutboundPage, StockPage
from pages.sale_order_page import SaleOrderPage

class TestOutbound:
    """出库管理测试用例"""

    def test_normal_outbound(self, driver):
        """正常出库：创建出库单 + 审核"""
        page = OutboundPage(driver)
        page.create_outbound(sku="SKU001", quantity=100, warehouse="主仓")
        page.submit_audit()
        assert page.get_audit_status() == "审核通过"

    def test_insufficient_stock(self, driver):
        """异常场景：库存不足，审核不通过"""
        page = OutboundPage(driver)

        # 先查当前库存
        stock_page = StockPage(driver)
        current_stock = int(stock_page.get_stock("SKU001"))

        # 出库数量设为库存+1
        page.create_outbound(sku="SKU001", quantity=current_stock + 1)
        page.submit_audit()

        assert page.get_audit_status() == "审核不通过"
        assert "库存不足" in page.get_error_message()

    def test_cancel_restore_stock(self, driver):
        """数据一致性：作废销售单后库存应该恢复"""
        stock_page = StockPage(driver)

        # 记录操作前库存
        stock_before = int(stock_page.get_stock("SKU001"))

        # 创建销售单 → 审核（库存扣减）
        sale_page = SaleOrderPage(driver)
        sale_page.create_order(sku="SKU001", quantity=10)
        sale_page.submit_audit()

        # 作废
        sale_page.cancel_order()

        # 验证库存是否恢复
        stock_after = int(stock_page.get_stock("SKU001"))
        assert stock_after == stock_before, \
            f"作废后库存未恢复！操作前:{stock_before} 操作后:{stock_after}"
```

---

### 三、POM 给你带来的实际好处

| 好处 | 没有 POM | 有了 POM |
|:---|:---|:---|
| **改一个元素定位** | 改 20 个测试文件 | 改 1 个 Page 类的 1 行 |
| **新人接手工件** | 看不懂代码 | 先看 page 就知道页面有哪些操作 |
| **测试用例可读性** | 满屏 xpath | 满屏业务方法（create_outbound） |
| **复用** | copy-paste 代码 | 调同一个 page 方法就行 |

---

### 四、面试回答模板

> **"我做的 UI 自动化用的是 POM 设计模式，一个页面一个类。比如我做 ERP 的出库管理，就写了一个 OutboundPage 类，里面把页面元素定位集中管理、把出库相关的操作封装成方法——click_new_outbound、select_sku、submit_audit。**
>
> **测试用例里完全不碰元素定位，只调 page 的方法。好处是页面改版时只改 page 类，不用动测试用例。比如改了一个按钮的 class，OutboundPage 里改一行就行，不用去 20 个测试文件里一个个找。**
>
> **我还写了一个 BasePage 基类，封装了显式等待的 click、input、get_text 这些通用操作。这样所有页面类继承 BasePage 就不用重复写等待逻辑。"**

---

**关键词**：POM / Page Object Model / 页面对象模型 / ERP 架构 / 三层结构 / 元素定位管理

---

## Q2. XPath 语法在测试中怎么用？常用写法有哪些？

### 问题描述

Selenium 元素定位有 8 种方式（id、name、class、xpath、css selector...），其中 XPath 是最万能的——id 没有、name 没有、class 不唯一时，全靠 XPath 兜底。面试官问你常用哪些 XPath 写法。

### 答案

XPath 一句话：**通过路径表达式在 DOM 树里找到你想要的元素。**

---

### 一、绝对路径 vs 相对路径

```python
# ❌ 绝对路径 —— 前端改一点就挂，永远不要用
"/html/body/div[3]/div[1]/form/input[2]"

# ✅ 相对路径 —— 从任意节点开始找，稳定得多
"//input[@name='quantity']"
"//button[text()='提交审核']"
```

**铁律**：绝对路径直接扔掉，只用相对路径。

---

### 二、8 种实战常用写法（按使用频率排序）

#### 写法 1：按标签 + 属性定位（最常用）

```python
"//input[@id='sku_select']"         # id 定位
"//input[@name='quantity']"         # name 定位
"//button[@class='btn-submit']"     # class 定位
"//div[@data-testid='warehouse']"   # 自定义属性（最稳定）
```

#### 写法 2：按文本内容定位

```python
"//button[text()='提交审核']"              # 完全匹配
"//button[contains(text(),'审核')]"        # 包含文本
"//span[text()='审核通过']"               # 验证状态文本
```

**ERP 业务场景**：页面上按钮文本是"提交审核"，用 `text()` 直接定位最直观。

#### 写法 3：按层级关系定位

```python
"//div[@class='outbound-form']//input[@name='quantity']"
#  ↑ 父元素                      ↑ 子元素

"//tr[td[text()='SKU001']]//button[text()='审核']"
#  ↑ 找到 SKU001 所在行         ↑ 该行的审核按钮
```

**ERP 业务场景**：表格里每行一个商品，审核按钮没有唯一 id，用层级定位"找到 SKU001 这行里的审核按钮"。

#### 写法 4：按索引定位

```python
"(//button[text()='审核'])[1]"     # 第 1 个审核按钮
"(//button[text()='审核'])[2]"     # 第 2 个审核按钮
"(//input[@type='text'])[3]"      # 第 3 个输入框（用户名/密码/验证码）
```

#### 写法 5：按属性模糊匹配

```python
"//div[contains(@class,'audit')]"         # class 包含 audit
"//input[starts-with(@id,'warehouse_')]"  # id 以 warehouse_ 开头
"//span[ends-with(@class,'-status')]"     # class 以 -status 结尾
```

#### 写法 6：按轴（axes）找亲戚

```python
"//td[text()='SKU001']/parent::tr"                    # 当前节点的父行
"//button[text()='审核']/ancestor::tr"                 # 往上找最近的 tr
"//label[text()='仓库']/following-sibling::input"      # 同级后面的 input
"//input[@name='sku']/preceding-sibling::label"        # 同级前面的 label
```

#### 写法 7：多条件组合

```python
"//button[@class='btn' and text()='提交审核']"
"//input[@type='text' and @name='quantity' and not(@disabled)]"
"//div[@class='form-item' and contains(.,'出库数量')]//input"
```

#### 写法 8：按属性存在判断

```python
"//input[@required]"              # 所有必填输入框
"//button[@disabled]"             # 所有禁用按钮
"//input[@readonly]"              # 所有只读输入框
```

---

### 三、XPath vs CSS Selector

| 维度 | XPath | CSS Selector |
|:---|:---|:---|
| 按文本找 | ✅ `//button[text()='提交']` | ❌ 不支持 |
| 往上找（父元素） | ✅ `parent::` / `ancestor::` | ❌ 不支持 |
| 按索引 | ✅ `[1]` | ✅ `:first-child` |
| 性能 | 慢一点 | 快一点 |
| 万能程度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

**结论**：CSS Selector 用于简单定位，XPath 用于复杂场景和动态页面。

---

### 四、ERP 实战场景

```python
# 场景 1：表格操作——找到指定 SKU 那行的审核按钮
audit_btn = f"//tr[td[contains(text(),'{sku_code}')]]//button[contains(text(),'审核')]"

# 场景 2：下拉选项——选择仓库
warehouse = "//li[contains(text(),'主仓')]"

# 场景 3：Toast 提示验证
toast = "//div[contains(@class,'toast') and contains(text(),'操作成功')]"

# 场景 4：表单验证——数量输入框
quantity = "//input[@name='quantity']"

# 场景 5：动态弹窗——确认按钮
confirm = "//div[@class='modal']//button[text()='确认']"
```

---

### 五、XPath 调试技巧

```python
# 浏览器 F12 → Console → 直接测试
$x("//button[text()='提交审核']")          # 返回匹配的元素数组
$x("//input[@name='quantity']")[0].value   # 获取第一个的值
$x("count(//tr)")                          # 数表格行数
```

---

### 六、面试回答模板

> **"Selenium 定位元素我用得最多的是 XPath，因为它最万能。常用写法有：按标签+属性定位 `//input[@name='quantity']`、按文本定位 `//button[text()='提交审核']`、按层级定位 `//tr[td[text()='SKU001']]//button`、按轴找 `ancestor::` 和 `following-sibling::`。**
>
> **比如做 ERP 表格审核，每行商品没有唯一 id，我就用 `//tr[td[contains(text(),'SKU001')]]//button` 这种层级定位，找到特定商品的审核按钮。拿不准的时候在浏览器 F12 Console 里用 `$x()` 先测一遍再写到代码里。**
>
> **相比 CSS Selector，XPath 支持文本定位和向上查找，在动态页面和复杂场景下更灵活。"**

---

**关键词**：XPath / 元素定位 / Selenium / 相对路径 / 层级定位 / CSS Selector

---

## Q3. Selenium 用下来有什么缺点？结合你的项目说一下。

### 问题描述

面试官问你：Selenium 有什么缺点？这题不是让你黑 Selenium，而是看你**是否真的深入用过，能否客观评价一个工具**。

### 答案

Selenium 是 UI 自动化的老牌工具，但在 ERP 项目中确实有 5 个明显的缺点。

---

### 缺点 1：元素定位脆弱，页面一改就挂

**这是 Selenium 最大的痛点**。

```python
# 这次还能用，下周前端重构 class 改了就不行了
driver.find_element(By.XPATH, "//div[@class='outbound-form']//button[text()='审核']")

# 前端一改 → 用例全挂
# 实际中大概每 2-3 个迭代就要调一次 XPath
```

**你项目里的真实感受**：

> 之前测出库管理页，前端把按钮 class 从 `btn-submit` 改成了 `btn-primary`，我改了一个 class，10 条 UI 用例挂了 6 条。好在用了 POM 模式，OutboundPage 类里改一行就行，不用动测试用例。

**缓解办法**：
- 用 POM 集中管理元素定位
- 优先用 `id`、`name`、`data-testid`，少用 `class`
- 和前端约定：核心按钮加 `data-testid`，不要随意改

---

### 缺点 2：执行速度慢

```text
接口自动化：90 条用例 → 5 秒跑完
UI 自动化：10 条用例 → 3 分钟

Selenium 慢的原因：
  - 每次都要启动浏览器
  - 每个操作都要等待 DOM 渲染
  - 页面加载、动画、网络请求都在消耗时间
```

**你项目里的真实感受**：

> 最早做 UI 自动化时，出库审核这条用例跑了 4 分钟——等登录页加载、等列表加载、等弹窗打开。后来用显式等待替代 `time.sleep`，把多余的等待砍掉，优化到 1.5 分钟，但和接口比还是慢很多。

**缓解办法**：
- `time.sleep(3)` → `WebDriverWait`
- 只做核心页面的 UI 自动化，其他用接口覆盖
- 用无头模式（`--headless`）跑 CI

---

### 缺点 3：不稳定，偶发性失败多

Selenium 的用例经常出现"这次过、下次不过"的 Flaky 问题：

```text
常见 Flaky 原因：
  - 网络慢了，页面还没加载完就点了按钮
  - 弹窗/Toast 遮挡了目标元素
  - 动画没结束就操作了
  - CI 环境比本地慢，同样的 wait 时间不够
```

**你项目里的真实感受**：

> 之前有 10 条 UI 用例接入 Jenkins，每次跑都有 1-2 条随机失败。排查发现是 CI 环境网络差，`WebDriverWait` 的 10 秒不够。改成 20 秒后好了，但跑得更慢了。

**缓解办法**：
- `WebDriverWait` 超时适当放大（CI 环境比本地慢）
- 失败自动重跑（`pytest --reruns 2`）
- 记录失败的截图便于排查

---

### 缺点 4：多窗口/多 Tab 切换复杂

```python
# 处理多窗口切换，代码冗长
main_window = driver.current_window_handle
driver.find_element(By.LINK_TEXT, "详情").click()
driver.switch_to.window(driver.window_handles[-1])  # 切到新窗口
driver.find_element(By.ID, "confirm").click()
driver.close()
driver.switch_to.window(main_window)  # 切回原窗口
```

对比 Playwright：
```python
# Playwright 自动管理多页面
with page.expect_popup() as popup:
    page.click("text=详情")
popup.value.click("text=确认")
```

**你项目里的真实感受**：

> 测出库单详情时需要在弹窗中操作，Selenium 需要手动 `switch_to`，容易写漏。Playwright 的写法优雅得多。

---

### 缺点 5：不能拦截网络请求

Selenium 没法直接 mock 接口返回，只能通过代理（BrowserMob）或修改页面。这在需要模拟异常场景时很不方便：

```text
Selenium 里想模拟"接口返回 500"：
  → 要搭一个 mock 服务器 或 用 Charles 拦截

Playwright 里：
  → page.route("/api/xxx", lambda route: route.fulfill(status=500))
  一行搞定的功能，Selenium 要绕一大圈
```

**你项目里的感受**：

> 想测"凭证生成接口超时"的场景，Selenium 只能等真实超时或者搭 mock，很费劲。

---

### 总结：我的真实评价

| 维度 | 评价 | 说明 |
|:---|:---|:---|
| 稳定性 | ⭐⭐ | Flaky 多，维护麻烦 |
| 速度 | ⭐⭐ | 和接口自动化比慢 100 倍 |
| 生态 | ⭐⭐⭐⭐⭐ | 资料多，问题基本有解 |
| 浏览器兼容 | ⭐⭐⭐⭐ | Chrome/Edge/Firefox 都行 |
| 网络拦截 | ⭐ | 不支持，绕弯子 |
| 上手难度 | ⭐⭐⭐ | POM + 显式等待学会就行 |

**总评**：**Selenium 像螺丝刀——基础够用、成熟稳定，但你需要冲击钻（Playwright）的时候它也有局限性。**

---

### 面试回答模板

> **"Selenium 用下来最大的痛点是元素定位脆弱——前端改个 class，用例就挂。好在用了 POM 模式，元素定位集中管理，改一处就行。第二个是执行太慢，一条 UI 用例跑 1-2 分钟，和接口秒级没法比。所以我们策略是用接口自动化覆盖核心链路，UI 只做最稳定的几个页面——登录和出库审核。**
>
> **多窗口切换、网络请求拦截、Flaky 多这几点 Selenium 确实比不过 Playwright。但 Selenium 的优势是生态成熟、社区大、碰到的问题基本有解。如果项目要从零开始选，我可能会推 Playwright；但在已有 Selenium 积累的项目上，稳定跑着就不轻易换。"**

---

**关键词**：Selenium 缺点 / Flaky / 元素定位脆弱 / 执行速度 / 维护成本

---

## Q4. 有没有遇到过动态加载导致的元素定位问题？

### 问题描述

面试官问你动态加载的问题——页面数据是异步加载的，元素还没渲染出来你就去定位了，直接报 `NoSuchElementException`。这种问题你怎么处理？

### 答案

这个问题在 ERP 项目里经常遇到。我们系统是前后端分离的 Vue 项目，表格数据、下拉选项全部是 Ajax 异步加载的。踩过的坑和对策如下：

---

### 痛点 1：列表数据还没加载完就去点了

```python
# ❌ 打开页面后直接找列表元素 → 数据还没渲染，直接报错
driver.get("http://erp.example.com/wms/outbound/list")
driver.find_element(By.XPATH, "//tr[1]//button[text()='审核']").click()
# NoSuchElementException!

# ✅ 等数据加载完再操作
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

driver.get("http://erp.example.com/wms/outbound/list")

# 等第一行出现（说明列表数据加载完了）
wait = WebDriverWait(driver, 10)
first_row = wait.until(
    EC.presence_of_element_located((By.XPATH, "//tbody/tr[1]"))
)
first_row.find_element(By.XPATH, ".//button[text()='审核']").click()
```

---

### 痛点 2：下拉选项是接口返回的，点快了选不到

```python
# ❌ 点了下拉框马上选 → 选项还没加载出来
driver.find_element(By.ID, "sku_select").click()
driver.find_element(By.XPATH, "//li[text()='SKU001']").click()
# ElementNotInteractableException!

# ✅ 等选项渲染完再选
driver.find_element(By.ID, "sku_select").click()
wait.until(
    EC.visibility_of_element_located((By.XPATH, "//li[text()='SKU001']"))
).click()
```

---

### 痛点 3：Loading 动画遮盖了元素

ERP 里每次翻页/搜索都有 loading 动画，这时候点按钮会报 `ElementClickInterceptedException`：

```python
# ✅ 等 loading 消失后再操作
wait.until(
    EC.invisibility_of_element_located((By.CLASS_NAME, "el-loading-mask"))
)
driver.find_element(By.XPATH, "//tr[1]//button[text()='审核']").click()
```

---

### 痛点 4：页面局部刷新后元素变旧了（Stale Element）

这是最隐蔽的动态加载问题——你找到的元素还在，但 DOM 被 Ajax 刷新了，元素引用失效：

```python
# ❌ 审核成功后状态文字变了，之前的引用失效
status = driver.find_element(By.CLASS_NAME, "audit-status")
driver.find_element(By.XPATH, "//button[text()='提交审核']").click()
print(status.text)  # StaleElementReferenceException!

# ✅ 重新获取
status_locator = (By.CLASS_NAME, "audit-status")
status = driver.find_element(*status_locator)
driver.find_element(By.XPATH, "//button[text()='提交审核']").click()
wait.until(
    EC.text_to_be_present_in_element(status_locator, "审核通过")
)
```

---

### 你项目里的通用解法：封装等待方法

我写了一个通用的等待方法，专门应对动态加载：

```python
class BasePage:
    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 15)  # CI 环境设大一点

    def wait_and_click(self, locator):
        """等元素可点击后点击"""
        element = self.wait.until(EC.element_to_be_clickable(locator))
        element.click()

    def wait_and_input(self, locator, text):
        """等元素可见后输入"""
        element = self.wait.until(EC.visibility_of_element_located(locator))
        element.clear()
        element.send_keys(text)

    def wait_for_text(self, locator, expected_text):
        """等元素文本变成预期值"""
        self.wait.until(EC.text_to_be_present_in_element(locator, expected_text))

    def wait_loading_disappear(self):
        """等 loading 消失"""
        try:
            self.wait.until(
                EC.invisibility_of_element_located((By.CLASS_NAME, "el-loading-mask"))
            )
        except:
            pass  # loading 没出现也是正常的

    def wait_table_loaded(self, row_xpath="//tbody/tr[1]"):
        """等表格数据加载完成"""
        self.wait.until(
            EC.presence_of_element_located((By.XPATH, row_xpath))
        )
```

---

### 面试回答模板

> **"动态加载在 ERP 项目里太常见了——前后端分离、Vue 渲染、Ajax 异步加载，页面打开时表格是空的、下拉选项是接口返回的。**

> **我的处理方法靠三层：第一，通用等待方法封装在 BasePage 里——`wait_and_click`、`wait_loading_disappear`、`wait_table_loaded`，所有 page 类继承即可。第二，按场景用不同的等待条件——列表数据等第一行出现、下拉选项等可见、表格等 loading 消失、状态变更等文本变化。第三，`time.sleep` 只在万不得已时用，能换为显式等待的一定换。**

> **比如之前在 CI 环境里，每隔几天就有一条用例随机报 `NoSuchElementException`。排查后发现是 CI 网络慢，列表数据加载了 12 秒，但 wait 只设了 10 秒。改成 15 秒后稳定了。"**

---

**关键词**：动态加载 / 显式等待 / WebDriverWait / Selenium / loading / 异步渲染