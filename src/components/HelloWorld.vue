<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, watchEffect } from 'vue'

type Stage = 'start' | 'typing' | 'game' | 'celebration'
type CellValue = '' | 'X' | 'O'

type ConfettiParticle = {
  x: number
  y: number
  width: number
  height: number
  color: string
  rotation: number
  rotationSpeed: number
  vx: number
  vy: number
  gravity: number
  alpha: number
}

const fullText =
  'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Praesent commodo eros in lectus tincidunt, sit amet condimentum est tincidunt. Vestibulum non nisi eu lacus dignissim facilisis, id luctus nibh luctus.'
const typingDelayMs = 68
const isDevMode = new URLSearchParams(window.location.search).get('dev')?.toLowerCase() === 'true'

const totalGames = 3
const requiredWins = 1

const stage = ref<Stage>('start')
const typedText = ref('')
const isTypingDone = ref(false)

const currentPoints = ref(0)
const currentGame = ref(1)

const board = ref<CellValue[]>(Array(9).fill(''))
const gameOver = ref(false)
const computerThinking = ref(false)
const gameMessage = ref('Your turn. You are X.')
const animatedCells = ref<number[]>([])
const showSadFace = ref(false)
const showIllegalAction = ref(false)

const confettiCanvas = ref<HTMLCanvasElement | null>(null)

const winningLines: number[][] = [
  [0, 1, 2],
  [3, 4, 5],
  [6, 7, 8],
  [0, 3, 6],
  [1, 4, 7],
  [2, 5, 8],
  [0, 4, 8],
  [2, 4, 6],
]

const confettiColors = ['#ff2d95', '#00d9ff', '#ffd400', '#70ff57', '#ff7a00', '#8a5cff']

let typingTimer: number | null = null
let textIndex = 0

let computerMoveTimer: number | null = null
let animationClearTimer: number | null = null
let illegalActionTimer: number | null = null
let audioContext: AudioContext | null = null

let animationFrameId: number | null = null
let cleanupTimerId: number | null = null
let particles: ConfettiParticle[] = []

const stopTyping = () => {
  if (typingTimer !== null) {
    window.clearInterval(typingTimer)
    typingTimer = null
  }
}

const startTyping = () => {
  typingTimer = window.setInterval(() => {
    if (textIndex >= fullText.length) {
      stopTyping()
      isTypingDone.value = true
      return
    }

    typedText.value += fullText[textIndex]
    textIndex += 1
  }, typingDelayMs)
}

const startExperience = () => {
  if (stage.value !== 'start') {
    return
  }

  stage.value = 'typing'
  if (isDevMode) {
    typedText.value = fullText
    isTypingDone.value = true
    stopTyping()
    return
  }

  startTyping()
}

const stopComputerMoveTimer = () => {
  if (computerMoveTimer !== null) {
    window.clearTimeout(computerMoveTimer)
    computerMoveTimer = null
  }
}

const stopAnimationClearTimer = () => {
  if (animationClearTimer !== null) {
    window.clearTimeout(animationClearTimer)
    animationClearTimer = null
  }
}

const stopIllegalActionTimer = () => {
  if (illegalActionTimer !== null) {
    window.clearTimeout(illegalActionTimer)
    illegalActionTimer = null
  }
}

const animateCells = (indices: number[]) => {
  stopAnimationClearTimer()
  animatedCells.value = [...indices]
  animationClearTimer = window.setTimeout(() => {
    animatedCells.value = []
    animationClearTimer = null
  }, 420)
}

const getAudioContext = async () => {
  if (!audioContext) {
    audioContext = new AudioContext()
  }

  if (audioContext.state === 'suspended') {
    await audioContext.resume()
  }

  return audioContext
}

const playLossSound = async () => {
  try {
    const context = await getAudioContext()
    const now = context.currentTime
    const oscillator = context.createOscillator()
    const gain = context.createGain()

    oscillator.type = 'triangle'
    oscillator.frequency.setValueAtTime(440, now)
    oscillator.frequency.exponentialRampToValueAtTime(170, now + 0.38)

    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.exponentialRampToValueAtTime(0.16, now + 0.04)
    gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.42)

    oscillator.connect(gain)
    gain.connect(context.destination)

    oscillator.start(now)
    oscillator.stop(now + 0.43)
  } catch {
    // Intentionally ignore audio playback failures.
  }
}

