这是一份为你量身定制的**《OpenClaw 二次开发与版本升级标准工作流 (SOP)》**。我将前面讨论的初始构建、分支规范以及“二开代码与官方新版本融合”的核心步骤，整合成了一份可以直接作为团队内部开发规范的完整指南。

---

### 👑 核心原则：上游只读，基于 Tag 部署，隔离二开代码。

### 第一阶段：初始仓库基建 (只需执行一次)

在开始任何开发前，必须将你的本地代码库与官方建立“血脉联系”。

```bash
# 1. 克隆你 Fork 后的私有仓库
git clone https://github.com/你的账号/openclaw.git
cd openclaw

# 2. 绑定 OpenClaw 官方仓库作为“上游 (upstream)”
git remote add upstream https://github.com/openclaw/openclaw.git

# 3. 验证绑定状态 (应显示 origin 和 upstream 各两个地址)
git remote -v

```

### 第二阶段：确立三级分支模型

彻底放弃在 `main` 分支上一把梭的习惯，严格按照以下职能划分分支：

1. **`main` 分支 (搬运工)：**

- **作用：** 纯粹用于同步官方最前沿的代码。**绝对不要在此分支上手动改代码或用于生产环境。**

2. **`production` 分支 (生产护城河)：**

- **作用：** 服务器实际运行的分支。它必须基于官方某个**稳定的 Tag（标签）**创建。
- **创建命令：** `git checkout -b production tags/v1.0.0`（假设官方当前稳定版是 v1.0.0）。

3. **`feature/xxx` 或 `custom` 分支 (二次开发区)：**

- **作用：** 如果你需要改底层逻辑，基于 `production` 拉出此分支进行二开，测试无误后再合并回 `production`。

```bash
# ==========================================
# 1. 初始化 main 分支（保持纯净，紧跟官方）
# ==========================================
# 确保你当前在 main 分支
git checkout main
# 拉取官方最新代码并合并，确保你的 main 是最新的官方原始状态
git fetch upstream
git merge upstream/main
# 顺手推送到你自己的 GitHub 仓库备份
git push origin main

# ==========================================
# 2. 初始化 production 分支（生产护城河）
# ==========================================
# 极其关键：获取官方所有的 Release 标签 (Tags)
git fetch upstream --tags

# 查看分支
git tag --sort=-creatordate | head

# 假设你查阅官方 Release 发现当前最稳的版本是 v1.0.0
# 直接基于这个 Tag 创建并切换到 production 分支
git checkout -b production tags/v1.0.0

# 此时，修改 docker-compose.yml，将镜像版本锁定
# 例如将 image: openclaw/backend:latest 改为 image: openclaw/backend:v1.0.0
# 提交这个锁定版本的修改
git add docker-compose.yml
git commit -m "chore: 初始化生产环境，锁定镜像至 v1.0.0"

# ==========================================
# 3. 初始化 custom / feature 分支（二次开发区）
# ==========================================
# 确保你是从稳定的 production 分支拉出二开分支，而不是从 main 拉！
git checkout production

# 创建并切换到你的专属开发分支（比如你要加一个 B2B 的数据清洗逻辑）
git checkout -b custom/data-cleaner

# --- 在这里尽情修改你的二次开发代码 ---

# 修改完毕且本地测试跑通后，提交你的二开代码
git add .
git commit -m "feat: 增加 B2B 专属数据清洗中间件"

# 最后，将二开代码合并回你的生产分支，准备发布到服务器
git checkout production
git merge custom/data-cleaner
git push origin production
```

---

### 第三阶段：带二开代码的安全升级流 (核心实操 SOP)

假设你的 `production` 分支中已经包含了你辛辛苦苦写的二次开发代码。此时官方发布了重磅更新 `v1.2.0`。请严格按以下 5 步执行安全升级：

#### Step 1: 制作“后悔药” (备份当前稳定版)

```bash
git checkout production
git branch backup-prod-20260224  # 以当天日期命名，万一搞砸了一秒切回

```

#### Step 2: 拉取官方最新“里程碑”

```bash
# 获取官方所有最新的 Release Tags（不要拉取 main 分支）
git fetch upstream --tags

```

#### Step 3: 融合官方新版本 (Merge 策略)

```bash
# 确保当前在 production 分支
git checkout production

# 将官方的新版本代码，合并到你带有二开代码的分支中
git merge tags/v1.2.0

```

#### Step 4: 解决合并冲突 (如有)

- **如果自动合并成功：** 终端提示 `Merge made by the 'ort' strategy`，直接进入下一步。
- **如果提示代码冲突 (Merge conflict)：**

1. 打开编辑器（如 VS Code），全局搜索 `<<<<<<<`。
2. 手动对比你的二开逻辑与官方新逻辑，谨慎决定保留哪一部分。
3. 保存文件，执行 `git add .` 和 `git commit -m "chore: 解决升级 v1.2.0 时的冲突"`。

#### Step 5: 重新构建与防腐测试

```bash
# 重新编译底层 Docker 镜像
docker-compose down
docker-compose build --no-cache
docker-compose up -d

```

- **终极测试：** 回到 n8n 画布，手动执行你的 `Tool_Search_SOP_OpenClaw` 子工作流。确认大模型依然能拿到格式正确的 SOP 文本。测试通过后，即可删除 Step 1 的备份分支。

---

### 💡 架构师的长远忠告

为了让未来的每一次升级都如同丝般顺滑，请尽量遵守**“非侵入式二开”**原则：
如果你修改代码仅仅是为了**清洗数据**、**转换 JSON 格式**或者**预处理查询词**，**请立刻停止修改 OpenClaw 源码！** 把这些逻辑全部转移到 n8n 的 `Code Node`（代码节点）或者子工作流里去处理。只有让 OpenClaw 保持尽可能的“纯洁”，你的升级成本才会降到最低。

这份合并后的 SOP 是否清晰？接下来，我们是着手准备 **n8n 侧的子工作流搭建**，还是先设计一下用来给你们**产品团队填写的“SOP 知识收集表”模板**？
