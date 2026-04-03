<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'

type Stage = 'start' | 'typing' | 'game' | 'rps' | 'celebration'
type CellValue = '' | 'X' | 'O'
type RpsChoice = 'rock' | 'paper' | 'scissors'
type RpsOutcome = 'user' | 'computer' | 'draw'
type RpsFlash = 'none' | 'win' | 'lose'

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
  'Alles Gute zum Geburtstag, Janalein! ❤️\nLeider hat ein böser Computer dein Geschenk geklaut... Besiege ihn in mindestens einmal in den folgenden Spielen, um es zurückzuholen!'
const typingDelayMs = 68
const isDevMode = new URLSearchParams(window.location.search).get('dev')?.toLowerCase() === 'true'

const totalGames = 3
const requiredWins = 1
const tttLossTarget = 3
const effectiveTttLossTarget = isDevMode ? 1 : tttLossTarget

const stage = ref<Stage>('start')
const typedText = ref('')
const isTypingDone = ref(false)

const currentGame = ref(1)
const tttLosses = ref(0)
const showTttDefeatModal = ref(false)
const showRpsWinModal = ref(false)

const board = ref<CellValue[]>(Array(9).fill(''))
const gameOver = ref(false)
const computerThinking = ref(false)
const gameMessage = ref('Dein Zug.')
const animatedCells = ref<number[]>([])
const showSadFace = ref(false)
const showIllegalAction = ref(false)

const rpsChoices: RpsChoice[] = ['rock', 'paper', 'scissors']
const rpsChoiceLabels: Record<RpsChoice, string> = {
  rock: 'Stein',
  paper: 'Papier',
  scissors: 'Schere',
}
const rpsChoiceIcons: Record<RpsChoice, string> = {
  rock: '🪨',
  paper: '📄',
  scissors: '✂️',
}
const rpsUserWins = ref(0)
const rpsComputerWins = ref(0)
const rpsDraws = ref(0)
const rpsThinking = ref(false)
const rpsThinkingStep = ref(0)
const rpsMessage = ref('Wähle Stein, Papier oder Schere.')
const rpsUserChoice = ref<RpsChoice | null>(null)
const rpsComputerChoice = ref<RpsChoice | null>(null)
const rpsRoundFlash = ref<RpsFlash>('none')

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
let rpsThinkingInterval: number | null = null
let rpsRevealTimer: number | null = null
let rpsFlashTimer: number | null = null
let tttDefeatModalTimer: number | null = null
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

const stopRpsTimers = () => {
  if (rpsThinkingInterval !== null) {
    window.clearInterval(rpsThinkingInterval)
    rpsThinkingInterval = null
  }
  if (rpsRevealTimer !== null) {
    window.clearTimeout(rpsRevealTimer)
    rpsRevealTimer = null
  }
}

const stopRpsFlashTimer = () => {
  if (rpsFlashTimer !== null) {
    window.clearTimeout(rpsFlashTimer)
    rpsFlashTimer = null
  }
}

const stopTttDefeatModalTimer = () => {
  if (tttDefeatModalTimer !== null) {
    window.clearTimeout(tttDefeatModalTimer)
    tttDefeatModalTimer = null
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

const playRpsWinSound = async () => {
  try {
    const context = await getAudioContext()
    const now = context.currentTime
    const oscillator = context.createOscillator()
    const gain = context.createGain()

    oscillator.type = 'sine'
    oscillator.frequency.setValueAtTime(420, now)
    oscillator.frequency.exponentialRampToValueAtTime(780, now + 0.2)

    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.exponentialRampToValueAtTime(0.12, now + 0.03)
    gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.24)

    oscillator.connect(gain)
    gain.connect(context.destination)
    oscillator.start(now)
    oscillator.stop(now + 0.26)
  } catch {
    // Intentionally ignore audio playback failures.
  }
}

const playRpsLoseSound = async () => {
  try {
    const context = await getAudioContext()
    const now = context.currentTime
    const oscillator = context.createOscillator()
    const gain = context.createGain()

    oscillator.type = 'triangle'
    oscillator.frequency.setValueAtTime(580, now)
    oscillator.frequency.exponentialRampToValueAtTime(210, now + 0.25)

    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.exponentialRampToValueAtTime(0.12, now + 0.03)
    gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.28)

    oscillator.connect(gain)
    gain.connect(context.destination)
    oscillator.start(now)
    oscillator.stop(now + 0.3)
  } catch {
    // Intentionally ignore audio playback failures.
  }
}

