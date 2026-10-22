# VSCode Remote-SSH + OpenAI Codex 远程安装完整指南

> 适用场景：本地 Windows 电脑通过 VSCode Remote-SSH 连接远程 Linux 服务器，在服务器上运行 Codex CLI Agent，图形化界面在本地 VSCode 侧边栏。
> 核心架构：**Codex Agent 跑在服务器，GUI 界面在本地 VSCode**，读写远程文件、调用服务器算力。

---

## 一、整体原理

```
本地 Windows (VSCode + Clash Verge)
        │
        │  SSH RemoteForward 7897
        ▼
远程 Linux 服务器
  ├─ bash 终端（继承 shell 环境变量代理）
  ├─ VSCode Extension Host（独立进程，需单独配置代理）
  └─ codex-cli（被 Extension Host 调用）
        │
        │  走代理访问 OpenAI
        ▼
  auth.openai.com / api.openai.com
```

**关键认知（最大坑点）：**
- bash 终端和 VSCode Extension Host 是**两个独立进程**，不共享环境变量。
- 终端 `curl` 能通，不代表 Codex 插件能通。
- Codex 插件走的是 Extension Host 的网络栈，需要单独在 VSCode 远程设置里配代理。

---

## 二、前置准备

### 2.1 本地电脑
- 安装最新版 VSCode
- 安装两个扩展：
  - `Remote - SSH`（微软官方）
  - `OpenAI ChatGPT`（OpenAI 官方，内含 Codex Agent，插件 ID：`openai.chatgpt`）
- 安装 Clash Verge（或其他代理工具），记下**混合代理端口**（默认 `7897`）

### 2.2 SSH 配置
编辑 `C:\Users\你的用户名\.ssh\config`：

```ssh-config
Host my-server
  HostName 服务器IP          # 例：192.168.50.25
  User 用户名                # 例：ywm
  Port 22
  IdentityFile ~/.ssh/id_rsa
  # 核心：把服务器的 127.0.0.1:7897 流量转发到本地 Clash
  RemoteForward 7897 127.0.0.1:7897
```

> ⚠️ 端口号必须和 Clash Verge 里的混合代理端口完全一致。
> ⚠️ `RemoteForward` 是 SSH 反向隧道，不需要在服务器上装任何代理软件。

---

## 三、连接服务器并安装 Codex CLI

### 3.1 通过 VSCode Remote-SSH 连接服务器
连接成功后，左下角应显示 `SSH: 服务器IP`。

### 3.2 在远程终端配置代理（临时，当前终端生效）

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export NO_PROXY=localhost,127.0.0.1,服务器IP
```

> ⚠️ 这三条只对当前 bash 终端有效，新开终端需要重新执行。

### 3.3 验证代理隧道是否通

```bash
curl -I https://api.openai.com/v1/models
```

- 返回 `401 Unauthorized` = **网络通了**（只是没带 token，正常）
- 返回超时 / connection refused = 隧道不通，检查 Clash 是否开启、SSH 是否重连

### 3.4 安装 nvm + Node.js 20

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc

# 安装 Node 20（Codex 要求 >= 18）
nvm install 20
nvm use 20
```

### 3.5 全局安装 Codex CLI

```bash
npm install -g @openai/codex
```

> ⚠️ **不要用 `snap install codex`**，那是同名的别的软件，不是 OpenAI 的 Codex。
> ⚠️ npm 必须走代理，否则会很慢或卡住。如长时间无输出，加 `--verbose` 看日志。

### 3.6 验证安装

```bash
codex --version
# 应输出类似：codex-cli 0.163.0
```

### 3.7 在远程环境安装 Codex 插件

连接 SSH 后，在 VSCode 扩展面板：
1. 确认顶部显示 `SSH: 服务器名`（远程环境）
2. 搜索 `OpenAI ChatGPT`
3. 点击 **Install in SSH: 服务器名**

