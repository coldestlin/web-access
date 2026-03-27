---
domain: weibo.com
aliases: [微博, s.weibo.com, passport.weibo.com]
updated: 2026-03-25
---

## 平台特征

- **登录需求**：必须登录才能访问搜索结果，未登录会跳转到 `passport.weibo.com` 登录页
- **内容加载**：服务端渲染，DOM 直接包含完整内容，无需等待异步加载
- **反爬机制**：有反爬限制，静态抓取（curl/WebFetch）通常失败，需 CDP 浏览器模式

## 有效模式

### 微博发布 (weibo.com 首页)

**发布框选择器**：
```javascript
// 文本输入框
'textarea'  // placeholder: "有什么新鲜事想分享给大家？"

// 图片/视频上传
'input[type=file]'  // accept: image/*, video/*

// 发送按钮
'button._btn_2z30i_68'  // 精确选择器，innerText === "发送"
```

**发布流程**：
1. `/setFiles` 设置图片：`{"selector":"input[type=file]","files":["/path/to/image.jpg"]}`
2. `/eval` 设置文本：`textarea.value = "内容"; textarea.dispatchEvent(new Event("input", {bubbles: true}))`
3. `/click` 点击发送：`button._btn_2z30i_68`
4. 发布成功标志：textarea 值被清空

**注意事项**：
- 发送按钮不要用 `button.woo-button-primary`，会误匹配到"管理"等其他按钮
- 设置 textarea.value 后需要触发 input 事件，否则可能不被识别

### 微博搜索 (s.weibo.com)

**搜索 URL 格式**：
```
https://s.weibo.com/weibo/{关键词}
```

**核心选择器**：
```javascript
// 搜索结果卡片
'.card-wrap[mid]'  // 带 mid 属性的卡片，排除广告等干扰

// 作者名
'.name[nick-name]'  // 用 getAttribute('nick-name') 获取，比 innerText 更准确

// 正文内容
'.node-type-text'              // 主选择器
'p[node-type=feed_list_content]'  // 备用选择器

// 发布时间
'.from a'  // 时间链接

// 互动数据（转评赞）
'li em'  // 三个 em 元素依次为：转发、评论、点赞
```

**页面结构标识**：
- `#pl_feedlist_index` - 搜索结果列表容器
- `.card-wrap` - 结果卡片（约20个）

**数据提取示例**：
```javascript
const cards = document.querySelectorAll('.card-wrap[mid]');
cards.forEach(card => {
  const author = card.querySelector('.name[nick-name]')?.getAttribute('nick-name');
  const text = card.querySelector('.node-type-text')?.innerText.trim();
  const time = card.querySelector('.from a')?.innerText.trim();
});
```

## 已知陷阱

1. **选择器混淆**：`.card-wrap` 不只包含搜索结果，还包含广告、推荐等干扰项。必须用 `.card-wrap[mid]` 或 `.card-wrap[action-type=feed_list_item]` 过滤
2. **登录跳转**：未登录访问搜索页会 302 跳转到登录页，需检查 `document.title` 是否包含"登录"
3. **文本截断**：部分微博正文有"展开"折叠，innerText 获取的是可见部分，可能需要点击展开或直接从 DOM 提取完整内容
4. **动态元素**：页面有"智搜回答"等 AI 生成内容块，结构与普通微博不同，需注意区分

## 登录处理

当检测到跳转登录页时：
1. 通知用户在当前 tab 登录
2. 登录完成后用 `/navigate` 刷新到目标 URL
3. 用户日常 Chrome 天然携带登录态，大多数情况已登录