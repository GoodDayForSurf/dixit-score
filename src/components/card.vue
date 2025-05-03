<script setup>
defineProps({
  msg: {
    type: Number,
    required: true
  },
  owner: {
    type: Number,
    required: false
  },
  players: {
    required: false
  },
  isSelected: {
    type: Boolean,
    required: false,
    value: false
  },
  isTarget: {
    type: Boolean,
    required: false,
    value: false
  }
})
</script>

<template>
  <div class="card" :class="{selected: isSelected, target: isTarget}">
    <div class="number">{{msg}}</div>
    <div v-if="players?.size" class="players">
      <div v-for="playerId in players"
           class="voted-payer"
           :class="{['color-' + playerId]: true}">
         &nbsp;
      </div>
        <!--{{[...players].join(' ')}}-->
    </div>
    <div v-if="owner" class="owner" :class="[`color-${owner}`]"><!--{{owner}}-->&nbsp;</div>
  </div>
</template>

<style scoped lang="scss">
:root {
  --player-button-size: 50px;
}

.card {
  cursor: pointer;
  position: relative;
  width: calc(var(--player-button-size) + var(--player-button-size) / 1.5);
  height: var(--player-button-size);
  display: flex;
  flex-flow: column;
  align-items: center;
  justify-content: center;

  background-color: #9478ae;
  color: whitesmoke;
  border-radius: 10px;
  border-width: 0;
  font-size: 24px;



  &.selected {
    box-shadow: 0 0 10px 3px greenyellow
  }

  &.target {
    background-color: fuchsia;
  }

  .number {
    margin-bottom: 5px;
  }

  .players {
    position: absolute;
    bottom: 0;
    left: 0;
    padding-left: 5px;
    font-size: 14px;
    display: flex;
    flex-flow: row;
    align-items: center;
    gap: calc(var(--player-button-size) / 15);
    padding-bottom: 5px;

    .voted-payer {
      width: calc(var(--player-button-size) / 5);
      height: calc(var(--player-button-size) / 5);
      border: 1px solid white;
    }
  }

  .owner {
    position: absolute;
    top: 0;
    right: 0px;
    font-size: 17px;
    border-radius: 50%;
    border: 1px solid white;
    width: 22px;
    height: 22px;
    display: flex;
    align-items: center;
    justify-content: center;
  }
}
</style>
