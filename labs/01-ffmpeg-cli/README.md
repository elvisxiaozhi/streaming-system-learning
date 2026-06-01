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

## Day 4：抽取裸流并对比

从 Day 2 的 MP4 中分别抽出视频和音频裸流：

```bash
# 抽视频裸流（Annex B 格式，-an 表示去掉音频）
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -an -c:v copy \
  labs/01-ffmpeg-cli/samples/day4_video.h264

# 抽音频裸流（ADTS 格式，-vn 表示去掉视频）
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vn -c:a copy \
  labs/01-ffmpeg-cli/samples/day4_audio.aac
```

文件大小对比：

| 文件 | 大小 | 说明 |
|---|---|---|
| `day2_testsrc_30s.mp4` | 912.4 KB | 视频 + 音频 + 容器开销 |
| `day4_video.h264` | 409.2 KB | 仅视频裸流 |
| `day4_audio.aac` | 479.3 KB | 仅音频裸流 |

关键 `ffprobe` 差异：

- `.h264` 裸流：`Duration: N/A`，`bitrate: N/A`，fps 被猜成 25（实际 30）——裸流没有容器，无时间戳索引。
- `.aac` 裸流：时长被估算为 31.43s（实际 30s），带警告 "Estimating duration from bitrate"——ADTS 格式无全局索引。
- MP4：所有字段精确，因为 moov 盒子存储了完整索引和元数据。
- 409 + 479 ≈ 888 KB，MP4 912 KB，差值 ~24 KB 是容器开销（moov 盒子）。

## Day 5：分辨率与码率转码

第一次真正的转码（重新编码，不是 `-c copy`）：

```bash
# 降分辨率：1280x720 → 640x360
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vf scale=640:360 -c:a copy \
  labs/01-ffmpeg-cli/samples/day5_360p.mp4

# 指定视频码率目标 200kb/s
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -b:v 200k -c:a copy \
  labs/01-ffmpeg-cli/samples/day5_200kbps.mp4
```

数据对比：

| 文件 | 分辨率 | 视频码率 | 文件大小 |
|---|---|---|---|
| `day2_testsrc_30s.mp4`（原始） | 1280x720 | 111 kb/s | 912 KB |
| `day5_360p.mp4` | 640x360 | 55 kb/s | 706 KB |
| `day5_200kbps.mp4` | 1280x720 | 163 kb/s | 1.1 MB |

关键结论：

- `-vf scale` 触发重新编码，`-c copy` 不触发。
- 分辨率降低导致文件变小（像素减少 → bit 需求减少）。
- 指定 `-b:v 200k`（高于原始 111k）反而使文件变大：原始用 CRF 质量优先，强制更高码率只是浪费 bit，不提升画质。
- CRF 比固定码率更适合"在合理质量下压到最小文件"的场景。

## Day 6：集中实验——截图、抽帧、截取、合并、提取音频

本节把五类常用 FFmpeg 操作集中演练，素材均基于 `day2_testsrc_30s.mp4`。

### 实验 1：截图（单帧 PNG）

从第 5 秒位置截取 1 张图：

```bash
ffmpeg -y -ss 5 -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -frames:v 1 \
  labs/01-ffmpeg-cli/samples/day6_screenshot.png
```

| 参数 | 含义 |
|---|---|
| `-ss 5` | 跳转到第 5 秒（放在 `-i` 前面是输入 seek，速度快） |
| `-frames:v 1` | 只输出 1 帧 |

输出：`day6_screenshot.png`，约 45 KB，1280×720 PNG。

### 实验 2：抽帧（每秒 1 帧）

把前 5 秒按每秒 1 帧抽成图片序列：

```bash
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vf "fps=1" -t 5 \
  labs/01-ffmpeg-cli/samples/day6_frames/frame_%03d.png
```

| 参数 | 含义 |
|---|---|
| `-vf "fps=1"` | 视频滤镜：每秒输出 1 帧 |
| `-t 5` | 只处理前 5 秒 |
| `frame_%03d.png` | 输出序列，`%03d` 表示三位数字编号 |

输出：`frame_001.png` ～ `frame_005.png`，每张约 44 KB。

