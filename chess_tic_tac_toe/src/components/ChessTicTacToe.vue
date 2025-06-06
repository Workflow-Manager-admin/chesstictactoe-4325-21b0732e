<template>
  <div class="cttt-container">
    <div class="cttt-status">
      <span v-if="!winner && !isDraw">
        Current turn:
        <span :class="['cttt-piece', currentPlayer]">
          <span v-if="currentPlayer === 'queen'">{{ queenIcon }}</span>
          <span v-else>{{ kingIcon }}</span>
        </span>
        <span class="cttt-player-label">
          {{ currentPlayer === 'queen' ? 'Queen' : 'King' }}
        </span>
      </span>
      <span v-if="winner">
        <span class="cttt-piece" :class="winner">
          <span v-if="winner === 'queen'">{{ queenIcon }}</span>
          <span v-else>{{ kingIcon }}</span>
        </span>
        <span class="cttt-player-label">
          {{ winner === 'queen' ? 'Queen' : 'King' }}
        </span> wins!
      </span>
      <span v-else-if="isDraw">
        <span class="cttt-draw-label">It's a draw!</span>
      </span>
      <span v-if="autoRestartCountdown !== null && (winner || isDraw)" class="cttt-auto-restart">
        Restarting in {{ autoRestartCountdown }}...
      </span>
    </div>
    <div class="cttt-board">
      <div
        v-for="(cell, idx) in board"
        :key="idx"
        class="cttt-cell"
        :class="getCellClass(idx)"
        @click="onCellClick(idx)"
      >
        <span v-if="cell === 'queen'" class="cttt-piece queen">
          {{ queenIcon }}
        </span>
        <span v-else-if="cell === 'king'" class="cttt-piece king">
          {{ kingIcon }}
        </span>
      </div>
    </div>
    <div class="cttt-controls">
      <button class="cttt-restart-btn" @click="restartGame">
        Restart Game
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, watch, onBeforeUnmount } from 'vue'

const BOARD_SIZE = 9
const lines = [
  // rows
  [0,1,2], [3,4,5], [6,7,8],
  // cols
  [0,3,6],[1,4,7],[2,5,8],
  // diagonals
  [0,4,8],[2,4,6]
]

// Unicode chess piece icons (with fallback)
const QUEEN_ICON = '♛'
const KING_ICON = '♚'

// PUBLIC_INTERFACE
export default {
  name: 'ChessTicTacToe',
  setup() {
    // State
    const board = ref<(null | 'queen' | 'king')[]>(Array(BOARD_SIZE).fill(null))
    const currentPlayer = ref<'queen' | 'king'>('queen')
    const winner = ref<null | 'queen' | 'king'>(null)
    const isDraw = ref(false)
    const winningLine = ref<number[]>([])
    const autoRestartCountdown = ref<null | number>(null)
    let intervalId: ReturnType<typeof setInterval> | null = null

    // PUBLIC_INTERFACE
    function onCellClick(idx: number): void {
      if (winner.value || isDraw.value || board.value[idx]) return
      board.value[idx] = currentPlayer.value
      // Check for win/draw
      const res = calculateWinner(board.value)
      if (res) {
        winner.value = res.player
        winningLine.value = res.line
      } else if (board.value.every(cell => cell !== null)) {
        isDraw.value = true
      } else {
        currentPlayer.value = currentPlayer.value === 'queen' ? 'king' : 'queen'
      }
    }

    // PUBLIC_INTERFACE
    function restartGame(): void {
      board.value = Array(BOARD_SIZE).fill(null)
      currentPlayer.value = 'queen'
      winner.value = null
      isDraw.value = false
      winningLine.value = []
      autoRestartCountdown.value = null
      if (intervalId) {
        clearInterval(intervalId)
        intervalId = null
      }
    }

    // PUBLIC_INTERFACE
    function getCellClass(idx: number): string {
      const arr = ['']
      if (winner.value && winningLine.value.includes(idx)) {
        arr.push('cttt-winning')
      }
      if (!board.value[idx]) arr.push('cttt-empty')
      return arr.join(' ')
    }

    // PUBLIC_INTERFACE
    function calculateWinner(
      b: (null | 'queen' | 'king')[]
    ): { player: 'queen' | 'king'; line: number[] } | null {
      for (const line of lines) {
        const [a, c, d] = line
        if (b[a] && b[a] === b[c] && b[a] === b[d]) {
          return { player: b[a] as 'queen' | 'king', line }
        }
      }
      return null
    }

    // Watch for game end (win or draw)
    watch(
      () => [winner.value, isDraw.value],
      ([newWinner, newIsDraw], [oldWinner, oldIsDraw]) => {
        // Only trigger when game status changes to ended
        if ((newWinner || newIsDraw) && (!oldWinner && !oldIsDraw)) {
          let countdown = 5
          autoRestartCountdown.value = countdown
          intervalId = setInterval(() => {
            countdown--
            autoRestartCountdown.value = countdown
            if (countdown === 0) {
              restartGame()
            }
          }, 1000)
        }
        // If game resets or restarts, clean up any interval/timer
        if ((!newWinner && !newIsDraw) && (oldWinner || oldIsDraw)) {
          if (intervalId) {
            clearInterval(intervalId)
            intervalId = null
          }
          autoRestartCountdown.value = null
        }
      }
    )

    onBeforeUnmount(() => {
      if (intervalId) {
        clearInterval(intervalId)
      }
    })

    return {
      board,
      currentPlayer,
      winner,
      isDraw,
      winningLine,
      autoRestartCountdown,
      onCellClick,
      restartGame,
      getCellClass,
      queenIcon: QUEEN_ICON,
      kingIcon: KING_ICON,
    }
  }
}
</script>

