# 🎬 DreamFactory — AI 短剧生成平台 / AI Short Drama Generation Platform

<div align="center">

**基于 TypeScript 全栈的 AI 短剧自动化生产平台**<br>
**A full-stack TypeScript platform for automated AI short-drama production**

[![Node Version](https://img.shields.io/badge/Node.js-20+-339933?style=flat&logo=node.js)](https://nodejs.org)
[![Vue Version](https://img.shields.io/badge/Vue-3.x-4FC08D?style=flat&logo=vue.js)](https://vuejs.org)
[![License](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

**中英双语 / Bilingual** | [简体中文](README.zh-CN.md) | [English](README.en.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

登录即可体验小说转短剧、分镜制作、视频生成和剪辑。

</div>

---

## 📖 项目简介 / Overview

DreamFactory 是一个基于 AI 的短剧自动化生产平台，实现从剧本生成、角色设计、分镜制作到视频合成的全流程自动化。<br>
DreamFactory is an AI-powered short-drama production platform that automates the entire pipeline: script generation, character design, storyboard breakdown, and video compositing.

### 🎯 核心价值 / Core Value

- **🤖 AI 驱动**：使用大语言模型解析剧本，提取角色、场景和分镜信息<br>
  **🤖 AI-Driven**: LLMs parse scripts and extract characters, scenes, and storyboard information

- **🎨 智能创作**：AI 绘图生成角色形象和场景背景<br>
  **🎨 Intelligent Creation**: AI image generation for character designs and scene backgrounds

- **📹 视频生成**：基于文生视频和图生视频模型自动生成分镜视频<br>
  **📹 Video Generation**: Text-to-video and image-to-video models automatically produce storyboard clips

- **🔄 工作流**：完整的短剧制作工作流，从创意到成片一站式完成<br>
  **🔄 End-to-End Workflow**: A complete pipeline from idea to finished episode

### 🛠️ 技术架构 / Architecture

```
frontend/   — Nuxt 3 + Vue 3 + TypeScript
backend/    — Hono + Drizzle ORM + Mastra AI Agents + better-sqlite3
backend/workspace/skills/ — Agent 技能定义，支持界面在线编辑
data/       — 生成资源文件与 SQLite 数据库
```

---

## ✨ 功能特性 / Features

### 🎭 角色管理 / Character Management

- ✅ AI 生成角色形象
- ✅ 批量角色生成
- ✅ 角色图片上传和管理

### 🎬 视频任务 / Video Tasks

- ✅ AI 自动生成视频任务
- ✅ 场景描述和视频提示词生成
- ✅ 按任务批量生成视频

### 🎥 视频生成 / Video Generation

- ✅ 文生视频自动生成
- ✅ FFmpeg 单镜头合成与字幕处理
- ✅ 整集拼接导出

### 📦 资源管理 / Asset Management

- ✅ 素材库统一管理
- ✅ 本地存储支持
- ✅ 任务进度追踪

### 🤖 AI Agents

内置 4 个 Mastra Agent，支持数据库配置和 Skill 扩展：

| Agent |职责 / Role|
|---|---|
| `script_rewriter` |小说 → 格式化剧本改写|
| `extractor` |角色 / 场景 / 道具智能提取与去重|
| `storyboard_breaker` |剧本 → 分镜序列拆解|
| `prompt_generator` |角色/场景/道具图片提示词 + 分镜视频提示词生成|

### 🌐 多语言界面 / Multi-Language UI

界面内置 **中文 / English / 日本語 / 한국어** 四种语言，并可全局设置 AI 生成内容的语言。

### 🔌 多厂商适配 / Multi-Provider Support

|类型 / Type|支持厂商 / Providers|
|---|---|
|**文本** / **Text**|OpenAI(兼容接口)、Gemini|
|**图片** / **Image**|OpenAI、Gemini、火山引擎、Grsai|
|**视频** / **Video**|火山引擎 Seedance 2.0、MiniMax H3、阿里云百炼 Wan 3.0、Grsai H3|

---

## 🚀 快速开始 / Quick Start

### 📋 环境要求 / Requirements

|软件 / Software|版本要求 / Version|说明 / Notes|
|---|---|---|
| **Node.js** | 20+ |前后端运行环境|
| **npm** | 9+ |包管理工具|

> **数据库零安装**：内置 SQLite（单文件，随项目数据目录存放），无需安装任何数据库服务。

> **FFmpeg 无需安装**：项目通过 `ffmpeg-static` / `ffprobe-static` npm 包内置二进制，开箱即用。

### 📥 安装依赖 / Installation

```bash
git clone <your-repo-url>
cd dreamfactory

# 安装后端依赖
cd backend && npm install

# 安装前端依赖
cd ../frontend && npm install
```

### 🎯 启动项目 / Running

#### 方式一：开发模式（推荐）
```bash
# 终端1：启动后端
cd backend
npm run dev

# 终端2：启动前端
cd frontend
npm run dev
```

#### 方式二：单服务模式
```bash
# 1. 构建前端
cd frontend && npm run generate
cp -r .output/public dist

# 2. 启动后端
cd ../backend && npm start
```

---

## 📄 许可证 / License

本项目采用 **[CC BY-NC-SA 4.0](LICENSE)** 许可证。

- ✅ 个人使用、学习研究、非商业项目均可自由使用
- ✅ 允许修改与再分发，但须署名并以相同许可证共享
- ❌ **禁止商用**——未经作者书面许可，不得将本项目整体或部分用于任何商业目的。
