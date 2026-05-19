# 2026-05-17

## 今天做了什么

- 完成 Day 4：从 MP4 中抽出 H.264 视频裸流和 AAC 音频裸流。
- 使用 `ffprobe` 对比三个文件：`day2_testsrc_30s.mp4`、`day4_video.h264`、`day4_audio.aac`。
- 观察裸流与有容器的文件在 `ffprobe` 输出上的具体差异。

## 实验命令

```bash
# 抽视频裸流（Annex B 格式）
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -an -c:v copy \
  labs/01-ffmpeg-cli/samples/day4_video.h264

# 抽音频裸流（ADTS 格式）
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vn -c:a copy \
  labs/01-ffmpeg-cli/samples/day4_audio.aac
```

## 文件大小对比

| 文件 | 大小 | 说明 |
|---|---|---|
| `day2_testsrc_30s.mp4` | 912.4 KB | 视频 + 音频 + 容器开销 |
| `day4_video.h264` | 409.2 KB | 仅视频裸流 |
| `day4_audio.aac` | 479.3 KB | 仅音频裸流 |

409.2 + 479.3 = 888.5 KB，而 MP4 是 912.4 KB，多出的约 **24 KB 是 MP4 容器开销**（moov 盒子：索引表、元数据、时间戳映射等结构）。

## ffprobe 输出关键差异

### H.264 裸流 vs MP4

| 字段 | MP4 | .h264 裸流 | 原因 |
|---|---|---|---|
| Duration | 00:00:30.00（精确） | N/A | 裸流没有容器，无时间戳索引 |
| bitrate | 111 kb/s（精确） | N/A | 无法从裸流计算 |
| fps | 30 fps | 25 fps（猜测） | ffprobe 从 SPS 默认值推断，不可靠 |
| tbn（时间基） | 15360 | 1200k | 前者来自 MP4 容器，后者来自 SPS |
| 编解码标记 | `h264 (High) (avc1 / 0x31637661)` | `h264 (High)` | 无 AVCC 封装，只有裸 Annex B |
| 容器元数据 | major_brand、handler_name、encoder | 无 | 这些字段存在 moov 盒子里 |

### AAC 裸流 vs MP4

| 字段 | MP4 | .aac 裸流 | 原因 |
|---|---|---|---|
| Duration | 00:00:30.00（精确） | 00:00:31.43（估算） | ADTS 没有全局索引，ffprobe 靠码率估算 |
| bitrate | 128 kb/s（精确） | 124 kb/s（估算） | 同上 |
| 警告 | 无 | "Estimating duration from bitrate, this may be inaccurate" | ADTS 固有限制 |

## 关键结论

1. **裸流没有容器，就没有精确的时间信息。** `.h264` 里没有 Duration 和 bitrate，ffprobe 只能从 SPS 猜 fps，结果不准（25 vs 实际 30）。

2. **AAC 从 MP4 中抽出后变成 ADTS 格式。** MP4 内部的 AAC 是"裸 raw_data_block"，写成独立文件时 FFmpeg 自动加上每帧 7 字节的 ADTS header 来保持自同步能力，代价是没有全局时长索引。

3. **文件大小 = 视频裸流 + 音频裸流 + 容器开销。** 409 + 479 ≈ 888 KB，MP4 912 KB，差值约 24 KB 是 moov 盒子的代价（换来了精确的 seek 和元数据能力）。

4. **`avc1` 和 Annex B 是同一套 H.264 数据的两种封装方式。** MP4 内用 AVCC（长度前缀 NALU），裸流文件用 Annex B（start code 前缀 NALU）。`-c:v copy` 抽流时 FFmpeg 自动完成了这个转换。

## 我现在能解释什么

- 为什么对 `.h264` 裸流跑 `ffprobe` 会看到 `Duration: N/A`：裸流没有容器，容器才是存放时长和时间戳索引的地方。
- 容器的价值：不只是"打包"，而是提供索引（可 seek）、精确时长、元数据、多流同步等能力。
- 抽出来的 `.aac` 为什么比 MP4 内的 AAC 多了开销：每帧多了 7 字节 ADTS header，但换来了无需容器即可自同步播放的能力。
- 文件大小的组成：媒体数据本身 + 容器索引开销，两者加起来才是最终文件大小。
