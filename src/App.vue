<template>
  <div class="e-whiteboard">
    <!-- 頂部標題區 -->
    <header class="header-settings">
      <div class="input-row serif-text">
        <input v-model="schoolName" class="ghost-input title-part"> 大學 
        <input v-model="department" class="ghost-input title-part"> 系 
        <input v-model="className" class="ghost-input short"> 班
      </div>
    </header>

    <div class="main-layout">
      <!-- 左側：互動功能 -->
      <aside class="side-panel left-panel">
        <div class="panel-header">互動功能</div>
        <div class="menu-list scrollable">
          <div class="menu-item section">
            <div class="tool-title"><span class="icon">🎲</span> 快速抽籤</div>
            <div class="tool-body">
              <div class="draw-setup"><span>人數:</span><input type="number" v-model="studentCount" class="num-input"></div>
              <div class="draw-options">
                <label class="checkbox-label"><input type="checkbox" v-model="excludeDrawn"> 排除重複</label>
                <button v-if="excludeDrawn" @click="drawnNumbers.clear()" class="reset-link">重置</button>
              </div>
              <!-- 抽籤數字顯示區 -->
              <div class="draw-number-display">{{ currentNumber || '--' }}</div>
              <button @click="drawNumber" :disabled="isRolling || (excludeDrawn && drawnNumbers.size >= studentCount)" class="action-btn">
                {{ isRolling ? '抽取中...' : '開始抽籤' }}
              </button>
            </div>
          </div>
          <!-- 分組與獎懲（保持不變） -->
          <div class="menu-item section">
            <div class="tool-title"><span class="icon">👥</span> 分組名單</div>
            <div class="tool-body">
              <button @click="addGroup" class="add-btn">+ 新增小組</button>
              <div v-for="(group, gIdx) in groups" :key="gIdx" class="group-card">
                <div class="group-card-header"><input v-model="group.name" class="group-name-input"><button @click="groups.splice(gIdx, 1)" class="del-btn">✕</button></div>
                <textarea v-model="group.membersRaw" placeholder="組員(逗號隔開)" class="group-textarea"></textarea>
              </div>
            </div>
          </div>
          <div class="menu-item section">
            <div class="tool-title"><span class="icon">🏆</span> 獎懲系統</div>
            <div class="tool-body">
              <div class="mode-tabs">
                <button @click="scoreTarget = 'group'" :class="{ active: scoreTarget === 'group' }">小組</button>
                <button @click="scoreTarget = 'person'" :class="{ active: scoreTarget === 'person' }">個人</button>
              </div>
              <div class="score-list">
                <div v-for="(item, idx) in scoreList" :key="idx" class="score-row">
                  <span class="subject-name">{{ item.name }}</span>
                  <div class="score-btns">
                    <button @click="item.score--" class="score-minus">-</button>
                    <span class="score-val" :class="{ plus: item.score > 0, neg: item.score < 0 }">{{ item.score }}</span>
                    <button @click="item.score++" class="score-plus">+</button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </aside>

      <!-- 中間：核心展示區 -->
      <main class="center-display" :class="displayMode + '-theme'">
        <div class="mode-switcher">
          <button @click="displayMode = 'normal'" :class="{active: displayMode === 'normal'}">常規</button>
          <button @click="displayMode = 'exam'" :class="{active: displayMode === 'exam'}">考試</button>
          <button @click="displayMode = 'quiz'" :class="{active: displayMode === 'quiz'}">測驗</button>
        </div>

        <div class="display-content">
          <!-- A. 常規模式 (優化佈局：時鐘置中偏下) -->
          <div v-if="displayMode === 'normal'" class="clock-view-container">
            <h1 class="top-info-title serif-text">{{ schoolName }}大學 {{ department }}系 {{ className }}班</h1>
            
            <div class="clock-center-box-shifted">
              <div class="time-display-large">{{ currentTime }}</div>
            </div>

            <div class="bottom-info-box-deep">
              <div class="date-display-text">{{ currentDate }}</div>
              <input v-model="currentActivity" class="activity-input-styled serif-text" placeholder="輸入目前活動狀態...">
            </div>
          </div>

          <!-- B. 考試模式 (保持條帶色塊) -->
          <div v-else-if="displayMode === 'exam'" class="exam-view-modern">
            <h1 class="exam-header serif-text">今日考程公告</h1>
            <div class="exam-list-container">
              <div v-for="(ex, idx) in exams" :key="idx" class="exam-stripe serif-text">
                <div class="ex-stripe-subject">{{ ex.subject }}</div>
                <div class="ex-stripe-time">{{ ex.time }}</div>
                <div class="ex-stripe-teacher">監考：{{ ex.teacher }}</div>
              </div>
            </div>
          </div>

          <!-- C. 隨堂測驗模式 (保持 2x2 色塊) -->
          <div v-else class="quiz-view-modern">
            <div v-if="activeQuizIdx !== null" class="quiz-container-inner">
              <div class="quiz-header-box">
                <span class="quiz-badge">{{ quizList[activeQuizIdx].type === 'single' ? '單選題' : '複選題' }}</span>
                <h2 class="quiz-q-text serif-text">{{ quizList[activeQuizIdx].q }}</h2>
              </div>
              <div class="quiz-options-grid">
                <div v-for="(opt, oIdx) in shuffledOptions" :key="oIdx" 
                     class="quiz-opt-card serif-text" :class="{ selected: userAnswers.includes(opt) }"
                     @click="toggleAnswer(opt)">
                  <div class="opt-index">{{ String.fromCharCode(65 + oIdx) }}</div>
                  <div class="opt-content">{{ opt }}</div>
                </div>
              </div>
              <div class="quiz-action-area">
                <button @click="checkQuiz" class="quiz-submit-btn">送出答案並核對</button>
                <div v-if="quizResult" class="quiz-result-banner" :class="quizResult.status">{{ quizResult.msg }}</div>
              </div>
            </div>
            <div v-else class="quiz-empty-state serif-text">請從右側選擇測驗題號</div>
          </div>
        </div>
      </main>

      <!-- 右側：管理區 -->
      <aside class="side-panel right-panel">
        <div class="panel-header">管理系統</div>
        <div class="menu-list scrollable">
          <div class="menu-item section">
            <div class="tool-title"><span class="icon">📝</span> 缺曠紀錄</div>
            <div class="tool-body">
              <div class="attendance-info">應到: {{ studentCount }} | 實到: {{ studentCount - absentList.length }}</div>
              <input v-model="absentInput" class="mini-input" placeholder="輸入座號">
            </div>
          </div>
          <div class="menu-item section">
            <div class="tool-title"><span class="icon">📅</span> 考程編輯</div>
            <div class="tool-body">
              <button @click="addExam" class="add-btn">+ 新增考科</button>
              <div v-for="(ex, idx) in exams" :key="idx" class="exam-edit-card">
                <input v-model="ex.subject" placeholder="科目" class="mini-input">
                <input v-model="ex.time" placeholder="時間" class="mini-input">
                <input v-model="ex.teacher" placeholder="監考老師" class="mini-input">
                <button @click="exams.splice(idx, 1)" class="del-link">移除</button>
              </div>
            </div>
          </div>
          <div class="menu-item section">
            <div class="tool-title"><span class="icon">📖</span> 隨堂測驗列表</div>
            <div class="tool-body quiz-grid-nav">
              <button v-for="(q, idx) in quizList" :key="idx" 
                   class="quiz-nav-btn" :class="{active: activeQuizIdx === idx}"
                   @click="selectQuiz(idx)">
                第 {{ idx + 1 }} 題
              </button>
            </div>
          </div>
        </div>
      </aside>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// 外部套件載入：使用 canvas-confetti 噴彩帶
