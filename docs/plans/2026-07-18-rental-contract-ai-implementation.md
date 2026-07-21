# 租房合同AI助手 实施计划

> **给 Claude：** 必须使用 `superpowers:executing-plans` 子技能，按任务逐项执行本计划。

**目标：** 构建一个 H5 移动端网页版租房合同 AI 辅助阅读工具，MVP 聚焦杭州大学生群体，基于 Dify + DeepSeek-V4-Pro + RAG 双知识库。

**架构方案：** React H5 前端 → Vercel 托管 → Cloudflare Tunnel → 笔记本 Dify 后端（编排工作流 + 管理知识库 + 调用 DeepSeek-V4-Pro API + 腾讯云 OCR）。前端做数据脱敏和本地存储，Dify 做检索增强生成和结果结构化输出。

**技术栈：** React + Vite + TailwindCSS（前端），Dify（后端编排），DeepSeek-V4-Pro API（生成），bge-large-zh-law（Embedding），腾讯云 OCR，localStorage（存储），Vercel（托管），Cloudflare Tunnel（穿透）

---

## 任务拆解

### 阶段一：基础设施搭建

---

### 任务 1：前端项目脚手架

**涉及文件：**
- 新建：`rental-contract-ai/`（项目根目录）

**步骤 1：创建 React + Vite 项目**

```bash
npm create vite@latest rental-contract-ai -- --template react
cd rental-contract-ai
npm install
```

**步骤 2：安装依赖**

```bash
npm install react-router-dom axios tailwindcss @tailwindcss/vite
```

**步骤 3：配置 TailwindCSS**

修改 `vite.config.js`，引入 tailwindcss 插件：

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

**步骤 4：初始化 TailwindCSS 基础样式**

修改 `src/index.css`：

```css
@import "tailwindcss";
```

**步骤 5：清理默认文件，创建基础目录结构**

```bash
rm src/App.css src/assets/react.svg
mkdir -p src/pages src/components src/utils src/hooks src/services
```

**步骤 6：运行开发服务器，确认能启动**

```bash
npm run dev
```

预期：浏览器打开 `http://localhost:5173`，看到空白页面（无报错）。

**步骤 7：提交变更**

```bash
git init
git add -A
git commit -m "feat: init React + Vite + TailwindCSS project scaffold"
```

---

### 任务 2：Dify 基础配置

**步骤 1：确认 Dify 自部署运行状态**

打开 Dify 管理页面（本地 `http://localhost:3000` 或你的部署地址），确认能正常登录。

**步骤 2：在 Dify 中创建"租房合同助手"应用**

- 进入 Dify → 创建应用 → 选择"聊天助手"类型
- 应用名称：`rental-contract-ai`
- 记录生成的 API Key（后续前端调用需要）

**步骤 3：配置 DeepSeek-V4-Pro 模型**

- Dify 设置 → 模型供应商 → 添加 DeepSeek
- 填入 DeepSeek API Key
- 确认模型列表中能看到 `deepseek-v4-pro`

**步骤 4：配置 Embedding 模型**

- 模型供应商 → 添加 OpenAI-API-compatible
- 配置 bge-large-zh-law（通过 Ollama 或其他本地服务，或配置云端兼容 API）
- 确认 Dify 知识库设置中可选该 Embedding 模型

**步骤 5：测试连通性**

在 Dify 应用内发送一条测试消息："你好，租房合同要注意什么"，确认能收到模型回复。

**步骤 6：提交 Dify 配置记录**

保存 Dify 工作流配置截图或导出 DSL 文件到 `docs/dify-config/` 目录。

---

### 任务 3：Cloudflare Tunnel 配置

**步骤 1：安装 cloudflared**

```bash
# Windows（以管理员身份运行）
winget install cloudflare.cloudflared

# 或下载安装包：https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
```

**步骤 2：登录 Cloudflare**

```bash
cloudflared tunnel login
```

浏览器会弹出 Cloudflare 授权页面，选择你的域名或使用默认的快速隧道。

**步骤 3：创建隧道**

```bash
cloudflared tunnel create rental-contract-tunnel
```

**步骤 4：配置隧道指向 Dify**

创建 `config.yml`（放在 `~/.cloudflared/` 目录）：

```yaml
tunnel: <隧道ID>
credentials-file: ~/.cloudflared/<隧道ID>.json

ingress:
  - hostname: rental-api.yourdomain.com  # 如果用快速隧道则不需要这行
    service: http://localhost:3000       # Dify API 端口
  - service: http_status:404
```

如果用快速隧道（不需要域名），跳过 `hostname` 配置，直接运行：

```bash
cloudflared tunnel run --url http://localhost:3000 rental-contract-tunnel
```

**步骤 5：记录隧道 URL**

运行后终端会输出类似 `https://xxx.trycloudflare.com` 的 URL，记录下来——这是前端要调用的 API 基础地址。

**步骤 6：验证隧道**

浏览器访问 `https://xxx.trycloudflare.com/api`（Dify API 端点），确认能返回响应（可能是 JSON 错误信息，说明已经连通到 Dify）。

---

### 阶段二：知识库建设

---

### 任务 4：法律知识库搭建

**步骤 1：爬取法律条文**

用一个脚本或手动收集以下法律文本，保存为 Markdown 文件：

| 法律文件 | 重点章节 | 文件名 |
|---|---|---|
| 《民法典》租赁合同章节 | 第 703-734 条 | `civil-code-lease.md` |
| 《商品房屋租赁管理办法》 | 全文 | `rental-management-measures.md` |
| 《杭州市住房租赁管理条例》 | 全文 | `hangzhou-rental-regulations.md` |
| 《浙江省房屋租赁管理条例》 | 全文 | `zhejiang-rental-regulations.md` |

