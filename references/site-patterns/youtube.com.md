---
domain: youtube.com
aliases: [YouTube, www.youtube.com, m.youtube.com]
updated: 2026-03-26
---

## 平台特征

- **登录需求**：无需登录即可观看视频和获取基本信息
- **内容加载**：SPA 架构，大量动态加载，需 CDP 浏览器模式
- **反爬机制**：
  - 静态抓取（curl/WebFetch/Jina）通常返回 429 错误
  - 页面内容通过 JavaScript 动态渲染
  - 描述文本需要点击"展开"才能获取完整内容

## 有效模式

### 视频信息获取

**基本选择器**：
```javascript
// 视频元素
document.querySelector("#movie_player video")  // 主播放器
document.querySelector("video")  // 备用选择器

// 视频控制
video.currentTime = N  // 跳转到 N 秒
video.duration  // 总时长
video.paused  // 是否暂停

// 标题
document.querySelector("#title")?.innerText
document.querySelector("h1.ytd-video-primary-info-renderer")?.innerText

// 描述（未展开）
document.querySelector("#description-inline-expander #text")?.innerText

// 描述展开按钮
document.querySelector("ytd-expander-button-renderer button")
```

### 时间戳导航

**直接导航到视频特定时间点**：
```
https://www.youtube.com/watch?v=VIDEO_ID&t=Ns
```
- `t` 参数单位为秒
- 例如：`&t=120s` 跳转到 2 分钟

**使用 navigate 命令**：
```bash
curl -s "http://localhost:3456/navigate?target=ID&url=https://www.youtube.com/watch?v=TqC1qOfiVcQ&t=120s"
```

### 章节时间戳获取

**章节标记选择器**：
```javascript
// 章节列表容器
document.querySelectorAll("ytd-macro-markers-list-item-renderer")

// 提取时间文本
Array.from(document.querySelectorAll("ytd-macro-markers-list-item-renderer"))
  .map(el => el.querySelector("#text")?.innerText)
  .filter(t => t)
```

### 截图采帧

**视频截图流程**：
1. 用 `/navigate` 跳转到目标时间点
2. 等待 3 秒让视频加载
3. 用 `/screenshot` 截取当前帧

```bash
# 流程示例
curl -s "http://localhost:3456/navigate?target=ID&url=...&t=120s"
sleep 3
curl -s "http://localhost:3456/screenshot?target=ID&file=output.png"
```

### 批量处理

**批量导航 + 截图**：
```bash
# 多点截图（适合长视频）
for t in 0 60 120 300 600 900 1800 3600; do
  curl -s "http://localhost:3456/navigate?target=ID&url=...&t=${t}s"
  sleep 3
  curl -s "http://localhost:3456/screenshot?target=ID&file=shot-${t}.png"
done
```

**PPT/幻灯片检测**：
- 视频中的 PPT 通常是全屏或大比例
- 截图后用视觉识别判断是否包含文字/图表
- 避免截取到"正在加载"或"缓冲"状态

**有效截图时机**：
- 导航后等待至少 3 秒让页面稳定
- 避开片头广告和赞助商标识（通常在前 30 秒）
- 长视频建议每 5-10 分钟截取一个关键帧

## 已知陷阱

1. **网络访问问题**：YouTube 需要代理访问。静态抓取返回 429 错误时，启用本地代理（详见 `references/cdp-api.md` 中的"代理设置"章节）

2. **Jina 抓取失败**：`r.jina.ai/https://www.youtube.com/...` 返回 429 Too Many Requests
   - **解决方案**：使用 CDP 浏览器模式，操作真实 Chrome

3. **描述需要展开**：视频描述默认折叠，需要点击展开按钮才能获取完整内容（包括时间戳）

4. **Eval 容易报错**：YouTube 的 JS 环境复杂，复杂 eval 脚本容易返回 "Uncaught"，建议用简单脚本分步执行

5. **页面重渲染**：导航后页面可能完全重新加载，之前的 DOM 引用失效

6. **滚动不加载章节**：YouTube 视频页的章节信息不是懒加载，滚动不会显示更多内容

7. **视频控制受限**：直接调用 `video.click()` 或 `video.play()` 可能被拦截，用 `navigate&t=` 更可靠

8. **字幕 API 无法直接下载**：
   - 从页面提取的字幕 URL（`https://www.youtube.com/api/timedtext?...`）无法通过 curl 直接下载
   - 测试：curl 返回超时或空响应
   - 原因：YouTube 字幕 API 有严格的反爬保护

## 关闭标签

任务完成后清理：
```bash
curl -s "http://localhost:3456/close?target=ID"
```

---

## 更新日志

### 2026-03-26

**新增内容**：
1. YouTube 视频处理完整模式
   - 时间戳导航方法
   - 章节信息提取
   - 批量截图采帧流程

2. **已知陷阱**：
   - Jina 抓取失败（429）
   - eval 复杂脚本报错
   - 字幕 API 无法直接下载

**验证来源**：
- 通过实际操作 YouTube 视频 `https://www.youtube.com/watch?v=TqC1qOfiVcQ` 验证
- 成功截取 14 张关键帧截图（PPT 页面）
- 成功提取章节时间戳信息