const loadConfetti = () => {
  return new Promise((resolve) => {
    if (window.confetti) return resolve(window.confetti);
    const script = document.createElement('script');
    script.src = 'https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js';
    script.onload = () => resolve(window.confetti);
    document.head.appendChild(script);
  });
}

const schoolName = ref('淡江'), department = ref('教科'), className = ref('A')
const currentActivity = ref('互動程式設計 課堂中'), displayMode = ref('normal')
const currentDate = ref(''), currentTime = ref('')

const updateTime = () => {
  const now = new Date()
  currentDate.value = now.toLocaleDateString('zh-TW', { year: 'numeric', month: 'long', day: 'numeric', weekday: 'long' })
  currentTime.value = now.toLocaleTimeString('zh-TW', { hour12: false })
}

// 抽籤邏輯 + 彩帶效果
const studentCount = ref(40), currentNumber = ref(''), isRolling = ref(false)
const excludeDrawn = ref(false), drawnNumbers = ref(new Set())

const fireConfetti = async () => {
  const confetti = await loadConfetti();
  // 噴射彩帶效果
  confetti({
    particleCount: 150,
    spread: 70,
    origin: { y: 0.6 },
    colors: ['#d32f2f', '#0288d1', '#388e3c', '#fbc02d', '#7b1fa2']
  });
}

