<template>
  <q-page class="game-page q-pa-md">
    <div class="game-shell">
      <header class="game-header">
        <div class="brand">
          <div class="brand-icon">
            <q-icon name="bolt" size="32px" />
          </div>
          <div>
            <div class="brand-name">30 SEGUNDOS</div>
            <div class="brand-subtitle">Pensa rápido. Vence o desafio.</div>
          </div>
        </div>
        <q-btn
          round
          flat
          :icon="soundEnabled ? 'volume_up' : 'volume_off'"
          aria-label="Alternar sons"
          @click="soundEnabled = !soundEnabled"
        />
      </header>

      <section v-if="screen === 'menu'" class="menu-screen">
        <div class="hero">
          <q-badge color="deep-purple-5" class="q-mb-md"> DESAFIO DE CONHECIMENTO </q-badge>
          <h1>30 segundos.<br /><span>Mostra o que sabes.</span></h1>
          <p>Responde corretamente, acumula pontos e supera os teus adversários.</p>
        </div>

        <div class="section-label">ESCOLHE O TEU MODO</div>

        <div class="mode-grid">
          <button
            class="mode-card"
            :class="{ chosen: selectedMode === 'computer' }"
            @click="selectedMode = 'computer'"
          >
            <q-icon name="smart_toy" size="34px" />
            <strong>Contra o computador</strong>
            <span>Tu contra a CPU</span>
          </button>

          <button
            class="mode-card"
            :class="{ chosen: selectedMode === 'duo' }"
            @click="selectedMode = 'duo'"
          >
            <q-icon name="groups" size="34px" />
            <strong>Equipas locais</strong>
            <span>2, 4, 6 ou 8 jogadores</span>
          </button>

          <button
            class="mode-card"
            :class="{ chosen: selectedMode === 'online' }"
            @click="selectedMode = 'online'"
          >
            <q-icon name="public" size="34px" />
            <strong>Equipas online</strong>
            <span>2, 4, 6 ou 8 jogadores</span>
          </button>
        </div>

        <div v-if="selectedMode !== 'online'" class="settings-panel">
          <div class="section-label">JOGADORES</div>

          <template v-if="selectedMode === 'computer'">
            <q-input
              v-model="playerName"
              dark
              outlined
              maxlength="20"
              label="O teu nome"
              class="q-mb-md"
            />
          </template>

          <template v-else>
            <q-select
              v-model="localPlayerCount"
              :options="playerCountOptions"
              emit-value
              map-options
              dark
              outlined
              label="Número de jogadores"
              class="q-mb-md"
            />

            <div class="section-label q-mt-md">NOMES DOS JOGADORES</div>

            <q-input
              v-for="(_, index) in localPlayerCount"
              :key="`local-player-${index}`"
              v-model="localPlayerNames[index]"
              dark
              outlined
              maxlength="20"
              :label="`Jogador ${index + 1}`"
              class="q-mb-sm"
            />

            <div class="note q-mb-md">
              Os primeiros {{ localPlayerCount / 2 }} jogadores ficam na Equipa A e os restantes na
              Equipa B.
            </div>
          </template>

          <div class="note">
            Cada turno dura 30 segundos. Durante esse tempo, a equipa responde a várias perguntas.
            Cada resposta correcta faz o peão avançar uma casa e apresenta uma nova pergunta. A vez
            só muda quando o tempo termina.
          </div>

          <q-btn
            class="start-button full-width q-mt-lg"
            unelevated
            icon="play_arrow"
            label="INICIAR PARTIDA"
            @click="startLocalGame"
          />
        </div>

        <div v-else class="settings-panel online-panel">
          <div class="section-label">SALA ONLINE — 2, 4, 6 OU 8 JOGADORES</div>

          <q-select
            v-model="roomPlayerCount"
            :options="playerCountOptions"
            emit-value
            map-options
            :disable="Boolean(roomCode)"
            dark
            outlined
            label="Número de jogadores na sala"
            class="q-mb-md"
          />

          <q-input
            v-model="roomPlayerName"
            dark
            outlined
            maxlength="20"
            label="O teu nome"
            class="q-mb-md"
          />

          <q-btn
            label="CRIAR SALA"
            icon="add"
            color="deep-purple-5"
            unelevated
            class="full-width q-mb-md"
            :loading="onlineLoading"
            @click="createOnlineRoom"
          />

          <q-separator dark class="q-my-md" />

          <q-input
            v-model="roomCodeInput"
            dark
            outlined
            maxlength="6"
            label="Código da sala"
            hint="Introduz o código de 6 caracteres"
            class="q-mb-md"
            @update:model-value="normalizeRoomCode"
          />

          <q-btn
            label="ENTRAR NA SALA"
            icon="login"
            color="blue-7"
            unelevated
            class="full-width"
            :loading="onlineLoading"
            @click="joinOnlineRoom"
          />

          <div v-if="roomCode" class="room-code-card q-mt-lg">
            <div class="text-caption">CÓDIGO DA SALA</div>
            <div class="room-code">{{ roomCode }}</div>

            <q-btn
              flat
              color="white"
              icon="content_copy"
              label="Copiar código"
              @click="copyRoomCode"
            />

            <p>{{ roomStatus }}</p>

            <div class="teams-grid">
              <div>
                <strong>Equipa A</strong>
                <div
                  v-for="p in roomPlayers.slice(0, roomPlayerCount / 2)"
                  :key="p.uid"
                  class="room-player"
                >
                  {{ p.name }}
                </div>
              </div>

              <div>
                <strong>Equipa B</strong>
                <div
                  v-for="p in roomPlayers.slice(roomPlayerCount / 2, roomPlayerCount)"
                  :key="p.uid"
                  class="room-player"
                >
                  {{ p.name }}
                </div>
              </div>
            </div>

            <q-btn
              v-if="
                isRoomHost && roomPlayers.length === roomPlayerCount && roomGameStatus !== 'playing'
              "
              class="start-button full-width q-mt-md"
              unelevated
              icon="play_arrow"
              label="INICIAR PARTIDA ONLINE"
              @click="startOnlineGame"
            />

            <q-banner
              v-if="roomGameStatus === 'playing'"
              class="bg-positive text-white q-mt-md"
              rounded
            >
              A partida começou.
            </q-banner>
          </div>

          <q-banner v-if="onlineMessage" rounded class="bg-grey-9 text-white q-mt-md">
            {{ onlineMessage }}
          </q-banner>
        </div>
      </section>

      <section v-else-if="screen === 'playing'" class="playing-screen">
        <div class="game-topline">
          <q-btn flat dense color="grey-5" icon="arrow_back" label="Sair" @click="confirmExit" />
          <q-badge color="deep-purple-6">
            {{ selectedMode === 'online' ? 'EQUIPAS ONLINE' : 'PARTIDA' }}
          </q-badge>
        </div>

        <GameBoard
          :spaces="boardSpaces"
          :pawn-positions="pawnPositions"
          :dice-values="diceValues"
          :phase="gamePhase"
          :active-team-name="activeTeamLabel"
          :next-category="nextCategory"
          :can-roll="canRoll"
          :can-draw="canDraw"
          @roll="rollStartDice"
          @draw="drawCard"
        />

        <div class="timer-panel" :class="{ urgent: timeLeft <= 5 }">
          <div class="timer-caption">TEMPO TOTAL RESTANTE</div>
          <div class="timer-value">00:{{ String(timeLeft).padStart(2, '0') }}</div>
          <q-linear-progress
            :value="timeLeft / 30"
            color="deep-purple-4"
            track-color="grey-9"
            rounded
            size="8px"
          />
        </div>

        <div v-if="selectedMode === 'online'" class="teams-grid score-teams">
          <div class="team-card team-a">
            <strong>Equipa A</strong>
            <span>{{ onlineScores[0] }} pontos</span>
            <small>
              {{
                roomPlayers
                  .slice(0, roomPlayerCount / 2)
                  .map((p) => p.name)
                  .join(' + ')
              }}
            </small>
          </div>

          <div class="team-card team-b">
            <strong>Equipa B</strong>
            <span>{{ onlineScores[1] }} pontos</span>
            <small>
              {{
                roomPlayers
                  .slice(roomPlayerCount / 2, roomPlayerCount)
                  .map((p) => p.name)
                  .join(' + ')
              }}
            </small>
          </div>
        </div>

        <div v-else-if="selectedMode === 'computer'" class="players-row">
          <div class="player-card">
            <q-avatar color="deep-purple-8" text-color="white">
              <q-icon name="person" />
            </q-avatar>
            <div class="player-info">
              <strong>{{ playerName || 'Jogador 1' }}</strong>
              <span>{{ scores[0] }} pontos</span>
            </div>
          </div>

          <div class="versus">VS</div>

          <div class="player-card">
            <q-avatar color="blue-9" text-color="white">
              <q-icon name="smart_toy" />
            </q-avatar>
            <div class="player-info">
              <strong>Computador</strong>
              <span>{{ scores[1] }} pontos</span>
            </div>
          </div>
        </div>

        <div v-else class="teams-grid score-teams">
          <div class="team-card team-a">
            <strong>Equipa A</strong>
            <span>{{ scores[0] }} pontos</span>
            <small>
              {{
                localPlayerNames
                  .slice(0, localPlayerCount / 2)
                  .map((n) => n || 'Jogador')
                  .join(' + ')
              }}
            </small>
          </div>

          <div class="team-card team-b">
            <strong>Equipa B</strong>
            <span>{{ scores[1] }} pontos</span>
            <small>
              {{
                localPlayerNames
                  .slice(localPlayerCount / 2, localPlayerCount)
                  .map((n) => n || 'Jogador')
                  .join(' + ')
              }}
            </small>
          </div>
        </div>

        <div v-if="currentQuestion" class="question-meta">
          <q-badge color="blue-grey-9">
            {{ currentQuestionCategory || currentQuestion.category || 'Conhecimentos gerais' }}
          </q-badge>
          <span>Pergunta {{ questionNumber }}</span>
        </div>

        <div v-if="currentQuestion" class="question-card">
          <h2>{{ currentQuestion.question }}</h2>

          <div class="answers-grid">
            <button
              v-for="(option, index) in currentQuestion.options"
              :key="`${currentQuestion.id}-${index}`"
              class="answer-button"
              :disabled="answerBusy"
              @click="submitAnswer(index)"
            >
              <span class="answer-letter">
                {{ ['A', 'B', 'C', 'D'][index] }}
              </span>
              <span>{{ option }}</span>
            </button>
          </div>

          <div v-if="answerMessage" class="answer-feedback">
            <strong>{{ answerMessage }}</strong>
          </div>
        </div>

        <div v-else class="question-card">
          <h2>A preparar a próxima pergunta…</h2>
        </div>

        <div class="bottom-actions">
          <span>+100 pontos por resposta correcta · 1 casa por acerto</span>
          <span v-if="selectedMode === 'online'">Sala: {{ roomCode }}</span>
        </div>
      </section>

      <section v-else-if="screen === 'result'" class="result-screen">
        <q-icon
          :name="winnerIndex === -1 ? 'handshake' : 'emoji_events'"
          size="76px"
          color="amber-5"
        />

        <div class="section-label">PARTIDA TERMINADA</div>
        <h1>{{ winnerText }}</h1>
        <p class="result-subtitle">Resultado final dos 30 segundos.</p>

        <div v-if="selectedMode === 'online'" class="teams-grid score-teams">
          <div class="team-card team-a">
            <strong>Equipa A</strong>
            <span>{{ onlineScores[0] }} pontos</span>
          </div>
          <div class="team-card team-b">
            <strong>Equipa B</strong>
            <span>{{ onlineScores[1] }} pontos</span>
          </div>
        </div>

        <div v-else-if="selectedMode === 'computer'" class="result-score-grid">
          <div class="result-score">
            <span>{{ playerName || 'Jogador 1' }}</span>
            <strong>{{ scores[0] }}</strong>
            <small>pontos</small>
          </div>
          <div class="result-score">
            <span>Computador</span>
            <strong>{{ scores[1] }}</strong>
            <small>pontos</small>
          </div>
        </div>

        <div v-else class="teams-grid score-teams">
          <div class="team-card team-a">
            <strong>Equipa A</strong>
            <span>{{ scores[0] }} pontos</span>
          </div>
          <div class="team-card team-b">
            <strong>Equipa B</strong>
            <span>{{ scores[1] }} pontos</span>
          </div>
        </div>

        <q-btn
          class="start-button full-width q-mt-xl"
          unelevated
          icon="replay"
          label="JOGAR NOVAMENTE"
          @click="selectedMode === 'online' ? backToMenu() : startLocalGame()"
        />

        <q-btn
          class="full-width q-mt-sm"
          flat
          color="grey-5"
          label="Voltar ao menu"
          @click="backToMenu"
        />
      </section>

      <footer class="game-footer">
        <span>30 SEGUNDOS</span>
        <span>DESAFIA A TUA MENTE</span>
      </footer>
    </div>
  </q-page>
