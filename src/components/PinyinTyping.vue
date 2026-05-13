<script setup>
import { ref, computed, onMounted, onUnmounted, watch, nextTick } from 'vue'
import { pinyin } from 'pinyin'
import CharacterBlock from './CharacterBlock.vue'

// 声调字母 -> 基础字母映射
const toneMap = {
  'ā':'a','á':'a','ǎ':'a','à':'a',
  'ē':'e','é':'e','ě':'e','è':'e',
  'ī':'i','í':'i','ǐ':'i','ì':'i',
  'ō':'o','ó':'o','ǒ':'o','ò':'o',
  'ū':'u','ú':'u','ǔ':'u','ù':'u',
  'ǖ':'ü','ǘ':'ü','ǚ':'ü','ǜ':'ü'
}

// 判断是否为标准键盘可输入的字符
const isTypableChar = (char) => {
  // 汉字、中文标点、字母、数字、基础标点 都需要输入
  if (/[一-鿿]/.test(char)) return true
  if (/[a-zA-Z0-9]/.test(char)) return true
  if (/[，。、！？；：""''（）【】《》\-…·]/.test(char)) return true
  return false
}

// 清理上传文本：去除多余空格，识别段落，段前空一格，去除段末空格
const cleanText = (text) => {
  let cleaned = text
    .trim()
    .replace(/[ \t]+/g, ' ')
    .split('\n')
    .map(line => line.trim())
    .join('\n')
    // 合并连续换行为段落分隔（2+换行 = 段落分界）
    .replace(/\n{2,}/g, '\n\n')

  // 按段落分割，每段前加全角空格，用换行连接段落
  const paragraphs = cleaned.split(/\n{2,}/)
  cleaned = paragraphs
    .filter(p => p.trim().length > 0)
    .map(p => '　' + p.replace(/\n/g, ''))
    .join('\n')

  return cleaned
}

// 标点符号到英文标点键的映射
const punctuationMap = {
  '，': ',',
  '。': '.',
  '、': '/',
  '！': '!',
  '？': '?',
  '；': ';',
  '：': ':',
  '“': '"',  // "
  '”': '"',  // "
  '‘': "'",  // '
  '’': "'",  // '
  '（': '(',
  '）': ')',
  '【': '[',
  '】': ']',
  '《': '<',
  '》': '>',
  '—': '-',
  '…': '.',
  '·': '.'
}

// 状态变量
const inputText = ref('春眠不觉晓，处处闻啼鸟。夜来风雨声，花落知多少。')
const characters = ref([])
const currentIndex = ref(0)
const isStarted = ref(false)
const isFinished = ref(false)
const startTime = ref(null)
const elapsedTime = ref(0)
const correctCount = ref(0)
const incorrectCount = ref(0)
const showUpload = ref(false)
const uploadedText = ref('')
const showTemplateSelect = ref(false)
const templateList = ref([])
const loadingTemplate = ref(false)
let timerInterval = null

// 获取拼音
const getPinyin = (char) => {
  if (char === '，' || char === '。' || char === '、' || char === '！' || char === '？') {
    return ''
  }
  const result = pinyin(char, { style: 'tone' })
  return result[0] ? result[0][0] : ''
}

// 将拼音字符串拆分为字母数组
const splitPinyin = (py) => {
  return py.split('').map((letter, i) => ({
    letter,
    base: toneMap[letter] || letter,
    status: i === 0 ? 'active' : 'pending'
  }))
}

