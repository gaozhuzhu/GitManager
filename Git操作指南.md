# Git 操作指南 —— 以 JiangWorkpalce 仓库为例

> **目标仓库**：`https://github.com/gaozhuzhu/JiangWorkpalce.git`
> **适用环境**：Windows 11 · Git 2.55.0.windows.3 · Git Bash / PowerShell
> **编写日期**：2026-11-20
> **建议工作目录**：`D:/AIWorkplace/Git技术`

本指南围绕上面这个真实仓库展开，从环境准备、首次克隆，到日常提交、分支协作、回退急救，最后给出针对该仓库实际出现的网络报错的专项排查。命令均可直接复制执行。

---

## 1. 先看清这个仓库

| 项目 | 实际情况 |
|---|---|
| HTTPS 地址 | `https://github.com/gaozhuzhu/JiangWorkpalce.git` |
| SSH 地址 | `git@github.com:gaozhuzhu/JiangWorkpalce.git` |
| 匿名读取 `git-upload-pack` | 返回 **401**，需要认证 |
| 仓库可见性 | 私有，或名称有误（GitHub 对不存在的私有仓库同样返回 401） |
| 认证方式 | Personal Access Token（PAT）或 SSH Key，账号密码已被 GitHub 禁用 |
| 本机网络 | 直连 `github.com:443` 正常（HTTP 200，约 1.5 秒） |

> **先核对仓库名**：`JiangWorkpalce` 中的 `Workpalce` 很像 `Workplace` 的笔误。若确认是笔误，后续所有命令中的 URL 都要改成正确地址，否则会一直卡在 401 / repository not found。

### 1.1 本次 `Connection was reset` 的诊断结论

| 路径 | 实测结果 |
|---|---|
| 直连 `github.com:443`（绕过代理） | HTTP **200**，网络层正常 |
| 环境变量代理 `127.0.0.1:55555` | 连接超时 / 被重置 |
| 系统代理 `127.0.0.1:7890` | `ProxyEnable=0`，端口未监听，残留配置 |
| 全局 git 配置 | 无 `http.proxy` / `https.proxy` |

根因是**代理指向了不可用的端口**，加上 GitHub 跨境链路在传输大数据包时容易被 RST。所以本指南的第 2.4 节是必读内容，不做那几步配置，后面的 clone/push 随时可能再次失败。

---

## 2. 环境准备

### 2.1 检查 Git 安装

Git Bash 或 PowerShell 中执行：

```bash
git --version
# 期望输出类似：git version 2.55.0.windows.3
```

