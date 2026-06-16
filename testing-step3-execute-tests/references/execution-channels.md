# 四条证据通道 · 执行细则与脚本范式

> 每条用例的"实际结果"必须来自真实证据。证据分级：接口/DB 实测 > 浏览器实测 > 代码佐证。

## 通道 1 · 浏览器（Playwright MCP）— UI 与面4 主通道
- 加载工具：ToolSearch `select:mcp__playwright__browser_navigate,mcp__playwright__browser_click,mcp__playwright__browser_type,mcp__playwright__browser_snapshot,mcp__playwright__browser_evaluate,mcp__playwright__browser_take_screenshot,...`
- 操作：navigate → snapshot 取 ref → click/type/press_key → 截图。表单 combobox 用 `browser_type` 填(合成事件可能不触发 React onChange，必要时 snapshot 确认)。
- **运行期 evaluate** 取代码读不到的真相：DOM 实际文案/枚举、`video.readyState/currentSrc`、`getBoundingClientRect`、`localStorage`、按钮 disabled、校验提示文案。比"看截图"精确。
- **门禁/上下文**：很多页面需先建立上下文(如某页需某业务 ID/会话态)——**走真实业务流进入受保护页面**(在前置页发起业务建立上下文，再进入)，而非硬闯 URL。
- 截图即证据：见 `pitfalls.md`(文件名禁括号/落盘校验/断链校验/MCP cwd≠shell cwd)。
- **★瞬时 toast/snackbar 文案的捕获（易截空漏证）**：成功/错误 toast 多有几秒 TTL，"操作完再 take_screenshot"必然截空 → 用例的可断言预期(toast 原文)漏取证。配方：**触发操作前先布捕获**——① `browser_evaluate` 注册 `MutationObserver` 监听 toast 容器、把出现的文本推进一个全局数组，操作后读该数组拿原文；或 ② `browser_wait_for`(等 toast 文本出现)与点击**并发**、命中即截图。**抓不到时**别硬等：以"Modal 关闭 + 列表出现/消失该行 + DB 写后回读"为主证据判通过，toast 子断言单列"未取证(TTL 过期)"——**不冒充"文案已逐字核对"**。

## 通道 1.5 · UI 的 API/函数化验证（★能测必测、省 token 的中间通道，别把 UI 框死成"只走浏览器 or 只看代码"）
很多 UI 用例不必真开浏览器，也能**程序化真测**——这比浏览器串行快、省 token，且算"真测"（接口真发请求/函数真执行）：
- **(a) 抓 UI 操作背后的真实 API 直接发请求**：UI 的提交/加载/搜索/筛选/状态流转/CRUD/上传，背后都是 API。读前端代码找到该操作调用的端点+入参，用通道2 的 node fetch **直接真发请求 + DB 写后回读**——等价验证了该 UI 操作的后端效果（如"点保存"=POST /xxx，真发请求看真入库）。
- **(b) 前端纯函数 node 复刻执行对值**：校验规则(正则/范围/必填)、计算(合计/时长/统计)、格式化、`statusMap`/枚举映射、`inferXxx` 推断等**纯前端逻辑**，把函数体抄出来在 node 真跑、对边界值断言输出——比"读代码说它对"强，是真执行。
- **何时仍必须浏览器**：必须看**渲染/交互态**的（布局、弹窗出现、可见性、disabled、拖拽、canvas、状态色、跳转、toast 文案）——这些没有 API/函数等价物，只能浏览器真操作+截图。
- **判断顺序**（呼应 SKILL §1 全局"能测必测"）：能(a)/(b) → 优先 API/函数真测；必须看渲染交互 → 浏览器真操作+截图；二者都不行且物理受限 → 阻塞+进专项；纯前端无 API 无函数可验 → 才源码佐证。
- **★判"纯前端无API"的最低门槛（防浅尝即止退源码佐证）**：判定一条 UI"无 API/函数可验、只能源码佐证"前，**必须做完**：① 在浏览器开发者 Network 面板触发一次该操作、看有没有发请求；② 在前端源码全局搜该操作的 handler/调用名，确认无 fetch/axios 调用、无可抽出的纯函数。**两步都做过仍无 → 才判"纯前端"，并把"查过 Network 无请求 + grep 哪些关键词无果"留痕**供审计抽查；没留痕的"纯前端无API"视为未尽力、应补查。
> **★面4 例外（三重，不是双重）**：面4 文档符合性**三者都要**，缺任一重不合格——① 浏览器逐条截图；② 对每条声称做 API/函数真检测（**"判有/判部分"发对应接口/调对应函数互证；"判无"=功能不存在，用"代码检索无端点/无路由/无前端函数" + "UI 无入口截图"佐证，不必对不存在的功能发请求**）；③ 逐条审计完成。

