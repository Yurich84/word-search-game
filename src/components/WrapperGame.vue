<template>
  <section class="main-content word-game">
      <h2>Find these {{ words.length }} words</h2>
      <CoreGame
          :matrix="matrix"
          :words="words"
          class="word-game__game"
      />
  </section>
</template>

<script setup>
import CoreGame from './CoreGame.vue'

const matrix = [
    ['A', 'M', 'C', 'N', 'T', 'S', 'K', 'B', 'A', 'N'],
    ['D', 'O', 'A', 'H', 'T', 'O', 'T', 'A', 'L', 'S'],
    ['A', 'J', 'N', 'X', 'E', 'V', 'R', 'U', 'O', 'R'],
    ['P', 'C', 'D', 'U', 'P', 'E', 'F', 'R', 'N', 'E'],
    ['R', 'B', 'Y', 'R', 'S', 'R', 'S', 'V', 'A', 'K'],
    ['E', 'Y', 'O', 'G', 'U', 'R', 'T', 'E', 'R', 'C'],
    ['T', 'L', 'N', 'I', 'B', 'A', 'R', 'S', 'G', 'A'],
    ['Z', 'K', 'T', 'G', 'I', 'I', 'J', 'X', 'W', 'R'],
    ['E', 'I', 'L', 'R', 'S', 'S', 'P', 'I', 'H', 'C'],
    ['L', 'S', 'E', 'I', 'K', 'O', 'O', 'C', 'U', 'F'],
]

const words = [
    'BARS',
    'CANDY',
    'CARROT',
    'CHEESE',
    'CHIPS',
    'COOKIES',
    'CRACKERS',
    'FRUIT',
    'GRANOLA',
    'NUTS',
    'PRETZEL',
    'YOGURT',
]
</script>