**步骤 2：格式化法律条文**

每条法条按以下格式整理：

```markdown
## 第713条 维修义务

**法律名称：** 《民法典》
**章节：** 租赁合同
**条款号：** 第713条
**主题：** 维修责任、出租人义务
**同义词：** 维修、修东西、家电坏了、房屋维修、谁出钱修

**原文：**
出租人应当履行租赁物的维修义务，但当事人另有约定的除外。
```

保存到 `docs/knowledge-base/law/` 目录。

**步骤 3：上传到 Dify 法律知识库**

- Dify → 知识库 → 创建知识库 → 名称：`法律知识库`
- Embedding 模型：bge-large-zh-law
- 分块设置：自定义，最大长度 500 字符，重叠 0
- 上传所有法律条文 Markdown 文件
- 等待索引完成

**步骤 4：测试法律知识库检索**

在 Dify 知识库页面，用以下 query 测试检索命中率：

| 测试 query | 期望命中 |
|---|---|
| "押金退不退" | 民法典关于押金/履约保证金的规定 |
| "空调坏了谁修" | 民法典第713条（维修义务） |
| "提前退租要赔多少" | 民法典关于提前解约的规定 |
| "房屋押金不给退" | 杭州市/浙江省租赁管理条例相关条款 |

如果命中不理想，回到步骤 2 补充同义词字段。

**步骤 5：提交知识库文件**

```bash
git add docs/knowledge-base/
git commit -m "feat: add legal knowledge base source files"
```

---

### 任务 5：风险知识库搭建（首批 15-20 个风险点）

**步骤 1：创建首批风险点文档**

保存到 `docs/knowledge-base/risk/` 目录，按主题分文件：

`rental-deposit-risks.md`（押金相关）：
```markdown
## 押金超过2个月租金
- **分类：** 押金相关
- **严重程度：** 高
- **检查方式：** 合同押金金额 ÷ 月租金 > 2
- **通俗解读：** 押金超过2个月租金属于偏高。市场惯例是押1付1或押1付3，押3付1意味着你要多压一大笔钱在房东手里
- **谈判建议：** 尽量谈到押1付1或押1付3，最差也不要超过押2付3
- **常见套路：** 房东会说"大家都这样"、"这是规矩"，但实际上是可以谈的

## 押金退还条件苛刻
- **分类：** 押金相关
- **严重程度：** 高
- **检查方式：** 合同中押金退还条款是否包含"需经房东验收合格"、"房屋需恢复原状"、"无任何损坏"等模糊表述
- **通俗解读：** 有些合同写"房屋无任何污渍才退押金"，这基本不可能做到——正常居住一定会有些痕迹
- **谈判建议：** 要求明确"自然损耗不扣押金"，拍照留底入住时的房屋状况
- **常见套路：** 退租时房东拿着合同说你弄脏了墙、地板有划痕，以此扣押金
```

按同样格式覆盖以下风险主题：

| 主题 | 至少覆盖 |
|---|---|
| 押金 | 金额过高、退还条件苛刻、退还时限过长、押金抵扣范围模糊 |
| 租金与费用 | 隐性杂费、水电燃气计价方式、年度涨租条款、物业费归属 |
| 违约金 | 双向不对等、提前退租罚则过高、房东违约无补偿 |
| 维修责任 | 责任全部推给租客、自然损耗与人为损坏不分、大件维修不明确 |
| 转租与退租 | 禁止转租无商量余地、退租通知期过长、恢复原状范围模糊 |
| 合同主体 | 二房东转租未经同意、出租方信息不完整 |

**步骤 2：上传到 Dify 风险知识库**

- Dify → 知识库 → 创建知识库 → 名称：`风险知识库`
- Embedding 模型：bge-large-zh-law
- 分块设置：自定义，最大长度 800 字符，重叠 0
- 上传所有风险点 Markdown 文件

**步骤 3：测试风险知识库检索**

用真实场景 query 验证命中率，不理想则补充同义表述。

**步骤 4：提交**

```bash
git add docs/knowledge-base/risk/
git commit -m "feat: add initial risk knowledge base (15-20 risk points)"
```

---

### 阶段三：Dify 工作流开发

---

### 任务 6：核心合同分析工作流

**步骤 1：创建工作流**

Dify → 创建应用 → 选择"工作流"类型 → 名称：`contract-analysis`

**步骤 2：设计工作流节点**

```
[开始] → [HTTP节点: 腾讯云OCR] → [代码节点: 条款拆分] 
      → [知识检索节点: 法律知识库] → [知识检索节点: 风险知识库]
      → [LLM节点: 风险分析] → [代码节点: 结果整理] → [结束]
```

**步骤 3：配置各节点**

**3a. HTTP 节点（OCR 调用）：**
- 请求方式：POST
- URL：`https://ocr.tencentcloudapi.com/`（腾讯云 OCR 合同识别接口）
- 传入：前端传来的文件 URL 或 base64
- 输出：识别后的合同文本

**3b. 代码节点（条款拆分）：**
- 输入：OCR 文本
- 逻辑：先用正则匹配编号模式（"第X条"、"一、"等）粗分，输出条款列表
- 输出：`{ "clauses": [{ "index": 1, "title": "押金条款", "content": "..." }, ...] }`

**3c. 知识检索节点 1（法律知识库）：**
- 知识库：法律知识库
- 检索模式：混合检索
- Top K：5
- 查询内容：当前条款内容

