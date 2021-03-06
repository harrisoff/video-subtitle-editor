
# Video-subtitle

不需要安装任何软件，在网页上编辑视频字幕。

目前仅支持 SRT 格式。

[在线 demo](https://harrisoff.github.io/video-subtitle-editor)

## 安装

1. `clone` 项目
2. `npm i`
3. `npm run serve`

## 使用

1. 点击“打开视频”添加视频

2. 添加文本

   - 复制文本到右侧文本框
   - 或点击“打开文本文档”，仅支持 `.txt` 文件

   **某行字幕为空时，需要添加一个空行**

3. 添加时间轴

   选中编辑框，然后可以使用以下快捷键：

   - F1: 设置该行字幕开始时间
   - F2: 设置该行字幕结束时间
   - F3: 播放视频
   - F4: 暂停视频

   视频下方显示当前字幕

4. 点击“导出 SRT”，下载编辑框中的文本，并保存为 .srt 格式

## 开发

把 `<textarea />` 中的每一行解析为一个对象，结构如下：

```ts
// 字幕的其中一行
interface SubLine {
  // 字幕文本
  rawText: string;
  // 字幕开始时间和结束时间，格式为 hh:mm:ss.SSS
  time: string[];
  // 该行字幕文本在整个 <textarea /> 文本中的 selectionStart 和 selectionEnd 值
  position: number[];
}
```

监听文本的变化，实时更新对应的对象。

有两种操作会对文本产生影响：

1. 直接编辑 `<textarea />`

   文本发生变化，这时只更新数据对象。

2. 点击 F1/F2 为当前行插入时间标签

   实际操作的是数据对象。从数据对象中找到当前行，修改后再使用新的数据对象更新 `<textarea />` 的内容。