如果提示找不到命令，先安装 [Git for Windows](https://git-scm.com/download/win)，安装时保持默认选项即可（会自带 Git Bash 和凭据管理器）。

### 2.2 设置提交身份

Git 用姓名和邮箱标识每一次提交，首次使用必须先配置：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
```

检查配置：

```bash
git config --global --list
```

> 这里的邮箱不必与 GitHub 账号邮箱完全一致，但如果希望提交记录关联到 GitHub 头像，请在 GitHub → Settings → Emails 中把该邮箱加入并勾选公开。

### 2.3 配置认证凭据（PAT）

GitHub 从 2021 年起不再接受账号密码，HTTPS 方式必须使用 Personal Access Token。

**创建 Token：**

1. 打开 GitHub → 右上角头像 → **Settings**
2. 左侧最下方 **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
3. 点击 **Generate new token**，填写名称、选择过期时间
4. Repository access 选择该仓库（或 All repositories）
5. Permissions → **Contents** 设置为 **Read and write**（只读用途选 Read）
6. 生成后**立刻复制并保存**，页面关闭后无法再次查看

**让 Git 记住凭据：**

```bash
git config --global --replace-all credential.helper manager
```

> **为什么用 `--replace-all`**：Git for Windows 安装时常会预置一条 `credential.helper`，如果你照着旧教程用 `git config --global credential.helper manager` 单值写入，而配置里已经有多条同名项，Git 就会报 `cannot overwrite multiple values with a single value`。`--replace-all` 会把所有旧值合并为一条。查看当前所有值：`git config --show-origin --get-all credential.helper`。

Windows 自带的 Git Credential Manager 会把凭据存入系统凭据管理器。第一次 clone/push 时按要求输入：

- Username：你的 GitHub 用户名
- Password：**粘贴 PAT**（不是账号密码）

**验证认证是否成功：**

```bash
git ls-remote https://github.com/gaozhuzhu/JiangWorkpalce.git
```

若返回一串 commit hash + 引用名，说明认证和网络都通了；若返回 401，说明 Token 无效或没有该仓库权限。

**凭据填错了要重来：**

打开「控制面板 → 凭据管理器 → Windows 凭据」，删除 `git:https://github.com` 条目，下次操作会重新询问。

### 2.4 网络专项配置（针对本仓库的报错）

#### 第一步：检查当前终端的代理变量

**Git Bash：**

```bash
env | grep -i proxy
```

**PowerShell：**

```powershell
Get-ChildItem Env: | Where-Object Name -match 'proxy'
```

常见的 `http_proxy` / `https_proxy` 若指向一个没在运行的本地端口，Git 就会报 `Recv failure: Connection was reset` 或超时。

#### 第二步：单次强制直连（最快验证）

```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy
git -c http.proxy= -c https.proxy= -c http.version=HTTP/1.1 \
    clone https://github.com/gaozhuzhu/JiangWorkpalce.git
```

PowerShell 对应写法：

```powershell
Remove-Item Env:HTTP_PROXY,Env:HTTPS_PROXY,Env:http_proxy,Env:https_proxy -ErrorAction SilentlyContinue
git -c http.proxy= -c https.proxy= -c http.version=HTTP/1.1 clone https://github.com/gaozhuzhu/JiangWorkpalce.git
```

`-c http.proxy=` 把代理置空，`http.version=HTTP/1.1` 避免 HTTP/2 长连接被重置 —— 这是提升成功率最关键的一处。

#### 第三步：确认可用后固化到全局

```bash
git config --global http.version HTTP/1.1
git config --global http.postBuffer 524288000
git config --global http.lowSpeedLimit 0
git config --global http.lowSpeedTime 999999
git config --global --unset http.proxy 2>/dev/null
git config --global --unset https.proxy 2>/dev/null
```

各参数含义：

| 参数 | 作用 |
|---|---|
| `http.version HTTP/1.1` | 避开 HTTP/2 在跨境链路上的 RST 问题 |
| `http.postBuffer` | 增大推送缓冲区，减少大仓库 push 时中断 |
| `http.lowSpeedLimit/Time` | 关闭低速超时判定，避免慢速传输被误判为断线 |

#### 第四步：直连仍不稳定时再启用代理

先启动你的代理客户端（本机 `127.0.0.1:7890` 当前未监听）。推荐只给 GitHub 配代理，避免影响国内仓库：

```bash
git config --global 'http.https://github.com/.proxy' http://127.0.0.1:7890
```

取消该配置：

```bash
git config --global --unset 'http.https://github.com/.proxy'
```

全面检查代理来源：

```bash
git config --show-origin --get-regexp '.*proxy'
```

#### 第五步：HTTPS 始终不通时的兜底方案 —— SSH over 443

生成密钥（已有则可跳过）：

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
```

把 `~/.ssh/id_ed25519.pub` 的全部内容复制到 GitHub → Settings → SSH and GPG keys → New SSH key。

在 `~/.ssh/config` 中加入（让 SSH 走 443 端口，绕开 22 端口封锁）：

```sshconfig
Host github.com
    HostName ssh.github.com
    Port 443
    User git
```

测试连接：

```bash
ssh -T git@github.com
# 成功输出：Hi <用户名>! You've successfully authenticated...
```

---

## 3. 首次获取仓库

### 3.1 HTTPS 克隆（推荐）

```bash
cd D:/AIWorkplace/Git技术
git clone https://github.com/gaozhuzhu/JiangWorkpalce.git
```

克隆完成后会自动创建 `JiangWorkpalce` 目录。

### 3.2 克隆到指定目录名

```bash
git clone https://github.com/gaozhuzhu/JiangWorkpalce.git my-project
```

### 3.3 浅克隆（只要最新历史，大仓库更快）

```bash
git clone --depth 1 https://github.com/gaozhuzhu/JiangWorkpalce.git
```

> 浅克隆只保留最近一次提交，无法查看更早历史，也不便于 rebase。日常开发建议用完整克隆。

### 3.4 克隆指定分支

```bash
git clone -b develop https://github.com/gaozhuzhu/JiangWorkpalce.git
```

### 3.5 已有本地目录，关联远程仓库

如果目录已经存在且有文件：

```bash
cd D:/AIWorkplace/Git技术
git init
git remote add origin https://github.com/gaozhuzhu/JiangWorkpalce.git
git pull origin main --allow-unrelated-histories
```

### 3.6 验证克隆结果

```bash
cd D:/AIWorkplace/Git技术/JiangWorkpalce
git remote -v
git status
git branch -a
```

`git remote -v` 应显示 `origin` 的 fetch 和 push 两个地址。

---

## 4. 核心概念：四个区域

理解这四个区域，Git 的绝大多数命令就都讲得通了。

| 区域 | 英文 | 说明 | 进入该区域的命令 |
|---|---|---|---|
| 工作区 | Working Directory | 你实际编辑的文件 | 直接编辑文件 |
| 暂存区 | Staging Area / Index | 待提交的改动清单 | `git add` |
| 本地仓库 | Local Repository | 已记录的提交历史 | `git commit` |
| 远程仓库 | Remote Repository | GitHub 上的仓库 | `git push` |

数据流向：

```text
工作区 --add--> 暂存区 --commit--> 本地仓库 --push--> 远程仓库
   ^                                               |
   +------------- pull / fetch / clone ------------+
```

`commit` 是本地操作，不联网；只有 `push` / `pull` / `fetch` / `clone` 才需要网络。所以网络出问题时，你依然可以正常提交，等网络恢复再统一推送。

---

## 5. 日常操作

### 5.1 查看状态

```bash
git status
git status -s          # 简短格式
```

状态符号：`??` 未跟踪，`A` 新增，`M` 修改，`D` 删除。

### 5.2 跟踪文件到暂存区

```bash
git add index.html              # 单个文件
git add src/                    # 整个目录
git add .                       # 当前目录全部改动（不含已删除文件的记录时需 -A）
git add -A                      # 全部改动，包括新增、修改、删除
```

### 5.3 提交

```bash
git commit -m "feat: 新增登录页面"
```

提交信息推荐约定式写法：`feat:` 新功能、`fix:` 修复、`docs:` 文档、`refactor:` 重构、`chore:` 杂项。

追加到上一次提交（未 push 时）：

```bash
git commit --amend -m "更准确的提交信息"
```

### 5.4 推送到远程

```bash
git push -u origin main         # 首次推送，-u 建立上游关联
git push                        # 之后直接推送
git push origin feature/login   # 推送指定分支
```

### 5.5 拉取远程更新

```bash
git pull                        # 拉取并合并到当前分支
git fetch                       # 只下载远程更新，不改变工作区
git fetch --all                 # 拉取所有远程
```

`git pull` 等价于 `git fetch` + `git merge`。想保持线性历史可用 `git pull --rebase`。

### 5.6 查看历史

```bash
git log --oneline --decorate --graph --all     # 图形化全分支历史
git log -5                                      # 最近 5 条
git log --author="名字"                         # 按作者筛选
git log --since="2026-01-01"                    # 按时间筛选
git show <commit-id>                            # 查看某次提交详情
```

### 5.7 比较差异

```bash
git diff                        # 工作区 vs 暂存区
git diff --staged               # 暂存区 vs 本地仓库
git diff HEAD                   # 工作区 vs 最新提交
git diff main..feature/login    # 两个分支之间
```

### 5.8 忽略文件

在项目根目录创建 `.gitignore`：

```gitignore
node_modules/
dist/
.env
*.log
.DS_Store
```

已经跟踪过的文件不会被忽略，需要先移除跟踪：

```bash
git rm --cached -r node_modules
git commit -m "chore: 停止跟踪 node_modules"
```

---

## 6. 分支管理

### 6.1 创建与切换

```bash
git branch feature/login                # 创建分支
git switch feature/login                # 切换分支
git switch -c feature/register          # 创建并切换（一步到位）
git branch -a                           # 查看所有分支（含远程）
```

旧写法 `git checkout -b <分支名>` 同样可用。

### 6.2 合并分支

```bash
git switch main
git merge feature/login                 # 快进合并
git merge --no-ff feature/login         # 保留分支结构，生成合并提交
```

### 6.3 变基（保持线性历史）

```bash
git switch feature/login
git rebase main
```

> **重要原则**：只对**尚未推送**的本地提交做 rebase。已推送到远程的提交被 rebase 会改写历史，协作者会出问题。

交互式整理最近 3 次提交：

```bash
git rebase -i HEAD~3
```

### 6.4 删除分支

```bash
git branch -d feature/login             # 删除已合并的本地分支
git branch -D feature/login             # 强制删除未合并分支
git push origin --delete feature/login  # 删除远程分支
```

### 6.5 分支命名建议

| 类型 | 格式 | 示例 |
|---|---|---|
| 功能 | `feature/<描述>` | `feature/user-profile` |
| 修复 | `fix/<描述>` | `fix/login-redirect` |
| 文档 | `docs/<描述>` | `docs/api-readme` |
| 热修复 | `hotfix/<版本>` | `hotfix/1.2.1` |

---

## 7. 撤销与回退（急救章节）

按"改动走到哪一步"选择对应命令，用错可能丢代码。

| 场景 | 命令 | 说明 |
|---|---|---|
| 改了文件，还没 add | `git restore <文件>` | 丢弃工作区修改 |
| 已 add，还没 commit | `git restore --staged <文件>` | 取消暂存，改动保留在工作区 |
| 已 commit，还没 push | `git reset --soft HEAD~1` | 撤销提交，改动留在暂存区 |
| 已 commit，还没 push | `git reset HEAD~1` | 撤销提交，改动留在工作区 |
| 已 commit，还没 push | `git reset --hard HEAD~1` | **危险**：撤销提交并丢弃所有改动 |
| 已 push 到远程 | `git revert <commit-id>` | 生成一个反向提交，安全且可追溯 |
| 误删分支 / 误 reset | `git reflog` | 查看所有历史操作，找回 commit-id |

**已推送的错误提交，正确做法：**

```bash
git revert <commit-id>
git push
```

> 绝对不要对已 push 的提交使用 `git reset --hard` + 强推，除非你确定没有其他人基于它工作。

**找回丢失的提交：**

```bash
git reflog
git switch -c rescue-branch <commit-id>
```

---

## 8. 冲突处理

### 8.1 冲突什么时候发生

两人修改了同一文件的同一处，或本地与远程在同一文件上都有独立提交，执行 `merge` / `pull` / `rebase` 时 Git 无法自动判断该保留哪一份。

### 8.2 处理步骤

```bash
git pull                        # 出现 CONFLICT 提示
git status                      # 查看哪些文件冲突
```

打开冲突文件，会看到如下标记：

```text
<<<<<<< HEAD
这是你本地的内容
=======
这是远程分支的内容
>>>>>>> feature/login
```

手工编辑成你想要的最终内容，**删除所有 `<<<<<<<` `=======` `>>>>>>>` 标记**，然后：

```bash
git add <冲突文件>              # 标记为已解决
git commit -m "merge: 解决登录页冲突"
git push
```

### 8.3 中途放弃

```bash
git merge --abort                # 放弃本次合并
git rebase --abort               # 放弃本次变基
```

---

## 9. 标签与协作

### 9.1 打标签

```bash
git tag v1.0.0                                  # 轻量标签
git tag -a v1.0.0 -m "首个正式版本"             # 附注标签，推荐
git push origin v1.0.0                          # 推送单个标签
git push origin --tags                          # 推送所有标签
git tag -d v1.0.0                               # 删除本地标签
git push origin --delete v1.0.0                 # 删除远程标签
```

### 9.2 远程仓库管理

```bash
git remote -v                                   # 查看远程
git remote add upstream <另一个仓库URL>          # 添加上游仓库
git remote set-url origin <新URL>               # 修改远程地址
git remote remove upstream                      # 删除远程
```

### 9.3 没有写权限时的协作流程

1. 在 GitHub 上 **Fork** 该仓库到自己的账号
2. 克隆自己的 fork：`git clone https://github.com/<你的账号>/JiangWorkpalce.git`
3. 添加上游：`git remote add upstream https://github.com/gaozhuzhu/JiangWorkpalce.git`
4. 在新分支上修改、提交、推送到自己的 fork
5. 在 GitHub 上发起 **Pull Request** 请求合并回原仓库

同步上游更新：

```bash
git fetch upstream
git switch main
git merge upstream/main
```

---

## 10. 专项排错

### 10.1 `Recv failure: Connection was reset` / `Failed to connect`

按顺序执行：

```bash
# 1. 看代理变量
env | grep -i proxy

# 2. 直连测试，应返回 200
curl --noproxy '*' -o /dev/null -w "%{http_code}\n" --connect-timeout 8 https://github.com

# 3. 走代理测试，对比结果
curl -o /dev/null -w "%{http_code}\n" --connect-timeout 8 https://github.com

# 4. 检查 git 代理配置来源
git config --show-origin --get-regexp '.*proxy'

# 5. 打开 Git 的 HTTP 详细日志
GIT_CURL_VERBOSE=1 git ls-remote https://github.com/gaozhuzhu/JiangWorkpalce.git
```

处理顺序：清掉错误代理 → 强制 `http.version=HTTP/1.1` → 换网络 / 开代理 → 改 SSH 443。

### 10.2 `401 Unauthorized` / `403 Forbidden` / `Invalid username or token`

GitHub 从 2021 年起完全禁用密码认证。如果弹出登录框时输入了**账号密码**，就会报：

```text
Invalid username or token. Password authentication is not supported for Git operations.
fatal: Authentication failed for 'https://github.com/gaozhuzhu/JiangWorkpalce.git/'
```

**用户名填 GitHub 账号，密码栏必须粘贴 PAT**，不是账号密码。

排查清单：

- Token 是否已过期或被删除
- Fine-grained token 是否选中了该仓库，且 `Contents` 权限为 Read and write（只读操作选 Read 即可）
- Classic token 需要勾选 `repo` 权限（私有仓库必需）
- 账号是否被仓库所有者授权（私有仓库）
- SSH 方式确认公钥已添加到 GitHub 且 `ssh -T git@github.com` 成功

**清掉错误的缓存凭据**（改完 PAT 后必须做，否则旧凭据会一直被复用）：

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential reject
```

或在「控制面板 → 凭据管理器 → Windows 凭据」中删除 `git:https://github.com`。

如果提示 `could not read Username ... terminal prompts disabled`，说明环境变量 `GIT_TERMINAL_PROMPT=0` 禁用了交互输入，先执行：

```bash
unset GIT_TERMINAL_PROMPT
```

然后再重试 clone / push。

### 10.3 `fatal: repository '...' not found`

- 核对仓库名拼写，特别是 `JiangWorkpalce`
- 私有仓库未认证时 GitHub 也会返回 not found，先确认凭据
- 确认 URL 末尾是 `.git`，用户名大小写正确

### 10.4 `SSL certificate problem: unable to get local issuer certificate`

临时绕过（不推荐长期使用）：

```bash
git -c http.sslVerify=false clone <URL>
```

根治办法：更新 Git for Windows 自带的 CA 证书，或检查公司网络是否在做 HTTPS 中间人代理。

### 10.5 `early EOF` / `pack has bad object` / 克隆中断

网络不稳定导致对象包传输出错。改用浅克隆分段获取：

```bash
git clone --depth 1 <URL>
git fetch --unshallow
```

或增大缓冲区：

```bash
git -c http.postBuffer=524288000 clone <URL>
```

### 10.6 `! [rejected] main -> main (non-fast-forward)`

远程有你本地没有的提交。先拉取再推送：

```bash
git pull --rebase
git push
```

### 10.7 中文文件名显示成八进制转义

Git **完全能识别**中文文件名：内部按 UTF-8 字节完整存储，add / commit / 查看历史都不受影响。只是 `git status`、`git ls-files` 默认开启 `core.quotepath`，把非 ASCII 字符显示成八进制转义，看起来像乱码：

```text
"Git\346\223\215\344\275\234\346\214\207\345\215\227.md"
```

关闭转义即可正常显示：

```bash
git config --global core.quotepath false
git config --global i18n.logOutputEncoding utf-8    # 提交信息中的中文也正常显示
```

设置后重新执行 `git status --short` 就会直接显示 `Git操作指南.md`。

> 这是**显示层**问题，不是编码损坏。如果文件真的出现问号或替换字符（如 `æ`),才是编码问题：确认用 UTF-8 保存文件，Windows 下避免不同编辑器混用 GBK 编码。