**3d. 知识检索节点 2（风险知识库）：**
- 知识库：风险知识库
- 检索模式：混合检索
- Top K：8
- 查询内容：当前条款内容

**3e. LLM 节点（风险分析）：**
- 模型：DeepSeek-V4-Pro
- 系统提示词（写入节点配置）：

```
你是租房合同辅助阅读助手。

## 硬性约束
- ✅ 可以：用通俗语言解释合同条款含义、指出值得注意的风险点、
  提供相关法律条文作为参考、给出谈判方向建议
- ❌ 禁止：出具"合规/不合规"结论、提供具体修改措辞、
  建议用户采取或不采取某具体法律行动、声称自己是法律建议
- ❌ 禁用词：违法、违规、无效、不合法、霸王条款、陷阱

## 风险分级
- 🟢 常规提示：行业内常见做法，和多数情况一致
- 🟡 建议关注：可能对你不利，建议核实
- 🔴 强烈建议核实：风险较高，建议慎重

## 输出格式
对每条条款，按以下格式输出：

【风险等级】🔴/🟡/🟢
【涉及主题】押金 / 违约金 / 维修责任 / 租金费用 / 转租退租 / 合同主体 / 合同文本
【通俗解读】用大学生能看懂的大白话解释
【风险说明】哪里可能对你不利、为什么（参考风险知识库检索结果）
【法律参考】
原文："{从法律知识库逐字引用法条原文}"
解读：{LLM 大白话解释}
来源：《法律名称》第X条
【谈判方向】方向性建议，不给具体措辞

## 引用规范
- 只引用知识库检索结果中实际存在的法条，不得编造
- 法条原文从知识库逐字拼入，不润色
- 未检索到相关法条时说"暂未检索到直接相关的法律条文"

## 免责
末尾附上："以上分析仅供参考，不构成法律意见。重大决策请咨询专业律师。"
```

- 用户提示词：`{{当前条款内容}}`
- 上下文：挂接两个知识检索节点的输出

**3f. 代码节点（结果整理）：**
- 输入：LLM 输出
- 逻辑：解析 LLM 输出的结构化文本，提取风险等级计数（N个高风险、N个中风险）
- 输出：`{ "summary": {...}, "clauses": [...], "overall_risk": "high/medium/low" }`

**步骤 4：测试工作流**

在 Dify 中"运行"面板，输入一段模拟合同文本：

```
第一条：押金为三个月租金，即人民币10500元，退租时经出租方验收合格后无息退还。
第二条：租赁期内，房屋内所有设施设备的维修由承租方自行承担。
第三条：承租方提前退租，需支付两个月租金作为违约金，出租方不承担任何违约责任。
```

验证输出是否：
- 按格式输出了每条条款的分析
- 风险等级合理（押金3个月 = 🔴，维修全由租客承担 = 🔴）
- 法条引用准确（第713条维修义务应该命中）
- 末尾有免责声明

**步骤 5：记录工作流配置**

导出 Dify DSL 文件到 `docs/dify-config/contract-analysis.yml`。

**步骤 6：发布工作流并记录 API 端点**

Dify → 发布 → 记录 API 调用地址。

---

### 任务 7：通用问答工作流

**步骤 1：创建工作流**

Dify → 创建应用 → "聊天助手"类型 → 名称：`contract-qa`

**步骤 2：配置**

- 模型：DeepSeek-V4-Pro
- 挂接法律知识库和风险知识库
- 系统提示词：复用任务 6 中的系统提示词，但去掉输出格式约束（问答模式不需要结构化输出）
- 添加上下文变量：支持前端传入历史对话摘要

**步骤 3：测试**

测试以下场景：
- "租房合同一般有哪些坑"
- "押金一般多久退"
- "杭州租房和上海租房有什么区别"

**步骤 4：发布并记录**

导出 DSL，记录 API 端点。

---

### 任务 8：追问对话工作流

**步骤 1：创建追问工作流**

Dify → 创建应用 → "聊天助手"类型 → 名称：`contract-followup`

**步骤 2：设计上下文管理**

在系统提示词中加入 `{{key_contract_info}}` 变量（永久常驻的关键合同信息 JSON）：

```
## 当前合同的關鍵信息（永久保留）
{{key_contract_info}}

## 用户正在追问的条款
{{current_clause}}
```

**步骤 3：配置对话记忆**

- Dify 对话设置 → 开启记忆
- 记忆窗口：最近 10 轮
- 超出窗口后自动摘要压缩

**步骤 4：测试追问场景**

模拟以下对话链：
1. 用户上传合同 → 分析完成
2. 用户追问"押金这条怎么谈"
3. 用户追问"如果房东不答应怎么办"
4. 用户问"那转租这条呢"（切换条款）

验证：第4轮时关键合同信息仍在上下文中，模型没有丢失之前的合同数据。

**步骤 5：发布并记录**

导出 DSL，记录 API 端点。

---

### 阶段四：前端核心页面

---

### 任务 9：项目目录结构与路由

**涉及文件：**
- 新建：`src/main.jsx`
- 修改：`src/App.jsx`
- 新建：`src/pages/Home.jsx`
- 新建：`src/pages/AnalysisResult.jsx`
- 新建：`src/pages/History.jsx`
- 新建：`src/pages/HistoryDetail.jsx`

**步骤 1：配置路由**

