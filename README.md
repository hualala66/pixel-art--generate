# Pixel Art Generator

一个基于 Vue 3 和 Vite 的像素艺术生成器。上传图片后，应用会在浏览器本地把图片转换成 600 x 600 的像素风预览，并支持导出 PNG。

线上体验：

[https://pixel-art-generator-wheat.vercel.app](https://pixel-art-generator-wheat.vercel.app)

## 功能特点

- 支持拖拽或点击上传图片
- 支持 PNG、JPG、WebP 等常见图片格式
- 实时生成像素风画布预览
- 可调整横向像素块数量
- 可调整明暗对比阈值
- 支持方块像素和圆点像素两种样式
- 支持本地 AI 背景移除
- 支持导出 PNG 图片
- 图片处理主要在浏览器本地完成

## 技术栈

- Vue 3
- Vite
- @imgly/background-removal
- lucide-vue-next
- HTML Canvas
- Vercel

## 本地运行

安装依赖：

```bash
npm install
```

启动开发服务器：

```bash
npm run dev
```

构建生产版本：

```bash
npm run build
```

本地预览生产构建：

```bash
npm run preview
```

## 使用方式

1. 打开页面。
2. 拖入图片，或点击上传区域选择图片。
3. 调整 `Tiles X` 控制像素密度。
4. 调整 `Contrast Threshold` 控制明暗效果。
5. 在 `Pixel Shape` 中选择 `Square` 或 `Dot`。
6. 可选开启 `Local AI Cutout` 移除背景。
7. 点击 `Export PNG` 下载结果。

## 部署说明

项目已经包含 `vercel.json`，可直接部署到 Vercel。

Vercel 配置要点：

- 构建命令：`npm run build`
- 输出目录：`dist`
- 框架：`vite`
- 已配置 `Cross-Origin-Opener-Policy` 和 `Cross-Origin-Embedder-Policy` 响应头，用于改善 wasm 推理体验

部署命令：

```bash
npx vercel --prod
```

## 注意事项

- 首次使用 `Local AI Cutout` 时，背景移除模型和 wasm 资源需要下载，等待时间会比后续使用更长。
- 背景移除依赖浏览器端计算，性能会受到设备、浏览器和图片尺寸影响。
- 当前项目没有后端服务，部署产物是静态站点。

## 项目结构

```text
.
├── public/          # 静态资源
├── src/             # 应用源码
│   ├── App.vue      # 主界面和核心图片处理逻辑
│   ├── main.js      # Vue 入口
│   └── style.css    # 全局样式
├── index.html       # HTML 入口
├── package.json     # 脚本和依赖
├── vercel.json      # Vercel 部署配置
└── vite.config.js   # Vite 配置
```
