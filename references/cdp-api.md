# CDP Proxy API 参考

## 基础信息

- 地址：`http://localhost:3456`
- 启动：`node ~/.claude/skills/web-access/scripts/cdp-proxy.mjs &`
- 启动后持续运行，不建议主动停止（重启需 Chrome 重新授权）
- 强制停止：`pkill -f cdp-proxy.mjs`

## 代理设置

部分网站（如 YouTube、Google、GitHub 等国际站点）需要通过本地 HTTP 代理访问。

**何时考虑启用代理**：
- 静态抓取（curl/WebFetch）返回 429 Too Many Requests
- 连接超时或无法访问特定站点
- 已知需要代理的站点类型

**设置方式**：
```bash
# 方式一：使用系统命令（如果环境变量已配置）
proxy-on

# 方式二：手动设置环境变量
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
```

**CDP 浏览器模式说明**：
CDP 浏览器模式使用用户日常 Chrome，天然携带代理设置。如果代理不工作，检查 Chrome 是否正确配置代理（通常通过系统代理或浏览器扩展设置）。

**常见代理端口**：
- `8118`：Privoxy（常见配置）
- `7890`：Clash
- `1080`：SOCKS5 代理

## API 端点

### GET /health
健康检查，返回连接状态。
```bash
curl -s http://localhost:3456/health
```

### GET /targets
列出所有已打开的页面 tab。返回数组，每项含 `targetId`、`title`、`url`。
```bash
curl -s http://localhost:3456/targets
```

### GET /new?url=URL
创建新后台 tab，自动等待页面加载完成。返回 `{ targetId }`.
```bash
curl -s "http://localhost:3456/new?url=https://example.com"
```

### GET /close?target=ID
关闭指定 tab。
```bash
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

### GET /navigate?target=ID&url=URL
在已有 tab 中导航到新 URL，自动等待加载。
```bash
curl -s "http://localhost:3456/navigate?target=ID&url=https://example.com"
```

### GET /back?target=ID
后退一页。
```bash
curl -s "http://localhost:3456/back?target=ID"
```

### GET /info?target=ID
获取页面基础信息（title、url、readyState）。
```bash
curl -s "http://localhost:3456/info?target=ID"
```

### POST /eval?target=ID
执行 JavaScript 表达式，POST body 为 JS 代码。
```bash
curl -s -X POST "http://localhost:3456/eval?target=ID" -d 'document.title'
```

### POST /click?target=ID
JS 层面点击（`el.click()`），POST body 为 CSS 选择器。自动 scrollIntoView 后点击。简单快速，覆盖大多数场景。
```bash
curl -s -X POST "http://localhost:3456/click?target=ID" -d 'button.submit'
```

### POST /clickAt?target=ID
CDP 浏览器级真实鼠标点击（`Input.dispatchMouseEvent`），POST body 为 CSS 选择器。先获取元素坐标，再模拟鼠标按下/释放。算真实用户手势，能触发文件对话框、绕过部分反自动化检测。
```bash
curl -s -X POST "http://localhost:3456/clickAt?target=ID" -d 'button.upload'
```

### POST /setFiles?target=ID
给 file input 设置本地文件路径（`DOM.setFileInputFiles`），完全绕过文件对话框。POST body 为 JSON。
```bash
curl -s -X POST "http://localhost:3456/setFiles?target=ID" -d '{"selector":"input[type=file]","files":["/path/to/file1.png","/path/to/file2.png"]}'
```

### GET /scroll?target=ID&y=3000&direction=down
滚动页面。`direction` 可选 `down`（默认）、`up`、`top`、`bottom`。滚动后自动等待 800ms 供懒加载触发。
```bash
curl -s "http://localhost:3456/scroll?target=ID&y=3000"
curl -s "http://localhost:3456/scroll?target=ID&direction=bottom"
```

### GET /screenshot?target=ID&file=/tmp/shot.png
截图。指定 `file` 参数保存到本地文件；不指定则返回图片二进制。可选 `format=jpeg`。
```bash
curl -s "http://localhost:3456/screenshot?target=ID&file=/tmp/shot.png"
```

## /eval 使用提示

- POST body 为任意 JS 表达式，返回 `{ value }` 或 `{ error }`
- 支持 `awaitPromise`：可以写 async 表达式
- 返回值必须是可序列化的（字符串、数字、对象），DOM 节点不能直接返回，需要提取属性
- 提取大量数据时用 `JSON.stringify()` 包裹，确保返回字符串
- 根据页面实际 DOM 结构编写选择器，不要套用固定模板

## 错误处理

| 错误 | 原因 | 解决 |
|------|------|------|
| `Chrome 未开启远程调试端口` | Chrome 未开启远程调试 | 提示用户打开 `chrome://inspect/#remote-debugging` 并勾选 Allow |
| `attach 失败` | targetId 无效或 tab 已关闭 | 用 `/targets` 获取最新列表 |
| `CDP 命令超时` | 页面长时间未响应 | 重试或检查 tab 状态 |
| `端口已被占用` | 另一个 proxy 已在运行 | 已有实例可直接复用 |
| `Uncaught` (eval 返回) | JS 执行错误或页面环境复杂 | 简化脚本，用更简单的表达式 |

## 视频处理模式

### 长视频截图采帧

**场景**：需要从视频节目中提取 PPT/幻灯片页面

**流程**：
1. 用 `/new` 打开视频页面
2. 用 `/navigate?url=...&t=Ns` 跳转到多个时间点（N 为秒数）
3. 每次导航后等待 3 秒让视频加载
4. 用 `/screenshot` 截取当前帧
5. 完成后用 `/close` 关闭 tab

**示例**（批量导航 + 截图）：
```bash
# 多点截图（适合长视频，时间点单位：秒）
for t in 0 60 120 300 600 900 1800 3600; do
  curl -s "http://localhost:3456/navigate?target=ID&url=...&t=${t}s"
  sleep 3
  curl -s "http://localhost:3456/screenshot?target=ID&file=shot-${t}.png"
done
```

**注意事项**：
- 导航后至少等待 3 秒，让视频帧渲染完成
- 避开片头广告和赞助商标识（通常在前 30 秒）
- 长视频建议每 5-10 分钟截取一个关键帧
- 用 `navigate&t=` 比直接调用 `video.currentTime` 更可靠
- 截图后用视觉识别判断是否包含 PPT 内容（全屏/文字/图表）

### YouTube 视频特殊处理

**时间戳导航**：
```
https://www.youtube.com/watch?v=VIDEO_ID&t=Ns
```

**章节信息获取**：
```javascript
document.querySelectorAll("ytd-macro-markers-list-item-renderer")
```

**已知限制**：
- YouTube 静态抓取（curl/WebFetch/Jina）通常返回 429
- 必须用 CDP 浏览器模式
-  eval 脚本尽量简单，复杂脚本容易返回 `Uncaught`