修改 `src/App.jsx`：

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import Home from './pages/Home'
import AnalysisResult from './pages/AnalysisResult'
import History from './pages/History'
import HistoryDetail from './pages/HistoryDetail'

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/result/:sessionId" element={<AnalysisResult />} />
        <Route path="/history" element={<History />} />
        <Route path="/history/:sessionId" element={<HistoryDetail />} />
      </Routes>
    </BrowserRouter>
  )
}
```

**步骤 2：创建页面占位组件**

每个页面组件先返回一个简单的标题，确认路由能正常跳转。

**步骤 3：提交**

```bash
git add -A
git commit -m "feat: add routing and page placeholders"
```

---

### 任务 10：首页（智能输入入口 + 空状态）

**涉及文件：**
- 修改：`src/pages/Home.jsx`
- 新建：`src/components/InputArea.jsx`
- 新建：`src/components/ExampleCards.jsx`
- 新建：`src/components/DisclaimerModal.jsx`

**步骤 1：实现 InputArea 组件**

`src/components/InputArea.jsx`：

```jsx
import { useState } from 'react'

export default function InputArea({ onSubmit }) {
  const [text, setText] = useState('')
  const [file, setFile] = useState(null)

  const handleSubmit = () => {
    if (!text && !file) return
    onSubmit({ text, file })
  }

  return (
    <div className="flex flex-col gap-3 p-4">
      <textarea
        className="w-full p-3 border rounded-lg resize-none text-base"
        placeholder="上传合同图片/PDF，或直接输入你的问题..."
        rows={4}
        value={text}
        onChange={(e) => setText(e.target.value)}
      />
      <div className="flex gap-3">
        <label className="flex items-center gap-2 px-4 py-2 bg-gray-100 rounded-lg cursor-pointer">
          📎 上传合同
          <input
            type="file"
            accept="image/*,.pdf"
            className="hidden"
            onChange={(e) => setFile(e.target.files[0])}
          />
        </label>
        {file && <span className="text-sm text-gray-500 self-center">{file.name}</span>}
      </div>
      <button
        className="w-full py-3 bg-blue-600 text-white rounded-lg font-medium disabled:opacity-50"
        disabled={!text && !file}
        onClick={handleSubmit}
      >
        开始分析
      </button>
    </div>
  )
}
```

**步骤 2：实现 ExampleCards 组件**

`src/components/ExampleCards.jsx`：

```jsx
const examples = [
  { icon: '📄', text: '拍一张押金条款试试', action: 'upload' },
  { icon: '❓', text: '"租房合同一般有哪些坑"', action: 'text' },
  { icon: '🔍', text: '"违约金多少算合理？"', action: 'text' },
]

export default function ExampleCards({ onExampleClick }) {
  return (
    <div className="flex flex-col gap-2 px-4">
      <p className="text-sm text-gray-400">💡 试试这些：</p>
      {examples.map((ex, i) => (
        <div
          key={i}
          className="p-3 bg-gray-50 rounded-lg text-sm cursor-pointer active:bg-gray-100"
          onClick={() => onExampleClick(ex)}
        >
          {ex.icon} {ex.text}
        </div>
      ))}
    </div>
  )
}
```

**步骤 3：组装 Home 页面**

`src/pages/Home.jsx`：

```jsx
import { useNavigate } from 'react-router-dom'
import InputArea from '../components/InputArea'
import ExampleCards from '../components/ExampleCards'
import DisclaimerModal from '../components/DisclaimerModal'
import { useState, useEffect } from 'react'

export default function Home() {
  const navigate = useNavigate()
  const [showDisclaimer, setShowDisclaimer] = useState(false)
  const [pendingSubmit, setPendingSubmit] = useState(null)
  const [agreed, setAgreed] = useState(false)

  // 首次访问弹隐私协议
  useEffect(() => {
    if (!localStorage.getItem('agreed_disclaimer')) {
      setShowDisclaimer(true)
    } else {
      setAgreed(true)
    }
  }, [])

  const handleSubmit = ({ text, file }) => {
    if (!agreed) {
      setPendingSubmit({ text, file })
      setShowDisclaimer(true)
      return
    }
    // 后续任务实现实际的上传和分析逻辑
    navigate('/result/demo-session')
  }

  const handleAgree = () => {
    localStorage.setItem('agreed_disclaimer', 'true')
    setAgreed(true)
    setShowDisclaimer(false)
  }

  return (
    <div className="min-h-screen bg-white flex flex-col">
      <header className="pt-12 pb-6 text-center">
        <h1 className="text-2xl font-bold">🏠 租房合同助手</h1>
        <p className="text-gray-500 mt-2">AI 帮你读懂租房合同里的门道</p>
      </header>

      <InputArea onSubmit={handleSubmit} />
      <ExampleCards onExampleClick={(ex) => {
        if (ex.action === 'text') {
          // 填入文字到输入框
        }
      }} />

      {showDisclaimer && (
        <DisclaimerModal onAgree={handleAgree} />
      )}
    </div>
  )
}
```

**步骤 4：实现 DisclaimerModal 组件**

`src/components/DisclaimerModal.jsx`：

```jsx
export default function DisclaimerModal({ onAgree }) {
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center p-6 z-50">
      <div className="bg-white rounded-xl p-6 max-w-sm">
        <h2 className="text-lg font-bold mb-4">⚠️ 使用须知</h2>
        <p className="text-sm text-gray-600 mb-4">
          本工具是 AI 辅助阅读助手，分析结果<strong>仅供参考，不构成法律建议</strong>。
        </p>
        <p className="text-sm text-gray-600 mb-6">
          如涉及重大决策（如签署合同、产生纠纷），请咨询专业律师。
        </p>
        <div className="flex gap-3">
          <a href="/privacy" className="flex-1 py-2 text-center text-sm text-blue-600 border rounded-lg">
            隐私政策
          </a>
          <button
            className="flex-1 py-2 bg-blue-600 text-white rounded-lg font-medium"
            onClick={onAgree}
          >
            开始分析
          </button>
        </div>
      </div>
    </div>
  )
}
```

**步骤 5：提交**

```bash
git add -A
git commit -m "feat: implement home page with input area, example cards, disclaimer modal"
```

---

### 任务 11：数据脱敏工具函数

**涉及文件：**
- 新建：`src/utils/desensitize.js`

**步骤 1：实现脱敏函数**

`src/utils/desensitize.js`：

```js
/**
 * 对文本中的个人敏感信息进行脱敏处理
 * 在数据离开浏览器之前调用
 */
