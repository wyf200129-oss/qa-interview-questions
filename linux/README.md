# Linux / 日志排查面试题

---

## Q1. 测试工作中常用的 Linux 命令有哪些？（结合日常工作场景）

### 问题描述

面试官问你在测试工作中用什么 Linux 命令，不是考你背命令大全，而是看你**有没有真上过服务器干活**。结合你实际的工作场景说明你最常用的几个。

### 答案

之前做 ERP 测试时，经常要上服务器查日志、看数据、排查问题。下面按日常工作场景列出最常用的命令。

---

### 场景一：页面报错了，上服务器查日志上下文

这是最频繁的场景——业务反馈某个操作报错，或者自己测试时看到了 500。

**完整排查流程**：

```bash
# 1. 上服务器
ssh user@test-server

# 2. 先看应用日志的最后 200 行，快速定位到问题时间段
tail -200 /var/log/app/order.log | grep -A 5 "ERROR"
#    -A 5 显示匹配行后面 5 行，看报错详情

# 3. 找到报错的关键信息后，搜它的上下文
grep -C 10 "ORD001" /var/log/app/order.log
#    -C 10 显示前后各 10 行，看报错时的完整业务链路

# 4. 如果知道大概的时间，按时间范围查
sed -n '/2026-06-21 09:55/,/2026-06-21 10:05/p' /var/log/app/order.log | grep -C 3 "ERROR"

# 5. 跨多个服务搜同一个订单号
grep -H -C 5 "ORD001" /var/log/sale/*.log /var/log/wms/*.log /var/log/finance/*.log
```

**实际案例**：

> 有次业务反馈出库审核报 500，我上服务器搜了 `grep -C 10 "出库审核失败" /var/log/app/wms.log`，看到报错日志前 3 秒显示"库存=10"，前 1 秒显示"批量出库扣了 10 个"，我的请求来的时候库存已经变 0 了。**定位到了是并发扣库存的问题。**

---

### 场景二：实时跟踪日志（调试中常用）

测试环境测某个功能时，一边操作一边看日志输出。

```bash
# 1. 实时看日志，只显示报错
tail -f /var/log/app/wms.log | grep --color=always "ERROR"

# 2. 实时看日志，只显示包含某个业务ID的行
tail -f /var/log/app/order.log | grep --color=always "ORD001"

# 3. 日志输出太多，显示匹配行的前后 3 行
tail -f /var/log/app/wms.log | grep -C 3 --color=always "超时"

# 4. 排除无关的日志（比如健康检查的日志）
tail -f /var/log/app/app.log | grep -v "healthcheck"
```

---

### 场景三：查日志文件大小、磁盘空间

测试环境磁盘满了导致服务挂掉，也是常见问题。

```bash
# 1. 看磁盘使用情况
df -h
# 输出示例：
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   48G   2G  96% /    ← 磁盘快满了！

# 2. 看当前目录下各文件/目录大小
du -sh *
# 输出示例：
# 12G log/          ← 日志占了 12G
# 2G  backups/
# 1.5G data/

# 3. 看日志目录下各子目录大小
du -sh /var/log/app/*

# 4. 处理日志太大：清空但不删除文件（不影响进程写入）
cat /dev/null > /var/log/app/order.log
# 或用 truncate
truncate -s 0 /var/log/app/order.log
```

---

### 场景四：看端口、进程、服务状态

接口调不通时，先确认服务是不是活着。

```bash
# 1. 看端口是否在监听（最常用）
netstat -tlnp | grep 8080
# 或新版
ss -tlnp | grep 8080

# 2. 看某个进程是否在跑
ps -ef | grep java
ps -ef | grep order-service

# 3. 看 CPU/内存占用高的进程
top -b -n 1 | head -20
# 或
ps aux --sort=-%mem | head -10

# 4. 看服务是否正常
curl -I http://localhost:8080/health

# 5. 测试端口通不通
telnet 10.0.0.1 3306
# 或
nc -zv 10.0.0.1 3306
```

