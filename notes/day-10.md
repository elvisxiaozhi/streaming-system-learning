# Day 10

完成日期：2026-05-20

## 今天做了什么

用 FFmpeg 把同一个源文件（720p 30fps）转成 15fps、30fps、60fps 三个版本，观察文件大小、码率和帧数的变化。

源文件：`labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4`，720p / 30fps / ~111kbps。

### 实验命令

三个版本均使用 `-crf 23`（恒定质量模式），只改帧率。

```bash
# 15fps
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vf fps=15 -c:v libx264 -crf 23 -c:a copy \
  labs/01-ffmpeg-cli/samples/day10_15fps.mp4

# 30fps（重编码对照基线）
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vf fps=30 -c:v libx264 -crf 23 -c:a copy \
  labs/01-ffmpeg-cli/samples/day10_30fps.mp4

# 60fps（从 30fps 源插帧/重复帧补齐）
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -vf fps=60 -c:v libx264 -crf 23 -c:a copy \
  labs/01-ffmpeg-cli/samples/day10_60fps.mp4
```

### 实验结果

| 版本 | 帧率 | 帧数 | 视频码率 | 文件大小 | 单帧平均码率 |
|------|------|------|----------|----------|--------------|
| 15fps | 15 | 450 | 90 kbps | 818K | 6.0 kbps/帧 |
| 30fps | 30 | 900 | 116 kbps | 928K | 3.9 kbps/帧 |
| 60fps | 60 | 1800 | 187 kbps | 1.2M | 3.1 kbps/帧 |

## 关键观察

**1. CRF 模式下，帧率越高文件越大**

`-crf 23` 是恒定质量模式，编码器为了保持"每帧质量一致"，帧数翻倍就要编更多帧，总码率随之上升。15fps → 60fps，码率从 90kbps 涨到 187kbps，约涨 2 倍。

**2. 60fps 并不等于更流畅——源帧率决定上限**

源文件是 30fps，用 `-vf fps=60` 只是让 FFmpeg 把每帧复制一份补足帧数，视频内容并没有新信息。帧数翻倍（900 → 1800），文件变大，但画面流畅度不会提升。真正的 60fps 必须在拍摄阶段就用高帧率相机采集。

**3. 帧率越高，单帧编码成本反而越低**

相邻帧差异越小（每帧间隔时间短），P/B 帧参考前帧越容易找到匹配块，编码效率更高，所以单帧平均比特数随帧率上升而下降（6.0 → 3.1 kbps/帧）。但总帧数多，总码率仍然涨了。

**4. 固定码率（CBR）下帧率越高单帧质量越差**

CBR 把总码率固定，帧率越高每帧能分配的比特越少，单帧细节损失越大。这是直播场景通常选 30fps 而非 60fps 的核心原因——同样的上行带宽，30fps 能给每帧更多比特，画质更稳定。

## 遇到的问题

- 60fps 版本文件最大，但内容与 30fps 版本完全一致，直观说明了"插帧没有意义"。如果想验证真实 60fps 的效果，需要找到原生 60fps 拍摄的素材做对比。

## 我现在能解释什么

- CRF 模式下帧率和文件大小的关系：帧率翻倍 → 码率约翻倍 → 文件约翻倍。
- 为什么后期用 FFmpeg 转出的 60fps 不比原始 30fps 流畅：FFmpeg 只是重复帧，没有新的图像信息。
- CBR 模式下帧率越高单帧质量越差的原因：固定预算分给更多帧，每帧份额减少。
- 直播推流为什么常见 30fps 而不是 60fps：带宽有限时，降低帧率能为每帧保留更多比特，画质更稳。
