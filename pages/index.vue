<template>
  <ClientOnly>
    <!--    <DiceBox></DiceBox>-->
<!--    <ThreeD_Dice />-->
    <TwoD_Dice/>
  </ClientOnly>
</template>

<script setup>

import ThreeD_Dice from "~/components/ThreeD_Dice.vue";
import TwoD_Dice from "~/components/TwoD_Dice.vue";
</script>