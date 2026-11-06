<template>
  <section class="board-panel">
    <div class="board-heading">
      <div>
        <span class="board-eyebrow">DESAFIO DE CONHECIMENTO</span>
        <h2>O TABULEIRO</h2>
      </div>

      <div class="active-badge">
        <span class="active-dot"></span>
        {{ activeTeamName }}
      </div>
    </div>

    <div class="board-scroll">
      <div class="board-grid">
        <!-- Casas do percurso -->
        <div
          v-for="space in spaces"
          :key="space.index"
          class="board-space"
          :class="
            space.isStart
              ? 'space-start'
              : space.isFinish
                ? 'space-finish'
                : `space-color-${space.colorIndex}`
          "
          :style="{
            gridRow: space.row,
            gridColumn: space.col,
          }"
        >
          <span v-if="!space.isStart && !space.isFinish" class="space-number">
            {{ space.index }}
          </span>

          <q-icon v-if="space.isStart" name="flag" size="22px" />

          <q-icon v-else-if="space.isFinish" name="emoji_events" size="22px" />

          <span class="space-label">
            {{ space.label }}
          </span>

          <div class="space-pawns">
            <span
              v-for="team in [0, 1]"
              :key="team"
              v-show="pawnPositions[team] === space.index"
              class="pawn"
              :class="team === 0 ? 'pawn-a' : 'pawn-b'"
              :title="team === 0 ? 'Equipa A' : 'Equipa B'"
            >
              {{ team === 0 ? 'A' : 'B' }}
            </span>
          </div>
        </div>

        <!-- Centro do tabuleiro -->
        <div class="board-center">
          <div class="board-logo">
            <div class="logo-stopwatch">
              <q-icon name="timer" size="36px" />
              <strong>30</strong>
            </div>

            <div class="logo-title"><span>30</span> SEGUNDOS</div>

            <div class="logo-subtitle">PENSA RÁPIDO. VENCE O DESAFIO.</div>
          </div>

          <!-- Lançamento dos dados -->
          <div v-if="phase === 'dice'" class="board-action">
            <span class="action-label">DECIDIR QUEM COMEÇA</span>

            <div class="dice-container">
              <div class="dice">
                <q-icon :name="diceValues[0] ? 'casino' : 'help_outline'" size="42px" />
                <strong>{{ diceValues[0] || '?' }}</strong>
              </div>

              <div class="dice">
                <q-icon :name="diceValues[1] ? 'casino' : 'help_outline'" size="42px" />
                <strong>{{ diceValues[1] || '?' }}</strong>
              </div>
            </div>

            <button class="board-button dice-button" :disabled="!canRoll" @click="$emit('roll')">
              <q-icon name="casino" size="21px" />
              LANÇAR OS DADOS
            </button>

            <p class="board-hint">Os dados determinam qual equipa começa.</p>
          </div>

          <!-- Tirar carta -->
          <div v-else-if="phase === 'draw'" class="board-action">
            <span class="action-label">É A VEZ DE</span>

            <div class="turn-title">
              {{ activeTeamName }}
            </div>

            <div class="card-preview">
              <q-icon name="style" size="42px" />
              <strong>?</strong>
              <span>PRÓXIMA CARTA</span>
            </div>

            <div class="category-preview">PRÓXIMA CASA: {{ nextCategory }}</div>

            <button class="board-button draw-button" :disabled="!canDraw" @click="$emit('draw')">
              <q-icon name="style" size="21px" />
              TIRAR CARTA
            </button>

            <p class="board-hint">Retira uma carta para começar os teus 30 segundos.</p>
          </div>

          <!-- Perguntas em andamento -->
          <div v-else-if="phase === 'answer'" class="board-action">
            <span class="action-label">PARTIDA EM ANDAMENTO</span>

            <div class="turn-title">
              {{ activeTeamName }}
            </div>

            <div class="card-preview card-active">
              <q-icon name="style" size="42px" />
              <strong>✓</strong>
              <span>CARTA ACTIVA</span>
            </div>

            <div class="category-preview">CATEGORIA: {{ nextCategory }}</div>

            <p class="board-hint">Responde correctamente para avançar uma casa!</p>
          </div>

          <!-- Final -->
          <div v-else class="board-action">
            <q-icon name="emoji_events" color="amber-5" size="64px" />
            <div class="turn-title">PARTIDA TERMINADA</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Legenda -->
    <div class="board-legend">
      <span>
        <i class="legend-pawn pawn-a"></i>
        Equipa A
      </span>

      <span>
        <i class="legend-pawn pawn-b"></i>
        Equipa B
      </span>

      <span>
        <q-icon name="flag" color="blue-4" />
        START
      </span>

      <span>
        <q-icon name="emoji_events" color="amber-5" />
        FINISH
      </span>
    </div>
  </section>
