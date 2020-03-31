<template>
  <div id="app">
    <div class="editor-field">
      <div class="video-wrapper">
        <video
          :src="videoSrc"
          ref="video"
          controls
          @timeupdate="handleTimeUpdate"
          @seeked="handleSeeked"
        ></video>
        <p class="sub-preview">{{ currentSub }}</p>
      </div>
      <div class="sub-wrapper">
        <textarea v-model="rawSubs" ref="textarea"></textarea>
      </div>
    </div>
    <div class="tool-field">
      <div class="tool-row">
        <button @click="handleImportVideo">打开视频</button>
        <input
          type="file"
          ref="inputVideo"
          accept="video/*"
          @change="handleChangeVideoInput"
        />
        <button @click="handleImportTxt">打开文本文档</button>
        <input
          type="file"
          ref="inputText"
          accept="text/plain"
          @change="handleChangeTxtInput"
        />
      </div>
      <div class="tool-row">
        字幕文件名：<input type="text" v-model="filename" />
        <button @click="exportSRT">导出SRT</button>
      </div>
      <div class="tool-row">
        <label>快捷键：</label>
        <span v-for="(item, index) in shortcuts" :key="index" class="shortcut">
          {{ item.label }}:{{ item.keycode }}
        </span>
      </div>
    </div>
  </div>
</template>

<script>
import { saveAs } from "file-saver";

const startTimeReg = /^(\[\d\d:\d\d:\d\d.\d\d\d\])(.*)/;
const endTimeReg = /(.*)(\[\d\d:\d\d:\d\d.\d\d\d\])$/;
const timeTemplate = "[00:00:00.000]";
const timeLength = timeTemplate.length;

