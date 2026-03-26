# Apifox 供应链投毒攻击 — 检测与应急工具集

> **攻击窗口**: 2026-03-04 ~ 2026-03-22 (18天)
> **影响范围**: 所有在此期间启动过 Apifox **桌面版**的用户 (Windows / macOS / Linux)
> **风险等级**: 🔴 严重 (Critical)

## 事件概述

2026年3月4日至22日，API 协作平台 [Apifox](https://apifox.com) 的 CDN 资源遭到供应链投毒攻击。攻击者篡改了 CDN 上的 `apifox-app-event-tracking.min.js` 文件（从 34KB 膨胀至 77KB），在合法代码后追加了约 42KB 的混淆恶意代码。

由于 Apifox 桌面版基于 Electron，恶意脚本在 Node.js 环境中运行，具备**完整的文件系统和网络访问权限**，可以窃取 SSH 密钥、Git 凭据、Shell 历史、K8s 配置等敏感信息，并通过远程代码执行机制运行攻击者下发的任意代码。

## 快速检测：你中招了吗？

恶意脚本会在 Electron 的 localStorage（LevelDB）中写入 `_rl_mc`（机器指纹）和 `_rl_headers`（采集信息缓存）。**即使你已更新 Apifox 到最新版本，这些标记仍然存在**，是最可靠的历史感染指标。

### macOS / Linux

```bash
grep -arlE "rl_mc|rl_headers" ~/Library/Application\ Support/apifox/Local\ Storage/leveldb
```

### Windows (PowerShell)

```powershell
Select-String -Path "$env:APPDATA\apifox\Local Storage\leveldb\*" -Pattern "rl_mc","rl_headers" -List | Select-Object Path
```

**如果以上命令输出了具体文件路径 → 确认中招，请立即按下方指引处理。**

如果无输出，则表示未发现感染标记。但若你确信在攻击窗口期间使用过 Apifox 桌面版，仍建议运行本仓库的完整检测脚本进行深度排查。

## 仓库结构

```
├── payload/                                # 恶意代码样本 (仅供安全分析，请勿执行)
│   ├── 01-deobfuscated-malicious-payload.js   # C2 beacon 还原代码 (持久化远控)
│   ├── 02-apifox-event.js                     # Stage-1 加载器 (344字节)
│   └── 03-02ab429d.js                         # Stage-2 v1 信息窃取 (~3400字节)
│
├── macOS/
│   ├── check_compromised.sh               # 中招检测脚本
│   └── check_leaked_info.sh               # 泄露信息评估脚本
│
├── Windows/
│   ├── check_compromised.ps1              # 中招检测脚本
│   └── check_leaked_info.ps1              # 泄露信息评估脚本
│
├── LICENSE
└── README.md
```

## 使用方法

### macOS

```bash
# 1. 检测是否中招
chmod +x macOS/check_compromised.sh
sudo bash macOS/check_compromised.sh

# 2. 如果中招，评估泄露范围
chmod +x macOS/check_leaked_info.sh
bash macOS/check_leaked_info.sh
```

### Windows

以**管理员权限**打开 PowerShell：

```powershell
# 允许执行脚本 (仅当前会话)
Set-ExecutionPolicy -Scope Process Bypass

# 1. 检测是否中招
.\Windows\check_compromised.ps1

# 2. 如果中招，评估泄露范围
.\Windows\check_leaked_info.ps1
```

## 攻击技术分析

### 攻击链

```
CDN 投毒 (篡改 JS 文件)
    │
    ▼
Apifox 桌面版启动时加载恶意脚本
    │
    ├── 采集机器指纹 (MAC/CPU/主机名/用户名/OS → SHA-256)
    ├── 窃取 Apifox accessToken → 调用官方 API 获取用户邮箱/用户名
    │
    ▼
向 C2 服务器 (apifox.it.com) 发送指纹信息
    │
    ▼
C2 返回 RSA 加密的 Stage-1 加载器 (344字节)
    │
    ▼
Stage-1 动态加载一次性 Stage-2 脚本 (随机路径，加载后自删除)
    │
    ├── [Stage-2 v1] 窃取 SSH密钥 / Shell历史 / Git凭据 / 进程列表
    ├── [Stage-2 v2] 额外窃取 K8s配置 / npm凭据 / SVN配置 / 目录结构
    │
    ▼
数据加密外传: JSON → Gzip → AES-256-GCM(密码:apifox/盐值:foxapi) → Base64
    │
    ▼
上传至 C2: /event/0/log (v1) 或 /event/2/log (v2)
    │
    ▼
每 30分钟~3小时 轮询 C2，支持远程下发并执行任意代码
```

### 混淆技术

入口恶意代码采用了 7 层混淆保护：

1. **字符串数组旋转** — 所有字符串抽取到数组并随机旋转
2. **Base64 + RC4 双层解密** — 字符串运行时解密
3. **代理函数** — 间接调用隐藏真实逻辑
4. **十六进制算术混淆** — 数值运算替换为十六进制表达式
5. **控制流扁平化** — `switch-case` 打乱执行顺序
6. **死代码注入** — 插入无效分支干扰分析
7. **反调试陷阱** — 检测 DevTools 和调试器

### 关键设计特征

| 特征 | 说明 |
|------|------|
| **域名伪装** | `apifox.it.com` — `.it.com` 并非意大利 ccTLD，而是商业二级域名服务，无公开 WHOIS 信息，易被误认为官方域名 |
| **一次性载荷** | Stage-2 URL 为随机 8 位 hex 路径，加载后 DOM 标签自动删除，增加取证难度 |
| **RSA 私钥外泄** | 攻击者将 RSA-2048 私钥硬编码在客户端，使安全研究人员能解密全部 C2 通信 (设计失误) |
| **多版本迭代** | 攻击期间观察到至少 10 次不同 Stage-2 载荷下发 (v1→v2 功能持续增强) |
| **代码矛盾** | 入口 7 层混淆 + 反调试，但 Stage-2 保留完整中文注释，暗示可能非同一开发者编写 |

## 窃取内容详情

### Stage-2 v1 (侦察阶段)

| 目标 | macOS/Linux | Windows |
|------|:-----------:|:-------:|
| `~/.ssh/` 全部文件 (含私钥) | ✅ | ✅ |
| `.zsh_history` / `.bash_history` | ✅ | - |
| `.git-credentials` | ✅ | - |
| `ps aux` 进程列表 | ✅ | - |
| `tasklist` 进程列表 | - | ✅ |

### Stage-2 v2 (纵深窃取)

| 目标 | macOS/Linux | Windows |
|------|:-----------:|:-------:|
| `~/.kube/` (K8s 配置) | ✅ | ✅ |
| `~/.npmrc` (npm token) | ✅ | ✅ |
| SVN 凭据 | ✅ | ✅ |
| 目录树结构 | ✅ | ✅ |

### C2 Beacon (持续运行)

| 目标 | 全平台 |
|------|:------:|
| Apifox accessToken | ✅ |
| Apifox 用户邮箱/用户名 | ✅ |
| 系统指纹 (MAC/CPU/主机名/用户名/OS) | ✅ |
| C2 远程代码执行 | ✅ |

## IOC (Indicators of Compromise)

### 网络指标

| 类型 | 值 |
|------|-----|
| C2 域名 | `apifox[.]it[.]com` |
| C2 IP (Cloudflare) | `104.21.2.104`, `172.67.129.21` |
| 被篡改文件 | `hxxps://cdn[.]apifox[.]com/www/assets/js/apifox-app-event-tracking.min.js` |
| Stage-1 URL | `hxxps://apifox[.]it[.]com/public/apifox-event.js` |
| Stage-2 URL | `hxxps://apifox[.]it[.]com/<随机8位hex>.js` |
| 数据回传 (v1) | `hxxps://apifox[.]it[.]com/event/0/log` |
| 数据回传 (v2) | `hxxps://apifox[.]it[.]com/event/2/log` |
| 被滥用的合法 API | `hxxps://api[.]apifox[.]com/api/v1/user` |

### 主机指标

| 类型 | 值 |
|------|-----|
| localStorage 标记 | `_rl_headers`, `_rl_mc` |
| 异常 HTTP 头 | `af_uuid`, `af_os`, `af_user`, `af_name`, `af_apifox_user`, `af_apifox_name` |
| AES 加密参数 | 密码: `apifox`, 盐值: `foxapi`, 算法: AES-256-GCM, IV: 12字节随机 |
| 密钥派生 | `crypto.scryptSync("apifox", "foxapi", 32)` |
| 被篡改文件大小 | 77KB (正常: 34KB) |

### Wayback Machine 存档

投毒版本已被存档，可用于比对分析：

`https://web.archive.org/web/20260305160602/https://cdn.apifox.com/www/assets/js/user-tracking.min.js`

## 紧急修复清单

如果确认中招，请按优先级依次执行：

### 🔴 紧急 — 立即处理

- [ ] **轮换所有 SSH 密钥**
  ```bash
  # 生成新密钥
  ssh-keygen -t ed25519 -C "your_email@example.com"
  # 在所有平台更新公钥 (GitHub/GitLab/服务器)
  # 检查服务器 authorized_keys 是否被注入后门密钥
  ```

- [ ] **撤销所有 Git Token**
  - GitHub: Settings → Developer settings → Personal access tokens → 全部 Revoke
  - GitLab: Preferences → Access Tokens → 全部 Revoke
  - 删除明文凭据: `rm -f ~/.git-credentials`

- [ ] **轮换 K8s 凭据** (如果使用 Kubernetes)
  - 重置 `~/.kube/config` 中的 OIDC token 和客户端证书
  - 审计集群操作日志

- [ ] **轮换 npm Token** (如果使用 npm 私有包)
  ```bash
  npm token revoke <token>
  npm login
  ```

- [ ] **在 Apifox 中注销并重新登录** (刷新 accessToken)

### 🟡 重要 — 尽快处理

- [ ] 检查 GitHub/GitLab 安全日志 (异常 Deploy Key / OAuth 授权 / 异常地区登录 / 异常 Push)
- [ ] 检查服务器 SSH 登录日志: `last -i` / `grep sshd /var/log/auth.log`
- [ ] 轮换 Shell 历史中暴露的所有密码/Token/API Key
- [ ] 清除 SVN 缓存凭据: `rm -rf ~/.subversion/auth/`

### 🔵 建议 — 后续处理

- [ ] 更新 Apifox 到最新版本
- [ ] 清理 Apifox 历史数据: `rm -rf ~/Library/Application\ Support/apifox/` (macOS) 或删除 `%APPDATA%\apifox\` (Windows)
- [ ] 清理 Shell 历史: `rm -f ~/.zsh_history ~/.bash_history`
- [ ] 启用所有平台的两步验证 (2FA)
- [ ] 对 SSH 私钥添加密码保护: `ssh-keygen -p -f ~/.ssh/id_ed25519`

## 重要提醒

已捕获的 Stage-2 v1/v2 **仅为侦察阶段**。该攻击架构支持任意后续攻击，包括后门植入、横向移动和定制化精准打击。

**高价值目标**（拥有生产集群权限、npm 发包权限、大型仓库管理权限的开发者）可能已遭受远超凭据窃取的深度入侵，应按**"已失陷"最高级别**进行应急响应。

## 参考资料

- [Apifox 供应链攻击技术分析 — 白帽酱](https://rce.moe/2026/03/25/apifox-supply-chain-attack-analysis/)
- [恶意代码样本 — phith0n (Gist)](https://gist.github.com/phith0n/7020c55bf241b2f3ccf5254192bd48a5)
- [Wayback Machine 投毒版本存档](https://web.archive.org/web/20260305160602/https://cdn.apifox.com/www/assets/js/user-tracking.min.js)

## 免责声明

本仓库中的恶意代码样本（`payload/` 目录）仅供安全研究和分析使用，**请勿执行**。检测脚本仅在本地运行，不会上传任何数据。使用者应自行承担使用风险。

## License

[MIT](LICENSE)