const drawNumber = () => {
  if (excludeDrawn.value && drawnNumbers.value.size >= studentCount.value) return
  isRolling.value = true; 
  let rollCount = 0
  const timer = setInterval(() => {
    currentNumber.value = Math.floor(Math.random() * studentCount.value) + 1
    if (++rollCount > 15) { 
      clearInterval(timer)
      isRolling.value = false
      if (excludeDrawn.value) drawnNumbers.value.add(currentNumber.value)
      // 結束時噴彩帶
      fireConfetti()
    }
  }, 60)
}

// 分組與獎懲邏輯
const groups = ref([{ name: '第1組', membersRaw: '小明, 小華', score: 0 }])
const addGroup = () => groups.value.push({ name: `第${groups.value.length + 1}組`, membersRaw: '', score: 0 })
const scoreTarget = ref('group'), personalScores = ref({})
const scoreList = computed(() => {
  if (scoreTarget.value === 'group') return groups.value
  const list = []
  groups.value.forEach(g => {
    g.membersRaw.split(/[, \n]+/).filter(n => n.trim()).forEach(name => {
      if (personalScores.value[name] === undefined) personalScores.value[name] = 0
      list.push({ name, get score() { return personalScores.value[name] }, set score(v) { personalScores.value[name] = v } })
    })
  })
  return list
})

const exams = ref([
  { subject: '互動程式設計', time: '10:10 - 12:00', teacher: '陳教授' },
  { subject: '教育心理學', time: '13:10 - 15:00', teacher: '李老師' }
])
const addExam = () => exams.value.push({ subject: '', time: '', teacher: '' })
const absentInput = ref(''), absentList = computed(() => absentInput.value.split(',').filter(s => s.trim()))

// 測驗邏輯
const activeQuizIdx = ref(null), userAnswers = ref([]), quizResult = ref(null), shuffledOptions = ref([])
const quizList = ref([
  { type: 'single', q: '誰是目前淡江教科系的系主任?', options: ['蔡森輝老師', '林逸農老師', '賴婷鈴老師', '陳慶帆老師'], ans: ['賴婷鈴老師'] },
  { type: 'multi', q: '淡江大學校園內目前有哪幾班公車?', options: ['紅27', '紅28', '756', '877', '藍67'], ans: ['紅27', '紅28', '756'] },
  { type: 'multi', q: '淡江大學哪裡可以影印需要的資料?', options: ['工館玻璃屋', '商館實習室', '美食廣場', '陳慶帆老師辦公室'], ans: ['工館玻璃屋', '商館實習室'] }
])

const selectQuiz = (idx) => {
  activeQuizIdx.value = idx; userAnswers.value = []; quizResult.value = null
  shuffledOptions.value = [...quizList.value[idx].options].sort(() => Math.random() - 0.5)
  displayMode.value = 'quiz'
}
const toggleAnswer = (opt) => {
  const q = quizList.value[activeQuizIdx.value]
  if (q.type === 'single') userAnswers.value = [opt]
  else {
    const i = userAnswers.value.indexOf(opt); i > -1 ? userAnswers.value.splice(i, 1) : userAnswers.value.push(opt)
  }
}
const checkQuiz = () => {
  const correct = [...quizList.value[activeQuizIdx.value].ans].sort().join(',')
  const user = [...userAnswers.value].sort().join(',')
  quizResult.value = correct === user ? { status: 'correct', msg: '完全正確！' } : { status: 'wrong', msg: '再試一次！' }
}

