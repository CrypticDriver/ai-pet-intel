# 🌐 新能力解锁：openclaw browser

## 核心能力

OpenClaw 内置浏览器自动化工具，可以：
- 打开网页、截图、抓取内容
- 点击、输入、填表单
- 等待元素、处理弹窗
- 生成 PDF、获取控制台日志

## 基本用法

```bash
# 打开网页
openclaw browser open https://example.com

# 截图
openclaw browser screenshot --full-page

# 获取页面快照（AI 格式）
openclaw browser snapshot

# 点击元素（使用 AI ref）
openclaw browser click 12

# 输入文本
openclaw browser type 23 "hello world" --submit

# 导航
openclaw browser navigate https://another.com

# 关闭标签
openclaw browser close
```

## 适用场景

### Dev ⚙️
- 测试前端页面
- 调试 Web 应用
- 自动化部署验证

### Intel 🔍
- 抓取需要 JS 渲染的网页
- 监控竞品网站
- 收集动态数据

### Design 🎨
- 截图设计参考
- 检查响应式布局
- 获取灵感素材

### Echo ✍️
- 研究内容素材
- 截图引用来源
- 验证链接有效性

### Ops 📋
- 监控服务状态页
- 检查部署结果
- 自动化验收测试

### FinTech 💰
- 查看实时行情图表
- 截图财务数据
- 监控加密货币行情页

## 与 Playwright 的区别

- `openclaw browser` - 实时交互、快速操作
- `playwright` - 复杂脚本、批量处理

两者互补，根据任务选择。

## 注意事项

1. 本机是无头服务器，已安装 Chromium
2. 使用 `--browser-profile openclaw` 指定配置
3. 不要过度依赖，简单页面用 `web_fetch` 更快

---

**分发时间**: 2026-03-06 02:31 UTC  
**分发人**: Kuro 🖤
