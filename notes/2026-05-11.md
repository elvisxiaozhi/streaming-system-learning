# 2026-05-11

## 今天做了什么

- 进入 Day 2：准备一个 30 秒 mp4 样例。
- 使用 FFmpeg 生成本地测试视频：`labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4`。
- 使用 `ffprobe` 查看样例文件的容器、视频流、音频流信息。
- 更新 `labs/01-ffmpeg-cli/README.md`，记录生成命令和分析结果。
- 更新 `docs/glossary.md`，新增容器、编码格式、媒体流、码率、帧率等术语。
- 更新 `.gitignore`，忽略本地生成的 mp4 样例。

## 样例文件

文件路径：

```text
labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

生成命令：

```bash
ffmpeg -y \
  -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -t 30 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

## ffprobe 关键结果

| 项 | 结果 |
|---|---|
| 容器格式 | MP4/MOV 家族 |
| 时长 | 30 秒 |
| 文件大小 | 934288 bytes，约 912 KB |
| 总码率 | 约 249 kb/s |
| 视频流 | H.264 / AVC，1280x720，30 fps，yuv420p，900 帧 |
| 音频流 | AAC LC，48000 Hz，mono，约 128 kb/s |

## 遇到的问题

- 仓库里没有现成 mp4 文件，所以用 FFmpeg 的 `lavfi` 生成了可复现样例。
- 生成的视频属于实验资产，不适合直接提交到 Git，因此加入 `.gitignore`。

## 我现在能解释什么

- MP4 是容器，不等于视频编码格式。
- H.264 是视频编码格式，AAC 是音频编码格式。
- 一个 mp4 文件可以包含多条媒体流，今天的样例包含 1 条视频流和 1 条音频流。
- `ffprobe` 的 `FORMAT` 更关注整个文件，`STREAM` 更关注某一路音频或视频。

## 今日验收

- 能说清楚：`MP4 = 容器`，`H.264 = 视频编码`，`AAC = 音频编码`。
- 能从 `ffprobe` 输出中找到分辨率、帧率、采样率、声道、码率和时长。

## Day 2 必掌握字段

| 字段 | 含义 | 常见位置 |
|---|---|---|
| `duration` | 时长 | `FORMAT` / `STREAM` |
| `size` | 文件大小 | `FORMAT` |
| `bit_rate` | 码率，单位通常是 bit/s | `FORMAT` / `STREAM` |
| `codec_name` | 编码格式，例如 `h264`、`aac` | `STREAM` |
| `codec_type` | 流类型，例如 `video`、`audio`、`subtitle` | `STREAM` |
| `width` / `height` | 视频宽高，也就是分辨率 | 视频 `STREAM` |
| `r_frame_rate` / `avg_frame_rate` | 帧率 | 视频 `STREAM` |
| `pix_fmt` | 像素格式，例如 `yuv420p` | 视频 `STREAM` |
| `sample_rate` | 音频采样率，例如 `48000` Hz | 音频 `STREAM` |
| `channels` | 声道数 | 音频 `STREAM` |
| `channel_layout` | 声道布局，例如 `mono`、`stereo` | 音频 `STREAM` |

掌握标准：能在 `ffprobe -show_format -show_streams` 输出中亲手找到这些字段，并用自己的话解释它们描述的是“整个文件”还是“某一路媒体流”。