</template>

<script setup>
import { computed, onBeforeUnmount, ref } from 'vue'
import { Dialog, Notify, copyToClipboard } from 'quasar'

import { authReady, database } from '../boot/firebase'
import { ref as dbRef, get, set, onValue, runTransaction } from 'firebase/database'

import { questions } from '../data/questions.js'
import { boardCategories } from '../data/boardCategories.js'
import GameBoard from '../components/GameBoard.vue'

import { playCorrect, playWrong, playVictory, playTick, playTimeUp } from '../utils/gameAudio.js'

// =====================================================
// CONFIGURAÇÕES E PERCURSO DO TABULEIRO
// =====================================================

const TURN_SECONDS = 30
const TURN_MS = TURN_SECONDS * 1000
const START_QUESTION_CATEGORY = 'Internacional'

const TRACK_COORDINATES = (() => {
  const coordinates = []

  // Lado esquerdo: de baixo para cima.
  for (let row = 11; row >= 1; row--) {
    coordinates.push({ row, col: 1 })
  }

  // Parte superior: da esquerda para a direita.
  for (let col = 2; col <= 11; col++) {
    coordinates.push({ row: 1, col })
  }

  // Lado direito: de cima para baixo.
  for (let row = 2; row <= 11; row++) {
    coordinates.push({ row, col: 11 })
  }

  // Parte inferior: da direita para a esquerda.
  for (let col = 10; col >= 1; col--) {
    coordinates.push({ row: 11, col })
  }

  return coordinates.slice(0, boardCategories.length + 2)
})()

const FINISH_POSITION = boardCategories.length + 1

// =====================================================
// ESTADO PRINCIPAL
// =====================================================

const screen = ref('menu')
const selectedMode = ref('computer')

const playerName = ref('Jogador 1')
const roomPlayerName = ref('')
const soundEnabled = ref(true)

const playerCountOptions = [
  { label: '2 jogadores', value: 2 },
  { label: '4 jogadores', value: 4 },
  { label: '6 jogadores', value: 6 },
  { label: '8 jogadores', value: 8 },
]

const localPlayerCount = ref(4)
const roomPlayerCount = ref(4)

const localPlayerNames = ref([
  'Jogador 1',
  'Jogador 2',
  'Jogador 3',
  'Jogador 4',
  'Jogador 5',
  'Jogador 6',
  'Jogador 7',
  'Jogador 8',
])

// =====================================================
// ESTADO DA PARTIDA
// =====================================================

const currentQuestion = ref(null)
const currentQuestionCategory = ref('')

const activeTeam = ref(0)

const scores = ref([0, 0])
const onlineScores = ref([0, 0])

const pawnPositions = ref([0, 0])
const diceValues = ref([0, 0])

const gamePhase = ref('dice')

const timeLeft = ref(TURN_SECONDS)
const questionNumber = ref(0)

const winnerIndex = ref(-1)

const answerBusy = ref(false)
const answerMessage = ref('')

// =====================================================
// ESTADO ONLINE
// =====================================================

const roomCodeInput = ref('')
const roomCode = ref('')
const roomStatus = ref('')
const roomPlayers = ref([])

const onlineMessage = ref('')
const onlineLoading = ref(false)
const roomGameStatus = ref('waiting')

const myUid = ref('')
const isRoomHost = ref(false)

const onlineUsedQuestionIds = ref([])

let roomListener = null
let timerId = null
let cpuTimeoutId = null

let usedQuestions = []

let turnStartedAt = 0
let lastTick = TURN_SECONDS

