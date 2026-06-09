# Katabump Server Auto-Renewal

基于 [XCQ0607/katabump](https://github.com/XCQ0607/katabump) 优化：全协议代理、随机时间签到、ALTCHA/Turnstile 验证码自动绕过。

> 每天自动登录 katabump 面板进行一次续期，防止免费服务器过期被回收。

---

## 🚀 GitHub Actions 云端运行（推荐）

配置一次即可每天自动执行，无需本地电脑。

### 1. Fork 本仓库

点击右上角 **Fork** 到你的 GitHub 账号。

### 2. 配置 Secrets

进入你的仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**：

#### 必填

| Secret | 说明 | 格式 |
|--------|------|------|
| `USERS_JSON` | Katabump 账号列表 | `[{"username":"your@email.com","password":"123456"}]` |

#### 选填

| Secret | 说明 |
|--------|------|
| `PROXY_URL` | 代理订阅链接（支持 vmess/vless/hy2/tuic/socks5 全协议），脚本自动用 sing-box 启动本地代理 |
| `HTTP_PROXY` | 简单 HTTP 代理，格式 `http://user:pass@host:port`（`PROXY_URL` 未设置时的 fallback） |
| `TG_BOT_TOKEN` | Telegram Bot Token，用于推送续期结果通知 |
| `TG_CHAT_ID` | Telegram 接收通知的 Chat ID |

### 3. 启用 Workflow

进入 **Actions** 页面，点击 **I understand my workflows, go ahead and enable them**。

| 触发方式 | 说明 |
|---------|------|
| ⏰ 定时触发 | 每天 **北京时间 08:00 (UTC 00:00)** 自动运行 |
| 👆 手动触发 | 点击 **Run workflow** 立即测试（无随机延迟） |

### 4. 查看结果

- **日志**：Actions 运行详情页的 `Run Renew Script` 步骤
- **截图**：每次运行后，通过 `Upload Screenshots` 步骤上传到 Artifacts，可下载查看每个账号的最终页面状态

### 防检测

定时触发的任务会自动随机延迟 **0~3 小时**再执行，防止被目标站识别为固定时间自动化。

---

## 💻 本地运行（Windows 调试用）

适合本地调试或想观察浏览器操作过程。

### 环境准备

```bash
node -v   # 需 v18+
npm install
```

### 配置账号

```bash
cp login.json.template login.json
```

编辑 `login.json`：

```json
[
    {
        "username": "your@email.com",
        "password": "your_password"
    }
]
```

> `login.json` 已在 `.gitignore` 中，不会被提交到 GitHub。

### 配置 Chrome

编辑 `renew.js` 顶部几行：

```javascript
const CHROME_PATH = "C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe"; // 你的 Chrome 路径
const USER_DATA_DIR = path.join(__dirname, 'ChromeData_Katabump');               // 缓存目录
const HEADLESS = true;  // true=后台运行, false=可见窗口
```

### 运行

```bash
# 不带代理
node renew.js

# 带代理
set HTTP_PROXY=http://127.0.0.1:7890
node renew.js
```

运行截图保存在 `screenshots/` 目录（每个账号一张）。

---

## 🛠️ 项目结构

```
├── action_renew.js          # GitHub Actions 用主脚本（含随机延迟、sing-box 代理）
├── renew.js                 # Windows 本地运行脚本
├── proxy_handler.py         # 代理协议解析器 → 生成 sing-box 配置
├── package.json             # Node.js 依赖
├── login.json.template      # 账号模板（本地用）
├── start_chrome.bat         # 快速启动 Chrome 远程调试（Windows）
└── .github/workflows/
    └── renew.yml            # GitHub Actions 定时任务配置
```

## 🔄 工作流程

```
GitHub Actions (UTC 00:00)
  │
  ├─ 随机延迟 0~3h（仅定时触发）
  │
  ├─ [选填] 下载 sing-box → 解析 PROXY_URL → 启动本地 HTTP 代理
  │
  ├─ 启动 Chrome（无头模式 via xvfb）
  │
  └─ 遍历每个账号 →
       ├─ 访问登录页
       ├─ 绕过 Cloudflare Turnstile（CDP 点击）
       ├─ 输入凭据登录
       ├─ 进入续期页面
       ├─ 绕过 ALTCHA 验证码（CDP 点击）
       ├─ 点击续期按钮
       └─ 截图 → 上传 Artifacts / 推送 Telegram
```

## 📸 验证码绕过说明

脚本使用了两种自动绕过方案：

### Cloudflare Turnstile

通过 CDP (Chrome DevTools Protocol) 监听 shadow DOM 中的 checkbox，获取其在页面中的精确坐标后模拟鼠标点击。

### ALTCHA

监听 DOM 中 ALTCHA widget 的状态，自动在验证成功的时刻获取 token 并提交表单。支持 `click` 和 `pointer` 两种交互模式。

---

## 📝 注意事项

- **PROXY_URL 优先于 HTTP_PROXY**：如果同时设置了两个，脚本优先使用 sing-box 代理
- **Token 安全**：`CF_TUNNEL_TOKEN` 等敏感值请通过 GitHub Secrets 传入，不要硬编码在代码中
- **免费额度**：请合理使用，流量过大可能导致厂商封禁

---

## 致谢

- [XCQ0607/katabump](https://github.com/XCQ0607/katabump) — 原版项目
- [SagerNet/sing-box](https://github.com/SagerNet/sing-box) — 通用代理平台
