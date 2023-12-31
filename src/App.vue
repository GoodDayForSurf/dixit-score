<script setup>
import { computed, reactive, ref } from 'vue';
import PlayerButton from './components/player-button.vue'
import Card from './components/card.vue'

const hints = {
  'set-players-amount': 'Number of players?',
  'on-off-player': 'Select player',
  'set-target-card-owner': 'Who is active player?',
  'setting-players-choice': 'Specify players votes',
  'ready-show-target-card': 'Specify target card',
  'set-selected-card-owners': 'Specify owners of voted cards',
  'score-counted': 'Round!'
};
let playersAmount = 0;
const step = ref('set-players-amount');
const previousStep = ref();
const score = reactive(new Map());
let targetCardOwner = ref(null);
const targetCardNmb = ref(null);
const cardsSelection = reactive(new Map());
const playersChoices = reactive(new Map());
const selectedCard = ref(null);
const selectedPlayers = reactive(new Set());
const disabledPlayers = reactive(new Set());
const cardsOwners = reactive(new Map());
let needConfirm = ref(false);

const isNewGame = computed(() => ![...score.values()].find((score) => score !== 0))

const lastState = localStorage.getItem('dixit-game-state');

if(lastState) {
  restoreState(JSON.parse(lastState));
  startRound();
}

function setPlayersAmount(amount) {
  playersAmount = amount;
  score.clear();
  new Array(amount).fill(0).forEach((val, i) => score.set(i + 1, 0));
  setStep('set-target-card-owner');
}

function setStep(newStep) {
  previousStep.value = step.value;
  step.value = newStep;
}

function onPLayerBtnClick(playerId) {
  if(disabledPlayers.has(playerId) && step.value !== 'on-off-player') {
    return;
  } else if(step.value === 'set-target-card-owner') {
    setTargetCardOwner(playerId);
  } else if(step.value === 'on-off-player') {
    if (disabledPlayers.has(playerId)){
      disabledPlayers.delete(playerId);
    } else {
      disabledPlayers.add(playerId);
    }
    setStep(previousStep.value);
    return;
  }  else if(playerId === targetCardOwner.value) {
    return;
  } else if(step.value === 'setting-players-choice') {
    needConfirm.value = false;

    if(selectedPlayers.has(playerId)) {
      selectedPlayers.delete(playerId);
      cardsSelection.get(selectedCard.value)?.delete(playerId);
    } else {
      selectedPlayers.add(playerId);
    }

    if(selectedCard.value) {
      setCardVotes(selectedCard.value);
    }

  } else if (step.value === 'set-selected-card-owners'){
    const wrongCard = [...cardsOwners.entries()].find(([_, vote]) => vote === playerId);

    wrongCard && cardsOwners.delete(wrongCard[0]);
    cardsOwners.set(selectedCard.value, playerId);

    selectedCard.value = null;

    if (![...cardsSelection.keys()].find((cardId) => !cardsOwners.get(cardId))) {
      countScore();
    }
  }
}

function isAllPlayersVoted(){
  return (playersAmount - 1 - disabledPlayers.size) === [...cardsSelection.values()]
      .map((players) => [...players])
      .flat()
      .filter(player => !disabledPlayers.has(player))
      .length;
}

function setTargetCardOwner(playerId) {
  if (!targetCardOwner.value) {
    targetCardOwner.value = playerId;
    needConfirm.value = true;
  } else {
    if(needConfirm.value && targetCardOwner.value === playerId) {
      needConfirm.value = false;
      setStep('setting-players-choice');
    } else {
      targetCardOwner.value = playerId;
    }
  }
}

function onCardClick(cardNmb) {
  if (step.value === 'setting-players-choice') {
    if(needConfirm.value && selectedCard.value === cardNmb) {
      needConfirm.value = false;
      selectedPlayers.clear();
      selectedCard.value = null;

      if(isAllPlayersVoted()) {
        setStep('ready-show-target-card');
      }
    } else if(selectedPlayers.size) {
      selectedCard.value = cardNmb;
      setCardVotes(cardNmb)
      needConfirm.value = true;
      return;
    } else {
      selectedCard.value = cardNmb;
    }
  } else if (step.value === 'ready-show-target-card') {
    if(needConfirm.value && selectedCard.value === cardNmb) {
      needConfirm.value = false;
      cardsOwners.set(cardNmb, targetCardOwner.value);
      targetCardNmb.value = cardNmb;
      selectedCard.value = null;

      if( cardsSelection.get(cardNmb)?.size === playersAmount - 1) {
        countScore();
        return;
      } else {
        setStep('set-selected-card-owners');
      }
    } else {
      selectedCard.value = cardNmb;
      needConfirm.value = true;
    }
  } else if (targetCardNmb.value !== cardNmb && step.value === 'set-selected-card-owners' && cardsSelection.get(cardNmb)?.size) {
    selectedCard.value = cardNmb;
  }
}