const playIllegalActionSound = async () => {
  try {
    const context = await getAudioContext()
    const now = context.currentTime
    const oscillator = context.createOscillator()
    const gain = context.createGain()

    oscillator.type = 'square'
    oscillator.frequency.setValueAtTime(620, now)
    oscillator.frequency.exponentialRampToValueAtTime(320, now + 0.14)

    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.exponentialRampToValueAtTime(0.11, now + 0.015)
    gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.18)

    oscillator.connect(gain)
    gain.connect(context.destination)

    oscillator.start(now)
    oscillator.stop(now + 0.2)
  } catch {
    // Intentionally ignore audio playback failures.
  }
}

const showIllegalActionFeedback = () => {
  stopIllegalActionTimer()
  showIllegalAction.value = true
  void playIllegalActionSound()

  illegalActionTimer = window.setTimeout(() => {
    showIllegalAction.value = false
    illegalActionTimer = null
  }, 1050)
}

const handleLoss = (message: string) => {
  gameOver.value = true
  gameMessage.value = message
  showSadFace.value = true
  void playLossSound()
  currentGame.value = Math.min(currentGame.value + 1, totalGames)
}

const resetGame = () => {
  stopComputerMoveTimer()
  stopAnimationClearTimer()
  board.value = Array(9).fill('')
  gameOver.value = false
  computerThinking.value = false
  gameMessage.value = 'Your turn. You are X.'
  animatedCells.value = []
  showSadFace.value = false
  showIllegalAction.value = false
}

const openGamesStage = () => {
  if (stage.value !== 'typing' || !isTypingDone.value) {
    return
  }

  stage.value = 'game'
  resetGame()
}

const findWinningLine = (cells: CellValue[], marker: 'X' | 'O') => {
  return winningLines.find((line) => line.every((index) => cells[index] === marker))
}

const getLineValues = (cells: CellValue[], line: number[]) => line.map((index) => cells[index])

const findSingleFlipWinningMove = (cells: CellValue[]) => {
  for (const line of winningLines) {
    const values = getLineValues(cells, line)
    const oCount = values.filter((value) => value === 'O').length
    const xCount = values.filter((value) => value === 'X').length

    if (oCount === 2 && xCount === 1) {
      const xPosition = values.findIndex((value) => value === 'X')
      return {
        line,
        indexToFlip: line[xPosition],
      }
    }
  }

  return null
}

const isIllegalPlayerMove = (index: number) => {
  const relevantLines = winningLines.filter((line) => line.includes(index))
  return relevantLines.some((line) => {
    const values = getLineValues(board.value, line)
    const xCount = values.filter((value) => value === 'X').length
    const emptyCount = values.filter((value) => value === '').length
    return xCount === 2 && emptyCount === 1
  })
}

const hasAnyLegalPlayerMove = (cells: CellValue[]) => {
  const emptyIndices = cells
    .map((value, index) => (value === '' ? index : -1))
    .filter((index) => index !== -1)

  return emptyIndices.some((index) => {
    const relevantLines = winningLines.filter((line) => line.includes(index))
    return !relevantLines.some((line) => {
      const values = getLineValues(cells, line)
      const xCount = values.filter((value) => value === 'X').length
      const emptyCount = values.filter((value) => value === '').length
      return xCount === 2 && emptyCount === 1
    })
  })
}

const forceComputerImmediateWin = () => {
  const next = [...board.value]
  const bestLine = winningLines
    .map((line) => ({
      line,
      oCount: line.filter((index) => next[index] === 'O').length,
    }))
    .sort((a, b) => b.oCount - a.oCount)[0]?.line ?? winningLines[0]

  for (const index of bestLine) {
    next[index] = 'O'
  }

  board.value = next
  animateCells(bestLine)
  handleLoss('Computer won this round.')
}

const findSetupMoveIndex = (cells: CellValue[]) => {
  const setupPriority = winningLines
    .map((line) => {
      const values = getLineValues(cells, line)
      const oCount = values.filter((value) => value === 'O').length
      const xCount = values.filter((value) => value === 'X').length
      const emptyIndices = line.filter((index) => cells[index] === '')

      return { line, oCount, xCount, emptyIndices }
    })
    .filter((entry) => entry.xCount >= 1 && entry.oCount <= 1 && entry.emptyIndices.length >= 1)
    .sort((a, b) => b.oCount - a.oCount)

  if (setupPriority.length === 0) {
    return null
  }

  return setupPriority[0].emptyIndices[0]
}