### 实验 3：截取 10 秒片段

从第 5 秒开始，截取 10 秒，不重新编码：

```bash
ffmpeg -y -ss 5 -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -t 10 -c copy \
  labs/01-ffmpeg-cli/samples/day6_clip_10s.mp4
```

| 参数 | 含义 |
|---|---|
| `-ss 5` | 从第 5 秒开始 |
| `-t 10` | 持续 10 秒 |
| `-c copy` | 流复制，不重新编码，速度极快 |

输出文件大小对比：

| 文件 | 时长 | 大小 |
|---|---|---|
| `day2_testsrc_30s.mp4` | 30s | 912 KB |
| `day6_clip_10s.mp4` | 10s | 459 KB |

注意：`-c copy` 截取时会对齐到最近的关键帧，实际时长可能略长于 10 秒。

### 实验 4：提取音频

把视频里的音频流单独提取为 AAC：

```bash
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vn -c:a copy \
  labs/01-ffmpeg-cli/samples/day6_audio_only.aac
```

| 参数 | 含义 |
|---|---|
| `-vn` | 去掉视频流（video none） |
| `-c:a copy` | 音频流直接复制，不重新编码 |

输出：`day6_audio_only.aac`，480 KB，与 Day 4 的 `day4_audio.aac` 一致。

### 实验 5：合并音视频

先从原视频提取纯视频（去掉音频），再与独立音频合并回一个 MP4：

```bash
# 步骤一：提取纯视频流
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -an -c:v copy \
  labs/01-ffmpeg-cli/samples/day6_video_only.mp4

# 步骤二：合并音视频
ffmpeg -y \
  -i labs/01-ffmpeg-cli/samples/day6_video_only.mp4 \
  -i labs/01-ffmpeg-cli/samples/day6_audio_only.aac \
  -c copy \
  labs/01-ffmpeg-cli/samples/day6_merged.mp4
```

| 参数 | 含义 |
|---|---|
| `-an` | 去掉音频流（audio none） |
| 两个 `-i` | 分别输入视频文件和音频文件 |
| `-c copy` | 两路流都不重新编码，直接合封装 |

文件大小对比：

| 文件 | 内容 | 大小 |
|---|---|---|
| `day6_video_only.mp4` | 仅视频 | 420 KB |
| `day6_audio_only.aac` | 仅音频 | 480 KB |
| `day6_merged.mp4` | 合并后 | 912 KB |

合并后大小 ≈ 视频 + 音频，容器开销可忽略不计（`-c copy` 无需额外编码）。

### Day 6 关键 ffprobe 对比

#### 截图（PNG）vs 视频帧

```bash
ffprobe -v error -show_streams labs/01-ffmpeg-cli/samples/day6_screenshot.png
```

| 字段 | PNG 截图 | 原始 MP4 视频流 |
|---|---|---|
| `codec_name` | `png` | `h264` |
| `pix_fmt` | `rgb24` | `yuv420p` |
| `color_space` | `gbr` | 未设置 |
| `duration` | `N/A` | `30.000000` |
| `start_time` | `N/A` | `0.000000` |

关键结论：
- `-frames:v 1` 输出 PNG 时，FFmpeg 自动把 `yuv420p` 转成了 `rgb24`。PNG 是无损 RGB 图像格式，不用 YUV。
- PNG 是静态图像，没有时间信息，所以 `duration` 和 `start_time` 均为 `N/A`。

#### 截取片段：时长为何是 10.1s 而非 10s

```bash
ffprobe -v error -show_format labs/01-ffmpeg-cli/samples/day6_clip_10s.mp4
```

| 文件 | 请求时长 | 实际时长 |
|---|---|---|
| `day6_clip_10s.mp4` | `-t 10`（10 秒）| `10.100000` s |

原因：`-c copy` 不重新编码，只能从关键帧（I 帧）处开始切割。第 5 秒处如果不是关键帧，FFmpeg 会自动向前对齐到最近的关键帧，导致实际时长略超 10 秒。

#### 提取音频：ADTS 裸流 vs 封装进 MP4