function setCardVotes(cardNmb) {
  cardsSelection.forEach(
      (card, cardId) => {
        selectedPlayers.forEach((playerId) => card?.delete(playerId));
        if(card.size === 0 ) {
          cardsSelection.delete(cardId);
        }
      }
  );

  selectedPlayers.size && cardsSelection.set(cardNmb, new Set([...selectedPlayers]));
}


function countScore() {
  const targetCardVotes = cardsSelection.get(targetCardNmb.value);
  const addScore = (playerId, scoreValue) => score.set(+playerId, (score.get(+playerId) || 0) + scoreValue);

  if(!targetCardVotes || targetCardVotes.size === 0 || targetCardVotes.size === playersAmount - 1) {
    [...score.keys()].forEach((playerId) => {
      if (!disabledPlayers.has(playerId) && playerId !== targetCardOwner.value) {
        addScore(playerId, 2);
      }
    });
  } else if(targetCardVotes?.size) {
    addScore(targetCardOwner.value, 3);
    [...targetCardVotes].forEach((playerId) => addScore(playerId, 3));
  }

  cardsSelection.forEach((selections, cardNumber) => {
    if(cardNumber === targetCardNmb.value) {
      return;
    }

    const cardOwnerId = cardsOwners.get(cardNumber);
    addScore(+cardOwnerId, selections.size);
  });

  setStep('score-counted');
  saveState();
}

function startRound() {
  targetCardOwner.value = null;
  cardsSelection.clear();
  playersChoices.clear();
  selectedCard.value = null;
  cardsOwners.clear();
  targetCardNmb.value = null;
  setStep('set-target-card-owner');
}

function isPlayerPointOwnCard(playerId) {
  return [...cardsOwners.values()].find((value) => playerId === value)
}

function isPlayerVoted(playerId) {
  return [...cardsSelection.values()].find((votes) => votes.has(playerId))
}

function saveState() {
  localStorage.setItem('dixit-game-state', JSON.stringify({
    playersAmount,
    score: Object.fromEntries(score)
  }))
}

function restoreState(lastState) {
  playersAmount = +lastState.playersAmount;
  score.clear();
  Object.entries(lastState.score).forEach(([key, value]) => key && score.set(+key, +value));
}

function startNewGame() {
  let text = "Reset score and start new Game?";
  if (confirm(text) == true) {
    localStorage.removeItem('dixit-game-state')
    location.reload();
  }
}

function onOffPlayer() {
  setStep('on-off-player');
}

function getPlace(playerId) {
  const playerScore = score.get(playerId);
  const place = [...new Set([...score.values()])].sort().reverse().indexOf(playerScore);
  return place + 1;
}
</script>

<template>
  <div class="board">
    <div class="header">
      <div v-if="!isNewGame && step === 'set-target-card-owner'"
           class="new-game-btn"
           @click="startNewGame"
      >New Game</div>
      <div v-if="step === 'set-target-card-owner' || step === 'on-off-player'
      || (step === 'setting-players-choice' && selectedPlayers.size === 0)"
           class="new-game-btn"
           @click="onOffPlayer"
      >On/Off player</div>
    </div>
    <div class="hint">{{hints[step]}}
      <div v-if="needConfirm" class="confirm-marker">Confirm?</div>
    </div>
    <div v-if="step === 'set-players-amount'"
         class="select-amount-panel"
    >
      <button v-for="i in 5"
              :key="i"
              @click="setPlayersAmount(i + 2)"
              style="cursor: pointer"
      >{{i + 2}}</button>
    </div>

    <div v-if="step !== 'set-players-amount'"
         id="score-board"
    >
      <div class="players-container">
        <div>Players</div>
        <player-button v-for="i in playersAmount"
                       :class="{disabled: disabledPlayers.has(i)}"
                       :key="i"
                       :id="i"
                       :isOwner="i === targetCardOwner"
                       :show-card="i !== targetCardOwner && (
                           (step === 'set-selected-card-owners' && !isPlayerPointOwnCard(i))
                            ||
                           (step === 'setting-players-choice' && !isPlayerVoted(i))
                           )"
                       :card="cardsOwners.get(i)"
                       :disabled="disabledPlayers.has(i)"
                       :score="score.get(i) || 0"
                       :place="getPlace(i)"
                       :selected="selectedPlayers.has(i)"

                       @click="onPLayerBtnClick(i)"
        />
      </div>

      <div class="center-panel">
        <div v-if="step === 'score-counted'"
             class="control"
             @click="startRound()"
             style="cursor: pointer"
        >&#10226;</div>
      </div>

      <div class="cards-container" :class="{disabled: ['set-target-card-owner', 'on-off-player'].includes(step)}">
        <div style="display: flex;justify-content: center;">Cards</div>
        <card v-for="i in (playersAmount - disabledPlayers.size)" :key="i"
              @click="onCardClick(i)"
              :players="cardsSelection.get(i)"
              :owner="cardsOwners.get(i)"
              :is-target="targetCardNmb === i"
              :is-selected="selectedCard === i"
              :msg="i" />
      </div>
    </div>
  </div>
</template>
<style src="./App.scss" lang="scss"></style>