> **`git restore` 必须指定路径**：直接敲 `git restore` 会报 `you must specify path(s) to restore`。要恢复当前目录全部文件，必须显式写 `.`（见 10.9 节）。

### 10.8 `cannot overwrite multiple values with a single value`

原因：同一个配置文件里存在多条同名的 `credential.helper`，而普通写入命令只会替换单条，Git 不知道该替换哪一条。

先查看所有值及其来源：

```bash
git config --show-origin --get-all credential.helper
```

把所有旧值合并成一条：

```bash
git config --global --replace-all credential.helper manager
```

或者先清空再设置：

```bash
git config --global --unset-all credential.helper
git config --global credential.helper manager
```

> 系统级配置中的那条（如 PortableGit 自带的 `manager`）与用户级配置不冲突，无需删除。

### 10.9 `you must specify path(s) to restore`

`git restore` **不会**默认作用于当前目录，必须显式给出路径（文件、目录或 `.`）。直接敲 `git restore` 就会报这个错。

正确用法：

```bash
git restore index.html             # 恢复单个文件（丢弃工作区改动）
git restore src/                   # 恢复整个目录
git restore .                      # 恢复当前目录下全部改动
git restore --staged index.html    # 取消暂存单个文件，改动保留在工作区
git restore --staged .             # 取消暂存全部文件
```