let onlineFinishPending = false
let onlineGameSeen = false

// =====================================================
// BANCO DE PERGUNTAS
// =====================================================

const validQuestions = computed(() => {
  return questions.filter((question) => {
    const category = String(question.category || question.subject || '')
    const optionsAreValid = Array.isArray(question.options) && question.options.length >= 2
    const answer = Number(question.answer)
    const answerIsValid =
      Number.isInteger(answer) && optionsAreValid && answer >= 0 && answer < question.options.length

    return !/matem[áa]tica/i.test(category) && optionsAreValid && answerIsValid
  })
})

// =====================================================
// LIGAÇÃO DAS CASAS ÀS CATEGORIAS DAS PERGUNTAS
// =====================================================

const BOARD_QUESTION_CATEGORIES = {
  'MZ Challenge': 'Moçambique',
  'História Nacional': 'Moçambique',
  GeoMZ: 'Moçambique',
  'Cultura Viva': 'Moçambique',
  Personalidades: 'Moçambique',

  'Global Challenge': 'Internacional',
  'World Atlas': 'Internacional',
  'World History': 'Internacional',
  'Africa Zone': 'Internacional',

  'Sports Arena': 'Desporto Africano',
  'Africa Sports': 'Desporto Africano',
  'Europe Sports': 'Desporto Europeu',
  'Football Pro': 'Desporto Europeu',
  'Game Legends': 'Desporto Europeu',

  'Cartoon World': 'Desenhos Animados',
  'Anime Zone': 'Desenhos Animados',
  'Cinema Plus': 'Desenhos Animados',
  'TV Series': 'Desenhos Animados',
  'Hero Universe': 'Desenhos Animados',
  'Music Arena': 'Desenhos Animados',

  'Language Master': 'Internacional',
  'Literature Club': 'Internacional',

  'Science Lab': 'Ciências',
  'Human Body': 'Ciências',
  'Eco Planet': 'Ciências',
  'Space Explorer': 'Ciências',

  'Tech World': 'Tecnologia',
  'Innovation Hub': 'Tecnologia',
  'Digital Life': 'Tecnologia',
  'Logic Master': 'Ciências',
  'Brain Power': 'Ciências',
  'Smart Life': 'Tecnologia',

  'Flag Master': 'Internacional',
  'Arts & Culture': 'Desenhos Animados',
  'Puzzle Zone': 'Ciências',
}

// =====================================================
// CASAS DO TABULEIRO
// =====================================================

const boardSpaces = computed(() => {
  return TRACK_COORDINATES.map((coordinate, index) => {
    const isStart = index === 0
    const isFinish = index === FINISH_POSITION
    const houseName = !isStart && !isFinish ? boardCategories[index - 1] || '' : ''

    const questionCategory = isStart
      ? START_QUESTION_CATEGORY
      : isFinish
        ? ''
        : BOARD_QUESTION_CATEGORIES[houseName] || START_QUESTION_CATEGORY

    return {
      index,
      row: coordinate.row,
      col: coordinate.col,
      isStart,
      isFinish,
      category: houseName,
      questionCategory,
      label: isStart ? 'START' : isFinish ? 'FINISH' : houseName,
      colorIndex: isStart || isFinish ? 0 : (index - 1) % 4,
    }
  })
})

function questionCategoryForPosition(position) {
  const numericPosition = Number(position)
  const safePosition = Number.isInteger(numericPosition)
    ? Math.min(Math.max(numericPosition, 0), FINISH_POSITION)
    : 0

  return boardSpaces.value[safePosition]?.questionCategory || START_QUESTION_CATEGORY
}

const nextCategory = computed(() => {
  return questionCategoryForPosition(pawnPositions.value[activeTeam.value])
})

function getQuestionCategory(question, fallback = START_QUESTION_CATEGORY) {
  return question?.category || question?.subject || fallback
}

// =====================================================
// EQUIPAS E TURNOS
// =====================================================

const playerTeamIndex = computed(() => {
  const player = roomPlayers.value.find((item) => item.uid === myUid.value)
  if (!player) return -1

  return Number(player.slot) < roomPlayerCount.value / 2 ? 0 : 1
})

const activeTeamLabel = computed(() => {
  if (selectedMode.value === 'computer') {
    return activeTeam.value === 0 ? 'A TUA VEZ' : 'VEZ DO COMPUTADOR'
  }

  return activeTeam.value === 0 ? 'VEZ DA EQUIPA A' : 'VEZ DA EQUIPA B'
})

const canRoll = computed(() => {
  if (gamePhase.value !== 'dice') return false
  return selectedMode.value !== 'online' || isRoomHost.value
})

const canDraw = computed(() => {
  if (gamePhase.value !== 'draw') return false
  if (selectedMode.value === 'computer') return activeTeam.value === 0
  if (selectedMode.value === 'online') return playerTeamIndex.value === activeTeam.value
  return true
})

const canAnswer = computed(() => {
  if (gamePhase.value !== 'answer') return false
  if (selectedMode.value === 'online') return playerTeamIndex.value === activeTeam.value
  if (selectedMode.value === 'computer') return activeTeam.value === 0
  return true
})

// =====================================================
// RESULTADO
// =====================================================

const winnerText = computed(() => {
  if (winnerIndex.value === -1) return 'Empate!'

  if (selectedMode.value === 'computer') {
    return winnerIndex.value === 0
      ? `${playerName.value || 'Jogador 1'} venceu!`
      : 'O computador venceu!'
  }

  return winnerIndex.value === 0 ? 'A Equipa A venceu!' : 'A Equipa B venceu!'
})

// =====================================================
// SELECÇÃO DE PERGUNTAS
// =====================================================

function selectQuestion(category, alreadyUsed = [], excludedId = null) {
  if (!validQuestions.value.length) return null

  const requestedCategory = String(category || START_QUESTION_CATEGORY)
  const categoryQuestions = validQuestions.value.filter((question) => {
    return (question.category || question.subject || 'Cultura Geral') === requestedCategory
  })

  // Fallback para o banco completo caso uma categoria não tenha perguntas.
  const pool = categoryQuestions.length ? categoryQuestions : validQuestions.value
  const usedIds = new Set(Array.isArray(alreadyUsed) ? alreadyUsed.map(String) : [])
  const excludedQuestionId =
    excludedId === null || excludedId === undefined ? null : String(excludedId)

  const unused = pool.filter((question) => {
    const id = String(question.id)
    return !usedIds.has(id) && id !== excludedQuestionId
  })

  const alternatives = pool.filter((question) => {
    return String(question.id) !== excludedQuestionId
  })

  const candidates = unused.length ? unused : alternatives.length ? alternatives : pool

  return candidates[Math.floor(Math.random() * candidates.length)] || null
}

function pickQuestion(category = nextCategory.value, excludedId = null) {
  const question = selectQuestion(category, usedQuestions, excludedId)
  if (question) usedQuestions.push(question.id)
  return question
}

// =====================================================
// TEMPORIZADORES
// =====================================================

function stopTimer() {
  if (timerId) clearInterval(timerId)
  if (cpuTimeoutId) clearTimeout(cpuTimeoutId)
  timerId = null
  cpuTimeoutId = null
}

function startLocalClock() {
  if (timerId) clearInterval(timerId)
  turnStartedAt = Date.now()
  lastTick = TURN_SECONDS
  timeLeft.value = TURN_SECONDS
  timerId = setInterval(updateRemaining, 200)
}

function updateRemaining() {
  if (screen.value !== 'playing' || gamePhase.value !== 'answer') return

  const elapsed = Math.floor((Date.now() - turnStartedAt) / 1000)
  const remaining = Math.max(0, TURN_SECONDS - elapsed)
  timeLeft.value = remaining

  if (remaining !== lastTick) {
    lastTick = remaining
    if (soundEnabled.value && remaining > 0) playTick(remaining)
  }

  if (remaining <= 0) finishLocalTurn()
}

// =====================================================
// INICIAR PARTIDA LOCAL
// =====================================================