## 通道 1.6 · 非点击交互的真实操作配方（★通用手段，不是画布特例）
**合成 DOM 事件普遍不触发真实 handler**：`browser_evaluate` 里 `dispatchEvent`/合成 PointerEvent **常不触发** React 受控组件、konva/canvas、拖拽、滑块、悬停菜单的真实 handler（不只 canvas——combobox onChange 也踩过）；**一切非点击交互必须用 Playwright 真实输入工具或底层真实指针轨迹，禁止用合成事件冒充操作**。逐类配方（每类附"操作后怎么回读验证真生效"）：
- **canvas / 自由绘制 / 拖拽轨迹**：`browser_run_code_unsafe` 的 `page.mouse` 做 mousedown → **多段 mousemove(逐点真实坐标轨迹)** → mouseup；回读 shapeCount / 撤销键 disabled→enabled / DOM 变化，验证真画上。
- **拖放排序 / 看板**：`browser_drag` + `browser_drop`；回读顺序/归属变化。
- **悬停菜单 / tooltip / 浮层预览**：`browser_hover`(或 page.mouse.move) 后 snapshot/截图，验浮层出现、预览区随鼠标变化。
- **滑块 / 进度条**：mousedown 到手柄 → 分段 mousemove → mouseup；回读 value/aria-valuenow。
- **下拉**：原生 select 用 `browser_select_option`；自定义下拉 click 展开 + click 选项。
- **快捷键**：`browser_press_key`；回读对应效果。
- **弹窗 / 二次确认**：原生 dialog 用 `browser_handle_dialog`；Modal 确认/取消按钮分别真点；弹窗内控件逐个真操作。
- **文件上传 / 拖放**：`browser_file_upload`。
> **兜底原则（防默认走源码佐证）**：canvas/拖拽默认用 page.mouse 真实轨迹真测；**仅在真实拖拽确证物理卡死(MCP hang)且小粒度/主控串行仍不可行时**才标阻塞并进 §10 专项——**绝不把"可真测但麻烦"默认沉淀为"源码佐证·待复核"**（画布自由绘制/区域缩放就是被这个默认兜底漏掉的）。