同时撤销暂存区和工作区的改动（回到最近一次提交的状态，**危险**）：

```bash
git restore --staged --worktree .
# 等价于 git reset --hard HEAD，执行前确认不需要这些改动
```

对照旧命令，便于理解：

| 新命令 | 旧命令 |
|---|---|
| `git restore <文件>` | `git checkout -- <文件>` |
| `git restore --staged <文件>` | `git reset HEAD -- <文件>` |

> 提示：`git restore .` 会丢弃当前目录下所有未暂存的修改且**无法撤销**，执行前先跑一次 `git status` 确认范围。如果只想看会改什么，先用 `git diff`。

### 10.10 新增文件显示红色，且 `pathspec did not match any file(s) known to git`

新增文件显示为红色 `??`（未跟踪，untracked），此时文件**还没有进入 Git 的索引**。而 `git restore` / `git checkout --` 只能恢复已跟踪文件，所以对它执行会报：

```text
error: pathspec 'AI时代20260805演讲稿1.docx' did not match any file(s) known to git
```

先确认状态：

```bash
git status --short
# ?? 开头 = 未跟踪（红色）
```

按你的**真实意图**选一种做法：

**① 想把文件纳入版本管理** → 用 `add`，不是 `restore`

```bash
git add "AI时代20260805演讲稿1.docx"
git status --short        # 变成 A（绿色）
git commit -m "docs: 添加演讲稿"
```