| 字段 | `day6_audio_only.aac`（ADTS 裸流）| `day6_merged.mp4` 音频流 |
|---|---|---|
| 容器格式 | `aac`（raw ADTS）| `mov,mp4,m4a...` |
| `duration` | `31.413501` s（估算）| `30.037333` s（精确）|
| `start_time` | `N/A` | `0.000000` |
| `bit_rate` | `124971` bit/s（估算）| `128072` bit/s（精确）|
| `extradata_size` | `N/A` | `2` bytes |

关键结论：
- ADTS 裸流没有全局索引，`duration` 和 `bit_rate` 都是 ffprobe 根据文件大小估算的，和 Day 4 `.aac` 的结论一致。
- 封装进 MP4 后，`duration` 变精确（由 moov box 记录），`extradata_size=2` 说明容器里存储了 AAC 的 codec config（AudioSpecificConfig），这是 ADTS 所没有的。
- 音频时长 `30.037333` 略大于 `30.000000`，是 AAC 帧边界对齐造成的正常误差。

#### 合并前后流数量对比

```bash
ffprobe -v error -show_format labs/01-ffmpeg-cli/samples/day6_video_only.mp4
ffprobe -v error -show_format labs/01-ffmpeg-cli/samples/day6_merged.mp4
```

| 文件 | `nb_streams` | 视频流 | 音频流 |
|---|---|---|---|
| `day6_video_only.mp4` | 1 | H.264 High | 无 |
| `day6_merged.mp4` | 2 | H.264 High | AAC LC |

合并后两路流的编解码参数与分离前完全一致（`-c copy` 不改变编码）。

### Day 6 命令速查

| 操作 | 核心参数 |
|---|---|
| 截图 | `-ss <秒> -frames:v 1` |
| 抽帧序列 | `-vf "fps=N" -t <秒>` + `%03d.png` |
| 截取片段 | `-ss <开始> -t <时长> -c copy` |
| 提取音频 | `-vn -c:a copy` |
| 提取视频 | `-an -c:v copy` |
| 合并音视频 | 两个 `-i` + `-c copy` |

## samples/ 目录归档清单

`samples/` 下的所有实验产物都不入 Git，丢失后需要按下表重新生成。两个**源文件**必须长期保留（其他 day 的实验都从它们派生），其余可按需重跑。

### 源文件（必留，丢失需重新生成）

| 文件 | 用途 | 生成命令 |
|---|---|---|
| `day2_testsrc_30s.mp4` | 合成源（720p / 30fps / 30s，testsrc 几何图案 + 1kHz 正弦）；Day 2-10 多个实验的输入 | 见上方 Day 2 一节 |
| `day11_noisy_src.mp4` | 带噪点源（720p / 30fps / 10s，testsrc + noise filter，无音频）；Day 11 / Day 12 画质对比实验的输入 | `ffmpeg -y -f lavfi -i "testsrc=duration=10:size=1280x720:rate=30,noise=alls=20:allf=t" -c:v libx264 -pix_fmt yuv420p -crf 23 labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4` |

### 派生产物（可丢可重生）

| 文件 | 所属 Day | 实验主题 |
|---|---|---|
| `day6_*.{png,mp4,aac}`、`day6_frames/` | Day 6 | 截图 / 抽帧 / 截取 / 提取 / 合并 |
| `day10_{15,30,60}fps.mp4` | Day 10 | 帧率转换 |
| `day12_noisy_{cbr_500k,cbr_1500k,vbr_500k,crf_capped}.mp4` | Day 12 | 带噪点源的 CBR / VBR / CRF+maxrate 对比 |
| `day12_synth_{cbr_500k,vbr_500k,crf_capped}.mp4` | Day 12 | 合成源的 CBR / VBR / CRF+maxrate 对比 |

### 经验规则

- **lavfi 生成或带 filter 的源必须显式 `-pix_fmt yuv420p`**：否则 libx264 可能选 yuv444p / High 4:4:4 Predictive，QuickTime / Finder 预览 / 浏览器 / Windows 自带播放器普遍不支持，看起来像"文件没坏但没画面"。
- **新增源文件时同步更新本表**，并记录"哪些 day 的实验依赖它"，避免源消失后下游实验找不到上游。
