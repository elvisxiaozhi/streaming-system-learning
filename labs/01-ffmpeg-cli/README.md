# 01 - FFmpeg 命令实验

本目录记录 FFmpeg / ffprobe 命令行实验。

## Day 1 环境

- FFmpeg：已安装
- ffprobe：已安装
- 后续 Day 2 需要准备一个 30-60 秒 mp4 样例

## Day 2：生成并分析第一个 mp4

本地生成一个 30 秒测试视频，视频为彩条测试图，音频为 1kHz 正弦波。生成的视频是实验资产，不提交到 Git，后续可用命令复现。

```bash
ffmpeg -y \
  -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -t 30 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

查看文件整体信息：

```bash
ffprobe -hide_banner labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

查看完整 format / stream 字段：

```bash
ffprobe -v error -show_format -show_streams labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

关键观察结果：

| 项 | 结果 |
|---|---|
| 容器格式 | `mov,mp4,m4a,3gp,3g2,mj2`，也就是 MP4/MOV 家族 |
| 总时长 | 30.000000 秒 |
| 文件大小 | 934288 bytes，约 912 KB |
| 总码率 | 249143 bit/s，约 249 kb/s |
| 流数量 | 2 条，1 条视频流 + 1 条音频流 |
| 视频编码 | H.264 / AVC，profile 为 High |
| 视频封装标记 | `avc1` |
| 视频分辨率 | 1280x720 |
| 视频帧率 | 30 fps |
| 视频像素格式 | `yuv420p` |
| 视频帧数 | 900 帧 |
| 视频码率 | 111684 bit/s，约 112 kb/s |
| 音频编码 | AAC LC |
| 音频采样率 | 48000 Hz |
| 音频声道 | mono，单声道 |
| 音频码率 | 128169 bit/s，约 128 kb/s |

当前理解：

- MP4 是容器，H.264 和 AAC 是编码格式。
- 一个 mp4 文件里可以同时包含视频流和音频流。
- `ffprobe` 的 `FORMAT` 描述整个文件，`STREAM` 描述文件里的每一路媒体流。

Day 2 必掌握字段：

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

## Day 3：容器与编码格式

把 Day 2 的 MP4 样例重新封装成 FLV 和 TS：

```bash
ffmpeg -y \
  -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c copy \
  labs/01-ffmpeg-cli/samples/day3_remux.flv

ffmpeg -y \
  -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c copy \
  labs/01-ffmpeg-cli/samples/day3_remux.ts
```

关键对比：

| 文件 | 容器格式 | 视频编码 | 音频编码 |
|---|---|---|---|
| `day2_testsrc_30s.mp4` | `mov,mp4,m4a,3gp,3g2,mj2` | H.264 High | AAC LC |
| `day3_remux.flv` | `flv` | H.264 High | AAC LC |
| `day3_remux.ts` | `mpegts` | H.264 High | AAC LC |

观察结论：

- 三个文件的容器格式不同，但视频编码和音频编码保持一致。
- `-c copy` 做的是重新封装，不重新编码。
- 容器变化后，`Duration`、`start`、`bitrate` 可能出现轻微变化，例如 FLV/TS 的起始时间和总码率与 MP4 不完全相同；这是容器组织方式、时间戳表示和封装开销不同造成的正常现象。
- `-c copy` 并不是对任何输出格式都一定成立，目标容器必须支持原有编码流。
