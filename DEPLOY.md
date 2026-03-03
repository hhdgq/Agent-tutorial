# 部署到 GitHub Pages 指南

本指南说明如何将 AI Agent 教程文档部署到 GitHub Pages，实现在线浏览。

---

## 快速部署（推荐方式）

### 步骤 1：准备 GitHub 仓库

1. 登录 GitHub (https://github.com)
2. 创建新仓库，例如 `ai-agent-tutorial`
   - 或者使用现有仓库

### 步骤 2：推送文件到 GitHub

将此 `docs-gh-pages` 文件夹的内容推送到仓库：

```bash
# 进入 docs-gh-pages 目录
cd docs-gh-pages

# 初始化 git（如果尚未初始化）
git init

# 添加所有文件
git add .

# 提交
git commit -m "Initial commit: AI Agent tutorial docs"

# 添加远程仓库（替换为你的仓库地址）
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git

# 推送到 main 分支
git push -u origin main
```

### 步骤 3：启用 GitHub Pages

1. 进入 GitHub 仓库页面
2. 点击 **Settings**（设置）
3. 在左侧边栏点击 **Pages**
4. 在 "Source" 下选择：
   - **Deploy from a branch**
   - Branch: **main**
   - Folder: **/(root)**
5. 点击 **Save**

### 步骤 4：访问你的网站

等待 1-2 分钟构建完成后，访问：
```
https://YOUR_USERNAME.github.io/YOUR_REPO_NAME/
```

---

## 备选方案：使用 /docs 文件夹

如果你希望将文档作为项目的一部分（而非独立仓库）：

### 步骤 1：复制 docs-gh-pages 到项目根目录

```bash
# 在项目根目录执行
cp -r docs-gh-pages/* ./docs/
```

### 步骤 2：推送整个项目到 GitHub

```bash
git add .
git commit -m "Add docs for GitHub Pages"
git push origin main
```

### 步骤 3：配置 GitHub Pages 源

1. Settings → Pages
2. 选择 **main branch /docs folder**
3. Save

---

## 验证部署

部署完成后，检查以下内容：

- [ ] 网站首页可以正常访问
- [ ] 所有章节链接正常工作
- [ ] Mermaid 图表正确渲染
- [ ] 代码高亮正常显示
- [ ] 搜索功能可用
- [ ] 分页导航正常

---

## 自定义域名（可选）

如果你希望使用自定义域名：

1. 在仓库 Settings → Pages → Custom domain
2. 输入你的域名，例如 `docs.example.com`
3. 在你的 DNS 提供商处添加 CNAME 记录：
   ```
   CNAME  YOUR_USERNAME.github.io
   ```

---

## 更新文档

当需要更新文档内容时：

```bash
# 修改 docs-gh-pages 中的文件
# ... 编辑文件 ...

# 提交并推送更改
git add .
git commit -m "Update: 描述你的更改"
git push origin main
```

GitHub Pages 会在 1-2 分钟内自动重新构建并部署。

---

## 故障排查

### 问题：页面显示 404

**解决方案**：
- 等待 2-3 分钟（GitHub 需要时间构建）
- 确认已正确启用 GitHub Pages
- 检查是否推送了 `index.html` 文件

### 问题：样式/脚本加载失败

**解决方案**：
- 检查浏览器控制台是否有 CORS 错误
- 确认所有 CDN 链接使用 HTTPS
- 清除浏览器缓存

### 问题：Mermaid 图表不显示

**解决方案**：
- 检查 mermaid 代码块是否使用正确的语言标识（```mermaid）
- 查看浏览器控制台是否有 JavaScript 错误

### 问题：搜索功能不工作

**解决方案**：
- 确认已加载 search.min.js 插件
- 检查文档文件是否都在正确路径

---

## 技术细节

### 为什么使用 hash 路由模式？

```javascript
routeMode: 'hash'
```

hash 模式（URL 中包含 `#`）是 GitHub Pages 的最佳选择，因为：

1. **无需服务器配置**：纯静态文件即可工作
2. **避免 404 问题**：刷新页面不会丢失路由
3. **兼容性更好**：所有托管平台都支持

### .nojekyll 文件的作用

`.nojekyll` 文件告诉 GitHub Pages：

- 不要使用 Jekyll 处理此网站
- 直接服务现有的 HTML 文件
- 跳过 Markdown 转换步骤

这确保了 docsify 构建的静态文件可以直接使用。

---

## 资源链接

- [GitHub Pages 官方文档](https://docs.github.com/en/pages)
- [docsify 官方文档](https://docsify.js.org/)
- [docsify 部署指南](https://docsify.js.org/#/deploy)

---

**祝你部署成功！** 如有问题，请查看 GitHub Pages 文档或提交 Issue。
