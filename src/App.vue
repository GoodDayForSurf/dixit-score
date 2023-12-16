<script setup>
import { reactive, ref } from 'vue';
import PlayerButton from './components/player-button.vue'
import Card from './components/card.vue'

const state = reactive({
  playersAmount: 0,
  step: 'set-players-amount',
  score: {},
})

const hints = {
  'set-players-amount': 'Select players amount',
  'set-target-card-owner': 'Set of target card player',
  'setting-players-choice': 'Set choices of players',
  'player-is-choosing-card': 'Select card',
  'ready-show-target-card': 'Show target card',
  'set-selected-card-owners': 'Set owners for other selected cards',
  'score-counted': 'Game round!'
};
let targetCardOwner = ref(null);
const activePlayer = ref(null);
const cardsSelection = reactive([]);
const playersChoices = reactive(new Map());
const selectedCard = ref(null);
const cardsOwners = ref(new Map());
let needConfirm = ref(false);

const lastState = localStorage.getItem('dixit-game-state');
if(lastState) {
  restoreState(JSON.parse(lastState));
  startRound();
}

function setPlayersAmount(amount) {
  state.playersAmount = amount;
  state.step = 'set-target-card-owner';
  state.score = {};
  new Array(amount).fill(0).forEach((val, i) => state.score[i+1]=0)
}

function onPlayerActive(playerId) {
  if (['setting-players-choice', 'ready-show-target-card'].includes(state.step)) {
    activePlayer.value = playerId
  } else if (activePlayer.value === playerId && state.step === 'player-is-choosing-card') {
    cancelActive();
    return;
  }

  state.step = 'player-is-choosing-card';
}

function onCardClick(nmb) {
  if (state.step === 'player-is-choosing-card') {
    cardsSelection.forEach((card) => card?.delete(activePlayer.value));
    cardsSelection[nmb] = cardsSelection[nmb] || new Set();
    cardsSelection[nmb].add(activePlayer.value);

    playersChoices.set(activePlayer.value, nmb);

    cancelActive();

    if (playersChoices.size === state.playersAmount - 1) {
      state.step = 'ready-show-target-card';
    }
  } else if (state.step === 'ready-show-target-card') {
    if(needConfirm.value && selectedCard.value === nmb) {
      needConfirm.value = false;
      cardsOwners.value.set(nmb, targetCardOwner.value);
      state.step = 'set-selected-card-owners';
      selectedCard.value = null;
    } else {
      selectedCard.value = nmb;
      needConfirm.value = true;
    }
  } else if (state.step === 'set-selected-card-owners' && cardsSelection[nmb]?.size) {
    selectedCard.value = nmb;
  }
}

function setTargetCardOwner(playerId) {
  if (!targetCardOwner.value) {
    targetCardOwner.value = playerId;
    needConfirm.value = true;
  } else {
    if(needConfirm.value && targetCardOwner.value === playerId) {
      needConfirm.value = false;
      state.step ='setting-players-choice';
    } else {
      targetCardOwner.value = playerId;
    }
  }
}

function setCardOwner(playerId) {
  const cardsOwnersMap = cardsOwners.value;

  if (needConfirm.value && cardsOwnersMap.get(selectedCard.value) === playerId) {
    needConfirm.value = false;
    selectedCard.value = null;

    if (!cardsSelection.find((card, i)=> {
      return card && !cardsOwners.value.get(i)
    })) {
      countScore();
    }
  } else {
    cardsOwnersMap.set(selectedCard.value, playerId);
    needConfirm.value = true;
  }
}

function countScore() {
  const targetCardNumber = Object.keys(state.score).find((playerId) => cardsOwners.value.get(+playerId) === targetCardOwner.value);
  const targetSelections = cardsSelection[targetCardNumber];

  if(!targetSelections || targetSelections.size === 0 || targetSelections.size === state.playersAmount - 1) {
    Object.keys(state.score).forEach((key) => {
      const playerId = +key;
      if (playerId !== targetCardOwner.value) {
        state.score[playerId] += 2
      }
    });
  } else if(targetSelections?.size) {
    state.score[targetCardOwner.value] += 3;
    [...targetSelections].forEach((playerId) => state.score[playerId] += 3)
  }

  cardsSelection.forEach((selections, cardNumber) => {
    if(cardNumber === +targetCardNumber) {
      return;
    }

    const cardOwnerId = cardsOwners.value.get(cardNumber);
    state.score[cardOwnerId] +=selections?.size || 0;
  });

  state.step = 'score-counted';
  save();
}

function startRound() {
  targetCardOwner.value = null;
  activePlayer.value = null;
  cardsSelection.length = 0;
  playersChoices.value = new Map();
  selectedCard.value = null;
  cardsOwners.value = new Map();
  state.step = 'set-target-card-owner';
}

function cancelActive() {
  state.step = 'setting-players-choice';
  activePlayer.value = 0;
}

function save() {
  localStorage.setItem('dixit-game-state', JSON.stringify(state))
}

function restoreState(lastState) {
  state.playersAmount = lastState.playersAmount;
  state.score = lastState.score;
}

function startNewGame(e) {
  let text = "Reset score and start new Game?";
  if (confirm(text) == true) {
    localStorage.removeItem('dixit-game-state')
    location.reload();
  }
}
</script>

<template>
  <div class="board">
    <div class="hint">{{hints[state.step]}}
      <template v-if="Object.keys(state.score).length && state.step === 'set-target-card-owner'">
        or
        <span v-if="Object.keys(state.score).length && state.step === 'set-target-card-owner'"
              class="new-game-btn"
              @click="startNewGame"
        >New Game</span>
      </template>
      <div v-if="needConfirm" class="confirm-marker">Confirm?</div>
    </div>
    <div v-if="state.step === 'set-players-amount'"
         class="select-amount-panel"
    >
      <button v-for="i in 4"
              :key="i"
              @click="setPlayersAmount(i + 2)"
              style="cursor: pointer"
      >{{i + 2}}</button>
    </div>

    <div v-if="state.step !== 'set-players-amount'"
         id="score-board"
    >
      <div class="players-container">
        <player-button v-for="i in state.playersAmount"
                       :key="i"
                       :id="i"
                       :isOwner="i === targetCardOwner"
                       :disabled="i !== activePlayer && activePlayer > 0"
                       :score="state.score[i]"
                       @activate="state.step === 'setting-players-choice' && targetCardOwner !== i ? onPlayerActive(i) : null"
                       @click="{
                         'set-target-card-owner': () => setTargetCardOwner(i),
                         'set-selected-card-owners': () => selectedCard && setCardOwner(i)
                       }[state.step]"
        />
      </div>

      <div class="center-panel">
        <div v-if="state.step === 'score-counted'"
             class="control"
             @click="startRound()"
             style="cursor: pointer"
        >R</div>
      </div>

      <div class="cards-container">
        <card v-for="i in state.playersAmount" :key="i"
              @click="onCardClick(i)"
              :players="cardsSelection[i]"
              :owner="cardsOwners.get(i)"
              :is-target="cardsOwners.get(i) === targetCardOwner"
              :is-selected="selectedCard === i"
              :msg="i" />
      </div>
    </div>
  </div>
</template>
<style src="./App.scss" lang="scss"></style>