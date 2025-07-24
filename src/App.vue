<script setup lang="ts">
import { ref, watch, computed } from 'vue'
import type { Ref } from 'vue'
import createClient from 'openapi-fetch';
import type { components, paths } from '@/api/v1';

let client = createClient<paths>();

interface PlayerInformation {
    index: number;
    name: string;
}

const names : Ref<PlayerInformation[]> = ref([{index: 0, name: ''}, {index: 1, name: ''}]);
const gameData : Ref<components["schemas"]["GameResponse"]|undefined> = ref();
const rerollSelection = ref([false, false, false, false, false]);
// the booking type selected in the dropdown box
const selectedBookingType : Ref<components["schemas"]["GameResponse"]["usedBookingTypes"]|undefined> = ref();
const apiServer = ref(`${__API_URL__}`);
const myName = ref('');
const joinGameId = ref('');
const gameEnded = ref(false);

async function createGame() {
    client = createClient<paths>({ baseUrl: apiServer.value });
    const { data, error } = await client.POST("/api/v1/game/{arg0}", {
        body: {
            playerNames: names.value.map(n => n.name)
        }
    });
    if (error) {
        console.error(error);
    } else {
        gameData.value = data;
    }
}

async function reroll() {
    const diceToKeep : number[] = [];
    if (gameData.value) {
        for (let i = 0; i < gameData.value.diceRolls.length; i++) {
            if (rerollSelection.value[i]) {
                diceToKeep.push(gameData.value.diceRolls[i]);
            }
        }
        const { data, error } = await client.POST("/api/v1/game/{game_id}/roll", {
            params: {
                path: {
                    game_id: gameData.value.gameId
                }
            },
            body: {
                diceToKeep
            }
        });
        if (error) {
            console.error(error);
        } else {
            gameData.value = data;
        }
        selectedBookingType.value = undefined;
        if (gameData.value) {
            for (let i = 0; i < gameData.value.diceRolls.length; i++) {
                const idxToKeep = diceToKeep.indexOf(gameData.value.diceRolls[i]);
                if (idxToKeep === -1) {
                    rerollSelection.value[i] = false;
                } else {
                    rerollSelection.value[i] = true;
                    diceToKeep.splice(idxToKeep, 1);
                }
            }
        }
    }
}

// simple REST API call to send the booking type
async function book() {
  if (gameData.value) {
    const { data, error } = await client.POST("/api/v1/game/{game_id}/book", {
      params: {
        path: {
            game_id: gameData.value.gameId
        }
      },
      body: {
        bookingType: selectedBookingType.value?.toString() ?? ""
      }
    });
    if (error) {
        console.error(error);
    } else {
        gameData.value = data;
        if (gameData.value.state === 'ENDED') {
            gameEnded.value = true;
        }
    }
    rerollSelection.value = [false, false, false, false, false];
  }
}

const joinGame = async () => {
    client = createClient<paths>({ baseUrl: apiServer.value });
    const { data, error } = await client.GET(`/api/v1/game/{game_id}`, {
        params: {
            path: {
                game_id: joinGameId.value
            }
        }
    });
    if (error) {
        console.error(error);
    } else {
        gameData.value = data;
    }
}

const reload = () => {
    setTimeout(async () => {
        const { data, error } = await client.GET(`/api/v1/game/{game_id}`, {
            params: {
                path: {
                    game_id: gameData.value?.gameId ?? "?"
                }
            }
        })
        if (error) {
            console.error(error);
        } else {
            gameData.value = data;
        }
        if (gameData.value?.currentPlayerName !== myName.value) {
            reload();
        } else {
            if (gameData.value.state === 'ENDED') {
                gameEnded.value = true;
            }
        }
    }, 500)
}

watch(() => gameData.value?.currentPlayerName, () => {
    if (gameData.value?.currentPlayerName !== myName.value) {
        reload();
    }
})