const playRpsVictoryCheer = async () => {
  try {
    const context = await getAudioContext()
    const now = context.currentTime

    // Cheer-like rising tone.
    const cheerOsc = context.createOscillator()
    const cheerGain = context.createGain()
    cheerOsc.type = 'sawtooth'
    cheerOsc.frequency.setValueAtTime(320, now)
    cheerOsc.frequency.exponentialRampToValueAtTime(760, now + 0.42)
    cheerGain.gain.setValueAtTime(0.0001, now)
    cheerGain.gain.exponentialRampToValueAtTime(0.09, now + 0.04)
    cheerGain.gain.exponentialRampToValueAtTime(0.0001, now + 0.46)
    cheerOsc.connect(cheerGain)
    cheerGain.connect(context.destination)
    cheerOsc.start(now)
    cheerOsc.stop(now + 0.48)

    // Short clap-like noise bursts.
    const burstDur = 0.05
    const clapTimes = [0.08, 0.18, 0.28, 0.38, 0.5]
    for (const offset of clapTimes) {
      const start = now + offset
      const frameCount = Math.floor(context.sampleRate * burstDur)
      const buffer = context.createBuffer(1, frameCount, context.sampleRate)
      const data = buffer.getChannelData(0)
      for (let i = 0; i < frameCount; i += 1) {
        data[i] = (Math.random() * 2 - 1) * (1 - i / frameCount)
      }

      const noise = context.createBufferSource()
      const noiseGain = context.createGain()
      noise.buffer = buffer
      noiseGain.gain.setValueAtTime(0.0001, start)
      noiseGain.gain.exponentialRampToValueAtTime(0.1, start + 0.005)
      noiseGain.gain.exponentialRampToValueAtTime(0.0001, start + burstDur)
      noise.connect(noiseGain)
      noiseGain.connect(context.destination)
      noise.start(start)
      noise.stop(start + burstDur)
    }
  } catch {
    // Intentionally ignore audio playback failures.
  }
}

const playTttDefeatCrowdSadSound = async () => {
  try {
    const context = await getAudioContext()
    const now = context.currentTime

    // Low "crowd sadness" bed.
    const crowdOsc = context.createOscillator()
    const crowdGain = context.createGain()
    crowdOsc.type = 'sawtooth'
    crowdOsc.frequency.setValueAtTime(180, now)
    crowdOsc.frequency.exponentialRampToValueAtTime(120, now + 0.7)
    crowdGain.gain.setValueAtTime(0.0001, now)
    crowdGain.gain.exponentialRampToValueAtTime(0.07, now + 0.05)
    crowdGain.gain.exponentialRampToValueAtTime(0.0001, now + 0.75)
    crowdOsc.connect(crowdGain)
    crowdGain.connect(context.destination)
    crowdOsc.start(now)
    crowdOsc.stop(now + 0.78)

    // A few soft descending "ohhh" tones.
    const offsets = [0.08, 0.22, 0.36]
    for (const offset of offsets) {
      const start = now + offset
      const osc = context.createOscillator()
      const gain = context.createGain()
      osc.type = 'triangle'
      osc.frequency.setValueAtTime(320, start)
      osc.frequency.exponentialRampToValueAtTime(210, start + 0.24)
      gain.gain.setValueAtTime(0.0001, start)
      gain.gain.exponentialRampToValueAtTime(0.045, start + 0.03)
      gain.gain.exponentialRampToValueAtTime(0.0001, start + 0.26)
      osc.connect(gain)
      gain.connect(context.destination)
      osc.start(start)
      osc.stop(start + 0.27)
    }
  } catch {
    // Intentionally ignore audio playback failures.
  }
}

