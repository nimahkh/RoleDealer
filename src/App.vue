<template>
  <div class="app">
    <RoleInput v-if="gameState === 'setup'" @start="startGame" @resume="resumeGame" :hasSavedGame="hasSavedGame" />
    <GameBoard v-else-if="gameState === 'playing'" :roles="gameRoles" :scenario="scenario" @finish="finishGame" />
    <GameSummary
      v-else-if="gameState === 'summary'"
      :players="players"
      :scenario="scenario"
      @updatePlayers="updatePlayers"
      @playAgain="playAgain"
      @backToSetup="resetGame"
    />
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import RoleInput from './components/RoleInput.vue'
import GameBoard from './components/GameBoard.vue'
import GameSummary from './components/GameSummary.vue'
import { getGameState, getGameRoles, getGameScenario, getPlayers, saveGameState, saveGameRoles, saveGameScenario, savePlayers, clearAllGameData } from './utils/storage.js'

export default {
  name: 'App',
  components: {
    RoleInput,
    GameBoard,
    GameSummary
  },
  setup() {
    const gameState = ref('setup')
    const gameRoles = ref([])
    const players = ref([])
    const scenario = ref(null)
    const hasSavedGame = ref(false)

    onMounted(() => {
      // Check if there's saved game data
      const savedState = getGameState()
      if (savedState) {
        hasSavedGame.value = true
      }
    })

    const startGame = ({ scenario: selectedScenario, assignments }) => {
      clearAllGameData()
      scenario.value = selectedScenario
      gameRoles.value = assignments
      players.value = []
      gameState.value = 'playing'
      hasSavedGame.value = true
      saveGameState('playing')
      saveGameScenario(selectedScenario)
      saveGameRoles(assignments)
    }

    const resumeGame = () => {
      const savedState = getGameState()
      const savedRoles = getGameRoles()
      const savedScenario = getGameScenario()
      const savedPlayers = getPlayers()

      scenario.value = savedScenario

      if (savedState === 'summary' && savedPlayers) {
        // Resume to summary
        players.value = savedPlayers
        gameState.value = 'summary'
      } else if (savedState === 'playing' && savedRoles) {
        // Resume to playing
        gameRoles.value = savedRoles
        gameState.value = 'playing'
      }
    }

    const finishGame = (playerList) => {
      players.value = playerList
      gameState.value = 'summary'
      saveGameState('summary')
      savePlayers(playerList)
    }

    const updatePlayers = (playerList) => {
      players.value = playerList
      const playersById = new Map(playerList.map((player) => [player.id, player]))
      gameRoles.value = gameRoles.value.map((assignment) => ({
        ...assignment,
        ...(playersById.get(assignment.id) || {})
      }))
    }

    const shuffleArray = (items) => {
      const shuffled = [...items]
      for (let i = shuffled.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1))
        ;[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]]
      }
      return shuffled
    }

    const playAgain = () => {
      // Reset revealed state on all assignments but keep the same role assignments and players
      gameRoles.value = gameRoles.value.map((assignment) => ({
        ...assignment,
        revealed: false
      }))

      gameState.value = 'playing'
      // Keep players.value the same - they already know their roles!
      saveGameScenario(scenario.value)
      saveGameRoles(gameRoles.value)
      saveGameState('playing')
    }

    const resetGame = () => {
      // Full reset - go back to setup
      gameState.value = 'setup'
      gameRoles.value = []
      players.value = []
      scenario.value = null
      hasSavedGame.value = false
      clearAllGameData() // Clear all data including notes and roles
    }

    return {
      gameState,
      gameRoles,
      players,
      scenario,
      hasSavedGame,
      startGame,
      resumeGame,
      finishGame,
      playAgain,
      updatePlayers,
      resetGame
    }
  }
}
</script>

<style scoped>
.app {
  width: 100%;
  min-height: 100vh;
  overflow-x: hidden;
}
</style>
