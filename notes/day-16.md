# Day 16

完成日期：2026-09-17

## 今天做了什么

从同一个 10 秒、1000 Hz 正弦波出发，对比 WAV、裸 PCM、AAC/M4A 和 Opus/Ogg，并将 AAC、Opus 解码回 PCM 做 SHA-256 对比。

实验环境：FFmpeg 8.1.1；基准音频为 48000 Hz、16-bit、单声道、10 秒。

## 理论基线

```text
PCM 码率 = 48000 × 16 × 1 = 768000 bit/s
PCM 数据大小 = 768000 × 10 ÷ 8 = 960000 bytes
AAC 128k 理论大小 = 128000 × 10 ÷ 8 = 160000 bytes
Opus 64k 理论大小 = 64000 × 10 ÷ 8 = 80000 bytes
```

## 实验结果

| 文件 | 编码 / 容器 | 实际大小 | 总平均码率 | 关键观察 |
|---|---|---:|---:|---|
| `day16_source.wav` | PCM / WAV | 960078 bytes | 768062 bit/s | 比纯 PCM 多 78 bytes 格式头与元数据 |
| `day16_raw.pcm` | PCM / 无容器 | 960000 bytes | 768000 bit/s | 不指定参数时 `ffprobe` 报 `Invalid data found` |
| `day16_aac.m4a` | AAC / M4A | 162211 bytes | 129768 bit/s | 音频流约 127347 bit/s，接近 128k 目标 |
| `day16_opus.ogg` | Opus / Ogg | 120193 bytes | 96091 bit/s | `libopus` 默认 `-vbr on`，64k 目标未形成严格上限 |
| `day16_opus_cbr.ogg` | Opus / Ogg | 81095 bytes | 64833 bit/s | `-vbr off` 后紧贴 64k，额外约 1095 bytes 为封装等开销 |

裸 PCM 在 FFmpeg 8.1.1 中的正确探测命令：

```bash
ffprobe -v error \
  -f s16le \
  -sample_rate 48000 \
  -ch_layout mono \
  -show_streams -show_format \
  labs/01-ffmpeg-cli/samples/day16_raw.pcm
```

`s16le` demuxer 的参数为 `-sample_rate` 和 `-ch_layout`；本机 `ffprobe` 不接受之前尝试的 `-ac 1`。这些参数不会改写文件，而是告诉工具如何解释裸字节。

## 有损编码验证

将 AAC 和 Opus 都解码成 `pcm_s16le` 后：

| 文件 | 大小 | SHA-256 前缀 |
|---|---:|---|
| 原始裸 PCM | 960000 bytes | `15a7d88b...` |
| AAC 解码 PCM | 960512 bytes | `787a3a78...` |
| Opus 解码 PCM | 960000 bytes | `8d97ea8e...` |

- 三份哈希全部不同，证明 AAC/Opus 解码后只能得到听感接近的 PCM，不能逐字节恢复原始样本。
- Opus 解码文件与原始 PCM 大小相同但哈希不同，证明“文件大小相同”不等于“内容相同”。
- AAC 解码后多 512 bytes，即 256 个 16-bit mono 样本；结果与 480000 个输入样本补齐到 469 个、每个 1024 samples 的 AAC 帧一致。

## 关键理解

- WAV 是容器；本实验的 WAV 内部是 PCM。WAV 格式头记录采样率、位深、声道等解释参数。
- 裸 PCM 只有振幅样本，不包含采样率、位深、大小端和声道数；同一组字节可以按多种方式解释。
- 采样率描述一秒对应多少个采样时刻；编码码率描述压缩后每秒使用多少 bit。有损压缩会减少表达声音的数据量，不等于重采样。
- AAC/Opus 的 `sample_fmt=fltp` 表示解码器的浮点 planar 输出格式，不表示压缩文件内保存浮点 PCM。`bits_per_sample=0` 表示它们不像 PCM 那样具有固定的每样本位深。
- VBR 中的目标码率不是严格上限；编码器可以根据内容调整码率。

## 遇到的问题

- 第一次计算 PCM 码率时少写一个 0：`48000 × 16 = 768000`，不是 76800。可用“码率 × 时长 ÷ 8”反向校验。
- 一度认为 `ffprobe` 能自动知道裸 PCM 的参数；实际无参数探测直接失败。
- 一度预测 AAC/Opus 解码回 PCM 后哈希相同；实验证明有损编码不可逆。
- `ffprobe -ac 1` 在 FFmpeg 8.1.1 中报 `Option not found`，通过 `ffprobe -h demuxer=s16le` 查到当前参数名。

## 我现在能解释什么

- WAV 与裸 PCM 的区别，以及裸 PCM 为什么必须由外部提供格式参数。
- 采样率和编码码率描述的是两个不同维度。
- 为什么 AAC/Opus 体积远小于 PCM，但解码后不能恢复原始哈希。
- 为什么 Opus VBR 中的 64k 是目标码率而非严格上限，以及 `-vbr off` 对文件大小的影响。

## 今日验收

闭卷能说清：裸 PCM 只有振幅样本，AAC/Opus 压缩减少的是表达声音的数据量，VBR 目标码率不是严格上限。

下一步：Day 17，观察原始与压缩后的波形，建立“听感接近但样本不同”的直观认识。