</template>

<script setup>
defineProps({
  spaces: {
    type: Array,
    required: true,
  },

  pawnPositions: {
    type: Array,
    default: () => [0, 0],
  },

  diceValues: {
    type: Array,
    default: () => [0, 0],
  },

  phase: {
    type: String,
    default: 'dice',
  },

  activeTeamName: {
    type: String,
    default: 'Equipa A',
  },

  nextCategory: {
    type: String,
    default: 'Conhecimentos gerais',
  },

  canRoll: {
    type: Boolean,
    default: true,
  },

  canDraw: {
    type: Boolean,
    default: true,
  },
})

defineEmits(['roll', 'draw'])
</script>

<style scoped>
.board-panel {
  margin: 24px 0;
  padding: 20px;
  border: 1px solid rgba(76, 164, 255, 0.4);
  border-radius: 22px;
  background: linear-gradient(145deg, rgba(8, 26, 65, 0.96), rgba(4, 12, 37, 0.97));
  box-shadow: 0 18px 45px rgba(0, 0, 0, 0.3);
}

.board-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 18px;
}

.board-eyebrow {
  color: #75caff;
  font-size: 10px;
  font-weight: 900;
  letter-spacing: 2px;
}

.board-heading h2 {
  color: white;
  font-size: 22px;
  font-weight: 950;
  margin: 5px 0 0;
}

.active-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 9px 13px;
  color: #ffda68;
  background: rgba(255, 185, 47, 0.1);
  border: 1px solid rgba(255, 185, 47, 0.4);
  border-radius: 30px;
  font-size: 12px;
  font-weight: 900;
}

.active-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #22c55e;
  box-shadow: 0 0 12px #22c55e;
}

.board-scroll {
  width: 100%;
  overflow-x: auto;
  padding-bottom: 8px;
}

.board-grid {
  display: grid;
  grid-template-columns: repeat(11, minmax(48px, 1fr));
  grid-template-rows: repeat(11, minmax(48px, 1fr));
  min-width: 620px;
  aspect-ratio: 1 / 1;
  gap: 5px;
  padding: 9px;
}

.board-center {
  grid-column: 2 / 11;
  grid-row: 2 / 11;
}

.board-space {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 3px;
  overflow: hidden;

  min-width: 0;
  min-height: 0;
  padding: 3px;

  border: 1px solid rgba(255, 255, 255, 0.55);
  border-radius: 7px;

  text-align: center;

  box-shadow:
    inset 0 1px 0 rgba(255, 255, 255, 0.35),
    0 3px 7px rgba(0, 0, 0, 0.3);
}

