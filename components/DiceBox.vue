<template>
  <div id="dice-container" class="dice-box-container"></div>
  <button @click="start">Roll em!</button>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import DiceBox from '@3d-dice/dice-box'

const colors = [
  "#348888",
  "#22BABB",
  "#9EF8EE",
  "#FA7F08",
  "#F24405",
  "#F25EB0",
  "#B9BF04",
  "#F2B705",
  "#F27405",
  "#F23005",
];

const scale = [2, 3, 4, 5, 6, 7, 8];
const configRolls = ["4d20", "4d12", "4d10", "4d8", "4d6", "4d4"]

const diceContainer = ref(null)
let box

onMounted(() => {
  box = new DiceBox({
    assetPath: "assets/",
    origin: "https://unpkg.com/@3d-dice/dice-box@1.1.3/dist/",
    container: "#dice-container",
    theme: 'default', // Тема должна поддерживать форс значения
    themeColor: "#feea03",
    externalThemes: {
      diceOfRolling: "https://www.unpkg.com/@3d-dice/theme-dice-of-rolling@0.2.1",
    },
    // offscreen: false,
    // scale: 6,
    // throwForce: 5,
    gravity: 1,
    mass: 1,
    // spinForce: 6,
    result: [5],
    rollResult: 5
  })


  box.init().then(() => {
    // Initial roll with forced value
    box.roll('1d12!');
  });
})

function get_random(list) {
  return list[Math.floor(Math.random() * list.length)];
}

function start(){
  
  // Force the die to show 5
  box.roll('1d12!');
}
</script>

<style scoped lang="scss">
.dice-box-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 450px;
  background: #791a1a;
  border-radius: 8px;
  box-shadow: inset 0 0 10px rgba(0,0,0,0.1);
  position: relative;

  :deep(canvas) {
    width: 100% !important;
    height: 100% !important;
  }
}

button {
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background: #feea03;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  
  &:hover {
    background: darken(#feea03, 10%);
  }
}
</style>