const triggerRpsRoundFlash = (flash: RpsFlash) => {
  stopRpsFlashTimer()
  rpsRoundFlash.value = flash
  if (flash === 'none') {
    return
  }
  rpsFlashTimer = window.setTimeout(() => {
    rpsRoundFlash.value = 'none'
    rpsFlashTimer = null
  }, 620)
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
  tttLosses.value = Math.min(tttLosses.value + 1, effectiveTttLossTarget)
  if (tttLosses.value >= effectiveTttLossTarget) {
    stopTttDefeatModalTimer()
    tttDefeatModalTimer = window.setTimeout(() => {
      showTttDefeatModal.value = true
      void playTttDefeatCrowdSadSound()
      tttDefeatModalTimer = null
    }, 1800)
  }
}

const resetGame = () => {
  stopComputerMoveTimer()
  stopAnimationClearTimer()
  board.value = Array(9).fill('')
  gameOver.value = false
  computerThinking.value = false
  gameMessage.value = 'Du bist dran. Du bist X.'
  animatedCells.value = []
  showSadFace.value = false
  showIllegalAction.value = false
}

const openGamesStage = () => {
  if (stage.value !== 'typing' || !isTypingDone.value) {
    return
  }

  stage.value = 'game'
  tttLosses.value = 0
  showTttDefeatModal.value = false
  showRpsWinModal.value = false
  stopTttDefeatModalTimer()
  currentGame.value = 1
  resetGame()
}

const continueAfterTttDefeat = () => {
  stopTttDefeatModalTimer()
  showTttDefeatModal.value = false
  stage.value = 'rps'
  rpsUserWins.value = 0
  rpsComputerWins.value = 0
  rpsDraws.value = 0
  rpsThinking.value = false
  rpsThinkingStep.value = 0
  rpsUserChoice.value = null
  rpsComputerChoice.value = null
  rpsMessage.value = 'Wähle Stein, Papier oder Schere.'
  rpsRoundFlash.value = 'none'
  stopRpsFlashTimer()
}

const rpsOutcomePools = (computerWins: number): RpsOutcome[] =>
  computerWins < 2 ? ['draw', 'computer'] : ['draw', 'user']

const pickComputerChoice = (userChoice: RpsChoice, outcome: RpsOutcome): RpsChoice => {
  if (outcome === 'draw') {
    return userChoice
  }
  if (outcome === 'computer') {
    if (userChoice === 'rock') return 'paper'
    if (userChoice === 'paper') return 'scissors'
    return 'rock'
  }
  if (userChoice === 'rock') return 'scissors'
  if (userChoice === 'paper') return 'rock'
  return 'paper'
}