**② 不想让 Git 跟踪这个文件**（比如临时稿、私密文档）→ 写进 `.gitignore`

```bash
echo "AI时代20260805演讲稿1.docx" >> .gitignore
git status --short        # 红色消失，文件仍留在磁盘上
```

忽略一整类文件用通配符，例如 `*.docx`、`draft*/`。

**③ 文件确实不要了** → 直接从磁盘删除，Git 管不着未跟踪文件

```bash
rm "AI时代20260805演讲稿1.docx"
```

> 第 ③ 种是不可逆操作，删除前先确认内容不再需要。如果只是想让它从 git 状态里消失但保留文件，请用第 ② 种。

**补充**：如果 `add` 之后反悔，想让文件回到红色未跟踪状态：

```bash
git restore --staged "AI时代20260805演讲稿1.docx"
```

### 10.11 `~$xxx.docx` 这类临时锁文件出现在 Untracked files

用 WPS / Microsoft Word 打开文档时，会在**同一目录**生成一个以 `~$` 开头的隐藏小文件，例如 `~$时代20260805演讲稿.docx`（通常 162 字节，内容是打开者的用户名）。它用来标记"该文档正在被编辑"，防止多人同时修改冲突。文档正常关闭后会自动删除，所以**它不是你的文档，也不需要提交**。