const makeFairComputerMove = () => {
  const setupIndex = findSetupMoveIndex(board.value)
  if (setupIndex !== null) {
    board.value[setupIndex] = 'O'
    animateCells([setupIndex])
    return
  }

  const empty = board.value
    .map((value, index) => (value === '' ? index : -1))
    .filter((index) => index !== -1)

  if (empty.length === 0) {
    return
  }

  const center = 4
  const index = board.value[center] === '' ? center : empty[Math.floor(Math.random() * empty.length)]
  board.value[index] = 'O'
  animateCells([index])
}

const makeCheatingWin = () => {
  const singleFlipWin = findSingleFlipWinningMove(board.value)
  if (!singleFlipWin) {
    return false
  }

  const next = [...board.value]
  next[singleFlipWin.indexToFlip] = 'O'
  const computerLine = findWinningLine(next, 'O')

  if (!computerLine) {
    return false
  }

  board.value = next
  animateCells([singleFlipWin.indexToFlip, ...computerLine])
  handleLoss('You were close. Computer wins this round.')
  return true
}

const onCellClick = (index: number) => {
  if (stage.value !== 'game' || gameOver.value || computerThinking.value || board.value[index] !== '') {
    return
  }

  if (isIllegalPlayerMove(index)) {
    showIllegalActionFeedback()
    if (!hasAnyLegalPlayerMove(board.value)) {
      forceComputerImmediateWin()
    }
    return
  }

  showIllegalAction.value = false
  stopIllegalActionTimer()

  board.value[index] = 'X'
  animateCells([index])

  gameMessage.value = 'Computer is thinking...'
  computerThinking.value = true

  computerMoveTimer = window.setTimeout(() => {
    if (stage.value !== 'game' || gameOver.value) {
      computerThinking.value = false
      return
    }

    if (makeCheatingWin()) {
      computerThinking.value = false
      return
    }

    makeFairComputerMove()

    const computerLine = findWinningLine(board.value, 'O')
    if (computerLine) {
      animateCells(computerLine)
      handleLoss('Computer won this round.')
      computerThinking.value = false
      return
    }

    const hasEmptyCells = board.value.some((value) => value === '')
    if (!hasEmptyCells) {
      gameOver.value = true
      gameMessage.value = 'Draw. Try again.'
      currentGame.value = Math.min(currentGame.value + 1, totalGames)
      computerThinking.value = false
      return
    }

    if (!hasAnyLegalPlayerMove(board.value)) {
      forceComputerImmediateWin()
      computerThinking.value = false
      return
    }

    gameMessage.value = 'Your turn. You are X.'
    computerThinking.value = false
  }, 620)
}

const resizeCanvas = () => {
  const canvas = confettiCanvas.value
  if (!canvas) {
    return
  }

  canvas.width = window.innerWidth
  canvas.height = window.innerHeight
}

const spawnParticles = () => {
  const canvas = confettiCanvas.value
  if (!canvas) {
    return
  }

  const centerX = canvas.width / 2
  const centerY = canvas.height * 0.44

  particles = Array.from({ length: 140 }, (_, index) => {
    const angle = (Math.PI * 2 * index) / 140 + Math.random() * 0.2
    const speed = 6 + Math.random() * 10

    return {
      x: centerX,
      y: centerY,
      width: 12 + Math.random() * 18,
      height: 14 + Math.random() * 22,
      color: confettiColors[Math.floor(Math.random() * confettiColors.length)],
      rotation: Math.random() * Math.PI,
      rotationSpeed: (Math.random() - 0.5) * 0.34,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed - (6 + Math.random() * 6),
      gravity: 0.16 + Math.random() * 0.12,
      alpha: 1,
    }
  })
}

const stopConfetti = () => {
  if (animationFrameId !== null) {
    window.cancelAnimationFrame(animationFrameId)
    animationFrameId = null
  }

  if (cleanupTimerId !== null) {
    window.clearTimeout(cleanupTimerId)
    cleanupTimerId = null
  }

  particles = []

  const canvas = confettiCanvas.value
  if (!canvas) {
    return
  }

  const ctx = canvas.getContext('2d')
  ctx?.clearRect(0, 0, canvas.width, canvas.height)
}