function startLocalGame() {
  if (selectedMode.value === 'online') return

  if (!validQuestions.value.length) {
    Dialog.create({
      title: 'Sem perguntas',
      message: 'Não existem perguntas válidas no banco de dados.',
      ok: 'Entendi',
    })
    return
  }

  stopTimer()

  scores.value = [0, 0]
  pawnPositions.value = [0, 0]
  diceValues.value = [0, 0]
  activeTeam.value = 0
  gamePhase.value = 'dice'
  questionNumber.value = 0
  timeLeft.value = TURN_SECONDS
  winnerIndex.value = -1
  currentQuestion.value = null
  currentQuestionCategory.value = ''
  answerBusy.value = false
  answerMessage.value = ''
  usedQuestions = []
  turnStartedAt = 0

  screen.value = 'playing'
}

// =====================================================
// LANÇAMENTO DOS DADOS
// =====================================================

function randomDie() {
  return Math.floor(Math.random() * 6) + 1
}

async function rollStartDice() {
  if (gamePhase.value !== 'dice') return

  const dice = [randomDie(), randomDie()]

  if (selectedMode.value === 'online') {
    if (!isRoomHost.value || !roomCode.value) return

    try {
      const winner = dice[0] === dice[1] ? null : dice[0] > dice[1] ? 0 : 1
      const result = await runTransaction(
        dbRef(database, `rooms/${roomCode.value}/game`),
        (game) => {
          if (!game || game.status !== 'playing' || game.phase !== 'dice') return

          return {
            ...game,
            diceValues: dice,
            activeTeam: winner === null ? game.activeTeam || 0 : winner,
            phase: winner === null ? 'dice' : 'draw',
          }
        },
      )

      if (result.committed) {
        if (winner === null) {
          Notify.create({
            message: 'Empate nos dados! Lancem novamente.',
            color: 'warning',
            icon: 'casino',
          })
        } else {
          Notify.create({
            message: `A Equipa ${winner === 0 ? 'A' : 'B'} começa!`,
            color: 'positive',
            icon: 'flag',
          })
        }
      }
    } catch (error) {
      console.error('Erro ao lançar dados online:', error)
      onlineMessage.value = 'Não foi possível lançar os dados.'
    }
    return
  }

  diceValues.value = dice

  if (dice[0] === dice[1]) {
    Notify.create({
      message: 'Empate nos dados! Lança novamente.',
      color: 'warning',
      icon: 'casino',
    })
    return
  }

  activeTeam.value = dice[0] > dice[1] ? 0 : 1
  gamePhase.value = 'draw'

  Notify.create({
    message: `${activeTeamLabel.value} começa a partida!`,
    color: 'positive',
    icon: 'flag',
  })

  if (selectedMode.value === 'computer' && activeTeam.value === 1) {
    scheduleCpuDraw()
  }
}

// =====================================================
// RETIRAR CARTA
// =====================================================

async function drawCard(isComputer = false) {
  if (gamePhase.value !== 'draw' || screen.value !== 'playing') return

  if (selectedMode.value === 'online') {
    if (!canDraw.value) return
    await drawOnlineCard()
    return
  }

  if (isComputer) {
    if (selectedMode.value !== 'computer' || activeTeam.value !== 1) return
  } else if (!canDraw.value) {
    return
  }

  const category = questionCategoryForPosition(pawnPositions.value[activeTeam.value])
  const question = pickQuestion(category)

  if (!question) {
    Dialog.create({
      title: 'Sem perguntas',
      message: 'Não existem perguntas válidas disponíveis.',
      ok: 'Entendi',
    })
    skipLocalTurnWithoutQuestion()
    return
  }

  currentQuestion.value = question
  currentQuestionCategory.value = getQuestionCategory(question, category)
  questionNumber.value += 1
  answerMessage.value = ''
  answerBusy.value = false
  gamePhase.value = 'answer'

  startLocalClock()

  if (selectedMode.value === 'computer' && activeTeam.value === 1) {
    scheduleCpu()
  }
}

// =====================================================
// RETIRAR CARTA ONLINE
// =====================================================

async function drawOnlineCard() {
  if (!roomCode.value || playerTeamIndex.value !== activeTeam.value) return

  const team = activeTeam.value
  const position = pawnPositions.value[team]
  const category = questionCategoryForPosition(position)
  const question = selectQuestion(category, onlineUsedQuestionIds.value)

  if (!question) {
    onlineMessage.value = 'Não existem perguntas válidas disponíveis.'
    return
  }

  const gameRef = dbRef(database, `rooms/${roomCode.value}/game`)

  try {
    const result = await runTransaction(gameRef, (game) => {
      if (
        !game ||
        game.status !== 'playing' ||
        game.phase !== 'draw' ||
        Number(game.activeTeam) !== team
      ) {
        return
      }

      if (Number(game.pawnPositions?.[team] || 0) !== position) return

      const usedIds = Array.isArray(game.usedQuestionIds) ? game.usedQuestionIds : []

      return {
        ...game,
        phase: 'answer',
        currentQuestion: question,
        currentQuestionCategory: getQuestionCategory(question, category),
        questionNumber: (game.questionNumber || 0) + 1,
        turnStartedAt: Date.now(),
        timeLeft: TURN_SECONDS,
        usedQuestionIds: [...usedIds, question.id],
      }
    })

    if (!result.committed) {
      onlineMessage.value = 'A carta já foi retirada ou a vez mudou.'
    }
  } catch (error) {
    console.error(error)
    onlineMessage.value = 'Erro ao retirar a carta online.'
  }
}

// =====================================================
// COMPUTADOR
// =====================================================

function scheduleCpuDraw() {
  if (
    selectedMode.value !== 'computer' ||
    activeTeam.value !== 1 ||
    gamePhase.value !== 'draw' ||
    screen.value !== 'playing'
  ) {
    return
  }

  if (cpuTimeoutId) clearTimeout(cpuTimeoutId)

  cpuTimeoutId = setTimeout(() => {
    cpuTimeoutId = null

    if (
      screen.value !== 'playing' ||
      selectedMode.value !== 'computer' ||
      activeTeam.value !== 1 ||
      gamePhase.value !== 'draw'
    ) {
      return
    }

    drawCard(true)
  }, 1000)
}

function scheduleCpu() {
  if (
    selectedMode.value !== 'computer' ||
    activeTeam.value !== 1 ||
    gamePhase.value !== 'answer' ||
    screen.value !== 'playing' ||
    timeLeft.value <= 0
  ) {
    return
  }

  if (cpuTimeoutId) clearTimeout(cpuTimeoutId)

  cpuTimeoutId = setTimeout(
    () => {
      cpuTimeoutId = null

      if (
        screen.value !== 'playing' ||
        gamePhase.value !== 'answer' ||
        activeTeam.value !== 1 ||
        !currentQuestion.value ||
        timeLeft.value <= 0
      ) {
        return
      }

      const correct = Math.random() < 0.72
      const correctIndex = Number(currentQuestion.value.answer)

      const wrongOptions = currentQuestion.value.options
        .map((_, index) => index)
        .filter((index) => index !== correctIndex)

      const wrongIndex = wrongOptions.length
        ? wrongOptions[Math.floor(Math.random() * wrongOptions.length)]
        : correctIndex

      submitLocalAnswer(correct ? correctIndex : wrongIndex, true)
    },
    1100 + Math.random() * 900,
  )
}

// =====================================================
// RESPONDER
// =====================================================

function submitAnswer(index) {
  if (!canAnswer.value || answerBusy.value || !currentQuestion.value || timeLeft.value <= 0) {
    return
  }

  if (selectedMode.value === 'online') {
    submitOnlineAnswer(index)
  } else {
    submitLocalAnswer(index)
  }
}

// =====================================================
// RESPOSTAS LOCAIS
// =====================================================