> ⚠️ **必须装在远程环境**，只装本地无法操作服务器文件。

---

## 四、关键步骤：给 VSCode 远程扩展单独配置代理

> 这一步是解决 `token_exchange_failed` 的核心，绝大多数人卡在这里。

### 4.1 打开远程设置 JSON

在已连接服务器的 VSCode 窗口中：
1. `Ctrl + Shift + P`
2. 搜索并选择：`Preferences: Open Remote Settings (JSON)`
3. 在打开的 `settings.json` 中追加以下配置（**不要覆盖原有内容**）：

```json
{
    "http.proxy": "http://127.0.0.1:7897",
    "http.proxySupport": "override",
    "http.noProxy": [
        "localhost",
        "127.0.0.1",
        "::1"
    ],
    "http.useLocalProxyConfiguration": false
}
```

### 4.2 配置项说明

| 配置项 | 作用 |
|---|---|
| `http.proxy` | 给远程 VSCode 工作区指定代理地址（服务器侧回环端口，经 SSH 转发到本地 Clash） |
| `http.proxySupport: "override"` | 强制扩展优先使用 VSCode 配置的代理，而不是系统默认 |
| `http.noProxy` | 本地回环地址不走代理 |
| `http.useLocalProxyConfiguration: false` | **关键**：禁止 Remote-SSH 把 Windows 本机代理配置同步到远程，避免地址错乱 |

### 4.3 重载生效

方案 A（优先）：
- `Ctrl + Shift + P` → `Developer: Reload Window`

方案 B（A 无效时，彻底重启远程扩展进程）：
- `Ctrl + Shift + P` → `Remote-SSH: Kill VS Code Server on Host...`
- 选择当前服务器
- 重新连接 SSH

---

## 五、登录认证

### 5.1 方式一：Sign in with ChatGPT（推荐，走 Plus 订阅）

在 VSCode 侧边 Codex 面板点击登录，或在远程终端执行：

```bash
codex login --device-auth
```

终端会输出：
```
Visit https://chat.openai.com/device and enter code: XXXX-XXXX
```

1. 在**本地浏览器**打开链接
2. 登录你的 ChatGPT Plus 账号
3. 输入验证码，授权
4. 终端自动登录成功，凭证保存在 `~/.codex/auth.json`

```bash
# 安全加固
chmod 600 ~/.codex/auth.json
```

### 5.2 方式二：Use API Key（走平台余额）

```bash
echo "sk-xxxx" | codex login --with-api-key
```

> ⚠️ Plus 订阅和 API 余额是两套计费体系，API Key 不会走 Plus 额度。

---

## 六、常见报错与排错

### 6.1 `Command 'codex' not found`

**原因：** 服务器上没装 codex-cli，只装了 VSCode 插件。
**解决：** 执行 `npm install -g @openai/codex`（见 3.5）。

### 6.2 `token_exchange_failed`（最高频报错）

**现象：** 浏览器授权成功，但终端报 token 交换失败。
**排查顺序：**

1. **检查代理环境变量是否还在**
   ```bash
   echo $HTTPS_PROXY
   ```
   为空则重新 export（见 3.2）。

2. **测试 OAuth 接口连通性**
   ```bash
   curl -v https://auth.openai.com/oauth/token
   ```
   返回 `405 Method Not Allowed` = 网络通（GET 方法不对而已）。
   返回超时 = SSH 隧道假死，断开重连 SSH。

3. **检查 VSCode 远程扩展代理配置**（90% 的最终解决方案）
   - 按第四章配置 `settings.json`
   - Reload Window 或 Kill VS Code Server 后重试

4. **检查 IP 一致性**
   - 浏览器授权出口 IP 和 codex token 交换出口 IP 必须一致。
   - 换一条干净的代理线路（住宅 IP > 机房 IP）重试。

### 6.3 npm install 长时间卡住无输出