如果 `git status` 里看到它，说明文档当前还开着，或上次异常退出导致残留。

**处理方式（按情况选）：**

```bash
# ① 文档还在 WPS/Word 中打开 → 直接关闭该文档，锁文件自动消失，无需任何 git 操作

# ② 文档已关闭但文件残留 → 直接删除
rm "~\$时代20260805演讲稿.docx"
```

**长期方案**：在项目根目录创建 `.gitignore`，把这类临时文件永久忽略：

```gitignore
~$*
~$*.docx
~$*.xlsx
~$*.pptx
```

一行 `~$*` 就能覆盖所有 Office 临时锁文件。之后 `git status` 再也不会显示它们。

> 同样的道理，`.tmp`、`~` 结尾的备份文件、`.DS_Store` 等系统/软件自动生成的文件都应该写进 `.gitignore`，而不是 `git add`。

### 10.12 本地已 `git init`，如何把目录内容推到远程仓库

场景：本地目录已执行 `git init`（尚未 commit），远程是一个**已有初始提交**的仓库（GitHub 建仓时勾选了 README）。

**关键坑点**：本地默认分支是 `master`，远程是 `main`；且两边各有一次互不相关的提交，直接 push 会被拒绝。

完整步骤（在目标目录中执行）：

```bash
cd "D:/AIWorkplace/Git技术"

# 1. 先忽略不该上传的目录/文件（按实际情况写 .gitignore）
echo ".workbuddy/" >> .gitignore

# 2. 暂存并提交
git add .gitignore Git操作指南.md Git操作指南.html
git commit -m "docs: 添加 Git 操作指南"

# 3. 本地分支改名为 main，与远程一致
git branch -M main

# 4. 关联远程仓库
git remote add origin https://github.com/gaozhuzhu/Git-.git

# 5. 拉取远程初始提交（历史不相关必须加 --allow-unrelated-histories）
git pull origin main --allow-unrelated-histories

# 6. 推送并建立上游关联
git push -u origin main
```