function submitLocalAnswer(index, isComputer = false) {
  if (
    screen.value !== 'playing' ||
    gamePhase.value !== 'answer' ||
    !currentQuestion.value ||
    answerBusy.value ||
    Date.now() - turnStartedAt >= TURN_MS
  ) {
    return
  }

  if (isComputer) {
    if (selectedMode.value !== 'computer' || activeTeam.value !== 1) return
  } else if (!canAnswer.value) {
    return
  }

  answerBusy.value = true

  const question = currentQuestion.value
  const correct = Number(index) === Number(question.answer)
  const team = activeTeam.value

  if (cpuTimeoutId) {
    clearTimeout(cpuTimeoutId)
    cpuTimeoutId = null
  }

  if (correct) {
    scores.value[team] += 100

    const positions = [...pawnPositions.value]
    positions[team] = Math.min(positions[team] + 1, FINISH_POSITION)
    pawnPositions.value = positions

    if (soundEnabled.value) playCorrect()

    answerMessage.value = 'Resposta correcta! +100 pontos. Avançaste uma casa.'

    if (positions[team] >= FINISH_POSITION) {
      answerBusy.value = false
      finishLocalGame(team)
      return
    }
  } else {
    if (soundEnabled.value) playWrong()
    answerMessage.value = 'Resposta incorrecta! O peão mantém-se. Continua a tentar.'
  }

  const category = questionCategoryForPosition(pawnPositions.value[team])
  const nextQuestion = pickQuestion(category, question.id)

  if (!nextQuestion) {
    answerBusy.value = false
    finishLocalTurn()
    return
  }

  currentQuestion.value = nextQuestion
  currentQuestionCategory.value = getQuestionCategory(nextQuestion, category)
  questionNumber.value += 1
  answerBusy.value = false

  if (selectedMode.value === 'computer' && activeTeam.value === 1) {
    scheduleCpu()
  }
}

// =====================================================
// PASSAR TURNO SEM PERGUNTAS
// =====================================================

function skipLocalTurnWithoutQuestion() {
  if (screen.value !== 'playing' || gamePhase.value !== 'draw') return

  stopTimer()

  currentQuestion.value = null
  currentQuestionCategory.value = ''
  answerBusy.value = false

  activeTeam.value = activeTeam.value === 0 ? 1 : 0
  gamePhase.value = 'draw'
  timeLeft.value = TURN_SECONDS
  turnStartedAt = 0

  answerMessage.value = `Não existem perguntas disponíveis. ${activeTeamLabel.value}.`

  if (selectedMode.value === 'computer' && activeTeam.value === 1) {
    scheduleCpuDraw()
  }
}

// =====================================================
// TERMINAR TURNO LOCAL
// =====================================================

function finishLocalTurn() {
  if (screen.value !== 'playing' || gamePhase.value !== 'answer') return

  stopTimer()

  if (soundEnabled.value) playTimeUp()

  currentQuestion.value = null
  currentQuestionCategory.value = ''
  answerBusy.value = false

  activeTeam.value = activeTeam.value === 0 ? 1 : 0
  gamePhase.value = 'draw'
  timeLeft.value = TURN_SECONDS
  turnStartedAt = 0

  answerMessage.value = `${activeTeamLabel.value}. Retirem uma nova carta.`

  if (selectedMode.value === 'computer' && activeTeam.value === 1) {
    scheduleCpuDraw()
  }
}

// =====================================================
// TERMINAR PARTIDA LOCAL
// =====================================================

function finishLocalGame(teamIndex) {
  if (screen.value !== 'playing') return

  stopTimer()

  winnerIndex.value = teamIndex
  gamePhase.value = 'finished'
  currentQuestion.value = null
  currentQuestionCategory.value = ''
  answerBusy.value = false
  timeLeft.value = 0
  screen.value = 'result'

  if (soundEnabled.value) {
    playVictory()
    playTimeUp()
  }
}

// =====================================================
// CÓDIGOS DAS SALAS
// =====================================================

function makeRoomCode() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'
  return Array.from({ length: 6 }, () => chars[Math.floor(Math.random() * chars.length)]).join('')
}

function normalizeRoomCode(value) {
  roomCodeInput.value = String(value || '')
    .toUpperCase()
    .replace(/[^A-Z0-9]/g, '')
    .slice(0, 6)
}

// =====================================================
// CRIAR SALA ONLINE
// =====================================================

async function createOnlineRoom() {
  if (onlineLoading.value) return

  onlineLoading.value = true
  onlineMessage.value = ''

  try {
    const user = await authReady
    myUid.value = user.uid

    const name = roomPlayerName.value.trim() || 'Jogador 1'
    let code = ''
    let created = false

    for (let attempt = 0; attempt < 8 && !created; attempt++) {
      code = makeRoomCode()
      const roomRef = dbRef(database, `rooms/${code}`)

      const result = await runTransaction(roomRef, (currentRoom) => {
        if (currentRoom !== null) return

        return {
          hostId: user.uid,
          playerLimit: roomPlayerCount.value,
          status: 'waiting',
          createdAt: Date.now(),
          players: {
            [user.uid]: {
              uid: user.uid,
              name,
              joinedAt: Date.now(),
              slot: 0,
            },
          },
          game: {
            status: 'waiting',
            phase: 'waiting',
            activeTeam: 0,
            scores: [0, 0],
            pawnPositions: [0, 0],
            diceValues: [0, 0],
            questionNumber: 0,
            usedQuestionIds: [],
            currentQuestion: null,
            currentQuestionCategory: '',
            timeLeft: TURN_SECONDS,
            turnStartedAt: 0,
          },
        }
      })

      if (result.committed) created = true
    }

    if (!created) {
      throw new Error('Não foi possível criar a sala. Tenta novamente.')
    }

    roomCode.value = code
    roomCodeInput.value = code
    isRoomHost.value = true
    selectedMode.value = 'online'

    listenToRoom(code)
  } catch (error) {
    console.error(error)
    onlineMessage.value = error?.message || 'Erro ao criar sala. Verifica o Firebase.'
  } finally {
    onlineLoading.value = false
  }
}

// =====================================================
// ENTRAR NUMA SALA ONLINE
// =====================================================

async function joinOnlineRoom() {
  if (onlineLoading.value) return

  onlineLoading.value = true
  onlineMessage.value = ''

  try {
    const user = await authReady
    myUid.value = user.uid

    const code = roomCodeInput.value.trim().toUpperCase()

    if (!/^[A-Z0-9]{6}$/.test(code)) {
      onlineMessage.value = 'Introduz um código válido de 6 caracteres.'
      return
    }

    const roomRef = dbRef(database, `rooms/${code}`)
    const snapshot = await get(roomRef)

    if (!snapshot.exists()) {
      onlineMessage.value = 'Sala não encontrada.'
      return
    }

    const room = snapshot.val()
    const players = Object.values(room.players || {})
    const existing = players.find((player) => player.uid === user.uid)
    const limit = Number(room.playerLimit) || 4

    if (!existing && room.game?.status === 'playing') {
      onlineMessage.value = 'A partida já começou.'
      return
    }

    if (!existing && players.length >= limit) {
      onlineMessage.value = 'A sala já está cheia.'
      return
    }

    if (!existing) {
      const result = await runTransaction(
        dbRef(database, `rooms/${code}/players`),
        (currentPlayers) => {
          const updated = { ...(currentPlayers || {}) }
          if (updated[user.uid]) return updated
          if (Object.keys(updated).length >= limit) return

          const slots = Object.values(updated).map((player) => Number(player.slot))
          let slot = 0
          while (slots.includes(slot)) slot++

          updated[user.uid] = {
            uid: user.uid,
            name: roomPlayerName.value.trim() || `Jogador ${slot + 1}`,
            joinedAt: Date.now(),
            slot,
          }

          return updated
        },
      )

      if (!result.committed) {
        onlineMessage.value = 'A sala está cheia. Escolhe outra sala.'
        return
      }
    }

    roomCode.value = code
    roomPlayerCount.value = limit
    isRoomHost.value = room.hostId === user.uid
    selectedMode.value = 'online'

    listenToRoom(code)
  } catch (error) {
    console.error(error)
    onlineMessage.value =
      'Não foi possível entrar. Confirma a autenticação e as regras do Firebase.'
  } finally {
    onlineLoading.value = false
  }
}

// =====================================================
// SINCRONIZAR SALA ONLINE
// =====================================================