**实际案例**：

> 有次接口返回 502，我上服务器先 `curl -I http://localhost:8080/health` 发现应用还在，再 `netstat -tlnp | grep 8080` 看到端口也在监听，说明服务没挂。继续看了 `tail -200 /var/log/app/error.log` 发现是数据库连接池满了，连不上数据库了。

---

### 场景五：查服务器上的文件

```bash
# 1. 按名字找文件
find /var/log -name "*.log" | head -20

# 2. 按内容找文件（搜某个关键词在哪些文件里出现过）
grep -r "ORD001" /var/log/app/

# 3. 按时间找文件（最近修改过的文件）
find . -mmin -60 -type f   # 最近 60 分钟修改过的
find . -mtime -1 -type f   # 最近 1 天修改过的

# 4. 看文件行数
wc -l /var/log/app/order.log
# 输出：1234567 /var/log/app/order.log    ← 123 万行，够大的
```

---

### 场景六：处理文本数据（接口返回校验）

测试接口时，经常需要处理服务端返回的 JSON 或文本。

```bash
# 1. 从日志里提取 JSON 字段
grep "订单金额" /var/log/app/order.log | awk '{print $NF}'

# 2. 统计某个关键词出现的次数
grep -c "ERROR" /var/log/app/order.log
# 输出：128

# 3. 统计不同状态的订单数量
grep -oP 'status=\K[a-zA-Z]+' /var/log/app/order.log | sort | uniq -c
# 输出：
#  2345 CREATED
#  1890 PAID
#   123 CANCELED

# 4. 只显示特定列
# 日志格式：2026-06-21 10:00:01 ERROR [order-service] 库存不足
# 只看时间 + 日志级别
awk '{print $1, $2, $3}' /var/log/app/order.log

# 5. 去掉重复行
sort file.txt | uniq
# 或显示每行出现次数
sort file.txt | uniq -c | sort -rn
```

---

### 场景七：压缩和解压（传日志/传文件）

```bash
# 1. 压缩日志文件（发给开发）
tar -czf order_logs.tar.gz /var/log/app/order.log

# 2. 解压
tar -xzf order_logs.tar.gz

# 3. 看压缩文件内容（不解压）
zcat order_logs.tar.gz | grep "ERROR" | head -20
zgrep "ORD001" order_logs.tar.gz
```

---

### 场景八：快速统计和分析

```bash
# 1. 统计每秒请求数（看接口压力）
grep -oP '\d{2}:\d{2}:\d{2}' access.log | sort | uniq -c | head -10

# 2. 统计 IP 访问次数
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 3. 统计返回状态码分布
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 4. 找到最耗时的请求
awk '{print $NF, $0}' access.log | sort -rn | head -10
```

---

### 面试常用命令速查表

| 场景 | 命令 | 一句话说明 |
|:---|:---|:---|
| 查日志上下文 | `grep -C 10 "关键词" 文件` | 显示匹配行前后 10 行 |
| 实时看日志 | `tail -f 文件 \| grep 关键词` | 过滤只看想要的 |
| 看日志结尾 | `tail -200 文件` | 看最后 200 行 |
| 按时间切片 | `sed -n '/开始/,/结束/p' 文件` | 只看某个时间段 |
| 看磁盘 | `df -h` | 磁盘满了没 |
| 看端口 | `netstat -tlnp \| grep 端口` | 服务在不在 |
| 看进程 | `ps -ef \| grep 服务名` | 进程活着没 |
| 找文件 | `find 目录 -name "*.log"` | 找日志文件 |
| 统计次数 | `sort \| uniq -c \| sort -rn` | 各种统计 |
| 压缩传文件 | `tar -czf xxx.tar.gz 目录` | 打包发给别人 |

---

### 面试回答模板

