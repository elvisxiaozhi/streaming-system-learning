# Day 9

完成日期：2026-05-19

## 今天做了什么

学习 I/P/B 帧的基本作用；用 `ffprobe -show_frames` 找出视频里的关键帧，并与 Day 8 的 `-show_packets` 结果对比。

### 命令 1：查看前 20 帧的帧类型

```bash
ffprobe -v error -show_frames -select_streams v:0 \
  labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

前 20 帧关键字段（已按显示顺序排列）：

| # | pts_time | pict_type | key_frame | pkt_size |
|---|---|---|---|---|
| 1 | 0.000s | I | 1 | 8686 |
| 2 | 0.033s | B | 0 | 225 |
| 3 | 0.067s | B | 0 | 208 |
| 4 | 0.100s | B | 0 | 287 |
| 5 | 0.133s | P | 0 | 968 |
| 6 | 0.167s | B | 0 | 295 |
| 7 | 0.200s | B | 0 | 237 |
| 8 | 0.233s | B | 0 | 266 |
| 9 | 0.267s | P | 0 | 872 |
| 10 | 0.300s | B | 0 | 287 |
| 11 | 0.333s | B | 0 | 269 |
| 12 | 0.367s | B | 0 | 200 |
| 13 | 0.400s | P | 0 | 871 |
| 14 | 0.433s | B | 0 | 255 |
| 15 | 0.467s | B | 0 | 275 |
| 16 | 0.500s | B | 0 | 288 |
| 17 | 0.533s | P | 0 | 923 |
| 18 | 0.567s | B | 0 | 264 |
| 19 | 0.600s | B | 0 | 310 |
| 20 | 0.633s | B | 0 | 279 |

### 命令 2：统计全片帧类型数量

```bash
ffprobe -v error -show_frames -select_streams v:0 \
  labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 | \
  grep "^pict_type" | sort | uniq -c
```

| 帧类型 | 数量 | 占比 |
|---|---|---|
| I | 4 | 0.4% |
| P | 228 | 25.3% |
| B | 668 | 74.2% |
| **合计** | **900** | 30s × 30fps = 900 ✓ |

### 命令 3：找出所有 I 帧的位置

```bash
ffprobe -v error -show_frames -select_streams v:0 \
  labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 | \
  grep -E "^(pict_type|pts_time|pkt_size)" | \
  paste - - - | grep "pict_type=I"
```

| I 帧出现时间 | pkt_size |
|---|---|
| 0.000s | 8686 |
| 8.333s | 9034 |
| 16.667s | 9022 |
| 25.000s | 9200 |

GOP 间隔 = 8.333s × 30fps = **250 帧**，这是 libx264 的默认 GOP 大小。

## 关键观察

**GOP 结构**

从显示顺序看，每个 GOP 的模式是：`I, BBB, P, BBB, P, BBB, P...`，每个 P 帧前有 3 个 B 帧。250 帧 / (3B + 1P) = 62.5 个小组，其中第 1 个是 I 帧，后续都是 P 帧打头带 3 个 B 帧。

**三种帧的大小差异**

| 帧类型 | 典型 pkt_size | 原因 |
|---|---|---|
| I 帧 | ~8686 - 9200 字节 | 完整图像，不依赖其他帧 |
| P 帧 | ~828 - 968 字节 | 只存与前一帧的差异 |
| B 帧 | ~200 - 343 字节 | 同时参考前后帧，冗余最少 |

B 帧只有 I 帧的 1/30 左右，这就是为什么加入 B 帧能显著压缩文件大小。

**`-show_frames` vs `-show_packets` 的关键区别**

| | `-show_packets` | `-show_frames` |
|---|---|---|
| 顺序 | DTS 顺序（解码顺序） | PTS 顺序（显示顺序）|
| PTS 是否单调 | 不单调（B 帧乱序）| 严格单调递增 |
| 能看到 `pict_type` | 否 | 是 |
| 能看到 `key_frame` | 是（flags=K__）| 是（key_frame=1）|

`-show_frames` 的结果相当于解码器"看到的视图"——已经按显示顺序排好，pts 单调递增。`-show_packets` 是容器"存储的视图"——按解码顺序排列，pts 乱序。

## 遇到的问题

- `-show_frames` 处理完整 30s 视频较慢（需要完整解码所有帧），比 `-show_packets` 慢很多。

## 我现在能解释什么

- I/P/B 三种帧的压缩原理和大小关系。
- GOP 结构：I 帧打头，后跟 P/B 帧，本视频 GOP = 250 帧（约 8.3s）。
- `-show_frames` 给出显示顺序（PTS 单调），`-show_packets` 给出解码顺序（DTS 单调）。
- B 帧占全片 74%，是文件小的主要原因之一。
