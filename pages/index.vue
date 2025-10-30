<template>
  <ClientOnly>
    <!--    <DiceBox></DiceBox>-->
<!--    <ThreeD_Dice />-->
    <TwoD_Dice v-if="isSHowDice" :endNumber="endNumber"/>
    <input type="text" v-model="endNumber">
  <button @click="restartDice" :disabled="isRolling" class="restart-button">
    {{ isRolling ? 'Бросаем...' : 'Новый бросок' }}
  </button>
  </ClientOnly>
</template>

<script setup>
import ThreeD_Dice from "~/components/ThreeD_Dice.vue";
import TwoD_Dice from "~/components/TwoD_Dice.vue";
import { ref } from "vue";


let endNumber = ref(20);
let isSHowDice = ref(true);

function restartDice() {
  isSHowDice.value = false;
  setTimeout(()=>{
    isSHowDice.value = true;
  }, 500)
}

</script>