**原因：** npm 默认输出安静，下载慢时看起来像卡死。
**解决：**
```bash
# Ctrl+C 终止后，加 verbose 看日志
npm install -g @openai/codex --verbose
```

> ⚠️ **不要换淘宝 npm 镜像**，`@openai/codex` 不在镜像源里，会 404。必须走官方源 `registry.npmjs.org`，靠代理访问。

### 6.4 SSH 隧道长时间连接后假死

**现象：** 终端里 export 了代理，curl 之前能通，现在突然超时。
**原因：** `RemoteForward` 隧道在大流量（如 npm 下载 8 分钟）后容易断流，但 SSH 连接本身还在。
**解决：** 断开 VSCode SSH 连接，重新连接，重建隧道。

### 6.5 Codex 在本地执行命令而不是服务器

**原因：** Codex 插件装在了本地环境，不是远程环境。
**检查：** VSCode 左下角绿色标识必须是 `SSH: xxx`，扩展面板顶部必须显示 `SSH: 服务器名` 状态下再安装插件。

### 6.6 登录后一会就退出

**原因：** `auth.json` 过期或权限不对。
**解决：**
```bash
chmod 600 ~/.codex/auth.json
# 重新登录
codex login --device-auth
```

---

## 七、使用配置建议

### 7.1 审批模式（安全）

VSCode 设置 → 搜索 `Codex: Approval Mode`：
- `ask`（推荐）：每条 shell 命令、文件修改都弹窗确认
- `auto`：自动执行，风险高，不推荐
- `chat-only`：只聊天，不修改文件

### 7.2 永久代理配置（可选）

如果想每次登录服务器自动带代理，写入 `~/.bashrc`：

```bash
echo 'export HTTP_PROXY=http://127.0.0.1:7897' >> ~/.bashrc
echo 'export HTTPS_PROXY=http://127.0.0.1:7897' >> ~/.bashrc
echo 'export NO_PROXY=localhost,127.0.0.1' >> ~/.bashrc
source ~/.bashrc
```

> ⚠️ 缺点：如果 Clash 没开或 SSH 没连，所有外网命令都会超时。临时 export 反而更灵活。

---

## 八、完整流程速查表

| 步骤 | 操作位置 | 命令/动作 |
|---|---|---|
| 1 | 本地 SSH config | 添加 `RemoteForward 7897 127.0.0.1:7897` |
| 2 | VSCode Remote-SSH | 连接服务器 |
| 3 | 远程终端 | `export HTTP_PROXY/HTTPS_PROXY=...` |
| 4 | 远程终端 | `curl -I https://api.openai.com/v1/models` 验证 |
| 5 | 远程终端 | 安装 nvm + Node 20 |
| 6 | 远程终端 | `npm install -g @openai/codex` |
| 7 | VSCode 扩展面板 | 在 SSH 环境安装 OpenAI ChatGPT 插件 |
| 8 | VSCode Remote Settings JSON | 添加四项 http.proxy 配置 |
| 9 | VSCode 命令面板 | `Developer: Reload Window` |
| 10 | 远程终端 / Codex 面板 | `codex login --device-auth` 完成授权 |

---

## 九、核心踩坑总结

1. **终端通 ≠ 插件通**：VSCode Extension Host 是独立进程，不继承 bash 环境变量，必须单独配 `http.proxy`。
2. **RemoteForward 隧道会假死**：大流量后需要重连 SSH 重建隧道。
3. **npm 不能换国内源**：`@openai/codex` 只在官方源有，必须走代理。
4. **插件必须装在远程环境**：本地安装无法操作服务器文件。
5. **`token_exchange_failed` 八成是代理问题**：先查环境变量，再查 VSCode 远程代理配置，最后才考虑换 IP/风控。
6. **端口号三方一致**：Clash 混合端口 = SSH RemoteForward 端口 = VSCode http.proxy 端口，必须全是同一个数字。