const onRpsChoice = (choice: RpsChoice) => {
  if (stage.value !== 'rps' || rpsThinking.value || showRpsWinModal.value) {
    return
  }

  rpsUserChoice.value = choice
  rpsComputerChoice.value = null
  rpsMessage.value = 'Computer denkt nach...'
  rpsThinking.value = true
  rpsThinkingStep.value = 0
  stopRpsTimers()

  rpsThinkingInterval = window.setInterval(() => {
    rpsThinkingStep.value = (rpsThinkingStep.value + 1) % rpsChoices.length
  }, 130)

  rpsRevealTimer = window.setTimeout(() => {
    stopRpsTimers()
    const pool = rpsOutcomePools(rpsComputerWins.value)
    const outcome = pool[Math.floor(Math.random() * pool.length)]
    const computerChoice = pickComputerChoice(choice, outcome)
    rpsComputerChoice.value = computerChoice

    if (outcome === 'user') {
      rpsUserWins.value += 1
      rpsMessage.value = `Du gewinnst diese Runde! (${rpsChoiceLabels[choice]} schlägt ${rpsChoiceLabels[computerChoice]})`
      triggerRpsRoundFlash('win')
      void playRpsWinSound()
    } else if (outcome === 'computer') {
      rpsComputerWins.value += 1
      rpsMessage.value = `Computer gewinnt diese Runde. (${rpsChoiceLabels[computerChoice]} schlägt ${rpsChoiceLabels[choice]})`
      triggerRpsRoundFlash('lose')
      void playRpsLoseSound()
    } else {
      rpsDraws.value += 1
      rpsMessage.value = `Unentschieden. Beide haben ${rpsChoiceLabels[choice]} gewählt.`
      triggerRpsRoundFlash('none')
    }

    if (rpsUserWins.value >= 3) {
      showRpsWinModal.value = true
      void playRpsVictoryCheer()
    }

    rpsThinking.value = false
  }, 980)
}

const openGiftFromRps = () => {
  showRpsWinModal.value = false
  startCelebration()
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
  handleLoss('Computer gewinnt diese Runde.')
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
  handleLoss('Knapp daneben. Computer gewinnt diese Runde.')
  return true
}