export function desensitize(text) {
  let result = text

  // 身份证号：18位或17位+X
  result = result.replace(
    /\b\d{17}[\dXx]\b/g,
    '[身份证号]'
  )

  // 手机号：1开头的11位数字
  result = result.replace(
    /\b1[3-9]\d{9}\b/g,
    '[手机号]'
  )

  // 银行卡号：16-19位数字
  result = result.replace(
    /\b\d{16,19}\b/g,
    '[银行卡号]'
  )

  // 详细地址（省份+市+区+详细）：命中省市区关键词的长文本
  result = result.replace(
    /(?:浙江省|浙江省|北京市|上海市|杭州市)\S{1,30}(?:路|街|巷|号|小区|栋|幢|单元|室)/g,
    '[地址]'
  )

  // 中文姓名（2-4个字，紧跟"甲方/乙方/出租方/承租方/姓名"等标识后）
  result = result.replace(
    /((?:甲方|乙方|出租方|承租方|承租人|出租人|姓名|签字)[：:]\s*)[一-龥]{2,4}(?=\s|[，,。\.\n]|$)/g,
    '$1[姓名]'
  )

  return result
}
```

**步骤 2：提交**

```bash
git add -A
git commit -m "feat: add data desensitization utility"
```

---

### 任务 12：API 服务层

**涉及文件：**
- 新建：`src/services/api.js`
- 新建：`src/services/dify.js`

**步骤 1：封装 HTTP 请求基类**

`src/services/api.js`：

```js
import axios from 'axios'

// Dify API 基础地址（Cloudflare Tunnel URL）
const DIFY_BASE_URL = import.meta.env.VITE_DIFY_BASE_URL || 'https://xxx.trycloudflare.com'

const apiClient = axios.create({
  baseURL: DIFY_BASE_URL,
  timeout: 60000, // 合同分析可能需要较长时间
  headers: {
    'Content-Type': 'application/json',
  },
})

// 请求拦截器：添加进度回调
apiClient.interceptors.request.use((config) => {
  return config
})

// 响应拦截器：统一错误处理
apiClient.interceptors.response.use(
  (response) => response.data,
  (error) => {
    if (error.code === 'ECONNABORTED') {
      throw new Error('请求超时，请稍后重试')
    }
    if (!error.response) {
      throw new Error('网络连接异常，请检查网络后重试')
    }
    throw new Error(error.response.data?.message || '服务异常，请稍后重试')
  }
)

export default apiClient
```

**步骤 2：封装 Dify 接口**

`src/services/dify.js`：

```js
import apiClient from './api'

/**
 * 分析合同
 * @param {string} contractText - 脱敏后的合同文本
 * @param {string} userQuestion - 用户附带的问题（可选）
 * @param {function} onProgress - 进度回调
 */
export async function analyzeContract(contractText, userQuestion = '', onProgress) {
  const API_KEY = import.meta.env.VITE_DIFY_CONTRACT_ANALYSIS_KEY

  return apiClient.post('/v1/workflows/run', {
    inputs: {
      contract_text: contractText,
      user_question: userQuestion,
    },
    response_mode: 'streaming',
    user: 'anonymous',
  }, {
    headers: { 'Authorization': `Bearer ${API_KEY}` },
    responseType: 'stream',
    onDownloadProgress: (progressEvent) => {
      // 解析 streaming 事件，提取进度阶段
      // Dify workflow 返回的是 SSE 事件流
    },
  })
}

/**
 * 通用问答
 */
export async function chatQuery(question, conversationId = '') {
  const API_KEY = import.meta.env.VITE_DIFY_QA_KEY

  return apiClient.post('/v1/chat-messages', {
    inputs: {},
    query: question,
    response_mode: 'streaming',
    conversation_id: conversationId,
    user: 'anonymous',
  }, {
    headers: { 'Authorization': `Bearer ${API_KEY}` },
  })
}

/**
 * 追问对话
 */
export async function followUpQuery(question, conversationId, keyContractInfo, currentClause) {
  const API_KEY = import.meta.env.VITE_DIFY_FOLLOWUP_KEY

  return apiClient.post('/v1/chat-messages', {
    inputs: {
      key_contract_info: JSON.stringify(keyContractInfo),
      current_clause: currentClause,
    },
    query: question,
    response_mode: 'streaming',
    conversation_id: conversationId,
    user: 'anonymous',
  }, {
    headers: { 'Authorization': `Bearer ${API_KEY}` },
  })
}
```

**步骤 3：创建环境变量配置文件**

`.env.example`：

```
VITE_DIFY_BASE_URL=https://xxx.trycloudflare.com
VITE_DIFY_CONTRACT_ANALYSIS_KEY=app-xxxx
VITE_DIFY_QA_KEY=app-xxxx
VITE_DIFY_FOLLOWUP_KEY=app-xxxx
```

**步骤 4：提交**

```bash
git add -A
git commit -m "feat: add API service layer with Dify integration"
```

---

### 任务 13：分析结果页面（核心页面）

**涉及文件：**
- 修改：`src/pages/AnalysisResult.jsx`
- 新建：`src/components/ContractHighlight.jsx`
- 新建：`src/components/RiskCard.jsx`
- 新建：`src/components/ProgressBar.jsx`
- 新建：`src/components/FollowUpChat.jsx`

**步骤 1：实现 ProgressBar 组件**

`src/components/ProgressBar.jsx`：

```jsx
const stages = [
  { key: 'reading', icon: '📄', label: '正在读取文件...' },
  { key: 'ocr', icon: '🔍', label: '正在识别合同条款...' },
  { key: 'rag', icon: '📚', label: '正在检索相关法规和案例...' },
  { key: 'llm', icon: '🧠', label: '正在分析风险点...' },
  { key: 'done', icon: '✅', label: '分析完成' },
]