const animateConfetti = () => {
  const canvas = confettiCanvas.value
  if (!canvas) {
    return
  }

  const ctx = canvas.getContext('2d')
  if (!ctx) {
    return
  }

  ctx.clearRect(0, 0, canvas.width, canvas.height)

  particles = particles.filter((particle) => particle.alpha > 0)

  for (const particle of particles) {
    particle.vy += particle.gravity
    particle.x += particle.vx
    particle.y += particle.vy
    particle.rotation += particle.rotationSpeed
    particle.alpha -= 0.004

    ctx.save()
    ctx.globalAlpha = Math.max(particle.alpha, 0)
    ctx.translate(particle.x, particle.y)
    ctx.rotate(particle.rotation)
    ctx.fillStyle = particle.color
    ctx.fillRect(-particle.width / 2, -particle.height / 2, particle.width, particle.height)
    ctx.restore()
  }

  if (particles.length > 0) {
    animationFrameId = window.requestAnimationFrame(animateConfetti)
  }
}

const startCelebration = () => {
  stage.value = 'celebration'
  resizeCanvas()
  spawnParticles()
  animateConfetti()

  cleanupTimerId = window.setTimeout(() => {
    stopConfetti()
  }, 3600)
}

watchEffect(() => {
  if (stage.value === 'game' && currentPoints.value >= requiredWins) {
    startCelebration()
  }
})

onMounted(() => {
  window.addEventListener('resize', resizeCanvas)
})

onBeforeUnmount(() => {
  stopTyping()
  stopComputerMoveTimer()
  stopAnimationClearTimer()
  stopIllegalActionTimer()
  stopConfetti()
  window.removeEventListener('resize', resizeCanvas)
  if (audioContext) {
    void audioContext.close()
  }
})
</script>