.space-color-0 {
  background: linear-gradient(145deg, #20b3ed, #0876bd);
  color: #06152e;
}

.space-color-1 {
  background: linear-gradient(145deg, #ffd64a, #f59e0b);
  color: #412000;
}

.space-color-2 {
  background: linear-gradient(145deg, #31d1c5, #078d9b);
  color: #062b3a;
}

.space-color-3 {
  background: linear-gradient(145deg, #9b8aff, #6349c9);
  color: white;
}

.space-start {
  background: linear-gradient(145deg, #0879dc, #043c86);
  color: white;
  border: 2px solid #7dd3fc;
}

.space-finish {
  background: linear-gradient(145deg, #ffce35, #f07818);
  color: #351900;
  border: 2px solid #fff0a0;
}

.space-number {
  position: absolute;
  top: 3px;
  left: 5px;
  font-size: 9px;
  opacity: 0.75;
  font-weight: 900;
}

.space-label {
  font-size: clamp(7px, 1.1vw, 10px);
  line-height: 1.1;
  font-weight: 950;
  overflow-wrap: anywhere;
}

.space-pawns {
  display: flex;
  gap: 3px;
  min-height: 17px;
  align-items: center;
}

.pawn {
  display: grid;
  place-items: center;
  width: 19px;
  height: 19px;
  border: 2px solid white;
  border-radius: 50%;
  font-size: 9px;
  font-weight: 950;
  color: white;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.4);
}

.pawn-a {
  background: #087bfa;
}

.pawn-b {
  background: #f47716;
}

.board-center {
  grid-column: 2 / 11;
  grid-row: 2 / 11;

  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;

  min-width: 0;
  padding: 10px;

  border: 1px solid rgba(83, 173, 255, 0.45);
  border-radius: 14px;

  background:
    radial-gradient(ellipse at center, rgba(14, 75, 154, 0.5), transparent 75%),
    linear-gradient(145deg, #071630, #030817);
}

.board-logo {
  text-align: center;
}

.logo-stopwatch {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 5px;
  color: #ffd34f;
}

.logo-stopwatch strong {
  font-size: 32px;
  font-weight: 950;
}

.logo-title {
  margin-top: 3px;
  color: #ffae25;
  font-size: clamp(14px, 2.5vw, 23px);
  font-weight: 950;
  letter-spacing: 1px;
}

.logo-title span {
  color: white;
}

.logo-subtitle {
  margin-top: 5px;
  color: #8bcaff;
  font-size: 8px;
  font-weight: 900;
  letter-spacing: 1px;
}

.board-action {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  gap: 8px;
  text-align: center;
}

.action-label {
  color: #94cfff;
  font-size: 9px;
  font-weight: 900;
  letter-spacing: 1.5px;
}

.dice-container {
  display: flex;
  justify-content: center;
  gap: 16px;
}

.dice {
  position: relative;
  display: grid;
  place-items: center;
  width: 57px;
  height: 57px;
  border: 2px solid #ffdf73;
  border-radius: 12px;
  background: linear-gradient(145deg, #fff2b0, #f9ad23);
  color: #17203d;
  box-shadow: 0 5px 15px rgba(255, 177, 33, 0.2);
}

.dice strong {
  position: absolute;
  color: #17203d;
  font-size: 21px;
  font-weight: 950;
}

.turn-title {
  color: #ffd34f;
  font-size: clamp(14px, 2vw, 21px);
  font-weight: 950;
}

.card-preview {
  position: relative;
  display: grid;
  place-items: center;
  width: 68px;
  height: 80px;
  border: 2px solid #ffd34f;
  border-radius: 10px;
  background: linear-gradient(145deg, #ffca38, #ed6b15);
  color: #16213f;
  box-shadow: 0 7px 18px rgba(255, 142, 23, 0.22);
}

.card-preview strong {
  position: absolute;
  font-size: 36px;
  font-weight: 950;
}

.card-preview span {
  font-size: 7px;
  font-weight: 950;
}

.card-active {
  background: linear-gradient(145deg, #2ab9f0, #0879d7);
  border-color: #85d8ff;
  color: #fff;
}

.category-preview {
  max-width: 100%;
  padding: 6px 10px;
  border: 1px solid rgba(76, 164, 255, 0.5);
  border-radius: 20px;
  color: #b8ddff;
  font-size: 9px;
  font-weight: 900;
}

.board-button {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;

  width: 100%;
  max-width: 220px;
  min-height: 39px;
  padding: 8px 10px;

  border: none;
  border-radius: 9px;

  color: #11182f;
  font-size: 10px;
  font-weight: 950;

  cursor: pointer;
  transition:
    transform 0.2s,
    filter 0.2s;
}

.board-button:hover:not(:disabled) {
  transform: translateY(-2px);
  filter: brightness(1.1);
}

.board-button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.dice-button {
  background: linear-gradient(90deg, #ffd34f, #ff941e);
}

.draw-button {
  color: white;
  background: linear-gradient(90deg, #087bf1, #14b7e8);
}

.board-hint {
  max-width: 230px;
  margin: 0;
  color: #a8c8ec;
  font-size: 9px;
  line-height: 1.4;
}

.board-legend {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 18px;
  margin-top: 16px;
  color: #c0d9f7;
  font-size: 11px;
  font-weight: 700;
}

.board-legend span {
  display: flex;
  align-items: center;
  gap: 6px;
}

.legend-pawn {
  width: 12px;
  height: 12px;
  border: 1px solid white;
  border-radius: 50%;
}

@media (max-width: 600px) {
  .board-panel {
    padding: 12px;
  }

  .board-grid {
    min-width: 480px;
    gap: 4px;
    padding: 6px;
    grid-template-columns: repeat(11, minmax(48px, 1fr));
   grid-template-rows: repeat(11, minmax(48px, 1fr));
   min-width: 620px;
  }

  .board-heading h2 {
    font-size: 18px;
  }

  .board-legend {
    gap: 10px;
  }
}
</style>
