# Day 11

完成日期：2026-05-31

## 今天做了什么

用 FFmpeg 对同一源文件做 CRF 18 / 23 / 28 三档转码，观察文件大小、码率和主观画质的变化。发现合成源看不出画质差异，于是补充了一组带噪点的真实感源文件对比。

源文件：`labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4`，720p / 30fps / 合成图案。

### 实验一：合成源 CRF 对比

```bash
ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c:v libx264 -crf 18 -c:a copy labs/01-ffmpeg-cli/samples/day11_crf18.mp4

ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c:v libx264 -crf 23 -c:a copy labs/01-ffmpeg-cli/samples/day11_crf23.mp4

ffmpeg -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c:v libx264 -crf 28 -c:a copy labs/01-ffmpeg-cli/samples/day11_crf28.mp4
```

#### 结果

| 版本 | CRF | 文件大小 | 视频码率 | 主观画质 |
|------|-----|----------|----------|----------|
| day11_crf18 | 18 | ~1.05 MB | ~152 kbps | 肉眼无差异 |
| day11_crf23 | 23 | ~930 KB  | ~114 kbps | 肉眼无差异 |
| day11_crf28 | 28 | ~791 KB  | ~77 kbps  | 肉眼无差异 |

三个版本文件大小相差约 1.3 倍，但画质肉眼无法区分。

### 实验二：带噪点源 CRF 对比

```bash
# 生成带噪点的源
ffmpeg -f lavfi -i "testsrc=duration=10:size=1280x720:rate=30,noise=alls=20:allf=t" \
  -c:v libx264 -crf 23 labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4

ffmpeg -i labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4 \
  -c:v libx264 -crf 18 -c:a copy labs/01-ffmpeg-cli/samples/day11_noisy_crf18.mp4

ffmpeg -i labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4 \
  -c:v libx264 -crf 28 -c:a copy labs/01-ffmpeg-cli/samples/day11_noisy_crf28.mp4
```

#### 结果

| 版本 | CRF | 文件大小 | 视频码率 | 主观画质 |
|------|-----|----------|----------|----------|
| noisy_crf18 | 18 | ~1.84 MB | ~1,537 kbps | 噪点细腻，接近原始 |
| noisy_crf28 | 28 | ~339 KB  | ~274 kbps   | 噪点区域出现明显块状感 |

文件大小相差 **5.5 倍**，码率相差 **5.6 倍**，画质差异肉眼可见。

## 关键观察

**1. CRF 控制的是质量，不是码率**

CRF 告诉编码器"保持这个质量等级"，码率由内容决定。同样的 CRF 23，合成源只需 114 kbps，带噪点源需要数倍码率才能维持同等质量。码率是结果，不是输入。

**2. 内容复杂度决定 CRF 的实际效果**

合成图案（纯色、几何形状）信息量少，CRF 28 已经够用，几乎无损；带噪点的真实感内容信息量大，CRF 28 强行丢弃大量细节，块状感明显。这就是为什么 CRF 无法作为码率上限的保证——内容越复杂，文件越大。

**3. CRF 每变化 6，码率约翻倍**

从数据推算：CRF 18 → 23 码率从 152 降到 114（约 ×0.75），CRF 23 → 28 从 114 降到 77（约 ×0.67），合成源因为内容太简单压缩空间有限，体现不明显。带噪点源的 CRF 18 vs 28（跨 10 格）码率相差 5.6 倍，更接近"每 6 格翻倍"的理论值。

**4. 合成测试源不适合做画质实验**

`testsrc` 是验证工具行为的源，不是验证画质感知的源。今后做画质对比实验要用真实拍摄素材或带噪点/复杂纹理的测试源。

## 遇到的问题

- 合成源三个 CRF 版本肉眼完全看不出差异，一度以为实验有问题。原因是合成图案过于规则，编码器在低质量模式下仍能精确重建，并非实验失败。
- 补充带噪点源后，CRF 28 的块状感立刻可见，验证了 CRF 机制本身是正常的。

## 我现在能解释什么

- CRF 是质量目标，不是码率目标：相同 CRF 下，复杂内容产生大文件，简单内容产生小文件。
- 为什么直播不能单纯用 CRF：内容突然变复杂（快速运动、场景切换），码率可能瞬间飙升，超出上行带宽。直播需要 CBR 或 VBR+码率上限来约束带宽。
- CRF 经验规律：每升高 6，码率约减半；每降低 6，码率约翻倍。
- 做画质感知实验必须用真实或复杂内容，合成源无法体现 CRF 的画质差距。