<style scoped>
.cttt-container {
  background: #2e2e2e;
  color: #bfa14a;
  max-width: 400px;
  margin: 60px auto 0 auto;
  border-radius: 14px;
  box-shadow: 0 5px 25px 0 #000a, 0 2px 5px 0 #bfa14a50;
  padding: 2.2rem 1.5rem 1.5rem 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  border: 3px solid #bfa14a;
}

.cttt-status {
  font-size: 1.15rem;
  margin-bottom: 1.3rem;
  min-height: 2em;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.cttt-player-label {
  margin-left: 0.35em;
  font-size: 1em;
  color: #bfa14a;
  letter-spacing: 0.03em;
}

.cttt-draw-label {
  color: #bfa14a;
  font-weight: bold;
  font-size: 1.1em;
}

.cttt-auto-restart {
  display: block;
  font-size: 1em;
  color: #fff0b0;
  margin-top: 8px;
  font-weight: 500;
  letter-spacing: 0.06em;
}

.cttt-board {
  display: grid;
  grid-template-columns: repeat(3, 64px);
  grid-template-rows: repeat(3, 64px);
  gap: 0.5em;
  background: #232323;
  padding: 1.1em;
  border-radius: 10px;
  border: 2.5px solid #bfa14a;
  margin-bottom: 1.3rem;
}

.cttt-cell {
  width: 64px;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  background: #f5f5dc;
  color: #2e2e2e;
  border-radius: 10px;
  border: 2.5px solid #bfa14a;
  transition: background 0.18s, box-shadow 0.18s;
  position: relative;
  overflow: hidden;
  box-sizing: border-box;
}

/* For responsive design: shrink cells on small screens */
@media (max-width: 500px) {
  .cttt-board {
    grid-template-columns: repeat(3, 15vw);
    grid-template-rows: repeat(3, 15vw);
    padding: 0.5em;
  }
  .cttt-cell {
    width: 15vw;
    height: 15vw;
    min-width: 45px;
    min-height: 45px;
    max-width: 80px;
    max-height: 80px;
  }
}

/* Highlight for winning cells stays the same */
.cttt-cell.cttt-winning {
  background: #bfa14a;
  color: #2e2e2e;
  box-shadow: 0 0 10px 0 #bfa14a99;
  border-color: #fff0b0;
  z-index: 2;
}

.cttt-cell.cttt-empty:hover {
  background: #bfa14a75;
  color: #232323;
  box-shadow: 0 0 7px 0 #bfa14a99 inset;
  outline: 1.5px solid #bfa14a98;
}

.cttt-cell .cttt-piece {
  /* Fix icon overflow on all screens */
  font-size: 2.4em;
  font-weight: bold;
  user-select: none;
  transition: transform 0.18s;
  line-height: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  max-width: 100%;
  max-height: 100%;
}

/* For really small cells, scale down icons */
@media (max-width: 500px) {
  .cttt-cell .cttt-piece {
    font-size: 2em;
  }
}

.cttt-piece.queen {
  color: #2e2e2e;
  filter: drop-shadow(1px 1px 1px #bfa14a60);
}

.cttt-piece.king {
  color: #bfa14a;
  filter: drop-shadow(1px 1px 1px #23232370);
  text-shadow: 0 0 5px #23232320;
}

.cttt-controls {
  margin-top: 0.5em;
  display: flex;
  justify-content: center;
  width: 100%;
}

.cttt-restart-btn {
  background: #bfa14a;
  color: #2e2e2e;
  border: none;
  padding: 0.7em 1.6em;
  font-size: 1.005em;
  font-weight: bold;
  border-radius: 7px;
  cursor: pointer;
  margin: 0 auto;
  box-shadow: 0 2px 10px #23232333;
  transition: background 0.14s, color 0.14s;
}
.cttt-restart-btn:hover,
.cttt-restart-btn:focus {
  background: #2e2e2e;
  color: #bfa14a;
  border: 1.5px solid #bfa14a;
  outline: none;
}
</style>