export default function ProgressBar({ currentStage }) {
  return (
    <div className="flex flex-col items-center justify-center py-20 gap-4">
      <div className="animate-spin text-3xl">⏳</div>
      {stages.map((stage, i) => (
        <div
          key={stage.key}
          className={`text-sm ${stages.findIndex(s => s.key === currentStage) >= i
            ? 'text-gray-900'
            : 'text-gray-300'}`}
        >
          {stage.icon} {stage.label}
        </div>
      ))}
    </div>
  )
}
```

**步骤 2：实现 ContractHighlight 组件**

`src/components/ContractHighlight.jsx`：

```jsx
export default function ContractHighlight({ clauses, activeClauseIndex, onClauseClick }) {
  const riskColor = {
    high: 'bg-red-100 border-l-4 border-red-500',
    medium: 'bg-yellow-100 border-l-4 border-yellow-500',
    low: 'bg-green-100 border-l-4 border-green-500',
  }

  return (
    <div className="p-3 text-sm">
      <h3 className="font-bold mb-3 text-base">📄 合同条款</h3>
      {clauses.map((clause, i) => (
        <div
          key={i}
          className={`p-2 mb-2 rounded cursor-pointer transition-all ${
            riskColor[clause.riskLevel] || ''
          } ${activeClauseIndex === i ? 'ring-2 ring-blue-500' : ''}`}
          onClick={() => onClauseClick(i)}
        >
          {clause.riskLevel === 'high' && '🔴 '}
          {clause.riskLevel === 'medium' && '🟡 '}
          {clause.riskLevel === 'low' && '🟢 '}
          <span className="font-medium">{clause.title || `第${i + 1}条`}</span>
          <p className="text-gray-600 mt-1 line-clamp-2">{clause.content}</p>
        </div>
      ))}
    </div>
  )
}
```

**步骤 3：实现 RiskCard 组件**

`src/components/RiskCard.jsx`：

```jsx
import { useState } from 'react'

export default function RiskCard({ clause, onFollowUp, onFeedback }) {
  const [feedback, setFeedback] = useState(null)
  const [showReason, setShowReason] = useState(false)

  return (
    <div className="p-4 border-b">
      <div className="flex items-center gap-2 mb-2">
        <span className="text-lg">
          {clause.riskLevel === 'high' && '🔴 强烈建议核实'}
          {clause.riskLevel === 'medium' && '🟡 建议关注'}
          {clause.riskLevel === 'low' && '🟢 常规提示'}
        </span>
        <span className="text-xs bg-gray-100 px-2 py-0.5 rounded">
          {clause.topic}
        </span>
      </div>

      <div className="mb-3">
        <h4 className="font-medium text-sm text-gray-500 mb-1">通俗解读</h4>
        <p className="text-sm">{clause.plainExplanation}</p>
      </div>

      <div className="mb-3">
        <h4 className="font-medium text-sm text-gray-500 mb-1">风险说明</h4>
        <p className="text-sm">{clause.riskExplanation}</p>
      </div>

      <div className="mb-3 bg-gray-50 p-3 rounded">
        <h4 className="font-medium text-sm text-gray-500 mb-1">法律参考</h4>
        <blockquote className="text-sm text-gray-700 italic border-l-2 border-gray-300 pl-3 mb-1">
          "{clause.lawReference?.original}"
        </blockquote>
        <p className="text-sm text-gray-600">{clause.lawReference?.explanation}</p>
        <p className="text-xs text-gray-400 mt-1">来源：{clause.lawReference?.source}</p>
      </div>

      <div className="mb-3">
        <h4 className="font-medium text-sm text-gray-500 mb-1">谈判方向</h4>
        <p className="text-sm">{clause.negotiationAdvice}</p>
      </div>

      <div className="flex items-center justify-between pt-2 border-t">
        <button
          className="text-sm text-blue-600"
          onClick={() => onFollowUp(clause)}
        >
          💬 追问这条
        </button>
        <div className="flex gap-2">
          <button
            className={`text-sm px-3 py-1 rounded ${feedback === 'up' ? 'bg-blue-100 text-blue-600' : 'bg-gray-100'}`}
            onClick={() => { setFeedback('up'); onFeedback('up', clause) }}
          >
            👍 有用
          </button>
          <button
            className={`text-sm px-3 py-1 rounded ${feedback === 'down' ? 'bg-red-100 text-red-600' : 'bg-gray-100'}`}
            onClick={() => {
              setFeedback('down')
              setShowReason(true)
              onFeedback('down', clause)
            }}
          >
            👎 没用
          </button>
        </div>
      </div>

      {showReason && (
        <input
          className="w-full mt-2 p-2 border rounded text-sm"
          placeholder="哪里不满意？（选填）"
          onBlur={(e) => {/* 保存反馈原因 */}}
        />
      )}
    </div>
  )
}
```

**步骤 4：实现 FollowUpChat 组件**

`src/components/FollowUpChat.jsx`：

```jsx
import { useState } from 'react'

