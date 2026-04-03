<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'

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

const hasStarted = ref(false)
const typedText = ref('')
const isTypingDone = ref(false)
const isCelebrating = ref(false)

const confettiCanvas = ref<HTMLCanvasElement | null>(null)

let typingTimer: number | null = null
let textIndex = 0

let animationFrameId: number | null = null
let cleanupTimerId: number | null = null
let particles: ConfettiParticle[] = []

const confettiColors = ['#ff2d95', '#00d9ff', '#ffd400', '#70ff57', '#ff7a00', '#8a5cff']

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
  if (hasStarted.value) {
    return
  }

  hasStarted.value = true
  startTyping()
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
  if (isCelebrating.value) {
    return
  }

  isCelebrating.value = true
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
  stopConfetti()
  window.removeEventListener('resize', resizeCanvas)
})
</script>

<template>
  <main class="gift-screen">
    <canvas ref="confettiCanvas" class="confetti-layer" aria-hidden="true"></canvas>

    <section v-if="!hasStarted" class="typing-stage">
      <button
        class="proceed-button"
        type="button"
        @click="startExperience"
        @touchstart.passive="startExperience"
      >
        Start
      </button>
    </section>

    <section v-else-if="!isCelebrating" class="typing-stage" aria-live="polite">
      <p class="typed-message">
        {{ typedText }}
        <span class="cursor" aria-hidden="true">|</span>
      </p>

      <button
        v-if="isTypingDone"
        class="proceed-button"
        type="button"
        @click="startCelebration"
        @touchstart.passive="startCelebration"
      >
        Proceed
      </button>
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
        <path
          d="M210 56 C194 74 196 90 210 96 C224 90 226 74 210 56"
          fill="#ff8a00"
        />
        <path
          d="M210 66 C201 78 202 87 210 91 C218 87 219 78 210 66"
          fill="#fff06a"
        />
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
.cake-stage {
  position: relative;
  z-index: 2;
  width: min(900px, 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
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

.proceed-button {
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
.proceed-button:focus-visible {
  background: #fff;
  color: #000;
  transform: translateY(-1px);
  outline: none;
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

@media (max-width: 640px) {
  .gift-screen {
    padding: 20px;
  }

  .typed-message {
    max-width: 30ch;
  }

  .proceed-button {
    margin-top: 28px;
    padding: 12px 24px;
  }
}
</style>
