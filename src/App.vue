<template>
  <div class="app">
    <img src="/logo.svg" alt="Logo" class="logo" />
    <h1>Quick Dice</h1>

    <div class="roller">
      <input
        class="main-input"
        v-model="formula"
        @keyup.enter="rollDice"
        @input="clear()"
        placeholder="e.g. 3d8+5"
      />
      <button class="primary-button" @click="rollDice">Roll</button>
      <button class="icon-button" @click="save">
        <svg
          xmlns="http://www.w3.org/2000/svg"
          height="20px"
          viewBox="0 -960 960 960"
          width="20px"
          fill="currentColor"
        >
          <path
            d="M840-680v480q0 33-23.5 56.5T760-120H200q-33 0-56.5-23.5T120-200v-560q0-33 23.5-56.5T200-840h480l160 160Zm-80 34L646-760H200v560h560v-446ZM480-240q50 0 85-35t35-85q0-50-35-85t-85-35q-50 0-85 35t-35 85q0 50 35 85t85 35ZM240-560h360v-160H240v160Zm-40-86v446-560 114Z"
          />
        </svg>
      </button>
    </div>

    <div v-if="result !== null" class="result">
      <div class="result-wrapper">
        <span class="total" :class="{ invalid: isInvalid }">{{ result }}</span>
        <div v-if="resultBreakdown" class="breakdown below">{{ resultBreakdown }}</div>
      </div>
    </div>

    <div v-if="rolls.length" class="history">
      <div v-for="(roll, index) in rolls" :key="index" class="roll-card">
        <div class="top-row">
          <input v-model="roll.name" class="roll-name-input" placeholder="Roll Name" />
          <div class="roll-details">
            <input
              v-model="roll.formula"
              class="roll-formula-input"
              placeholder="Roll Formula"
              @input="clearResult(roll)"
              @keyup.enter="reroll(index)"
            />
            <button class="primary-button reroll" @click="reroll(index)">Roll</button>
            <button class="icon-button delete" @click="deleteRoll(index)">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                height="20px"
                viewBox="0 -960 960 960"
                width="20px"
                fill="currentColor"
              >
                <path
                  d="M280-120q-33 0-56.5-23.5T200-200v-520h-40v-80h200v-40h240v40h200v80h-40v520q0 33-23.5 56.5T680-120H280Zm400-600H280v520h400v-520ZM360-280h80v-360h-80v360Zm160 0h80v-360h-80v360ZM280-720v520-520Z"
                />
              </svg>
            </button>
          </div>
        </div>

        <div class="roll-result" v-if="roll.result !== null">
          <span class="total" :class="{ invalid: roll.isInvalid }">{{ roll.result }}</span>
          <span v-if="roll.breakdown" class="breakdown">{{ roll.breakdown }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue'

const formula = ref('')
const result = ref(null)
const resultBreakdown = ref('')
const isInvalid = ref(false)
const rolls = ref([])

watch(
  rolls,
  (val) => {
    localStorage.setItem('rolls', JSON.stringify(val))
  },
  { deep: true },
)

onMounted(() => {
  const saved = localStorage.getItem('rolls')
  if (saved) rolls.value = JSON.parse(saved)
})

function strip(string) {
  return string.replace(/\s/gi, '')
}

function parseFormula(f) {
  const groups = strip(f).match(/([+-]?\d*d\d+|[+-]?\d+)/gi)
  if (!groups) return null

  return groups
    .map((g) => {
      const dice = g.match(/^([+-]?)(\d*)d(\d+)$/i)
      if (dice) {
        const sign = dice[1] === '-' ? -1 : 1
        const count = parseInt(dice[2] || 1)
        const sides = parseInt(dice[3])
        return { type: 'dice', sign, count, sides }
      }

      const flat = g.match(/^([+-]?)(\d+)$/)
      if (flat) {
        const sign = flat[1] === '-' ? -1 : 1
        const value = parseInt(flat[2])

        if (groups.length === 1 && sign === 1 && flat[1] !== '+') {
          return { type: 'dice', sign, count: 1, sides: value }
        }

        return { type: 'modifier', sign, value }
      }

      return null
    })
    .filter(Boolean)
}

function executeRoll(parts) {
  let total = 0
  const segments = []

  for (const p of parts) {
    if (p.type === 'dice') {
      const rolls = Array.from({ length: p.count }, () => Math.floor(Math.random() * p.sides) + 1)
      const subtotal = rolls.reduce((a, b) => a + b, 0) * p.sign
      total += subtotal
      segments.push(`${p.sign < 0 ? '- ' : segments.length > 0 ? '+ ' : ''}[${rolls.join(', ')}]`)
    } else if (p.type === 'modifier') {
      total += p.value * p.sign
      segments.push(`${p.sign < 0 ? '- ' : segments.length > 0 ? '+ ' : ''}${p.value}`)
    }
  }

  const breakdown =
    parts.length === 1 && (parts[0].count === 1 || parts[0].type === 'modifier')
      ? ''
      : segments.join(' ')

  return { total, breakdown }
}

function rollDice() {
  const parsed = parseFormula(formula.value)
  if (!parsed) {
    result.value = formula.value === '' ? 'Enter a Formula' : 'Invalid Format'
    resultBreakdown.value = 'Examples: d20, 2d6, d10-2, d10+d4'
    isInvalid.value = true
    return
  }

  const { total, breakdown } = executeRoll(parsed)
  result.value = total
  resultBreakdown.value = breakdown
  isInvalid.value = false
}

function save() {
  rolls.value.unshift({
    name: '',
    formula: formula.value.trim(),
    result: null,
    breakdown: '',
  })
}

function reroll(index) {
  const r = rolls.value[index]
  const parsed = parseFormula(r.formula)
  if (!parsed) {
    r.result = r.formula === '' ? 'Enter a Formula' : 'Invalid Format'
    r.breakdown = ''
    r.isInvalid = true
    return
  }
  const { total, breakdown } = executeRoll(parsed)
  r.result = total
  r.breakdown = breakdown
  r.isInvalid = false
}

function clear() {
  result.value = null
  resultBreakdown.value = ''
}

function clearResult(roll) {
  roll.result = null
  roll.breakdown = ''
}

function deleteRoll(index) {
  rolls.value.splice(index, 1)
}
</script>

<style>
:root {
  --bg: #fafafa;
  --card-bg: #ffffff;
  --border: #ddd;
  --text: #222;
  --muted: #aaa;
  --accent: #2d6cdf;
  --accent-hover: #1c4cad;
  --red: #e74c3c;
  --font: 'Segoe UI', 'Helvetica Neue', 'Arial', 'Verdana', sans-serif;
}

body {
  background: var(--bg);
  color: var(--text);
  font-family: var(--font);
  margin: 0 1rem;
}

.app {
  max-width: 520px;
  margin: 6rem auto 4rem;
  text-align: center;
}

.logo {
  width: 100%;
  max-width: 5rem;
}

h1 {
  font-size: 2rem;
  font-weight: 600;
  margin-bottom: 3rem;
}

.roller {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  padding: 0.8rem 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.main-input {
  flex: 1;
  min-width: 50%;
}

input {
  padding: 0.6rem 0.8rem;
  font-family: var(--font);
  font-size: 1rem;
  border: 1.5px solid var(--border);
  border-radius: 6px;
  width: 200px;
}

button {
  padding: 0.6rem 1.2rem;
  font-family: var(--font);
  font-size: 1rem;
  border: 1.5px solid var(--accent);
  color: var(--accent);
  background-color: rgba(0, 0, 0, 0);
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s ease;
}

button:hover {
  background-color: rgba(0, 0, 0, 0.1);
}

.primary-button {
  background: var(--accent);
  color: #fff;
}

.primary-button:hover {
  background: var(--accent-hover);
}

.icon-button {
  padding: 0.3rem 0.6rem;
}

.result {
  margin: 3rem 0 4rem;
  text-align: center;
}

.result-wrapper {
  display: inline-flex;
  flex-direction: column;
  align-items: center;
  font-size: 3rem;
  font-weight: 700;
}

.invalid {
  font-size: 2rem;
  color: var(--red);
}

.breakdown.below {
  margin-left: 0;
  margin-top: 0.3rem;
  font-size: 1.1rem;
  font-weight: 400;
  color: var(--muted);
}

.breakdown {
  margin-left: 0.5rem;
  font-size: 1.1rem;
  font-weight: 400;
  color: var(--muted);
}

.history {
  text-align: left;
  margin-top: 3rem;
}

.roll-card {
  background: var(--card-bg);
  border: 1.5px solid var(--border);
  border-radius: 10px;
  padding: 0.8rem 1rem;
  margin-bottom: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.top-row,
.roll-details {
  display: flex;
  gap: 0.5rem;
  justify-content: space-between;
}

.top-row {
  flex-wrap: wrap;
}

.roll-details {
  flex: 1;
  min-width: 50%;
}

.roll-name-input,
.roll-formula-input {
  flex: 1;
  padding: 0.5rem;
  font-size: 0.95rem;
  border-radius: 6px;
}

.roll-formula-input {
  min-width: 0;
}

.roll-result {
  display: flex;
  gap: 0.6rem;
  justify-content: center;
  align-items: baseline;
  font-size: 2rem;
  font-weight: 600;
}

.actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
}

.reroll,
.delete {
  flex: 0 0 auto;
}

.delete {
  color: var(--red);
  border-color: var(--red);
}
</style>