const onCellClick = (index: number) => {
  if (
    stage.value !== 'game' ||
    showTttDefeatModal.value ||
    gameOver.value ||
    computerThinking.value ||
    board.value[index] !== ''
  ) {
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

  gameMessage.value = 'Computer denkt nach...'
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
      handleLoss('Computer gewinnt diese Runde.')
      computerThinking.value = false
      return
    }

    const hasEmptyCells = board.value.some((value) => value === '')
    if (!hasEmptyCells) {
      gameOver.value = true
      gameMessage.value = 'Unentschieden. Bitte noch einmal.'
      currentGame.value = Math.min(currentGame.value + 1, totalGames)
      computerThinking.value = false
      return
    }

    if (!hasAnyLegalPlayerMove(board.value)) {
      forceComputerImmediateWin()
      computerThinking.value = false
      return
    }

    gameMessage.value = 'Dein Zug.'
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

onMounted(() => {
  window.addEventListener('resize', resizeCanvas)
})

onBeforeUnmount(() => {
  stopTyping()
  stopComputerMoveTimer()
  stopAnimationClearTimer()
  stopIllegalActionTimer()
  stopRpsTimers()
  stopRpsFlashTimer()
  stopTttDefeatModalTimer()
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
        Weiter
      </button>
    </section>

    <section v-else-if="stage === 'game'" class="game-stage" aria-label="Mini-Spiel-Herausforderung">
      <div class="game-layout">
        <article class="game-card">
          <h2>Tic-Tac-Toe</h2>
          <p class="game-message">{{ gameMessage }}</p>

          <div class="board" role="grid" aria-label="Tic-Tac-Toe Spielfeld">
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
              :class="{ 'retry-button-visible': gameOver && tttLosses < effectiveTttLossTarget && !showTttDefeatModal }"
              type="button"
              :disabled="!gameOver || showTttDefeatModal || tttLosses >= effectiveTttLossTarget"
              :aria-hidden="!gameOver || showTttDefeatModal || tttLosses >= effectiveTttLossTarget"
              @click="resetGame"
            >
              Erneut versuchen
            </button>
          </div>
        </article>

        <aside class="score-card" aria-label="Punktestand">
          <h3>Punktestand</h3>
          <p><span>Spiele</span><strong>{{ totalGames }}</strong></p>
          <p><span>Benötigte Siege</span><strong>{{ requiredWins }}</strong></p>
          <p><span>Spiel</span><strong>{{ currentGame }} / {{ totalGames }}</strong></p>
          <small>Du brauchst nur 1 Sieg um dein Geschenk zu retten! Du hast 3 Versuche.</small>
        </aside>
      </div>
    </section>

    <section v-else-if="stage === 'rps'" class="game-stage" aria-label="Schere Stein Papier Herausforderung">
      <div class="game-layout">
        <article class="game-card rps-card">
          <h2>Schere Stein Papier</h2>
          <p class="game-message">{{ rpsMessage }}</p>

          <div class="rps-choice-grid" role="group" aria-label="Schere Stein Papier Auswahl">
            <button
              v-for="choice in rpsChoices"
              :key="choice"
              class="choice-button"
              type="button"
              :aria-label="rpsChoiceLabels[choice]"
              :disabled="rpsThinking || showRpsWinModal"
              @click="onRpsChoice(choice)"
            >
              <span class="choice-icon" aria-hidden="true">{{ rpsChoiceIcons[choice] }}</span>
            </button>
          </div>

          <div
            class="rps-live-score"
            :class="{ 'rps-live-score-win': rpsRoundFlash === 'win', 'rps-live-score-lose': rpsRoundFlash === 'lose' }"
            aria-label="Aktueller Spielstand"
          >
            <span class="live-team">Du</span>
            <strong class="live-score">{{ rpsUserWins }} : {{ rpsComputerWins }}</strong>
            <span class="live-team">Computer</span>
          </div>

          <div class="rps-duel" aria-live="polite">
            <div class="duel-side">
              <span class="duel-name">Du</span>
              <strong class="duel-icon" aria-label="Deine Wahl">
                {{ rpsUserChoice ? rpsChoiceIcons[rpsUserChoice] : '◌' }}
              </strong>
            </div>
            <span class="duel-vs">gegen</span>
            <div class="duel-side">
              <span class="duel-name">Computer</span>
              <strong class="duel-icon" aria-label="Computer Wahl">
                {{ rpsThinking ? rpsChoiceIcons[rpsChoices[rpsThinkingStep]] : rpsComputerChoice ? rpsChoiceIcons[rpsComputerChoice] : '◌' }}
              </strong>
            </div>
          </div>
        </article>

        <aside class="score-card" aria-label="RPS Punktestand">
          <h3>Punktestand</h3>
          <p><span>Spiel</span><strong>RPS</strong></p>
          <p><span>Deine Siege</span><strong>{{ rpsUserWins }} / 3</strong></p>
          <p><span>Computer</span><strong>{{ rpsComputerWins }}</strong></p>
          <p><span>Unentschieden</span><strong>{{ rpsDraws }}</strong></p>
          <small>Erreiche 3 Siege, um dein Geschenk zu öffnen.</small>
        </aside>
      </div>
    </section>

    <section v-else class="cake-stage" aria-label="Geburtstagskuchen Feier">
      <svg
        class="cake-svg"
        viewBox="0 0 420 420"
        xmlns="http://www.w3.org/2000/svg"
        role="img"
        aria-label="Geburtstagskuchen"
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

    <div v-if="showTttDefeatModal" class="modal-backdrop" role="dialog" aria-modal="true">
      <div class="modal-jumbotron">
        <h3>Schade, verloren. Probiere dein Glück beim nächsten Spiel!</h3>
        <button class="modal-button" type="button" @click="continueAfterTttDefeat">Weiter</button>
      </div>
    </div>

    <div v-if="showRpsWinModal" class="modal-backdrop" role="dialog" aria-modal="true">
      <div class="modal-jumbotron">
        <h3>Mega! Ein verdienter Sieg! Öffne jetzt dein Geschenk!</h3>
        <button class="modal-button" type="button" @click="openGiftFromRps">Öffnen</button>
      </div>
    </div>
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
  white-space: pre-line;
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

.rps-card {
  gap: 14px;
}

.rps-choice-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}

.choice-button {
  border: 1px solid rgba(255, 255, 255, 0.4);
  background: rgba(255, 255, 255, 0.06);
  color: #fff;
  border-radius: 12px;
  aspect-ratio: 1 / 1;
  min-height: 62px;
  cursor: pointer;
  transition: transform 0.2s ease, border-color 0.2s ease;
  display: grid;
  place-items: center;
}

.choice-button:hover,
.choice-button:focus-visible {
  border-color: #fff;
  transform: translateY(-1px);
  outline: none;
}

.choice-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.choice-icon {
  font-size: 1.8rem;
  line-height: 1;
}

.rps-live-score {
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.08);
  padding: 8px 10px;
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  width: 100%;
  gap: 8px;
}

.rps-live-score-win {
  animation: rps-win-flash 0.62s ease-out;
}

.rps-live-score-lose {
  animation: rps-lose-flash 0.62s ease-out;
}

.live-team {
  font-size: 0.78rem;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.75);
}

