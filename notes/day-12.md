# Day 12

完成日期：2026-06-01

## 今天做了什么

在 Day 11（CRF 质量驱动）之后，反向验证码率优先的三种模式：CBR（严格固定码率）、VBR（带峰值上限的目标码率）、以及直播常用的 **CRF + maxrate** 混合模式。两套源文件横向对比，看同样的码率策略在简单内容和复杂内容上的差异。

### 源文件

- `labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4`：合成源（720p / 30fps / 30 秒，testsrc 几何图案 + AAC 静音音轨）
- `labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4`：带噪点源（720p / 30fps / 10 秒，testsrc + `noise=alls=20:allf=t`，无音频）

> 这次重新生成带噪点源时显式加了 `-pix_fmt yuv420p`，原因见后文「遇到的问题」。

### 实验一：CBR（严格固定码率）

```bash
# 严格 CBR 500 kbps（带噪点源）
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4 \
  -c:v libx264 -pix_fmt yuv420p -b:v 500k -minrate 500k -maxrate 500k -bufsize 1000k \
  -x264-params "nal-hrd=cbr" \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_noisy_cbr_500k.mp4

# 严格 CBR 1500 kbps（带噪点源，作为宽松对照组）
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4 \
  -c:v libx264 -pix_fmt yuv420p -b:v 1500k -minrate 1500k -maxrate 1500k -bufsize 3000k \
  -x264-params "nal-hrd=cbr" \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_noisy_cbr_1500k.mp4
```

`-minrate = -maxrate = -b:v` 是严格 CBR 的标准写法；`-x264-params "nal-hrd=cbr"` 告诉 x264 严格遵守 HRD 模型，必要时插入 padding NAL 单元填满目标码率。

### 实验二：VBR（带峰值上限）

```bash
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4 \
  -c:v libx264 -pix_fmt yuv420p -b:v 500k -maxrate 1000k -bufsize 2000k \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_noisy_vbr_500k.mp4
```

平均往 500k 靠，瞬时允许冲到 1000k。

### 实验三：CRF + maxrate（直播常用）

```bash
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day11_noisy_src.mp4 \
  -c:v libx264 -pix_fmt yuv420p -crf 23 -maxrate 1000k -bufsize 2000k \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_noisy_crf_capped.mp4
```

CRF 给质量目标，maxrate 兜带宽上限，冲突时 maxrate 优先。

### 实验四：合成源也跑同样三种模式

```bash
ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c:v libx264 -pix_fmt yuv420p -b:v 500k -minrate 500k -maxrate 500k -bufsize 1000k \
  -x264-params "nal-hrd=cbr" \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_synth_cbr_500k.mp4

ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c:v libx264 -pix_fmt yuv420p -b:v 500k -maxrate 1000k -bufsize 2000k \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_synth_vbr_500k.mp4

ffmpeg -y -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c:v libx264 -pix_fmt yuv420p -crf 23 -maxrate 1000k -bufsize 2000k \
  -c:a copy labs/01-ffmpeg-cli/samples/day12_synth_crf_capped.mp4
```

## 实验结果

### 带噪点源（复杂内容，10 秒）

| 模式 | 文件大小 | 实际平均码率 | 主观画质 |
|---|---|---|---|
| Day 11 CRF 23 参考 | ~3.0 MB | ~3000 kbps | 噪点细腻，接近源 |
| CBR 500k | 614 KB | 503 kbps | 块状感强烈，噪点纹理被抹平，高频细节大量丢失 |
| CBR 1500k | 1.87 MB | 1565 kbps | 块状感大幅减轻，噪点基本保留，整体接近源 |
| VBR 500k / maxrate 1000k | 624 KB | 511 kbps | 与 CBR 500k 接近，峰值段稍有缓解；均匀复杂内容下差异不显著 |
| CRF 23 + maxrate 1000k | 1.22 MB | 1023 kbps（顶到 maxrate） | 介于 CBR 500k 和 CBR 1500k 之间，噪点保留优于 CBR 500k，仍有可见压缩感 |

### 合成源（简单内容，30 秒）

| 模式 | 文件大小 | 实际平均码率（含音频） | 主观画质 |
|---|---|---|---|
| Day 11 CRF 23 参考 | ~930 KB | ~114 kbps | 肉眼无差异 |
| CBR 500k | 2.27 MB | 635 kbps | 肉眼无差异（码率被浪费，无质量增益） |
| VBR 500k / maxrate 1000k | 1.63 MB | 455 kbps | 肉眼无差异 |
| CRF 23 + maxrate 1000k | 928 KB | 253 kbps | 肉眼无差异（maxrate 未生效，行为同纯 CRF 23） |

合成源的码率包含约 130 kbps 的 AAC 音频，视频部分要相应减去。

## 关键观察

**1. CBR 在简单内容上会主动"填满"目标码率**

合成源用 CRF 23 只要 114 kbps，但 CBR 500k 强制把视频部分撑到约 500 kbps，文件足足大了 2 倍多。多出来的码率没有变成更好的画质（合成内容本来就压缩极限了），实际上是被编码器以更低 QP（更精细的量化）和填充 NAL 单元的形式"浪费"掉了。

CBR 的本质是"占满带宽"，不是"得到最好画质"。