// 初始化字符数组
const initCharacters = () => {
  const text = inputText.value
  characters.value = text.split('').map((char, i) => {
    // 换行符：不渲染
    if (char === '\n') {
      return {
        char,
        pinyin: '',
        pinyinLetters: [],
        pinyinIndex: 0,
        status: 'pending',
        hasError: false,
        isParagraphStart: false,
        isLineBreak: true
      }
    }
    // 段首全角空格：显示为空白，自动跳过
    if (char === '　' && (i === 0 || text[i - 1] === '\n')) {
      return {
        char,
        pinyin: '',
        pinyinLetters: [],
        pinyinIndex: 0,
        status: 'pending',
        hasError: false,
        isParagraphStart: true
      }
    }
    // ASCII字母和数字：直接输入对应键
    if (/[a-zA-Z0-9]/.test(char)) {
      return {
        char,
        pinyin: char,
        pinyinLetters: splitPinyin(char),
        pinyinIndex: 0,
        status: 'pending',
        hasError: false,
        isParagraphStart: false
      }
    }
    // 标点符号：映射为英文标点，需要输入
    if (punctuationMap[char]) {
      const mappedKey = punctuationMap[char]
      return {
        char,
        pinyin: mappedKey,
        pinyinLetters: splitPinyin(mappedKey),
        pinyinIndex: 0,
        status: 'pending',
        hasError: false,
        isParagraphStart: false
      }
    }
    // 非标准键盘字符：自动跳过不显示
    if (!isTypableChar(char)) {
      return {
        char,
        pinyin: '',
        pinyinLetters: [],
        pinyinIndex: 0,
        status: 'pending',
        hasError: false,
        isParagraphStart: false
      }
    }
    // 汉字：正常拼音输入
    const py = getPinyin(char)
    return {
      char,
      pinyin: py,
      pinyinLetters: py ? splitPinyin(py) : [],
      pinyinIndex: 0,
      status: 'pending',
      hasError: false,
      isParagraphStart: false
    }
  })
}

// 开始练习
const startPractice = () => {
  if (isStarted.value) return
  isStarted.value = true
  startTime.value = Date.now()
  timerInterval = setInterval(() => {
    elapsedTime.value = Math.floor((Date.now() - startTime.value) / 1000)
  }, 1000)
  // 强制重新渲染，确保光标立即显示
  characters.value.forEach(c => { c.status = 'pending' })
  nextTick(() => {
    if (characters.value.length > 0) {
      characters.value[0].status = 'current'
    }
    scrollToCurrent()
  })
}

// 重置练习
const resetPractice = () => {
  clearInterval(timerInterval)
  isStarted.value = false
  isFinished.value = false
  startTime.value = null
  elapsedTime.value = 0
  correctCount.value = 0
  incorrectCount.value = 0
  currentIndex.value = 0
  initCharacters()
}

// 推进到下一个字符，自动跳过特殊字符
const advanceToNextChar = () => {
  const prev = characters.value[currentIndex.value]
  if (prev) prev.status = 'correct'
  currentIndex.value++
  // 自动跳过段首空白、无拼音的特殊字符
  while (currentIndex.value < characters.value.length) {
    const next = characters.value[currentIndex.value]
    if (next.isParagraphStart || (!next.pinyin && next.pinyinLetters.length === 0)) {
      next.status = 'correct'
      currentIndex.value++
    } else {
      break
    }
  }
  if (currentIndex.value < characters.value.length) {
    characters.value[currentIndex.value].status = 'current'
    nextTick(() => scrollToCurrent())
  } else {
    finishPractice()
  }
}

// 自动滚动到当前光标位置，使其居中显示
const scrollToCurrent = () => {
  const typingArea = document.querySelector('.typing-area')
  const currentEl = document.querySelector('.character-block.current')
  if (!typingArea || !currentEl) return

  const areaRect = typingArea.getBoundingClientRect()
  const elRect = currentEl.getBoundingClientRect()

  // 计算光标元素相对于 typing-area 的位置
  const elOffsetTop = elRect.top - areaRect.top + typingArea.scrollTop
  const targetScroll = elOffsetTop - (areaRect.height / 2) + (elRect.height / 2)

  typingArea.scrollTo({
    top: Math.max(0, targetScroll),
    behavior: 'smooth'
  })
}

// 处理键盘输入
const handleKeydown = (event) => {
  if (!isStarted.value || isFinished.value) return

  const key = event.key
  if (key.length !== 1) return
  event.preventDefault()

  const currentChar = characters.value[currentIndex.value]
  if (!currentChar) return

  // 无拼音的字符：直接跳过
  if (!currentChar.pinyin || currentChar.pinyinLetters.length === 0) {
    advanceToNextChar()
    return
  }

  const letter = currentChar.pinyinLetters[currentChar.pinyinIndex]
  if (!letter) return

  if (key.toLowerCase() === letter.base.toLowerCase()) {
    // 输入正确：标记当前字母为 done，推进 pinyinIndex
    letter.status = 'done'
    currentChar.hasError = false
    correctCount.value++
    currentChar.pinyinIndex++

    // 整个拼音输入完成，推进到下一个字符
    if (currentChar.pinyinIndex >= currentChar.pinyinLetters.length) {
      advanceToNextChar()
    }
  } else {
    // 输入错误
    letter.status = 'error'
    currentChar.hasError = true
    incorrectCount.value++
  }
}