<template>
  <main class="gift-screen">
    <canvas ref="confettiCanvas" class="confetti-layer" aria-hidden="true"></canvas>

    <section v-if="stage === 'start'" class="typing-stage">
      <button class="proceed-button" type="button" @click="startExperience" @touchstart.passive="startExperience">
        Start
      </button>
    </section>

    <section v-else-if="stage === 'typing'" class="typing-stage" aria-live="polite">
      <p class="typed-message">
        {{ typedText }}
        <span class="cursor" aria-hidden="true">|</span>
      </p>

      <button
        v-if="isTypingDone"
        class="proceed-button"
        type="button"
        @click="openGamesStage"
        @touchstart.passive="openGamesStage"
      >
        Proceed
      </button>
    </section>

    <section v-else-if="stage === 'game'" class="game-stage" aria-label="Mini game challenge">
      <div class="game-layout">
        <article class="game-card">
          <h2>Tic-Tac-Toe</h2>
          <p class="game-message">{{ gameMessage }}</p>

          <div class="board" role="grid" aria-label="Tic tac toe board">
            <button
              v-for="(cell, index) in board"
              :key="index"
              class="cell"
              :class="{ 'cell-animated': animatedCells.includes(index) }"
              type="button"
              :disabled="cell !== '' || gameOver || computerThinking"
              @click="onCellClick(index)"
            >
              {{ cell }}
            </button>
          </div>

          <div class="game-actions">
            <p class="sad-face" :class="{ 'sad-face-visible': showSadFace }" aria-live="assertive">:(</p>
            <p class="illegal-action" :class="{ 'illegal-action-visible': showIllegalAction }">Illegale Aktion</p>
            <button
              class="retry-button"
              :class="{ 'retry-button-visible': gameOver }"
              type="button"
              :disabled="!gameOver"
              :aria-hidden="!gameOver"
              @click="resetGame"
            >
              Try Again
            </button>
          </div>
        </article>

        <aside class="score-card" aria-label="Scoreboard">
          <h3>Score</h3>
          <p><span>Current points</span><strong>{{ currentPoints }}</strong></p>
          <p><span>Games</span><strong>{{ totalGames }}</strong></p>
          <p><span>Required wins</span><strong>{{ requiredWins }}</strong></p>
          <p><span>Current game</span><strong>{{ currentGame }} / {{ totalGames }}</strong></p>
          <small>You need wins to unlock the present. This is game 1.</small>
        </aside>
      </div>
    </section>

    <section v-else class="cake-stage" aria-label="Birthday cake celebration">
      <svg
        class="cake-svg"
        viewBox="0 0 420 420"
        xmlns="http://www.w3.org/2000/svg"
        role="img"
        aria-label="Birthday cake"
      >
        <defs>
          <linearGradient id="cakeBody" x1="0%" y1="0%" x2="0%" y2="100%">
            <stop offset="0%" stop-color="#ffd1dc" />
            <stop offset="100%" stop-color="#ff8fab" />
          </linearGradient>
          <linearGradient id="plate" x1="0%" y1="0%" x2="100%" y2="0%">
            <stop offset="0%" stop-color="#5bc0ff" />
            <stop offset="100%" stop-color="#8f67ff" />
          </linearGradient>
        </defs>

        <ellipse cx="210" cy="338" rx="140" ry="26" fill="url(#plate)" opacity="0.92" />
        <rect x="100" y="150" width="220" height="170" rx="24" fill="url(#cakeBody)" />
        <rect x="100" y="136" width="220" height="34" rx="17" fill="#fff3f7" />

        <path
          d="M110 168 C124 184 136 182 150 168 C164 186 178 186 192 168 C206 186 220 186 234 168 C248 186 262 186 276 168 C290 186 304 186 318 168"
          fill="none"
          stroke="#fff3f7"
          stroke-width="12"
          stroke-linecap="round"
        />

        <rect x="199" y="92" width="22" height="48" rx="10" fill="#ffe66d" />
        <path d="M210 56 C194 74 196 90 210 96 C224 90 226 74 210 56" fill="#ff8a00" />
        <path d="M210 66 C201 78 202 87 210 91 C218 87 219 78 210 66" fill="#fff06a" />
      </svg>
    </section>
  </main>
</template>

<style scoped>
.gift-screen {
  position: relative;
  min-height: 100svh;
  background: #000;
  color: #fff;
  overflow: hidden;
  display: grid;
  place-items: center;
  padding: 24px;
  box-sizing: border-box;
}

.typing-stage,
.cake-stage,
.game-stage {
  position: relative;
  z-index: 2;
  width: min(960px, 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
  container-type: inline-size;
}

.typed-message {
  margin: 0;
  text-align: center;
  font-size: clamp(1.1rem, 1rem + 1.2vw, 1.75rem);
  line-height: 1.65;
  letter-spacing: 0.01em;
  max-width: 34ch;
}

.cursor {
  margin-left: 2px;
  animation: blink 0.8s steps(1) infinite;
}

.proceed-button,
.retry-button {
  margin-top: 36px;
  padding: 14px 28px;
  border-radius: 999px;
  border: 1px solid #fff;
  background: transparent;
  color: #fff;
  font-size: 1rem;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: transform 0.2s ease, background 0.2s ease, color 0.2s ease;
}

.proceed-button:hover,
.proceed-button:focus-visible,
.retry-button:hover,
.retry-button:focus-visible {
  background: #fff;
  color: #000;
  transform: translateY(-1px);
  outline: none;
}

.game-layout {
  display: grid;
  grid-template-columns: 278px 278px;
  gap: 22px;
  width: min(580px, 100%);
  align-items: start;
  justify-content: center;
}

.game-card,
.score-card {
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 16px;
  padding: 20px;
  background: rgba(255, 255, 255, 0.04);
}

.game-card {
  width: 278px;
  min-width: 278px;
  max-width: 278px;
  justify-self: center;
  display: flex;
  flex-direction: column;
}

.game-card h2,
.score-card h3 {
  margin: 0;
  font-size: 1.2rem;
  font-weight: 600;
}

.game-message {
  margin: 12px 0 18px;
  line-height: 1.4;
  min-height: calc(1.4em * 3);
  color: rgba(255, 255, 255, 0.9);
}

.board {
  display: grid;
  width: 100%;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
}

.cell {
  width: 100%;
  aspect-ratio: 1 / 1;
  min-height: 72px;
  border-radius: 14px;
  border: 1px solid rgba(255, 255, 255, 0.35);
  background: rgba(255, 255, 255, 0.06);
  color: #fff;
  font-size: clamp(1.4rem, 2.4vw, 2rem);
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.25s ease, border-color 0.25s ease;
}

.cell:not(:disabled):hover {
  transform: translateY(-2px) scale(1.02);
  border-color: rgba(255, 255, 255, 0.72);
}

.cell-animated {
  animation: cell-pop 0.42s cubic-bezier(0.2, 0.8, 0.3, 1);
  box-shadow: 0 0 28px rgba(255, 255, 255, 0.2);
}

.cell:disabled {
  cursor: not-allowed;
  opacity: 0.9;
}

.game-actions {
  margin-top: 10px;
  min-height: 112px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 6px;
}

.sad-face {
  margin: 0;
  min-height: 2rem;
  font-size: 2rem;
  line-height: 1;
  color: #ff9aa9;
  opacity: 0;
  transform: translateY(-14px) scale(0.75);
}

.retry-button {
  margin-top: 0;
  min-height: 44px;
  opacity: 0;
  visibility: hidden;
  pointer-events: none;
}

.sad-face-visible {
  opacity: 1;
  animation: sad-drop 0.44s ease-out;
  transform: translateY(0) scale(1);
}

.illegal-action {
  margin: 0;
  min-height: 1.3em;
  color: #ffd26b;
  font-size: 1rem;
  letter-spacing: 0.015em;
  opacity: 0;
  transform: translateY(-10px);
}

.illegal-action-visible {
  opacity: 1;
  transform: translateY(0);
  animation: illegal-flash 0.34s ease-out;
}

.retry-button-visible {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
}

.score-card {
  width: 278px;
  min-width: 278px;
  max-width: 278px;
  display: grid;
  gap: 10px;
}

.score-card p {
  margin: 0;
  display: flex;
  justify-content: space-between;
  gap: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.14);
}

.score-card strong {
  font-size: 1.1rem;
}

.score-card small {
  margin-top: 8px;
  color: rgba(255, 255, 255, 0.72);
  line-height: 1.4;
}

.confetti-layer {
  position: fixed;
  inset: 0;
  width: 100%;
  height: 100%;
  z-index: 3;
  pointer-events: none;
}

.cake-svg {
  width: min(440px, 82vw);
  height: auto;
  filter: drop-shadow(0 0 36px rgba(255, 161, 208, 0.45));
}

@keyframes blink {
  0%,
  49% {
    opacity: 1;
  }
  50%,
  100% {
    opacity: 0;
  }
}

@keyframes cell-pop {
  0% {
    transform: scale(0.72) rotate(-10deg);
    filter: brightness(1.3);
  }
  60% {
    transform: scale(1.16) rotate(6deg);
  }
  100% {
    transform: scale(1) rotate(0deg);
    filter: brightness(1);
  }
}

@keyframes sad-drop {
  0% {
    opacity: 0;
    transform: translateY(-14px) scale(0.75);
  }
  100% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes illegal-flash {
  0% {
    opacity: 0;
    transform: translateY(-10px) scale(0.95);
  }
  70% {
    opacity: 1;
    transform: translateY(0) scale(1.03);
  }
  100% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@container (max-width: 760px) {
  .game-layout {
    grid-template-columns: 1fr;
    gap: 16px;
    width: min(300px, 100%);
  }

  .board {
    width: min(100%, 320px);
  }

  .game-card {
    width: 300px;
    min-width: 300px;
    max-width: 300px;
  }

  .score-card {
    width: 300px;
    min-width: 300px;
    max-width: 300px;
    justify-self: center;
  }
}

@container (max-width: 560px) {
  .typed-message {
    max-width: 30ch;
    font-size: clamp(1rem, 4.4vw, 1.35rem);
  }

  .proceed-button {
    margin-top: 28px;
    padding: 11px 22px;
    font-size: 0.95rem;
  }

  .retry-button {
    padding: 11px 22px;
    font-size: 0.95rem;
  }

  .game-card,
  .score-card {
    padding: 16px;
    width: 300px;
    min-width: 300px;
    max-width: 300px;
  }

  .cell {
    border-radius: 12px;
    font-size: clamp(1.3rem, 5.8vw, 1.65rem);
  }

  .game-actions {
    min-height: 102px;
  }
}

@media (max-width: 640px) {
  .gift-screen {
    padding: 18px;
  }
}
</style>