export default function FollowUpChat({ clause, messages, onSend, onReset }) {
  const [input, setInput] = useState('')

  const handleSend = () => {
    if (!input.trim()) return
    onSend(input)
    setInput('')
  }

  return (
    <div className="border-t mt-4">
      <div className="flex items-center justify-between p-3 border-b">
        <h4 className="font-bold text-sm">💬 追问对话</h4>
        <button className="text-xs text-gray-400" onClick={onReset}>
          重新开始
        </button>
      </div>

      <div className="p-3 max-h-64 overflow-y-auto">
        {messages.map((msg, i) => (
          <div key={i} className={`mb-3 ${msg.role === 'user' ? 'text-right' : ''}`}>
            <div
              className={`inline-block p-2 rounded-lg text-sm max-w-[80%] ${
                msg.role === 'user'
                  ? 'bg-blue-600 text-white'
                  : 'bg-gray-100 text-gray-900'
              }`}
            >
              {msg.content}
            </div>
          </div>
        ))}
      </div>

      <div className="flex gap-2 p-3 border-t">
        <input
          className="flex-1 p-2 border rounded-lg text-sm"
          placeholder="继续追问..."
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && handleSend()}
        />
        <button
          className="px-4 py-2 bg-blue-600 text-white rounded-lg text-sm"
          onClick={handleSend}
        >
          发送
        </button>
      </div>
    </div>
  )
}
```

**步骤 5：组装 AnalysisResult 页面**

`src/pages/AnalysisResult.jsx`：

完整页面包含：
- 顶部：合同标题 + 整体风险摘要（N个高风险/N个中风险/N个低风险）
- 移动端用 Tab 切换"合同原文"和"风险解读"
- 桌面端左右分栏（但 MVP 移动端优先，先做 Tab 模式）
- 底部：追问对话区域

**步骤 6：提交**

---

### 任务 14：历史记录功能

**涉及文件：**
- 新建：`src/utils/storage.js`
- 修改：`src/pages/History.jsx`
- 修改：`src/pages/HistoryDetail.jsx`

**步骤 1：实现 localStorage 封装**

`src/utils/storage.js`：

```js
const STORAGE_KEY = 'rental_contract_history'

export function getHistoryList() {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]')
  } catch {
    return []
  }
}

export function saveAnalysis(session) {
  const list = getHistoryList()
  // 不存合同原文和OCR原始文本
  const record = {
    id: Date.now().toString(),
    title: session.title || '未命名合同',
    date: new Date().toISOString(),
    riskSummary: session.riskSummary,
    clauses: session.clauses.map(c => ({
      riskLevel: c.riskLevel,
      topic: c.topic,
      plainExplanation: c.plainExplanation,
      riskExplanation: c.riskExplanation,
      lawReference: c.lawReference,
      negotiationAdvice: c.negotiationAdvice,
    })),
    keyContractInfo: session.keyContractInfo,
  }
  list.unshift(record)
  localStorage.setItem(STORAGE_KEY, JSON.stringify(list.slice(0, 100))) // 最多100条
  return record
}

export function getAnalysis(id) {
  const list = getHistoryList()
  return list.find(r => r.id === id)
}

export function deleteAnalysis(id) {
  const list = getHistoryList().filter(r => r.id !== id)
  localStorage.setItem(STORAGE_KEY, JSON.stringify(list))
}
```

**步骤 2：实现 History 列表页**

- 显示历史记录列表（时间、标题、风险摘要）
- 空状态提示
- 点击进入详情
- 底部"新建分析"按钮

**步骤 3：实现 HistoryDetail 详情页**

- 从 localStorage 读取记录
- 渲染 AnalysisResult 的只读版本
- 仍可发起追问（关键合同信息已保存）

**步骤 4：提交**

---

### 任务 15：分享功能

**涉及文件：**
- 新建：`src/utils/share.js`
- 修改：`src/pages/AnalysisResult.jsx`

**步骤 1：实现分享卡片生成**

`src/utils/share.js`：

```js
export function generateShareCard(analysis) {
  const highCount = analysis.clauses.filter(c => c.riskLevel === 'high').length
  const mediumCount = analysis.clauses.filter(c => c.riskLevel === 'medium').length

  return {
    title: `我用AI分析了这份租房合同，发现${highCount}个高风险条款`,
    description: `🔴 ${highCount}个强烈建议核实 🟡 ${mediumCount}个建议关注`,
    text: `🔴 ${highCount}个高风险 🟡 ${mediumCount}个中风险\n点击查看完整分析→`,
  }
}

export function shareAnalysis(analysis) {
  const card = generateShareCard(analysis)

  if (navigator.share) {
    // 移动端原生分享
    navigator.share({
      title: card.title,
      text: card.text,
      url: window.location.href,
    }).catch(() => {
      // 用户取消分享，忽略
    })
  } else {
    // 降级：复制链接
    navigator.clipboard.writeText(
      `${card.title}\n${card.text}\n${window.location.href}`
    ).then(() => {
      alert('链接已复制，粘贴分享给同学吧')
    })
  }
}
```

**步骤 2：在 AnalysisResult 页面加入分享按钮**

- 分析完成后，顶部显示"📤 分享给同学"按钮
- 点击触发分享

**步骤 3：提交**

---

### 阶段五：异常处理与打磨

---

### 任务 16：网络状态检测与异常处理

**涉及文件：**
- 新建：`src/hooks/useNetworkStatus.js`
- 新建：`src/components/ErrorBoundary.jsx`
- 新建：`src/components/Toast.jsx`

**步骤 1：实现网络状态 Hook**

`src/hooks/useNetworkStatus.js`：

```js
import { useState, useEffect } from 'react'