// 完成练习
const finishPractice = () => {
  isFinished.value = true
  clearInterval(timerInterval)
}

// 计算每分钟输入字数
const wpm = computed(() => {
  if (elapsedTime.value === 0) return 0
  return Math.round((correctCount.value / elapsedTime.value) * 60)
})

// 计算正确率
const correctRate = computed(() => {
  const total = correctCount.value + incorrectCount.value
  if (total === 0) return 0
  return Math.round((correctCount.value / total) * 100)
})

// 格式化时间
const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60)
  const secs = seconds % 60
  return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
}

// 处理文件上传
const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (e) => {
      uploadedText.value = cleanText(e.target.result)
      inputText.value = uploadedText.value
      resetPractice()
      showUpload.value = false
    }
    reader.readAsText(file)
  }
}

// 使用上传的文本
const useUploadedText = () => {
  if (uploadedText.value.trim()) {
    uploadedText.value = cleanText(uploadedText.value)
    inputText.value = uploadedText.value
    resetPractice()
    showUpload.value = false
  }
}

// 加载模板列表
const baseUrl = import.meta.env.BASE_URL
const loadTemplateList = async () => {
  try {
    const res = await fetch(`${baseUrl}templates/templates.json`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    templateList.value = await res.json()
  } catch (e) {
    console.error('加载模板列表失败:', e)
    templateList.value = []
  }
}

// 选择并加载模板
const selectTemplate = async (template) => {
  loadingTemplate.value = true
  try {
    const res = await fetch(`${baseUrl}templates/${template.file}`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const text = await res.text()
    uploadedText.value = cleanText(text)
    inputText.value = uploadedText.value
    resetPractice()
    showTemplateSelect.value = false
  } catch (e) {
    console.error('加载模板失败:', e)
  } finally {
    loadingTemplate.value = false
  }
}

// 初始化
onMounted(() => {
  initCharacters()
  loadTemplateList()
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  clearInterval(timerInterval)
  window.removeEventListener('keydown', handleKeydown)
})

// 监听输入文本变化
watch(inputText, () => {
  initCharacters()
})
</script>

<template>
  <div class="container">
    <!-- 标题 -->
    <div class="header">
      <h1>拼音打字练习</h1>
      <p>根据拼音输入对应的汉字</p>
    </div>

    <!-- 控制按钮 -->
    <div class="controls">
      <button @click="startPractice" :disabled="isStarted && !isFinished" class="btn start-btn">
        {{ isStarted && !isFinished ? '练习中...' : '开始练习' }}
      </button>
      <button @click="resetPractice" class="btn reset-btn">结束练习</button>
      <button @click="showUpload = true" class="btn upload-btn">上传文字</button>
      <button @click="showTemplateSelect = true; loadTemplateList()" class="btn template-btn">加载模板</button>
    </div>

    <!-- 打字区域 -->
    <div class="typing-area">
      <div class="characters-container">
        <template v-for="(item, index) in characters" :key="index">
          <div v-if="item.isParagraphStart" class="paragraph-break"></div>
          <CharacterBlock
            v-if="!item.isLineBreak"
            :char="item"
            :is-current="item.status === 'current'"
            :is-paragraph-start="item.isParagraphStart"
          />
        </template>
      </div>
    </div>

    <!-- 上传弹窗 -->
    <div v-if="showUpload" class="modal-overlay" @click.self="showUpload = false">
      <div class="modal">
        <h3>上传文字</h3>
        <p>支持txt、md等文本文件，或直接输入文字</p>
        <input type="file" @change="handleFileUpload" accept=".txt,.md,.text" class="file-input" />
        <textarea
          v-model="uploadedText"
          placeholder="或在此输入文字..."
          class="text-input"
          rows="6"
        ></textarea>
        <div class="modal-buttons">
          <button @click="useUploadedText" class="btn confirm-btn">确认使用</button>
          <button @click="showUpload = false" class="btn cancel-btn">取消</button>
        </div>
      </div>
    </div>

    <!-- 模板选择弹窗 -->
    <div v-if="showTemplateSelect" class="modal-overlay" @click.self="showTemplateSelect = false">
      <div class="modal">
        <h3>选择模板</h3>
        <p>选择一个内置模板开始练习</p>
        <div class="template-list">
          <div v-for="tpl in templateList" :key="tpl.file"
               class="template-item" @click="selectTemplate(tpl)">
            <span class="template-name">{{ tpl.name }}</span>
            <span class="template-desc" v-if="tpl.desc">{{ tpl.desc }}</span>
          </div>
          <div v-if="templateList.length === 0" class="template-empty">暂无模板</div>
        </div>
        <div class="modal-buttons">
          <button @click="showTemplateSelect = false" class="btn cancel-btn">取消</button>
        </div>
      </div>
    </div>

    <!-- 统计信息（底部） -->
    <div class="stats">
      <div v-if="isFinished" class="stat-item stat-finish">
        <span class="stat-label">&nbsp;</span>
        <span class="stat-value finish-text">完成</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">时间</span>
        <span class="stat-value">{{ formatTime(elapsedTime) }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">正确字数</span>
        <span class="stat-value">{{ correctCount }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">错误字数</span>
        <span class="stat-value">{{ incorrectCount }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">速度(字/分)</span>
        <span class="stat-value">{{ wpm }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">正确率</span>
        <span class="stat-value">{{ correctRate }}%</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  height: 100vh;
  box-sizing: border-box;
}

.header {
  text-align: center;
  margin-bottom: 30px;
}

.header h1 {
  color: #333;
  margin-bottom: 10px;
}

.header p {
  color: #666;
  font-size: 16px;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-bottom: 30px;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.3s;
}

.start-btn {
  background-color: #4CAF50;
  color: white;
}

.start-btn:hover:not(:disabled) {
  background-color: #45a049;
}

.start-btn:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.reset-btn {
  background-color: #2196F3;
  color: white;
}

.reset-btn:hover {
  background-color: #1976D2;
}

.upload-btn {
  background-color: #FF9800;
  color: white;
}

.upload-btn:hover {
  background-color: #F57C00;
}

.stats {
  display: flex;
  justify-content: center;
  gap: 30px;
  margin-top: 30px;
  padding: 20px;
  background-color: white;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.stat-item {
  text-align: center;
}

.stat-label {
  display: block;
  color: #666;
  font-size: 14px;
  margin-bottom: 5px;
}

.stat-value {
  display: block;
  color: #333;
  font-size: 24px;
  font-weight: bold;
}

.typing-area {
  flex: 1;
  min-height: 130px;
  overflow-y: auto;
  scroll-behavior: smooth;
  padding: 30px 20px;
  background-color: white;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.typing-area::-webkit-scrollbar {
  width: 6px;
}

.typing-area::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.typing-area::-webkit-scrollbar-thumb {
  background: #ccc;
  border-radius: 3px;
}

.typing-area::-webkit-scrollbar-thumb:hover {
  background: #999;
}

.characters-container {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  justify-content: flex-start;
}

.paragraph-break {
  flex-basis: 100%;
  height: 0;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background-color: white;
  padding: 30px;
  border-radius: 10px;
  width: 400px;
  max-width: 90%;
}

.modal h3 {
  margin-top: 0;
  color: #333;
}

.modal p {
  color: #666;
  font-size: 14px;
  margin-bottom: 20px;
}

.file-input {
  width: 100%;
  margin-bottom: 15px;
}

.text-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
  resize: vertical;
  font-family: inherit;
}

.modal-buttons {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.confirm-btn {
  background-color: #4CAF50;
  color: white;
}

.confirm-btn:hover {
  background-color: #45a049;
}

.cancel-btn {
  background-color: #f44336;
  color: white;
}

.cancel-btn:hover {
  background-color: #d32f2f;
}

.finish-text {
  color: #4CAF50;
  font-size: 20px;
}

.template-btn {
  background-color: #9C27B0;
  color: white;
}

.template-btn:hover {
  background-color: #7B1FA2;
}

.template-list {
  max-height: 300px;
  overflow-y: auto;
}

.template-item {
  padding: 12px 15px;
  border: 1px solid #eee;
  border-radius: 5px;
  margin-bottom: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.template-item:hover {
  background-color: #f5f5f5;
  border-color: #9C27B0;
}

.template-name {
  display: block;
  font-weight: bold;
  color: #333;
}

.template-desc {
  display: block;
  font-size: 13px;
  color: #999;
  margin-top: 4px;
}

.template-empty {
  text-align: center;
  color: #999;
  padding: 20px;
}
</style>