function listenToRoom(code) {
  if (roomListener) roomListener()
  stopTimer()

  roomListener = onValue(
    dbRef(database, `rooms/${code}`),
    (snapshot) => {
      if (!snapshot.exists()) {
        roomStatus.value = 'A sala não existe ou foi removida.'
        return
      }

      const room = snapshot.val()
      const limit = Number(room.playerLimit) || 4
      const game = room.game || {}

      roomPlayerCount.value = limit
      roomPlayers.value = Object.values(room.players || {}).sort(
        (a, b) => Number(a.slot) - Number(b.slot),
      )

      roomStatus.value =
        `${roomPlayers.value.length}/${limit} jogadores ligados. ` +
        (roomPlayers.value.length < limit
          ? 'À espera dos restantes jogadores.'
          : 'As equipas estão completas.')

      roomGameStatus.value = game.status || 'waiting'
      onlineScores.value = game.scores || [0, 0]

      if (game.status === 'playing') {
        selectedMode.value = 'online'
        screen.value = 'playing'
        gamePhase.value = game.phase || 'dice'
        activeTeam.value = Number(game.activeTeam) || 0
        pawnPositions.value = game.pawnPositions || [0, 0]
        diceValues.value = game.diceValues || [0, 0]
        currentQuestion.value = game.currentQuestion || null
        currentQuestionCategory.value = game.currentQuestionCategory || ''
        questionNumber.value = game.questionNumber || 0
        onlineUsedQuestionIds.value = game.usedQuestionIds || []

        if (game.phase === 'answer') {
          turnStartedAt = Number(game.turnStartedAt) || Date.now()
          const elapsed = Math.floor((Date.now() - turnStartedAt) / 1000)
          timeLeft.value = Math.max(0, TURN_SECONDS - elapsed)
          startOnlineClock()
        } else {
          stopTimer()
          timeLeft.value = TURN_SECONDS
          turnStartedAt = 0
        }
      } else if (game.status === 'finished') {
        stopTimer()
        selectedMode.value = 'online'
        screen.value = 'result'
        gamePhase.value = 'finished'
        onlineScores.value = game.scores || [0, 0]
        pawnPositions.value = game.pawnPositions || [0, 0]

        winnerIndex.value =
          typeof game.winnerTeam === 'number'
            ? game.winnerTeam
            : onlineScores.value[0] === onlineScores.value[1]
              ? -1
              : onlineScores.value[0] > onlineScores.value[1]
                ? 0
                : 1

        timeLeft.value = 0

        if (soundEnabled.value && !onlineGameSeen) {
          playVictory()
          playTimeUp()
        }

        onlineGameSeen = true
      }
    },
    (error) => {
      console.error(error)
      onlineMessage.value = 'Erro ao sincronizar a sala. Verifica as regras do Firebase.'
    },
  )
}

// =====================================================
// INICIAR PARTIDA ONLINE
// =====================================================

async function startOnlineGame() {
  if (!isRoomHost.value || roomPlayers.value.length !== roomPlayerCount.value) {
    onlineMessage.value = `São necessários ${roomPlayerCount.value} jogadores para iniciar.`
    return
  }

  if (!validQuestions.value.length) {
    onlineMessage.value = 'Não existem perguntas disponíveis.'
    return
  }

  try {
    onlineGameSeen = false
    onlineUsedQuestionIds.value = []

    await set(dbRef(database, `rooms/${roomCode.value}/game`), {
      status: 'playing',
      phase: 'dice',
      activeTeam: 0,
      scores: [0, 0],
      pawnPositions: [0, 0],
      diceValues: [0, 0],
      questionNumber: 0,
      usedQuestionIds: [],
      currentQuestion: null,
      currentQuestionCategory: '',
      timeLeft: TURN_SECONDS,
      turnStartedAt: 0,
      startedAt: Date.now(),
    })
  } catch (error) {
    console.error(error)
    onlineMessage.value = 'Não foi possível iniciar. Verifica as regras do Firebase.'
  }
}

// =====================================================
// RELÓGIO ONLINE
// =====================================================

function startOnlineClock() {
  if (timerId || !roomCode.value) return

  lastTick = timeLeft.value

  timerId = setInterval(() => {
    if (
      screen.value !== 'playing' ||
      selectedMode.value !== 'online' ||
      gamePhase.value !== 'answer'
    ) {
      stopTimer()
      return
    }

    const elapsed = Math.floor((Date.now() - turnStartedAt) / 1000)
    const remaining = Math.max(0, TURN_SECONDS - elapsed)
    timeLeft.value = remaining

    if (remaining !== lastTick) {
      lastTick = remaining
      if (soundEnabled.value && remaining > 0) playTick(remaining)
    }

    if (remaining <= 0) finishOnlineTurn()
  }, 200)
}

// =====================================================
// TERMINAR TURNO ONLINE
// =====================================================

async function finishOnlineTurn() {
  if (onlineFinishPending || !roomCode.value) return

  onlineFinishPending = true

  try {
    await runTransaction(dbRef(database, `rooms/${roomCode.value}/game`), (game) => {
      if (!game || game.status !== 'playing' || game.phase !== 'answer') return

      const elapsed = Date.now() - Number(game.turnStartedAt || 0)
      if (elapsed < TURN_MS) return

      return {
        ...game,
        phase: 'draw',
        activeTeam: Number(game.activeTeam) === 0 ? 1 : 0,
        currentQuestion: null,
        currentQuestionCategory: '',
        timeLeft: TURN_SECONDS,
        turnStartedAt: 0,
      }
    })
  } catch (error) {
    console.error(error)
    onlineMessage.value = 'Erro ao mudar o turno online.'
  } finally {
    onlineFinishPending = false
  }
}

// =====================================================
// RESPOSTAS ONLINE
// =====================================================

async function submitOnlineAnswer(index) {
  if (
    !roomCode.value ||
    !currentQuestion.value ||
    answerBusy.value ||
    timeLeft.value <= 0 ||
    playerTeamIndex.value !== activeTeam.value ||
    gamePhase.value !== 'answer' ||
    Date.now() - turnStartedAt >= TURN_MS
  ) {
    return
  }

  answerBusy.value = true

  try {
    const question = currentQuestion.value
    const questionId = question.id
    const correct = Number(index) === Number(question.answer)
    const team = activeTeam.value
    const currentPosition = Number(pawnPositions.value[team]) || 0

    const newPosition = correct ? Math.min(currentPosition + 1, FINISH_POSITION) : currentPosition

    const reachedFinish = correct && newPosition >= FINISH_POSITION
    const category = questionCategoryForPosition(newPosition)

    const nextQuestion = reachedFinish
      ? null
      : selectQuestion(category, onlineUsedQuestionIds.value, questionId)

    if (!reachedFinish && !nextQuestion) {
      throw new Error('Não existem perguntas válidas disponíveis.')
    }

    const gameRef = dbRef(database, `rooms/${roomCode.value}/game`)

    const result = await runTransaction(gameRef, (game) => {
      if (
        !game ||
        game.status !== 'playing' ||
        game.phase !== 'answer' ||
        game.currentQuestion?.id !== questionId ||
        Number(game.activeTeam) !== team
      ) {
        return
      }

      const startedAt = Number(game.turnStartedAt || 0)
      if (Date.now() - startedAt >= TURN_MS) return

      const positions = [...(game.pawnPositions || [0, 0])]
      if (Number(positions[team] || 0) !== currentPosition) return

      const updatedScores = [...(game.scores || [0, 0])]

      if (correct) {
        updatedScores[team] = (updatedScores[team] || 0) + 100
        positions[team] = newPosition
      }

      const baseState = {
        ...game,
        scores: updatedScores,
        pawnPositions: positions,
        answeredBy: myUid.value,
        lastAnswerCorrect: correct,
        lastAnswerAt: Date.now(),
      }

      if (reachedFinish) {
        return {
          ...baseState,
          status: 'finished',
          phase: 'finished',
          winnerTeam: team,
          timeLeft: 0,
          finishedAt: Date.now(),
          currentQuestion: null,
          currentQuestionCategory: '',
        }
      }

      const usedIds = Array.isArray(game.usedQuestionIds) ? game.usedQuestionIds : []

      return {
        ...baseState,
        currentQuestion: nextQuestion,
        currentQuestionCategory: getQuestionCategory(nextQuestion, category),
        questionNumber: (game.questionNumber || 0) + 1,
        usedQuestionIds: [...usedIds, nextQuestion.id],
      }
    })

    if (result.committed) {
      if (soundEnabled.value) {
        if (correct) playCorrect()
        else playWrong()
      }

      answerMessage.value = correct
        ? 'Resposta correcta! +100 pontos. Avançaste uma casa.'
        : 'Resposta incorrecta! O peão mantém-se. Continua a tentar.'
    } else {
      answerMessage.value = 'A resposta já foi processada ou o tempo terminou.'
    }
  } catch (error) {
    console.error(error)
    onlineMessage.value = error?.message || 'A resposta não foi sincronizada. Tenta novamente.'
  } finally {
    answerBusy.value = false
  }
}