## 通道 2 · 接口（node 全局 fetch / curl）— API 类主通道（必须真跑）
登录取凭证，按本项目鉴权方式携带，打端点看真实状态码 + 错误文案原文：
```js
// 登录拿 token —— /api/v1/auth/login、data.access_token、Bearer、username 均为【示例端点/字段，按本项目鉴权方式替换】
const lg = await (await fetch(BASE+'/api/v1/auth/login',{method:'POST',
  headers:{'Content-Type':'application/json'},body:JSON.stringify({username:'<示例账号>',password:'***'})})).json();
const token = lg.data.access_token;
// 带凭证打端点 / 边界 / 越权三态(无凭证、低权、高权)各发一次
const r = await fetch(BASE+'/api/v1/users', {headers:{Authorization:'Bearer '+token}});
console.log(r.status, JSON.stringify(await r.json()).slice(0,200));
```
- **★鉴权形态各异，取凭证与携带方式相应调整**：上例是"POST 登录端点拿 Bearer token"的一种；实际鉴权形态可能是 **cookie-session（带 `Cookie`/`Set-Cookie`、CSRF token）/ API key（请求头或 query）/ OAuth（拿 access_token，可能需刷新）/ Bearer JWT / mTLS（客户端证书）**——登录端点路径、取凭证字段名、携带头按本项目实际替换，不要硬套 `/api/v1/auth/login` + `data.access_token` + `Bearer`。
- **每端点穷举**：正向 + 每条错误分支(逐状态码+文案) + 每字段边界(空/超长/特殊字符/中文/最值±1/格式正则/唯一重复) + 鉴权三态——各发真请求。
- **越权/匿名缺陷**：必须真发匿名(无凭证)/越权(低权)请求看真返回，**不能用带权请求冒充**。
- 内联命令对中文/反斜杠易出错（尤其某些 shell）→ **写 `.mjs` 文件再 `node x.mjs`**，别用长 `node -e`（环境特化注意见 `pitfalls.md`）。

## 通道 2.5 · WebSocket / 实时双工真连真测（★条件触发：有 WS/SSE/实时通道才适用；与通道2 同级强制——API 类能测必测、源码佐证=0）
有实时双工/推送通道（WebSocket / socket.io / SSE）的系统，**不能只读代码说"它会推"**——必须**真建连、真收帧、真断言**。在 node 用 `ws` 或 `socket.io-client`：建连 → 鉴权握手 → `send` 帧 → **断言收到的 server/广播帧**（类型 / payload / 顺序 / ack）。最小 `.mjs` 范式（端点/事件名/鉴权字段均为示例，按本项目替换）：
```js
// ws_probe.mjs —— 原生 WebSocket；socket.io 改用 io(URL,{auth:{token}}) + socket.on(event,cb)
import WebSocket from 'ws';
const ws = new WebSocket('wss://<示例HOST>/ws?token='+TOKEN);   // 鉴权握手按项目实际(query/header/连后首帧)
const got = [];                                                 // 收帧 buffer
ws.on('open',  () => ws.send(JSON.stringify({type:'subscribe', room:'<示例房间>'})));
ws.on('message', d => got.push(JSON.parse(d.toString())));      // 真实收到的帧
await new Promise(r => setTimeout(r, 3000));                     // 超时判定：限定时间内未收到=失败
ws.close();
// 断言：收到的帧类型/payload/顺序/ack 是否符合预期；未收到任何帧 = 推送缺陷(非"代码里有 emit 就算过")
console.log(JSON.stringify(got));
```
- **连接生命周期真测**：建连成功/鉴权失败被拒/心跳保活/断线后是否补发漏帧/重连后状态一致。
- **顺序与去重**：连发多帧断言**到达顺序**、断言**不重复投递**；需要 ack 的断言收到 ack。
- **协同/广播**：写操作触发的广播，用**第二个连接**断言它在限定时间内收到对应变更帧（呼应 SKILL §3 协同写双 oracle）。
- 没有实时通道的系统此通道整体 N/A，不必论证"不适用"。

## 通道 3 · 数据库写后回读 — 区分"前端态 vs 真落库"（最易漏）
**★原则（技术栈无关，永远适用）**：写操作后，**用任意可信手段直查数据存储**，核对**真落库 / 服务端默认值 / 级联**等副作用；**优先用与被测系统相同的连接配置**（同库同账号同 schema）查询，**避免连接配置漂移**（连错库/连错环境会得到假阴/假阳）。具体用什么客户端/ORM/CLI 按你的栈与运行环境选。