const canCreateGame = computed(() => {
    return names.value.some(player => player.name.trim() !== '');
});

</script>

<template>
  <div class="game-container animate-fadeIn">
    <!-- Game Setup Screen -->
    <div v-if="!gameData?.gameId" class="animate-slideIn">
      <div class="game-header">
        <h1>🎲 Kniffel Game</h1>
        <p class="text-lg text-soft">Welcome to the ultimate dice game experience</p>
      </div>

      <div class="form-group">
        <label for="api-server">Choose API Server:</label>
        <select id="api-server" v-model="apiServer" class="select">
          <option>https://api-kniffel.oglimmer.com</option>
          <option>https://api-rust-kniffel.oglimmer.com</option>
          <option>http://localhost:8080</option>
        </select>
      </div>

      <div class="game-setup">
        <div class="card">
          <h2>🆕 Create New Game</h2>
          <div class="form-group">
            <label>Players:</label>
            <ul class="player-list">
              <li v-for="ply in names" :key="ply.index" class="player-item">
                <span class="player-number">Player {{ ply.index + 1 }}:</span>
                <input 
                  type="text" 
                  v-model="ply.name" 
                  class="input" 
                  :placeholder="`Enter player ${ply.index + 1} name`"
                />
              </li>
            </ul>
          </div>
          <div class="form-row">
            <button @click="names.push({index: names.length, name: ''})" class="btn btn-secondary">
              ➕ Add Player
            </button>
            <button @click="createGame" class="btn btn-primary btn-lg" :disabled="!canCreateGame">
              🎮 Create Game
            </button>
          </div>
        </div>

        <div class="card">
          <h2>🔗 Join Existing Game</h2>
          <div class="form-group">
            <label for="game-id">Game ID:</label>
            <input 
              id="game-id"
              v-model="joinGameId" 
              class="input" 
              placeholder="Enter game ID"
            />
          </div>
          <button @click="joinGame" class="btn btn-primary btn-lg" :disabled="!joinGameId.trim()">
            🚪 Join Game
          </button>
        </div>
      </div>
    </div>

    <!-- Player Selection Screen -->
    <div v-if="gameData?.gameId && !myName" class="game-board animate-fadeIn">
      <div class="card">
        <h2>👤 Select Your Player</h2>
        <div class="form-group">
          <label for="player-select">Choose your player name:</label>
          <select id="player-select" v-model="myName" class="select">
            <option value="">-- Select Player --</option>
            <option v-for="ply in gameData?.playerData" :key="ply.name" :value="ply.name">
              {{ ply.name }}
            </option>
          </select>
        </div>
      </div>
    </div>

    <!-- Game Board -->
    <div v-if="myName && gameData?.gameId" class="game-board animate-fadeIn">
      <!-- Game Header -->
      <div class="game-status">
        <h2>🎯 Game {{ gameData.gameId }}</h2>
        <div class="players-grid">
          <div 
            v-for="ply in gameData?.playerData" 
            :key="ply.name" 
            class="card"
            :class="{ 'current-player': ply.name === gameData.currentPlayerName }"
          >
            <h4>{{ ply.name }}</h4>
            <div class="badge badge-primary">{{ ply.score }} points</div>
            <div v-if="ply.name === gameData.currentPlayerName" class="badge badge-success">
              Current Turn
            </div>
          </div>
        </div>
      </div>

      <!-- Game Ended -->
      <div v-if="gameEnded" class="game-ended animate-bounce">
        <h1>🏆 Game Finished!</h1>
        <p>Congratulations to all players!</p>
      </div>

      <!-- Current Player Turn -->
      <div v-if="gameData.currentPlayerName === myName && !gameEnded">
        <!-- Rolling Phase -->
        <div v-if="gameData?.state === 'ROLL'" class="card">
          <div class="roll-info">
            <div class="badge badge-primary">Round {{ gameData?.rollRound }}</div>
          </div>
          
          <div class="available-types">
            <strong>Available booking types:</strong> {{ gameData?.availableBookingTypes?.join(', ') }}
          </div>

          <h3>🎲 Select dice to keep:</h3>
          <div class="dice-grid">
            <div 
              v-for="(die, idx) in gameData.diceRolls" 
              :key="idx" 
              class="dice-item transition-all"
              :class="{ 'dice-selected': rerollSelection[idx] }"
              @click="rerollSelection[idx] = !rerollSelection[idx]"
            >
              <div class="dice-value">{{ die }}</div>
              <input 
                type="checkbox" 
                v-model="rerollSelection[idx]" 
                class="checkbox"
                :id="`dice-${idx}`"
              />
              <label :for="`dice-${idx}`" class="sr-only">Keep dice {{ die }}</label>
            </div>
          </div>
          
          <button @click="reroll" class="btn btn-primary btn-lg">
            🎲 Roll Dice
          </button>
        </div>

        <!-- Booking Phase -->
        <div v-if="gameData?.state === 'BOOK'" class="booking-section">
          <h2>🎯 Final Roll!</h2>
          <div class="final-dice">
            <div v-for="(die, idx) in gameData.diceRolls" :key="idx" class="final-dice-value">
              {{ die }}
            </div>
          </div>
          
          <div class="form-group">
            <label for="booking-type">Select booking category:</label>
            <select id="booking-type" v-model="selectedBookingType" class="select">
              <option value="">-- Choose Category --</option>
              <option v-for="cat in gameData.availableBookingTypes" :key="cat" :value="cat">
                {{ cat }}
              </option>
            </select>
          </div>
          
          <button 
            @click="book" 
            class="btn btn-success btn-lg" 
            :disabled="!selectedBookingType"
          >
            ✅ Book Score
          </button>
        </div>
      </div>

      <!-- Waiting for other players -->
      <div v-if="gameData.currentPlayerName !== myName && !gameEnded" class="card">
        <div class="text-center">
          <h3>⏳ Waiting for {{ gameData.currentPlayerName }}</h3>
          <div class="animate-pulse">It's their turn to play...</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.text-lg {
  font-size: 1.125rem;
  line-height: 1.75rem;
}