let timer
onMounted(() => { updateTime(); timer = setInterval(updateTime, 1000); loadConfetti(); })
onUnmounted(() => clearInterval(timer))
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Noto+Serif+TC:wght@500;700;900&display=swap');

:root { --primary-bg: #eceff1; --panel-bg: #ffffff; --accent: #37474f; --quiz-blue: #0288d1; }
* { box-sizing: border-box; }
body { margin: 0; background: var(--primary-bg); font-family: 'Noto Serif TC', serif; overflow: hidden; }
.serif-text { font-family: 'Noto Serif TC', serif !important; }

/* 基礎佈局 */
.header-settings { background: white; padding: 12px; text-align: center; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
.ghost-input { background: transparent; border: none; border-bottom: 1px dashed #999; font-size: 1.1rem; text-align: center; width: 80px;}
.main-layout { display: flex; flex: 1; padding: 15px; gap: 15px; height: calc(100vh - 60px); }
.center-display { flex: 2.5; background: white; border-radius: 16px; position: relative; padding: 40px; border: 2px solid #cfd8dc; display: flex; flex-direction: column; }

/* 常規模式佈局調整：時鐘置中偏下 */
.clock-view-container { display: flex; flex-direction: column; height: 100%; width: 100%; }
.top-info-title { margin-top: 0; font-size: 1.8rem; color: #455a64; font-weight: 700; text-align: center; }
.clock-center-box-shifted { flex: 1; display: flex; align-items: center; justify-content: center; padding-top: 40px; }
.time-display-large { font-size: clamp(3rem, 12vw, 6rem); color: #d32f2f; font-weight: 900; font-family: 'Courier New', monospace; }
.bottom-info-box-deep { width: 100%; text-align: center; display: flex; flex-direction: column; gap: 12px; margin-bottom: 10px; }
.date-display-text { font-size: 1.5rem; color: #78909c; }
.activity-input-styled { width: 80%; margin: 0 auto; font-size: 1.3rem; text-align: center; border: none; border-bottom: 2px solid #eee; padding: 5px; }

/* 抽籤樣式 */
.draw-number-display { font-size: 4rem; font-weight: 900; color: var(--accent); margin: 15px 0; text-align: center; background: #f8f9fa; border-radius: 12px; padding: 10px; border: 2px dashed #ccc; }
.action-btn { width: 100%; padding: 12px; background: var(--accent); color: white; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; }
.action-btn:disabled { opacity: 0.6; cursor: not-allowed; }

/* 測驗 2x2 格點 */
.quiz-options-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
.quiz-opt-card { display: flex; align-items: center; background: white; border: 2px solid #eee; padding: 20px; border-radius: 12px; cursor: pointer; }
.quiz-opt-card.selected { border-color: var(--quiz-blue); background: #e1f5fe; }
.opt-index { width: 30px; height: 30px; background: #eee; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin-right: 15px; font-weight: bold; }
.selected .opt-index { background: var(--quiz-blue); color: white; }

/* 考試條帶 */
.exam-stripe { display: flex; align-items: center; justify-content: space-between; background: #37474f; color: white; padding: 20px 30px; border-radius: 8px; margin-bottom: 15px; }

/* 側邊欄與通用 */  
.side-panel { width: 320px; background: white; border-radius: 16px; display: flex; flex-direction: column; }
.panel-header { background: var(--accent); color: white; padding: 15px; text-align: center; font-weight: bold; }
.scrollable { overflow-y: auto; flex: 1; padding: 15px; }
.section { border: 1px solid #f0f0f0; border-radius: 10px; padding: 12px; margin-bottom: 15px; }
.mode-switcher { position: absolute; top: 15px; right: 15px; z-index: 10; }
.mode-switcher button { margin-left: 5px; padding: 5px 10px; border-radius: 15px; border: 1px solid #ddd; cursor: pointer; }
.mode-switcher button.active { background: var(--accent); color: white; }
</style> 