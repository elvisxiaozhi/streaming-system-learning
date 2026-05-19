# 2026-05-19

## 今天做了什么

Day 6 集中实验，围绕 `day2_testsrc_30s.mp4` 跑了五类 FFmpeg 操作：

1. **截图**：`-ss 5 -frames:v 1` 从第 5 秒截一张 PNG，约 45 KB。
2. **抽帧**：`-vf "fps=1" -t 5` 每秒抽 1 帧，输出 5 张 PNG 序列。
3. **截取 10 秒**：`-ss 5 -t 10 -c copy` 不重新编码截出片段，459 KB。
4. **提取音频**：`-vn -c:a copy` 提取 AAC 裸流，480 KB。
5. **合并音视频**：先 `-an` 提纯视频，再用两个 `-i` + `-c copy` 合并回 MP4，912 KB。

所有命令整理到 `labs/01-ffmpeg-cli/README.md`。

## ffprobe 关键发现

- **PNG 像素格式**：`-frames:v 1` 截图时，FFmpeg 自动把 `yuv420p` 转成了 `rgb24`，因为 PNG 是 RGB 格式；截图没有时间信息，`duration=N/A`。
- **截取时长偏差**：请求 `-t 10`，实际 `duration=10.1s`。`-c copy` 必须对齐到关键帧，所以时长会向后多一点。
- **ADTS 音频 vs MP4 封装**：裸流 `.aac` 的 `duration=31.41s`（估算，和 Day 4 一样），封装进 MP4 后变成精确的 `30.04s`。MP4 音频多了 `extradata_size=2`（AudioSpecificConfig），是容器存储 codec config 的体现。
- **合并不改编码**：`-c copy` 合并后两路流的 codec 参数与分离前完全一致。

## 遇到的问题

- 公司电脑 ffmpeg 不在 PATH 里，需要用完整路径调用（`C:\Users\ZYB\Downloads\...`）。
- 素材文件不在 git 里，换电脑后需要先重新生成 `day2_testsrc_30s.mp4`。
- `-ss` 放在 `-i` 前（输入 seek）速度快，放在 `-i` 后（输出 seek）会逐帧解码，两者行为略有差异。

## 我现在能解释什么

- `-c copy` 是流复制，不重新编码，速度极快，但要求目标容器支持原始编码格式。
- `-vn` 去掉视频，`-an` 去掉音频，`-frames:v 1` 只输出一帧。
- `-vf "fps=N"` 是视频滤镜，会触发重新编码（解码再输出 PNG），不同于 `-c copy`。
- 合并音视频时，两路流用两个 `-i` 输入，`-c copy` 只做重新封装。
- 截取片段时 `-c copy` 对齐关键帧，实际时长可能略偏。