// =====================================================
// COPIAR CÓDIGO DA SALA
// =====================================================

async function copyRoomCode() {
  try {
    await copyToClipboard(roomCode.value)
    Notify.create({ type: 'positive', message: 'Código copiado!' })
  } catch {
    onlineMessage.value = `Partilha este código: ${roomCode.value}`
  }
}

// =====================================================
// SAIR DA PARTIDA
// =====================================================

function confirmExit() {
  Dialog.create({
    title: 'Sair da partida?',
    message: 'Queres voltar ao menu?',
    persistent: true,
    ok: { label: 'Sair', color: 'negative' },
    cancel: { label: 'Continuar', flat: true },
  }).onOk(backToMenu)
}

function backToMenu() {
  stopTimer()

  if (roomListener) {
    roomListener()
    roomListener = null
  }

  screen.value = 'menu'
  currentQuestion.value = null
  currentQuestionCategory.value = ''
  answerBusy.value = false
  answerMessage.value = ''

  roomCode.value = ''
  roomCodeInput.value = ''
  roomPlayers.value = []
  roomStatus.value = ''
  onlineMessage.value = ''
  roomGameStatus.value = 'waiting'

  onlineScores.value = [0, 0]
  scores.value = [0, 0]
  pawnPositions.value = [0, 0]
  diceValues.value = [0, 0]

  selectedMode.value = 'computer'
  activeTeam.value = 0
  gamePhase.value = 'dice'
  isRoomHost.value = false
  myUid.value = ''

  timeLeft.value = TURN_SECONDS
  questionNumber.value = 0
  winnerIndex.value = -1

  onlineUsedQuestionIds.value = []
  usedQuestions = []
  turnStartedAt = 0
  onlineFinishPending = false
}

// =====================================================
// LIMPEZA AO SAIR DA PÁGINA
// =====================================================

onBeforeUnmount(() => {
  stopTimer()

  if (roomListener) {
    roomListener()
    roomListener = null
  }
})
</script>

<style scoped>
/* =====================================================
   30 SEGUNDOS — TEMA PREMIUM
   Azul profundo • Azul eléctrico • Dourado • Laranja
===================================================== */

.game-page {
  position: relative;
  isolation: isolate;
  min-height: 100vh;
  color: #f8faff;
  background-color: #050d27;
  background-image:
    radial-gradient(ellipse at 50% 0%, rgba(0, 119, 255, 0.2), transparent 55%),
    radial-gradient(ellipse at 100% 50%, rgba(255, 133, 25, 0.09), transparent 42%),
    linear-gradient(180deg, rgba(4, 12, 37, 0.22), rgba(4, 9, 29, 0.62)),
    url('/capa-30-segundos.png');
  background-size: auto, auto, auto, cover;
  background-position:
    center,
    center,
    center,
    center top;
  background-repeat: no-repeat;
  background-attachment: scroll;
}

.game-shell {
  width: 100%;
  max-width: 1100px;
  margin: 0 auto;
  padding: 0 18px;
}

.game-header,
.brand,
.players-row,
.player-card,
.question-meta,
.bottom-actions {
  display: flex;
  align-items: center;
}

.game-header {
  justify-content: space-between;
  gap: 16px;
  padding: 20px 0 26px;
  margin-bottom: 12px;
  border-bottom: 1px solid rgba(91, 173, 255, 0.25);
}

.brand {
  gap: 14px;
}

.brand-icon {
  display: grid;
  place-items: center;
  flex-shrink: 0;
  width: 58px;
  height: 58px;
  border: 2px solid rgba(255, 216, 91, 0.8);
  border-radius: 17px;
  background: linear-gradient(145deg, #ffca28, #ff7a18 65%, #e84b12);
  color: #07112d;
  box-shadow:
    0 0 18px rgba(255, 174, 40, 0.35),
    inset 0 1px 0 rgba(255, 255, 255, 0.55);
}

.brand-name {
  color: #fff;
  font-size: 21px;
  font-weight: 950;
  letter-spacing: 1.3px;
  text-shadow: 0 2px 12px rgba(0, 0, 0, 0.5);
}

.brand-subtitle {
  color: #b9dcff;
  font-size: 12px;
}
.player-info span,
.result-subtitle {
  color: #bad5f5;
  font-size: 12px;
}

.hero {
  max-width: 650px;
  padding: 25px 0 34px;
}

.hero h1,
.result-screen h1 {
  margin: 10px 0;
  color: #fff;
  font-size: clamp(32px, 6vw, 55px);
  font-weight: 900;
  line-height: 1.08;
  text-shadow: 0 4px 22px rgba(0, 0, 0, 0.45);
}

.hero h1 span {
  color: #ffd34f;
  text-shadow: 0 0 24px rgba(255, 178, 24, 0.3);
}
.hero p {
  max-width: 520px;
  color: #d7e7ff;
  line-height: 1.7;
}

.section-label,
.timer-caption {
  margin-bottom: 14px;
  color: #78c9ff;
  font-size: 11px;
  font-weight: 900;
  letter-spacing: 2px;
}

.mode-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 16px;
}

.mode-card {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 12px;
  overflow: hidden;
  min-width: 0;
  min-height: 160px;
  padding: 22px;
  border: 1px solid rgba(77, 167, 255, 0.45);
  border-radius: 19px;
  background: linear-gradient(145deg, rgba(13, 47, 100, 0.93), rgba(7, 20, 54, 0.95));
  color: #fff;
  text-align: left;
  cursor: pointer;
  box-shadow:
    0 12px 35px rgba(0, 0, 0, 0.28),
    inset 0 1px 0 rgba(255, 255, 255, 0.08);
  transition:
    transform 0.25s ease,
    border-color 0.25s ease,
    box-shadow 0.25s ease,
    background 0.25s ease;
}

.mode-card:hover,
.mode-card.chosen {
  border-color: #ffd34f;
  background: linear-gradient(145deg, rgba(28, 81, 155, 0.97), rgba(20, 32, 78, 0.97));
  box-shadow:
    0 0 0 1px rgba(255, 211, 79, 0.13),
    0 0 26px rgba(34, 145, 255, 0.23),
    0 14px 32px rgba(0, 0, 0, 0.27);
  transform: translateY(-3px);
}

.mode-card strong {
  color: #fff;
  font-size: 16px;
  font-weight: 850;
}
.mode-card span {
  color: #c7defe;
  font-size: 12px;
}

.settings-panel,
.question-card,
.timer-panel {
  margin-top: 24px;
  padding: 24px;
  border: 1px solid rgba(76, 164, 255, 0.43);
  border-radius: 22px;
  background: linear-gradient(145deg, rgba(12, 34, 77, 0.94), rgba(6, 16, 43, 0.94));
  box-shadow:
    0 18px 45px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.07);
  -webkit-backdrop-filter: blur(12px);
  backdrop-filter: blur(12px);
}

.settings-panel {
  padding: 27px;
  border-color: rgba(110, 130, 255, 0.4);
}

:deep(.q-field--outlined .q-field__control) {
  border-radius: 12px;
  background: rgba(4, 14, 37, 0.78);
}
:deep(.q-field--outlined .q-field__control:before) {
  border-color: rgba(111, 182, 255, 0.55);
}
:deep(.q-field--outlined:hover .q-field__control:before) {
  border-color: #59bbff;
}
:deep(.q-field--outlined .q-field__control:after) {
  border-color: #ffd34f;
}
:deep(.q-field__label) {
  color: #9bcfff;
}
:deep(.q-field__native),
:deep(.q-field__input),
:deep(.q-field__marginal),
:deep(.q-field__prefix),
:deep(.q-field__suffix) {
  color: #fff;
}
:deep(.q-field--outlined.q-field--focused .q-field__control) {
  box-shadow: 0 0 0 3px rgba(58, 164, 255, 0.12);
}