**如果 pull 时提示合并冲突**（通常是 README.md），解决后：

```bash
git add README.md
git commit -m "merge: 合并远程初始提交"
git push -u origin main
```

**如果远程是全新空仓库**（没有任何 commit），可以跳过第 5 步的 pull，直接：

```bash
git push -u origin main
```

> 若远程内容与本地完全无关且你确定以本地为准，可用 `git push -u origin main --force` 强推，但会**覆盖远程历史**，执行前务必确认远程没有需要保留的内容。

验证推送结果：

```bash
git remote -v
git status -sb          # 应显示 ## main...origin/main
git log --oneline -3
```

### 10.13 修改 GitHub 仓库名称（Repository rename）

**网页端操作**（需要管理员权限）：

1. 打开仓库主页 → 点击 **Settings**（若看不到该标签，点仓库名旁的下拉菜单选 Settings）
2. 在 **General** 页找到 **Repository name** 输入框
3. 输入新名称 → 点击 **Rename**

命令行方式（需安装 GitHub CLI）：

```bash
gh repo rename 新名称 -R 当前仓库
```

**改名后的影响：**

- 提交历史、分支、Issues、Star、Watch、Wiki 全部保留，旧网址自动重定向到新网址
- `git clone` / `fetch` / `push` 用旧地址仍能工作，但**强烈建议更新本地远程地址**
- **不会**重定向：GitHub Pages 项目站点网址、引用了该仓库 Action 的工作流（会报 repository not found）
- 日后在同账号下新建仓库时**不要复用旧名称**，否则重定向会失效