export default function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine)

  useEffect(() => {
    const handleOnline = () => setIsOnline(true)
    const handleOffline = () => setIsOnline(false)

    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)

    return () => {
      window.removeEventListener('online', handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])

  return isOnline
}
```

**步骤 2：实现 Toast 提示组件**

轻量级 toast，用于网络异常提示、操作反馈等。

**步骤 3：实现 ErrorBoundary**

React 错误边界，全局捕获渲染异常，展示降级页面。

**步骤 4：在所有 API 调用中加入异常处理**

- 超时 → 重试一次 → 仍失败 → Toast 提示
- 网络断开 → Toast "网络连接异常"
- 服务端错误 → Toast "服务暂时不可用"

**步骤 5：提交**

---

### 任务 17：用户协议与隐私政策页面

**涉及文件：**
- 新建：`src/pages/Privacy.jsx`
- 新建：`src/pages/Terms.jsx`

**步骤 1：编写隐私政策页面**

参考行业模板（腾讯电子签、法大大），覆盖：
- 收集哪些信息（仅脱敏后的合同摘要）
- 信息如何存储（浏览器本地存储，不上传服务器）
- 数据不用于 AI 模型训练
- 分析后合同原文自动删除
- 用户权利（删除历史记录 = 清除所有本地数据）

**步骤 2：编写用户协议页面**

覆盖：
- 服务性质（AI 辅助阅读，不构成法律意见）
- 用户责任（自行判断分析结果适用性）
- 使用限制（禁止上传涉密内容）
- 免责声明

**步骤 3：提交**

---

### 阶段六：部署上线

---

### 任务 18：部署到 Vercel

**步骤 1：构建生产版本**

```bash
npm run build
```

确认 `dist/` 目录生成成功，无构建错误。

**步骤 2：部署到 Vercel**

```bash
# 方式一：Vercel CLI
npm install -g vercel
vercel --prod
```

或

```bash
# 方式二：连接 GitHub 仓库，Vercel 自动部署
git remote add origin <your-github-repo>
git push -u origin main
# 然后在 Vercel 控制台 import 项目
```

**步骤 3：配置环境变量**

在 Vercel 控制台 → Settings → Environment Variables：
- `VITE_DIFY_BASE_URL` = Cloudflare Tunnel URL
- `VITE_DIFY_CONTRACT_ANALYSIS_KEY` = Dify 工作流 API Key
- `VITE_DIFY_QA_KEY` = Dify 问答 API Key
- `VITE_DIFY_FOLLOWUP_KEY` = Dify 追问 API Key

**步骤 4：验证部署**

浏览器访问 Vercel 分配的域名（如 `https://xxx.vercel.app`），确认：
- 首页正常加载
- 隐私弹窗出现
- 上传测试合同 → 分析完成 → 结果展示
- 追问功能正常
- 分享功能可用

**步骤 5：提交**

---

### 任务 19：配置自定义域名（可选，后续）

- 购买域名（如通过 Cloudflare）
- 配置 DNS 指向 Vercel
- 配置 HTTPS 证书（自动）

---

## 验证方式

### 端到端测试场景

| 场景 | 操作 | 期望结果 |
|---|---|---|
| **首次访问** | 打开网址 | 显示首页 + 弹出免责弹窗 |
| **纯文字提问** | 输入"租房合同一般有哪些坑" | 返回避坑知识回答，无合同原文高亮 |
| **上传合同图片** | 上传一张租房合同照片 | OCR 识别 → 进度条展示 → 返回逐条分析 |
| **风险等级色** | 查看分析结果 | 🔴🟡🟢 颜色区分清晰 |
| **追问** | 点击某条"追问这条"，输入"怎么谈" | 返回针对该条款的追问回答，不丢失上下文 |
| **反馈** | 点击 👍 或 👎 | 按钮状态切换，👎 弹出原因输入框 |
| **历史记录** | 返回首页 → 点击历史 | 看到刚才的分析记录 |
| **分享** | 点击分享按钮 | 移动端弹出原生分享面板 / PC端复制链接 |
| **脱敏验证** | 上传一份含真实姓名的合同 | 分析结果中姓名显示为 [姓名] |
| **断网处理** | 关闭WiFi后提交分析 | Toast 提示"网络连接异常" |
| **免责弹窗（第二次）** | 分析完一份后，重新上传新合同 | 再次弹出免责确认 |

---

## 风险与注意事项

| 风险 | 应对 |
|---|---|
| Dify 工作流复杂度可能超出预期 | 任务6-8 可先简化为一个聊天助手完成全部功能，后续再拆分成多个工作流 |
| bge-large-zh-law 本地部署配置繁琐 | 若 Ollama 部署困难，暂时用 Dify 自带 Embedding，检索质量够用再换 |
| Cloudflare Tunnel 断开导致服务不可用 | 写一个守护脚本定时检查 Tunnel 状态；在首页展示"服务状态"标识 |
| 腾讯云 OCR 费用不可控 | 先接入百度 OCR（有免费额度），量上来再切换 |
| 风险知识库初始内容不够 | MVP 20 个风险点打底，配合用户反馈 👎 的数据优先补充高频不满意的点 |