.start-button {
  min-height: 54px;
  border: 1px solid rgba(255, 230, 134, 0.55);
  border-radius: 14px;
  background: linear-gradient(100deg, #ffb52e, #ff8325 52%, #ed5b22);
  color: #11172d;
  font-size: 15px;
  font-weight: 950;
  letter-spacing: 0.5px;
  box-shadow:
    0 8px 24px rgba(255, 119, 30, 0.24),
    inset 0 1px 0 rgba(255, 255, 255, 0.3);
  transition:
    transform 0.2s ease,
    filter 0.2s ease,
    box-shadow 0.2s ease;
}

.start-button:hover {
  filter: brightness(1.08);
  transform: translateY(-2px);
  box-shadow: 0 12px 30px rgba(255, 119, 30, 0.34);
}
.online-help,
.note {
  color: #bad5f5;
  font-size: 13px;
  line-height: 1.6;
}

.room-code-card {
  padding: 16px;
  border: 1px solid rgba(255, 211, 79, 0.5);
  border-radius: 14px;
  background: linear-gradient(135deg, rgba(18, 62, 120, 0.95), rgba(11, 24, 63, 0.95));
  text-align: center;
  box-shadow: 0 0 24px rgba(20, 137, 255, 0.13);
}

.room-code {
  color: #ffd34f;
  font-size: 36px;
  font-weight: 900;
  letter-spacing: 8px;
  text-shadow: 0 0 18px rgba(255, 190, 35, 0.22);
}
.room-player {
  padding: 7px;
  margin-top: 6px;
  border: 1px solid rgba(86, 154, 255, 0.22);
  border-radius: 8px;
  background: rgba(5, 17, 43, 0.8);
  color: #eee;
}

.teams-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}
.teams-grid > div {
  padding: 14px;
  border-radius: 15px;
}
.teams-grid > div strong {
  display: block;
  margin-bottom: 8px;
}
.teams-grid .team-a,
.team-a {
  border: 1px solid rgba(71, 151, 255, 0.8);
  border-radius: 15px;
  background: linear-gradient(135deg, rgba(17, 69, 139, 0.92), rgba(10, 31, 75, 0.94));
}
.teams-grid .team-b,
.team-b {
  border: 1px solid rgba(255, 171, 55, 0.8);
  border-radius: 15px;
  background: linear-gradient(135deg, rgba(126, 63, 21, 0.7), rgba(53, 32, 30, 0.93));
}

.playing-screen {
  padding-top: 8px;
}
.game-topline {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}

.timer-panel {
  text-align: center;
  border-color: rgba(255, 211, 79, 0.48);
  background: linear-gradient(145deg, rgba(15, 41, 88, 0.96), rgba(7, 16, 43, 0.96));
}
.timer-caption {
  margin-bottom: 8px;
}
.timer-value {
  margin-bottom: 14px;
  color: #ffd34f;
  font-size: clamp(48px, 7vw, 72px);
  font-weight: 900;
  font-variant-numeric: tabular-nums;
  letter-spacing: 2px;
  text-shadow: 0 0 30px rgba(255, 176, 33, 0.23);
}
.timer-panel.urgent .timer-value {
  color: #ff665d;
  text-shadow: 0 0 24px rgba(255, 72, 72, 0.3);
}

.players-row {
  justify-content: space-between;
  gap: 10px;
  margin-top: 22px;
}
.player-card {
  flex: 1;
  gap: 10px;
  min-width: 0;
  padding: 12px;
  border: 1px solid rgba(73, 155, 255, 0.42);
  border-radius: 15px;
  background: linear-gradient(135deg, rgba(13, 42, 87, 0.95), rgba(7, 21, 51, 0.95));
}
.player-info {
  display: flex;
  flex-direction: column;
  gap: 5px;
  min-width: 0;
}
.player-info strong {
  overflow-wrap: anywhere;
  color: #fff;
  font-size: 13px;
}
.player-info span {
  color: #b7d9ff;
}
.versus {
  color: #ffd34f;
  font-weight: 900;
}
.score-teams {
  margin-top: 20px;
}
.team-card {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.team-card strong {
  font-size: 18px;
}
.team-card span {
  color: #ffd34f;
  font-size: 24px;
  font-weight: 900;
}
.team-card small {
  overflow-wrap: anywhere;
  color: #c4d8f7;
}

.question-meta {
  justify-content: space-between;
  gap: 12px;
  margin-top: 26px;
  color: #b7d9ff;
  font-size: 13px;
}
.question-card h2 {
  margin: 0 0 22px;
  color: #fff;
  font-size: clamp(20px, 4vw, 28px);
  font-weight: 800;
  line-height: 1.35;
}
.answers-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.answer-button {
  display: flex;
  align-items: center;
  gap: 12px;
  min-height: 64px;
  padding: 14px;
  border: 1px solid rgba(75, 157, 255, 0.5);
  border-radius: 14px;
  background: linear-gradient(135deg, rgba(17, 55, 113, 0.94), rgba(10, 27, 64, 0.94));
  color: #fff;
  text-align: left;
  cursor: pointer;
  transition:
    border-color 0.2s ease,
    background 0.2s ease,
    transform 0.2s ease,
    box-shadow 0.2s ease;
}

.answer-button:hover:not(:disabled) {
  border-color: #ffd34f;
  background: linear-gradient(135deg, rgba(27, 86, 159, 0.97), rgba(15, 43, 94, 0.97));
  box-shadow: 0 0 18px rgba(43, 145, 255, 0.2);
  transform: translateY(-2px);
}
.answer-button:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}
.answer-letter {
  display: grid;
  place-items: center;
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border: 1px solid rgba(255, 211, 79, 0.45);
  border-radius: 8px;
  background: linear-gradient(135deg, #ffd34f, #ff922b);
  color: #111a36;
  font-weight: 900;
}
.answer-feedback {
  margin-top: 16px;
  color: #ffd34f;
  font-weight: 700;
}

.bottom-actions {
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 18px;
  color: #bad5f5;
  font-size: 12px;
}
.result-screen {
  padding: 45px 0;
  text-align: center;
}
.result-screen h1 {
  font-size: clamp(30px, 5vw, 48px);
}
.result-score-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  margin-top: 28px;
}
.result-score {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 20px;
  border: 1px solid rgba(80, 163, 255, 0.43);
  border-radius: 18px;
  background: linear-gradient(140deg, rgba(17, 47, 99, 0.94), rgba(8, 20, 49, 0.94));
}
.result-score strong {
  color: #ffd34f;
  font-size: 36px;
  text-shadow: 0 0 20px rgba(255, 185, 44, 0.18);
}
.result-score span,
.result-score small {
  color: #c4d8f7;
}

.game-footer {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  margin-top: 45px;
  padding: 20px 0;
  border-top: 1px solid rgba(96, 164, 255, 0.25);
  color: #a9c9f4;
  font-size: 10px;
  letter-spacing: 1.5px;
}

.mode-card:focus-visible,
.answer-button:focus-visible,
.start-button:focus-visible {
  outline: 3px solid #ffd34f;
  outline-offset: 3px;
}

@media (prefers-reduced-motion: reduce) {
  .mode-card,
  .answer-button,
  .start-button {
    transition: none;
  }
}

@media (max-width: 650px) {
  .game-shell {
    padding: 0 12px;
  }
  .game-header {
    padding: 14px 0 20px;
  }
  .brand {
    gap: 10px;
  }
  .brand-icon {
    width: 48px;
    height: 48px;
    border-radius: 14px;
  }
  .brand-name {
    font-size: 17px;
  }
  .hero {
    padding: 20px 0 26px;
  }
  .mode-grid {
    grid-template-columns: 1fr;
    gap: 12px;
  }
  .mode-card {
    min-height: 110px;
    padding: 17px;
  }
  .settings-panel,
  .question-card,
  .timer-panel {
    padding: 16px;
    border-radius: 17px;
  }
  .answers-grid {
    grid-template-columns: 1fr;
  }
  .players-row {
    align-items: stretch;
  }
  .player-card {
    flex-direction: column;
    align-items: flex-start;
  }
  .player-info strong {
    font-size: 11px;
  }
  .timer-value {
    font-size: 44px;
  }
  .room-code {
    font-size: 28px;
    letter-spacing: 5px;
  }
  .teams-grid {
    gap: 8px;
  }
  .teams-grid > div {
    padding: 10px;
  }
  .team-card span {
    font-size: 19px;
  }
  .game-footer {
    flex-direction: column;
    align-items: center;
    text-align: center;
    margin-top: 30px;
  }
}
</style>
