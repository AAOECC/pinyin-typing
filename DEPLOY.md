# 部署指南 - GitHub Pages

## 前置条件

- 已安装 Node.js 和 npm
- 已在 GitHub 创建仓库（本项目：https://github.com/AAOECC/pinyin-typing.git）

## 1. 安装依赖

```bash
npm install
```

## 2. 本地开发

```bash
npm run dev
```

访问 http://localhost:5173

## 3. 构建生产版本

```bash
npm run build
```

构建产物输出到 `dist/` 目录。

## 4. 本地预览生产版本

构建完成后，预览打包效果：

```bash
npm run preview
```

访问 http://localhost:4173

### 或使用其他 HTTP 服务器

```bash
# Python
cd dist && python -m http.server 8080

# Node.js
npx serve dist
```

访问 http://localhost:8080 或 http://localhost:3000

## 5. 部署到 GitHub Pages

### 方式一：使用 gh-pages 工具（推荐）

```bash
npx gh-pages -d dist
```

### 方式二：手动部署

```bash
# 创建 gh-pages 分支
git checkout --orphan gh-pages

# 清除工作区（保留 dist 内容）
git rm -rf .
cp -r dist/* .

# 添加 .nojekyll 文件（防止 GitHub 忽略 _ 开头的文件）
touch .nojekyll

# 提交并推送
git add .
git commit -m "deploy to github pages"
git push -u origin gh-pages

# 切回主分支
git checkout master
git branch -D gh-pages
```

## 6. 启用 GitHub Pages

1. 打开仓库 Settings → Pages
2. Source 选择 `gh-pages` 分支
3. 目录选择 `/ (root)`
4. 点击 Save

## 7. 访问地址

https://AAOECC.github.io/pinyin-typing/

## 附：添加新模板

在 `public/templates/` 目录下：

1. 新建 `.txt` 文件，内容为要练习的文字
2. 在 `templates.json` 中添加索引：

```json
{ "name": "模板名", "file": "文件名.txt", "desc": "作者或描述" }
```

3. 重新构建并部署：

```bash
npm run build
npx gh-pages -d dist
```