**示例：Python + SQLAlchemy + Docker 项目的一种实现**（其它栈见下方"多栈替代"）：
```python
# dbq.py：从环境变量 Q 读 SQL(;; 分隔)，用后端 SessionLocal 执行，打印 JSON
import os, json
from app.core.dependencies import SessionLocal   # 按项目实际模块路径调整
from sqlalchemy import text
db = SessionLocal()
for q in os.environ['Q'].split(';;'):
    if not q.strip(): continue
    try:
        rows = db.execute(text(q)).mappings().all()
        print(json.dumps([dict(r) for r in rows], default=str, ensure_ascii=False))
    except Exception as e:
        print('### ERR', str(e)[:200])
```
```bash
# 调用方式按你的运行环境调整（下例为容器内执行后端 Python 解释器查库的一种）
docker exec -i -e Q="SELECT COUNT(*) c FROM <表名>" <backend容器> python < dbq.py
```
- **★多栈替代（同原则、换工具）**：非 Python/SQLAlchemy 栈换对应 DB CLI/ORM——**psql**（PostgreSQL）/ **mongosh**（MongoDB）/ **prisma**（Node/Prisma：`prisma db execute` 或 client 脚本）/ **rails dbconsole / ActiveRecord**（Rails）/ **mysql** CLI 等。**远程/托管库（RDS / Supabase / PlanetScale / Atlas 等）**：不在容器内，**用独立 DB 客户端**（或托管方的只读控制台/数据 API）连同一实例查询；连接串/凭据按项目实际，仍遵守"同连接配置避免漂移"。
- **★API 微服务变体（远程库 / 异步 / gRPC）**：① **远程/托管库**：改用**服务自身的只读 API** 或**独立 DB 客户端**回读，别假设能 `docker exec`。② **异步副作用**（消息队列/事件/最终一致）：用 **"轮询最终状态 + correlation-id 关联请求与副作用 + 死信队列（DLQ）检查"** ——发请求拿 correlation-id，按它轮询目标状态直到达成或超时；副作用没出现就查 DLQ 看是否进了死信。③ **gRPC / 流式通道**：用 **`grpcurl`**（`grpcurl -d '{...}' host:port pkg.Svc/Method`）真发一元/流式调用看真实响应与状态码；流式额外真测**取消（cancel）/ 超时（deadline）/ 断线重连**这三种边界，断言服务端正确收尾、不泄漏、可恢复。
- 写后必回读：POST→新行+默认值；PATCH→改动+未传字段不变；DELETE→404+级联消失；上传→行+磁盘字节；状态流转→status+时间戳+副作用实体。
- **接口 200 ≠ 真入库**：曾发现 GET 触发写副作用、PATCH 绕状态机落脏数据——都是 DB 回读才抓到。
- 单条 SQL 用单次调用；多条 `;;` + 单引号混用在某些 shell 易碎，复杂多条分多次调用（shell 转义/编码踩坑见 `pitfalls.md`）。

## 通道 4 · 代码佐证（grep/read file:line）— 仅补充，不替代真跑
- 用途：定位实现解释行为、运行期确难触发项(并发 schema/底层时序)的判定依据。
- **硬约束**：API 类不得以代码佐证替代真跑(§1)；任何代码佐证条目**必须显式标"源码佐证(未真跑)"**，不得记成"已执行✅"。
- 源码词 ≠ UI 词(可能运行期文本替换)，按源码原词/路由/字段名 grep。

## 实战脚本范式（放被测项目的 `_probe/` 或工具目录，复用）
- `dbq.py`：DB 写后回读（上）。
- 造数脚本(`*_create.mjs`)：QA_ 前缀串建 父→子业务实体链(如 A→B→C…)，输出新建 ID 供后续步骤。
- 清理脚本：按子→父顺序 DELETE QA_ 链 + 验基线计数。
- 移图脚本(`mv_*.mjs`)：截图默认落 cwd 根目录，按前缀分发到 `证据/面X/`。
- 嵌图脚本(`embed_shots.mjs`)：把已截图按模块内嵌进执行记录(每图带说明)。
- 断链校验(`check_imgs.mjs`)：扫所有 `![](路径)` 验文件存在，0 断链才算图文并茂落地。
- 覆盖/真实性审计(`audit_*.mjs`)：见 `audit-and-honesty.md`。
