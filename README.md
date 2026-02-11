# 为艺术而技术博客

## 构建和部署

### Node.js 版本兼容性

- **推荐版本**: Node.js 20.x (LTS)
- **已测试版本**: Node.js v20.20.0 + Gatsby 5.16.1 + React 18.2.0

### 构建步骤

1. **安装依赖** (使用Node.js 20):
   ```bash
   npm install
   ```

2. **本地开发**:
   ```bash
   npm run develop
   ```

3. **构建生产版本**:
   ```bash
   npm run build
   ```

4. **本地预览构建结果**:
   ```bash
   npm run serve
   ```

5. **部署到Netlify**:
   - Netlify会自动检测到Gatsby项目
   - 构建命令: `npm run build`
   - 发布目录: `public`

### 升级历史

- ✅ **2026-02-11**: 成功升级到 Node.js 20 + Gatsby 5 + React 18
  - 移除了 `gatsby-theme-blog` 主题依赖
  - 添加了核心插件: `gatsby-plugin-image`, `gatsby-source-filesystem`
  - 更新了配置文件以支持新的插件架构
  - 修复了 GraphQL 查询语法

- ✅ **2026-02-11**: 成功从Node.js 16 + Gatsby 3 升级到 Node.js 20 + Gatsby 4
  - 移除了不兼容的 `gatsby-plugin-offline`
  - 降级了 `gatsby-theme-blog` 相关依赖
  - 保持了React 17以兼容现有主题

### 故障排除

如果遇到兼容性问题，可以尝试：
1. 使用 `npm install --legacy-peer-deps` 强制安装
2. 检查Node.js版本: `node --version`
3. 清理缓存: `rm -rf node_modules package-lock.json && npm install`