export default {
  name: "app",
  data() {
    return {
      rawSubs: "",
      tree: [],
      currentSub: "字幕预览",
      filename: "字幕.srt",
      videoSrc: "",
      video: null,
      textarea: null,
      lineBreak: "\r\n",
      shortcuts: {
        start: {
          keycode: "F1",
          label: "添加开始时间"
        },
        end: {
          keycode: "F2",
          label: "添加结束时间"
        },
        play: {
          keycode: "F3",
          label: "播放"
        },
        pause: {
          keycode: "F4",
          label: "暂停"
        }
      }
    };
  },
  computed: {
    lineBreakLength() {
      return this.lineBreak.length;
    },
    reservedKeys() {
      return Object.values(this.shortcuts).map(item => item.keycode);
    }
  },
  watch: {
    rawSubs: {
      handler(val) {
        this.lineBreak = this.detectLineBreak(val);
        const rawSubLines = val.split(this.lineBreak);
        let pointer = 0;
        const tree = [];
        rawSubLines.forEach(line => {
          const hasStartTime = startTimeReg.test(line);
          const hasEndTime = endTimeReg.test(line);
          let startTime = "",
            endTime = "";

          let rawText = line;
          if (hasStartTime) {
            startTime = rawText.substring(1, timeLength - 1);
            rawText = rawText.substring(timeLength);
          }
          if (hasEndTime) {
            endTime = rawText.substring(
              rawText.length - (timeLength - 1),
              rawText.length - 1
            );
            rawText = rawText.substring(0, rawText.length - timeLength);
          }

          const fullLength = line.length;

          tree.push({
            rawText,
            time: [startTime, endTime],
            position: [pointer, pointer + fullLength]
          });

          pointer += fullLength + 1;
        });
        this.tree = tree;
      },
      immediate: true
    }
  },
  created() {},
  mounted() {
    this.video = this.$refs.video;
    this.textarea = this.$refs.textarea;
    window.addEventListener("keydown", e => {
      if (this.reservedKeys.includes(e.key)) {
        e.preventDefault();
      }
    });
    window.addEventListener("keyup", e => {
      if (this.reservedKeys.includes(e.key)) {
        if (this.textarea === document.activeElement) {
          switch (e.key) {
            case this.shortcuts.start.keycode: {
              const line = this.getThisLine();
              const moveType = line.time[0] ? "keep" : "move";
              const time = this.getTime();
              line.time[0] = time;
              this.render(moveType);
              break;
            }
            case this.shortcuts.end.keycode: {
              const line = this.getThisLine();
              const time = this.getTime();
              line.time[1] = time;
              this.render("next");
              break;
            }
            case this.shortcuts.play.keycode: {
              this.video.play();
              break;
            }
            case this.shortcuts.pause.keycode: {
              this.video.pause();
              break;
            }
          }
        }
      }
    });
  },
  beforeUpdate() {},
  methods: {
    // editor
    getThisLine() {
      const pos = this.textarea.selectionEnd;
      return this.tree.find(
        line => pos >= line.position[0] && pos <= line.position[1]
      );
    },
    getNextLine(pos) {
      let isNext = false;
      let nextLine;
      for (const line of this.tree) {
        if (isNext) {
          nextLine = line;
          break;
        }
        if (pos >= line.position[0] && pos <= line.position[1]) {
          isNext = true;
        }
      }
      return nextLine;
    },
    render(cursorAction) {
      const prevSelectionEnd = this.textarea.selectionEnd;
      let nextSelectionEnd = "";
      let d = "";
      this.tree.forEach(line => {
        const { time, rawText } = line;
        let [startTime, endTime] = time;
        startTime = startTime ? `[${startTime}]` : "";
        endTime = endTime ? `[${endTime}]` : "";
        d += startTime + rawText + endTime + this.lineBreak;
      });
      d = d.substr(0, d.length - this.lineBreakLength);
      this.rawSubs = d;
      this.$nextTick(() => {
        if (cursorAction === "keep") {
          nextSelectionEnd = prevSelectionEnd;
        } else if (cursorAction === "move") {
          nextSelectionEnd = prevSelectionEnd + timeLength;
        } else {
          const nextLine = this.getNextLine(prevSelectionEnd);
          if (nextLine) {
            nextSelectionEnd = nextLine.position[1];
          } else {
            nextSelectionEnd = prevSelectionEnd;
          }
        }
        this.moveCursor(nextSelectionEnd);
      });
    },
    // video
    getTime() {
      const time = this.video.currentTime.toFixed(3);
      let h = parseInt(time / 3600);
      let m = parseInt((time - h * 3600) / 60);
      let s = parseInt(time) % 60;
      const ms = (time + "").split(".")[1];

      h = h >= 10 ? h : `0${h}`;
      m = m >= 10 ? m : `0${m}`;
      s = s >= 10 ? s : `0${s}`;
      return `${h}:${m}:${s}.${ms}`;
    },
    handleTimeUpdate() {
      this.currentSub = this.getCurrentSub();
    },
    handleSeeked() {
      this.currentSub = this.getCurrentSub();
    },
    toVideoTime(formattedTime) {
      let [t, ms] = formattedTime.split(".");
      let [hours, minutes, seconds] = t.split(":");
      ms = parseFloat(ms) / 1000;
      hours = parseInt(hours);
      minutes = parseInt(minutes);
      seconds = parseInt(seconds);
      return hours * 60 * 60 + minutes * 60 + seconds + ms;
    },
    getCurrentSub() {
      const time = this.video.currentTime;
      const currentLine = this.tree.find(line => {
        let [start, end] = line.time;
        start = this.toVideoTime(start);
        end = this.toVideoTime(end);
        if (!isNaN(start) && !isNaN(end)) {
          return time >= start && time <= end;
        } else {
          return time >= start;
        }
      });
      if (currentLine) return currentLine.rawText;
      return "";
    },
    // import
    handleImportVideo() {
      this.$refs.inputVideo.click();
    },
    handleChangeVideoInput(e) {
      const file = e.target.files[0];
      const t = file.name.split(".");
      t.pop();
      const rawFilename = t.join(".");
      const subFilename = rawFilename + ".srt";
      this.filename = subFilename;
      const reader = new FileReader();
      reader.onload = event => {
        this.videoSrc = event.target.result;
      };
      reader.readAsDataURL(file);
    },
    handleImportTxt() {
      this.$refs.inputText.click();
    },
    handleChangeTxtInput(e) {
      const file = e.target.files[0];
      const reader = new FileReader();
      reader.onload = event => {
        this.rawSubs = event.target.result;
      };
      reader.readAsText(file);
    },
    // export
    toSRT() {
      let subs = "";
      this.tree.forEach((line, index) => {
        const { rawText, time } = line;
        let [startTime, endTime] = time;
        startTime = startTime.replace(".", ",");
        endTime = endTime.replace(".", ",");
        subs += `${index + 1}${this.lineBreak}`;
        subs += `${startTime} --> ${endTime}${this.lineBreak}`;
        subs += `${rawText}${this.lineBreak}${this.lineBreak}`;
      });
      return subs;
    },
    exportSRT() {
      const blob = new Blob([this.toSRT()], {
        type: "text/plain;charset=utf-8"
      });
      saveAs(blob, this.filename);
    },
    // utils
    moveCursor(pos) {
      this.textarea.selectionStart = pos;
      this.textarea.selectionEnd = pos;
    },
    detectLineBreak(text) {
      if (!text) return "";
      if (text.split("\r\n")[0] === text) return "\n"; // LF
      return "\r\n"; // CRLF
    }
  }
};
</script>

<style lang="less" scoped>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  .editor-field {
    display: flex;
    width: 100%;
    height: 400px;
    .video-wrapper {
      width: 50%;
      video {
        width: 100%;
        height: calc(100% - 40px);
      }
      p.sub-preview {
        height: 20px;
        line-height: 20px;
        margin: 5px 0;
        font-weight: bold;
        text-align: center;
      }
    }
    .sub-wrapper {
      width: 50%;
    }
  }
  .tool-field {
    text-align: left;
    .tool-row {
      margin: 10px 0;
    }
    span.shortcut {
      margin-right: 10px;
    }
  }
}
textarea {
  box-sizing: border-box;
  width: 100%;
  height: 100%;
  resize: none;
}
input[type="file"] {
  display: none;
}
</style>
