# Day 3 任务单：容器与编码格式

## 今日目标

Day 3 的核心不是记住很多格式名，而是彻底区分两类概念：

- 容器：`mp4`、`flv`、`ts`
- 编码格式：`h264`、`aac`

今天要通过亲手转换和 `ffprobe` 对比，验证下面这句话：

> 同一份 H.264 + AAC 媒体内容，可以被放进不同容器里。

## 实验输入

使用 Day 2 已生成的样例：

```text
labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
```

如果本地没有这个文件，先执行 Day 2 的生成命令。

## 你要亲手执行的命令

### 1. 把 MP4 重新封装成 FLV

```bash
ffmpeg -y \
  -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c copy \
  labs/01-ffmpeg-cli/samples/day3_remux.flv
```

### 2. 把 MP4 重新封装成 TS

```bash
ffmpeg -y \
  -i labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4 \
  -c copy \
  labs/01-ffmpeg-cli/samples/day3_remux.ts
```

`-c copy` 的含义：不重新编码，只换容器。

## 你要亲手执行的 ffprobe

分别查看三个文件：

```bash
ffprobe -hide_banner labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
ffprobe -hide_banner labs/01-ffmpeg-cli/samples/day3_remux.flv
ffprobe -hide_banner labs/01-ffmpeg-cli/samples/day3_remux.ts
```

再查看完整字段：

```bash
ffprobe -v error -show_format -show_streams labs/01-ffmpeg-cli/samples/day2_testsrc_30s.mp4
ffprobe -v error -show_format -show_streams labs/01-ffmpeg-cli/samples/day3_remux.flv
ffprobe -v error -show_format -show_streams labs/01-ffmpeg-cli/samples/day3_remux.ts
```

## 你要自己填的对比表

执行完成后，自己把表填出来。你可以直接把结果发给我，我帮你校对。

| 文件 | 容器格式 | 视频编码 | 音频编码 | 你观察到的结论 |
|---|---|---|---|---|
| `day2_testsrc_30s.mp4` |  |  |  |  |
| `day3_remux.flv` |  |  |  |  |
| `day3_remux.ts` |  |  |  |  |

## 你必须想明白的 4 个问题

1. 为什么 `.mp4` 不能直接等同于“H.264 视频”？
2. `-c copy` 为什么能把 MP4 变成 FLV / TS？
3. 重新封装之后，视频编码和音频编码是否发生了变化？
4. 如果容器变了但编码没变，这说明容器和编码格式之间是什么关系？

## 今日通过标准

你能不用查资料回答下面这句话，就算 Day 3 通过：

```text
MP4、FLV、TS 是容器。
H.264、AAC 是编码格式。
重新封装只改变容器，不改变编码内容。
```

## 完成后发我什么

把下面任意一种发给我即可：

1. 你自己填好的对比表。
2. 三个文件的 `ffprobe -hide_banner` 输出。
3. 你对上面 4 个问题的回答。

我会据此检查你的理解，并把正式 Day 3 笔记沉淀到仓库文档里。
