# 🔐 n8n Security Checker

基于 **n8n** + **Docker** 的全自动安全情报采集与 AI 分析系统。每小时自动从全球主流安全资讯平台抓取最新漏洞与安全事件，由智谱 AI（GLM）生成专业安全时报，并推送至飞书群。

---

## ✨ 功能特性

- 🕐 **每小时自动采集**：从 BleepingComputer、CISA、The Hacker News 三大安全平台并行抓取 RSS 资讯
- 💾 **本地持久化归档**：所有原始资讯追加写入 `data/vulnerabilities_log.txt`，永久保存
- 🤖 **GLM AI 智能分析**：调用智谱 AI（GLM-4-Plus）对本次资讯进行分析，自动按威胁等级分类整理
- 📢 **飞书自动推送**：将生成的安全时报通过飞书自定义机器人 Webhook 发送到指定群聊

---

## 📁 项目结构

```
n8nSecurityChecker/
├── docker-compose.yml                  # Docker 编排配置
├── security_vulnerability_workflow.json # n8n 工作流定义
├── data/
│   └── vulnerabilities_log.txt         # 本地原始资讯归档（运行后自动生成）
└── README.md
```

---

## 🔄 工作流程

```
定时触发器（每小时）
        │
        ├──▶ BleepingComputer RSS ──┐
        ├──▶ CISA RSS              ─┼──▶ 合并 ──▶ 写入本地文件 + 准备AI提示词
        └──▶ The Hacker News RSS  ──┘                    │
                                                          ▼
                                              GLM API 安全分析
                                                          │
                                                          ▼
                                              发送安全时报到飞书
```

### 生成的安全时报格式

```
📅 安全时报 · 2026/5/7 14:00:00
共收录 45 条资讯

🔴 高危威胁
CVE-2026-XXXX: Apache XXX 远程代码执行漏洞，影响 2.4.x 以下版本...

🟡 中等风险
...

🟢 安全动态
...

📊 态势总结
本小时整体威胁态势较为活跃...
```

---

## 🚀 快速开始

### 前置要求

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) 已安装并运行
- 智谱 AI API Key（[获取地址](https://open.bigmodel.cn/usercenter/apikeys)）
- 飞书群自定义机器人 Webhook 地址

### 1. 启动 docker 服务（确保本地有docker环境）

```powershell
cd n8nSecurityChecker
docker-compose up -d
```

服务启动后访问 [http://localhost:5678](http://localhost:5678)

### 2. 导入工作流

1. 打开 n8n 界面，点击左上角 **+** 新建工作流
2. 点击右上角 **⋯** → **Import from File**
3. 选择项目根目录下的 `security_vulnerability_workflow.json`

### 3. 填写密钥配置

导入后，打开 **「AI分析并发送飞书」** 节点，修改顶部两行变量：

```javascript
const GLM_API_KEY = 'YOUR_GLM_API_KEY_HERE';           // ← 替换为你的智谱AI Key
const FEISHU_WEBHOOK = 'YOUR_FEISHU_WEBHOOK_URL_HERE'; // ← 替换为你的飞书Webhook地址
```

### 4. 激活工作流

- 点击右上角 **Publish** → 选择激活选项
- 或在工作流列表页找到该工作流，拨动右侧开关为**激活状态**

工作流激活后将每小时自动运行，无需手动操作。

---

## ⚙️ 配置说明

### docker-compose.yml 关键配置

| 环境变量 | 说明 |
|---|---|
| `NODE_FUNCTION_ALLOW_BUILTIN=*` | 允许 Code 节点使用 Node.js 内置模块（fs、https 等） |
| `NODE_FUNCTION_ALLOW_EXTERNAL=*` | 允许 Code 节点使用外部 npm 包 |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false` | 关闭文件写入沙箱限制 |
| `user: root` | 确保容器有权限写入挂载目录 |

### 数据卷挂载

```yaml
volumes:
  - n8n_data:/home/node/.n8n   # n8n 数据库与配置（持久化）
  - ./data:/data                # 本地资讯归档目录
```

### 更换 AI 模型

在 `AI分析并发送飞书` 节点中修改模型名称：

```javascript
model: 'glm-4-plus',   // 可替换为 glm-4、glm-4-air、glm-4-flash 等
```

### 添加/替换 RSS 数据源

在工作流中添加新的 **RSS Feed Read** 节点，连接到 **合并所有来源** 节点即可。推荐数据源：

| 名称 | RSS 地址 |
|---|---|
| BleepingComputer | `https://www.bleepingcomputer.com/feed/` |
| CISA 公告 | `https://www.cisa.gov/cybersecurity-advisories/all.xml` |
| The Hacker News | `https://feeds.feedburner.com/TheHackersNews` |
| Krebs on Security | `https://krebsonsecurity.com/feed/` |
| Dark Reading | `https://www.darkreading.com/rss.xml` |

---

## 🗂️ 本地归档日志格式

每条资讯以如下格式追加写入 `data/vulnerabilities_log.txt`：

```
[2026-05-07T06:00:00.000Z] Critical RCE Vulnerability in Apache
链接: https://www.bleepingcomputer.com/news/...
发布: Wed, 07 May 2026 05:30:00 GMT
描述: A critical remote code execution vulnerability...
--------------------------------------------------
```

---

## 🛠️ 常见问题

**Q: Code 节点报 `Module 'fs' is disallowed`**
> 检查 `docker-compose.yml` 中是否有 `NODE_FUNCTION_ALLOW_BUILTIN=*`，重启容器后重试。

**Q: data 目录没有生成文件**
> 确认 `docker-compose.yml` 中 `user: root` 已配置，且 `./data:/data` 挂载正确。

**Q: GLM API 报 `invalid_api_key`**
> 从 [智谱AI控制台](https://open.bigmodel.cn/usercenter/apikeys) 重新获取 Key，确认格式正确（通常以字母数字串形式展示）。

**Q: 飞书消息发送失败**
> 确认 Webhook 地址完整（通常以 `https://open.feishu.cn/open-apis/bot/v2/hook/` 开头）。

---

## 📄 License

MIT
