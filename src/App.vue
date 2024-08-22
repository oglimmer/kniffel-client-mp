<script setup lang="ts">
import { ref, watch } from 'vue'
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

</script>

<template>
    <div v-if="!gameData?.gameId">
        <h1>API Server</h1>
        <select v-model="apiServer">
            <option>https://api-kniffel.oglimmer.com</option>
            <option>https://api-rust-kniffel.oglimmer.com</option>
            <option>http://localhost:8080</option>
        </select>
        <br/><br/>
        <table border="1" cellpadding="0" cellspacing="0">
            <tr>
                <td>
                    <h1>Create New Game</h1>
                    <ul>
                        <li v-for="ply in names" :key="ply.index">
                            Player {{ ply.index+1 }}'s name: <input type="text" v-model="ply.name" />
                        </li>
                    </ul>
                    <button @click="names.push({index: names.length, name: ''})">Add Name</button> &nbsp;
                    <button @click="createGame">Create Game</button>
                </td>
                <td>
                    <div style="padding:20px">
                        <h1>Join Existing Game</h1>
                        Game-ID: <input v-model="joinGameId"><br />
                        <button @click="joinGame">Join Game</button>
                    </div>
                </td>
            </tr>
        </table>
    </div>
    <div v-if="gameData?.gameId">
        <div v-if="!myName">
            <h1>Pick your player name</h1>
            <select v-model="myName">
                <option v-for="ply in gameData?.playerData" :key="ply.name">{{ ply.name }}</option>
            </select>
        </div>
        <div v-if="myName">
            <h1>Game Scores</h1>
            for {{ gameData.gameId }}
            <ul>
                <li v-for="ply in gameData?.playerData" :key="ply.name">
                    Player {{ ply.name }} - Score: {{ ply.score }}
                </li>
            </ul>
            <h1 class="mt-20" v-if="!gameEnded">
                Current player: {{ gameData.currentPlayerName }}
            </h1>
            <div v-if="gameEnded">
                <h1>Game Ended</h1>
            </div>
            <div v-if="gameData.currentPlayerName === myName && !gameEnded">
                <div v-if="gameData?.state === 'ROLL'">
                    <h3>Roll round: {{ gameData?.rollRound }}</h3>
                    <div> These types are still available:
                        {{ gameData?.availableBookingTypes }}
                    </div>
                    <h3 style="margin-top: 30px;">Select the dice to keep:</h3>
                    <ul>
                        <li v-for="(die, idx) in gameData.diceRolls" :key="idx">
                            {{ die }} <input type="checkbox" v-model="rerollSelection[idx]" />
                        </li>
                    </ul>
                    <button @click="reroll">Roll</button>
                </div>
                <div v-if="gameData?.state === 'BOOK'">
                    <h1>Final dice rolls: {{  gameData.diceRolls }}</h1>
                    <div class="mt-20">
                    Select the booking type:
                    </div>
                    <select v-model="selectedBookingType">
                        <option v-for="cat in gameData.availableBookingTypes" :key="cat" :value="cat">{{ cat }}</option>
                    </select>
                    <button @click="book">Book</button>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
button,input {
    margin: 10px;
}
.mt-20 {
    margin-top: 20px;
}
</style>