> **"我测试工作中常用的 Linux 命令主要分三类。第一是查日志——`grep -C 10` 看报错上下文、`tail -f | grep` 实时跟踪、`sed` 按时间切片，这几个是每天都要用的。第二是看服务状态——`netstat` 查端口、`ps -ef` 查进程、`df -h` 查磁盘，接口调不通时先确认服务在不在。第三是数据处理——配合 awk/sort/uniq 统计日志中的错误频率、提取关键字段。**
>
> **之前做 ERP 测试时，业务反馈出库审核报 500，我上服务器用 `grep -C 10` 搜报错日志，上下文显示报错前库存被其他操作扣完了，1 分钟就定位到是并发问题。"**

---

**关键词**：grep / tail -f / netstat / ps -ef / df -h / awk / sed / Linux 日志排查

---

## Q2. Docker 在测试中怎么用？

### 答案

Docker 在测试里主要解决两个问题：**环境不一致**和**依赖服务搭建麻烦**。

### 你项目里怎么用

```yaml
# docker-compose.yml —— 一键启动测试环境
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: test123
      MYSQL_DATABASE: erp_test
    ports: ["3306:3306"]

  redis:
    image: redis:7
    ports: ["6379:6379"]

  app:
    build: .
    depends_on: [db, redis]
    ports: ["8080:8080"]
```

```bash
# 一条命令启动整套测试环境
docker-compose up -d

# 跑测试
pytest tests/

# 用完销毁
docker-compose down -v
```

### 三个核心场景

| 场景 | 怎么用 | 好处 |
|:---|:---|:---|
| **一键搭环境** | `docker-compose up` | 新人 5 分钟跑起来，不用配 MySQL/Redis |
| **多版本切换** | `docker tag` | 同一个机器同时跑 v2.3 和 v2.4 |
| **脏数据隔离** | `docker-compose down -v` | 全清干净，不影响下一轮 |

### 面试回答模板

> "Docker 我用在测试环境的搭建——docker-compose 一键起 MySQL+Redis+应用，新人不用配环境。多版本测试时用不同的 tag，同时验证新旧版本。跑完 `down -v` 全清干净，不会有脏数据。"

---

**关键词**：Docker / docker-compose / 测试环境 / 一键部署

---

## Q3. Git 分支管理和冲突怎么解决？

### 答案

测试工作中 Git 四步日常：

```bash
# 1. 拉最新代码
git pull origin main

# 2. 建分支干活
git checkout -b feature/add-test-outbound

# 3. 提交
git add tests/test_outbound.py
git commit -m "feat: 新增出库审核自动化用例"

# 4. 推送
git push origin feature/add-test-outbound
```

### 冲突解决

```bash
# 和同事同时改了 tests/conftest.py → 冲突了
git pull origin main
# CONFLICT in tests/conftest.py

# 打开文件，看到冲突标记
<<<<<<< HEAD
@pytest.fixture(scope="session")
def api_token():
=======
@pytest.fixture(scope="module")
def api_token():
>>>>>>> origin/main

# 保留你要的版本，删掉标记，然后
git add tests/conftest.py
git commit -m "fix: 解决 conftest 冲突，保留 session scope"
```

### 常用操作速查

| 命令 | 做什么 |
|:---|:---|
| `git status` | 看改了哪些文件 |
| `git diff` | 看具体改了什么 |
| `git log --oneline -10` | 看最近 10 次提交 |
| `git stash` | 暂存手头修改，先切分支 |
| `git stash pop` | 恢复暂存的修改 |
| `git checkout .` | 还原所有本地修改 |

### 面试回答模板

> "测试中我主要用 Git 管理自动化脚本和测试数据。改用例建 feature 分支，跑通后提 MR 让同事 review。遇到冲突时看 Git 标记，确认保留哪个版本，手动合并。保持提交信息清晰——feat 新增、fix 修复、refactor 重构。"

---

**关键词**：Git / 分支管理 / 冲突解决 / 版本控制