.live-team:last-child {
  text-align: right;
}

.live-score {
  font-size: 1.25rem;
  letter-spacing: 0.06em;
  font-variant-numeric: tabular-nums;
}

.rps-duel {
  border-top: 1px solid rgba(255, 255, 255, 0.2);
  margin-top: 4px;
  padding-top: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 14px;
}

.duel-side {
  display: grid;
  justify-items: center;
  gap: 6px;
}

.duel-name {
  font-size: 0.82rem;
  letter-spacing: 0.04em;
  color: rgba(255, 255, 255, 0.75);
}

.duel-icon {
  font-size: 2.3rem;
  line-height: 1;
}

.duel-vs {
  font-size: 0.8rem;
  letter-spacing: 0.08em;
  color: rgba(255, 255, 255, 0.65);
  text-transform: uppercase;
}

.rps-result p {
  margin: 0;
  display: flex;
  justify-content: space-between;
  gap: 10px;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 25;
  background: rgba(0, 0, 0, 0.72);
  display: grid;
  place-items: center;
  padding: 20px;
}

.modal-jumbotron {
  width: min(460px, 100%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 24px;
  background: linear-gradient(140deg, rgba(92, 92, 92, 0.4), rgba(52, 52, 52, 0.8));
  padding: clamp(24px, 3.6vw, 40px);
  text-align: center;
  box-shadow: 0 24px 50px rgba(0, 0, 0, 0.45);
}

.modal-jumbotron h3 {
  margin: 0;
  font-size: clamp(1.35rem, 2.2vw, 2rem);
  line-height: 1.4;
}

.modal-button {
  margin-top: 24px;
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

.modal-button:hover,
.modal-button:focus-visible {
  background: #fff;
  color: #000;
  transform: translateY(-1px);
  outline: none;
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

@keyframes rps-win-flash {
  0% {
    background: rgba(103, 255, 151, 0.5);
    border-color: rgba(103, 255, 151, 0.95);
    box-shadow: 0 0 0 rgba(103, 255, 151, 0);
  }
  45% {
    background: rgba(103, 255, 151, 0.24);
    border-color: rgba(103, 255, 151, 0.7);
    box-shadow: 0 0 26px rgba(103, 255, 151, 0.4);
  }
  100% {
    background: rgba(255, 255, 255, 0.08);
    border-color: rgba(255, 255, 255, 0.3);
    box-shadow: 0 0 0 rgba(103, 255, 151, 0);
  }
}

@keyframes rps-lose-flash {
  0% {
    background: rgba(255, 109, 109, 0.5);
    border-color: rgba(255, 109, 109, 0.95);
    box-shadow: 0 0 0 rgba(255, 109, 109, 0);
  }
  45% {
    background: rgba(255, 109, 109, 0.24);
    border-color: rgba(255, 109, 109, 0.7);
    box-shadow: 0 0 26px rgba(255, 109, 109, 0.4);
  }
  100% {
    background: rgba(255, 255, 255, 0.08);
    border-color: rgba(255, 255, 255, 0.3);
    box-shadow: 0 0 0 rgba(255, 109, 109, 0);
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
