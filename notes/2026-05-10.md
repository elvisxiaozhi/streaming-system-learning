# 2026-05-10

## 今天做了什么

- 创建学习目录：`notes/`、`docs/`、`labs/01-ffmpeg-cli` 到 `labs/08-sfu-room`。
- 初始化基础文档：`docs/glossary.md`、`docs/interview.md`、`docs/resume-draft.md`。
- 检查并补齐 Day 1 工具链。

## 环境检查结果

| 工具 | 状态 | 版本 |
|---|---|---|
| ffmpeg | 已安装 | 8.1.1 |
| ffprobe | 已安装 | 8.1.1 |
| cmake | 已安装 | 4.3.1 |
| ninja | 已安装 | 1.13.2 |
| git | 已安装 | 2.50.1 Apple Git-155 |
| Homebrew | 已安装 | 5.1.7 |

## 遇到的问题

- 初始环境里缺少 `ffmpeg`、`ffprobe` 和 `ninja`。
- 已通过 Homebrew 安装 `ffmpeg` 和 `ninja`。安装 `ffmpeg` 时同时带上了后续会用到的依赖，例如 SDL2、x264、x265、Opus。

## 我现在能解释什么

- `ffmpeg` 用来处理媒体文件，例如转码、抽流、截图、合并。
- `ffprobe` 用来分析媒体文件结构，后续会用它查看视频流、音频流、编码格式、码率、帧率等。
- `cmake` 负责生成 C++ 构建配置，`ninja` 负责执行编译。
- `ffmpeg` 更像是处理工具，`ffprobe` 更像是查看工具；一个负责改媒体文件，一个负责看媒体文件里有什么。

## 今日验收

- 目录结构已建立。
- 基础文档已建立。
- 后续 Day 2 可以直接进入第一个 mp4 样例分析。

## 你需要亲手跑一遍的命令

```bash
ffmpeg -version
ffprobe -version
cmake --version
ninja --version
git --version
```

