<script setup>
defineProps({
  char: {
    type: Object,
    required: true
  },
  isCurrent: {
    type: Boolean,
    default: false
  },
  isParagraphStart: {
    type: Boolean,
    default: false
  }
})
</script>

<template>
  <div class="character-block" :class="{ current: isCurrent }">
    <template v-if="isParagraphStart">
      <!-- 段首空白：只显示占位宽度 -->
      <div class="pinyin-row paragraph-space"></div>
      <div class="cross-box paragraph-space-box">
        <div class="char">&nbsp;</div>
      </div>
    </template>
    <template v-else>
      <!-- 拼音行：带五线谱背景 -->
      <div class="pinyin-row">
        <div class="pinyin-staff">
          <div class="staff-line" v-for="n in 5" :key="n"></div>
        </div>
        <div class="pinyin-content">
          <template v-for="(letter, li) in char.pinyinLetters" :key="li">
            <span class="pinyin-letter" :class="letter.status">{{ letter.letter }}</span>
            <span v-if="isCurrent && li === char.pinyinIndex" class="cursor">|</span>
          </template>
        </div>
      </div>
      <!-- 十字框 -->
      <div class="cross-box" :class="[char.status, { 'has-error': char.hasError }]">
        <div class="horizontal-line"></div>
        <div class="vertical-line"></div>
        <div class="char" :class="[char.status, { 'has-error': char.hasError }]">{{ char.char }}</div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.character-block {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 4px;
}

/* 拼音行容器 */
.pinyin-row {
  position: relative;
  min-width: 60px;
  height: 28px;
  margin-bottom: 2px;
}

/* 五线谱背景 */
.pinyin-staff {
  position: absolute;
  top: 0;
  left: -4px;
  right: -4px;
  bottom: 0;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 2px 0;
}

.staff-line {
  height: 1px;
  background-color: #ccc;
  opacity: 0.5;
}

/* 拼音内容层 */
.pinyin-content {
  position: relative;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  gap: 1px;
}

/* 拼音字母 */
.pinyin-letter {
  font-size: 18px;
  line-height: 24px;
  transition: color 0.15s;
  color: #333;
}

.pinyin-letter.active {
  color: #4CAF50;
  font-weight: bold;
}

.pinyin-letter.done {
  color: #999;
}

.pinyin-letter.error {
  color: #f44336;
}

/* 所有尚未输入的字母显示为绿色 */
.pinyin-letter.pending {
  color: #4CAF50;
}

/* 光标 */
.cursor {
  display: inline-block;
  color: #4CAF50;
  font-weight: bold;
  font-size: 18px;
  line-height: 24px;
  animation: blink 1s step-end infinite;
}

@keyframes blink {
  50% { opacity: 0; }
}

/* 十字框 */
.cross-box {
  position: relative;
  width: 56px;
  height: 56px;
  border: 2px solid #333;
  display: flex;
  align-items: center;
  justify-content: center;
}

.cross-box.current {
  border-color: #4CAF50;
  box-shadow: 0 0 8px rgba(76, 175, 80, 0.3);
}

.horizontal-line,
.vertical-line {
  position: absolute;
  background-color: #eee;
}

.horizontal-line {
  width: 100%;
  height: 1px;
  top: 50%;
  left: 0;
}

.vertical-line {
  width: 1px;
  height: 100%;
  top: 0;
  left: 50%;
}

.char {
  font-size: 26px;
  font-weight: bold;
  color: #333;
  z-index: 1;
}

.char.current {
  color: #4CAF50;
}

.char.current.has-error {
  color: #f44336;
}

.char.correct {
  color: #999;
}

/* 段首空白占位 */
.paragraph-space {
  min-width: 28px;
}

.paragraph-space-box {
  border: none !important;
  box-shadow: none !important;
}
</style>