.text-soft {
  color: var(--color-text-soft);
}

.text-center {
  text-align: center;
}

.dice-selected {
  border-color: var(--color-success) !important;
  background: var(--success-50);
  transform: translateY(-4px);
  box-shadow: var(--shadow-lg);
}

.dice-selected .dice-value {
  background: var(--color-success);
  color: var(--white);
  animation: bounce 0.5s ease-in-out;
}

@media (prefers-color-scheme: dark) {
  .dice-selected {
    background: var(--success-900);
  }
}

/* Enhanced dice animations */
.dice-item {
  cursor: pointer;
  position: relative;
}

.dice-item input[type="checkbox"] {
  position: absolute;
  opacity: 0;
  cursor: pointer;
}

.dice-item:hover {
  transform: translateY(-2px) scale(1.02);
}

.dice-item:active {
  transform: translateY(0) scale(0.98);
}

/* Loading states */
.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none !important;
}

/* Game status enhancements */
.current-player {
  animation: pulse 2s infinite;
  border-color: var(--color-primary) !important;
}

.current-player .badge-success {
  animation: bounce 1s infinite;
}

/* Responsive text sizing */
@media (max-width: 640px) {
  .text-lg {
    font-size: 1rem;
    line-height: 1.5rem;
  }
  
  h1 {
    font-size: 2rem;
  }
  
  h2 {
    font-size: 1.5rem;
  }
  
  h3 {
    font-size: 1.25rem;
  }
}

@media (max-width: 480px) {
  .game-header h1 {
    font-size: 1.75rem;
  }
  
  .card {
    padding: 1rem;
  }
  
  .btn-lg {
    padding: 0.75rem 1.5rem;
    font-size: 0.875rem;
  }
}
</style>