**每个本地 clone 都要同步更新远程地址：**

```bash
# HTTPS
git remote set-url origin https://github.com/用户名/新名称.git

# SSH
git remote set-url origin git@github.com:用户名/新名称.git

# 验证
git remote -v
git fetch origin        # 测试连接
```

还应手动更新项目文档、配置文件里引用旧仓库 URL 的地方（如 README.md、package.json），并通知协作者更新他们本地的 remote。

> 如果只想改分支名（如 master → main），不需要改名仓库，用 `git branch -M 新分支名` 即可。

---

## 11. 常用命令速查表

```text
git status                     查看状态
git add .                      暂存全部改动
git commit -m "message"        提交
git push                       推送到远程
git pull                       拉取并合并
git fetch                      只拉取不合并
git log --oneline --graph      查看历史
git diff                       查看差异
git switch -c <branch>         创建并切换分支
git merge <branch>             合并分支
git revert <commit>            撤销已推送的提交
git restore <file>             丢弃工作区改动
git reflog                     查看所有操作记录
git remote -v                  查看远程仓库
```

---

## 12. 推荐配置清单

新机器可一次性粘贴执行（把身份信息换成你自己的）：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
git config --global --replace-all credential.helper manager
git config --global core.quotepath false
git config --global i18n.logOutputEncoding utf-8
git config --global http.version HTTP/1.1
git config --global http.postBuffer 524288000
git config --global http.lowSpeedLimit 0
git config --global http.lowSpeedTime 999999
git config --global --unset http.proxy 2>/dev/null
git config --global --unset https.proxy 2>/dev/null
git config --global --list
```

---

## 13. 建议的操作顺序

如果你现在就要开始，按这个顺序走最稳：

1. 核对仓库名是否真的叫 `JiangWorkpalce`
2. 执行第 12 节的推荐配置
3. 创建 PAT 并执行第 2.3 节的凭据配置
4. 用 `git ls-remote <URL>` 验证认证与网络
5. 执行第 3.1 节克隆仓库
6. 日常开发重复第 5 节的 add → commit → push 循环
7. 出问题时先查第 10 节

---

## 14. 延伸学习

- [Pro Git 中文版（官方免费）](https://git-scm.com/book/zh/v2)
- [GitHub Docs：克隆仓库](https://docs.github.com/zh/repositories/creating-and-managing-repositories/cloning-a-repository)
- [GitHub Docs：管理个人访问令牌](https://docs.github.com/zh/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [交互式学习 Git](https://learngitbranching.js.org/?locale=zh_CN)