**2. CBR 在复杂内容上用画质换码率**

带噪点源 Day 11 CRF 23 需要约 3000 kbps 才能维持质量。CBR 500k 把它压到 500 kbps，是原始所需的 1/6，编码器只能加大 QP、丢弃细节，理论上块状感会非常明显。CBR 1500k 给到 1500 kbps，块状感应大幅减少但还没到 CRF 23 的水准。

CBR 的代价不是码率，是画质——内容越复杂，CBR 越吃亏。

**3. VBR 的优势在峰值缓冲**

VBR 500k / maxrate 1000k 和 CBR 500k 的最终码率几乎一样（511 vs 503 kbps），但是允许编码器在复杂段落临时冲到 1000k。同样的平均带宽消耗，VBR 总体画质应优于 CBR，因为它把 bit 优先分给最难压的帧。

合成源 VBR 跑出来 455 kbps（低于目标 500k），证明 VBR 在简单内容上不会硬撑——这是 VBR 比 CBR 更"省"的地方。

**4. CRF + maxrate 是质量优先 + 软上限**

带噪点源 CRF 23 + maxrate 1000k 最终顶到 1023 kbps，说明这个内容下 CRF 23 想要的码率远高于 1000k，maxrate 起到了硬约束作用——此时实际行为接近 VBR。

合成源 CRF 23 + maxrate 1000k 跑出 253 kbps（远低于上限），maxrate 完全没生效，行为就是纯 CRF。

这正是直播场景需要的："大部分时段质量恒定，只在画面突然复杂时退化为有上限的 VBR"。

**5. HRD / bufsize 决定上限的灵活程度**

`maxrate` 是峰值码率，`bufsize` 是 HRD 虚拟缓冲区大小。bufsize 决定编码器在多长一段时间内可以平均地接近 maxrate：
- bufsize = maxrate：编码器几乎要时时刻刻遵守上限，行为更接近 CBR
- bufsize = 2 × maxrate：允许短时间冲高，长期均值收敛，行为更接近 VBR

这次实验都用 bufsize = 2 × maxrate，是直播推流的典型配置（约 2 秒的缓冲窗口）。

## 遇到的问题

**问题：4 个 noisy 输出文件在 QuickTime / Finder 预览 / 浏览器里都看不到画面。**

排查路径：
1. `ffprobe ... stream=index,codec_type,nb_frames`：所有文件都有完整的 H.264 视频流和正确帧数，文件本身没坏。
2. `ffprobe ... stream=pix_fmt,profile`：发现 4 个 noisy 文件都是 `High 4:4:4 Predictive / yuv444p`，3 个 synth 文件是 `High / yuv420p`。
3. 根因：Day 11 重新生成 `day11_noisy_src.mp4` 时没指定 `-pix_fmt yuv420p`，`noise` filter 让 libx264 选择了 yuv444p 输出。所有后续从这个源转码的文件都继承了 yuv444p，消费级播放器普遍不支持这种 4:4:4 色度采样。
4. 解决：源文件和所有 day12 noisy 命令都加 `-pix_fmt yuv420p`，重跑后所有文件能正常播放。

教训：**任何 lavfi 生成源、或带 filter 的转码命令，都要显式指定 `-pix_fmt yuv420p`**。否则可能默认输出 yuv444p / yuv422p / 10bit 等专业格式，多数播放器无法显示。

## 我现在能解释什么

- **三种码率控制模式的本质**：
  - CRF：质量恒定，码率浮动 —— 适合点播、归档（你能下完整个文件）。
  - CBR：码率恒定，质量浮动 —— 适合带宽严格固定且要求实时的场景（老式直播推流、硬件传输）。
  - VBR：码率有目标 + 上限，质量在范围内尽量保持 —— 适合点播但有流量约束的场景。
- **`maxrate / bufsize` 描述 HRD**：HRD（Hypothetical Reference Decoder）是 H.264 标准里的"虚拟解码器缓冲区"模型，规定编码器输出的码率波动必须能被某个固定大小的缓冲区吸收，避免解码端缓冲溢出/欠载。maxrate 是峰值速率，bufsize 是缓冲容量，二者共同决定瞬时码率的灵活区间。
- **为什么直播倾向 CRF + maxrate**：大部分时段（静态画面、慢动作）按质量走，画质稳定；遇到场景切换、快速运动需要瞬时高码率时，maxrate 退化为软上限保护上行带宽。这比纯 CBR 更高效（简单段落不浪费），比纯 CRF 更安全（不会把上行打爆）。
- **CBR 在简单内容上会浪费的根因**：编码器必须满足 `output bits / time = target rate`，简单内容下编码器要么提高质量（更小 QP）、要么插入 padding NAL 单元，结果就是文件比 CRF 大很多但画质没增益。
- **VBR 优于 CBR 的根因**：VBR 把瞬时码率自由度还给编码器，复杂帧多分 bit、简单帧少分 bit，同样平均带宽下整体画质更平稳。

## 备忘

- 本日产物保留在 `labs/01-ffmpeg-cli/samples/day12_*.mp4`，共 7 个文件。
- 待 Day 13 把 Day 5（分辨率/码率）、Day 10（帧率）、Day 11（CRF）、Day 12（CBR/VBR）的数据汇总成一张总表。
