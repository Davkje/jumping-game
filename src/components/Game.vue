<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

// JUMP
const isJumping = ref(false);
const position = ref(0);
const gravity = 4;
const jumpHeight = 120;
const jumpSpeed = 15;
// OBSTACLES
const obstacles = ref<{ left: number }[]>([]);
const obstacleSpeed = 8;
const gameOver = ref(false);

const jump = () => {
  if (isPaused.value || isJumping.value) return; // Pausa hoppet om spelet är pausat
  isJumping.value = true;

  let jumpInterval = setInterval(() => {
    if (position.value < jumpHeight) {
      position.value += jumpSpeed;
    } else {
      clearInterval(jumpInterval);
      let fallInterval = setInterval(() => {
        if (position.value > 0) {
          position.value -= gravity;
        } else {
          clearInterval(fallInterval);
          isJumping.value = false;
        }
      }, 20);
    }
  }, 20);
};

const handleKeyDown = (event: KeyboardEvent) => {
	if (event.repeat) return;
	if (event.code === "Space") jump();
};

const createObstacle = () => {
	obstacles.value.push({ left: window.innerWidth });
};

// --- MOVE OBSTACLES ---
const moveObstacles = () => {
  if (isPaused.value) return; // Pausar spelet om isPaused är true

  obstacleInterval = setInterval(() => {
    if (gameOver.value) {
      clearInterval(obstacleInterval!);
      return;
    }

    obstacles.value.forEach((obstacle, index) => {
      obstacle.left -= obstacleSpeed;

      if (obstacle.left < -50) {
        obstacles.value.splice(index, 1);
      }

      if (
        obstacle.left + 20 > 50 &&
        obstacle.left < 50 + 40 &&
        position.value < 30
      ) {
        gameOver.value = true;
        isPaused.value = true; // Pausa spelet när kollisionen sker
      }
    });
  }, 50);
};


const isPaused = ref(false); // Ny status för paus
let obstacleInterval: ReturnType<typeof setInterval> | null = null;

onMounted(() => {
	window.addEventListener("keydown", handleKeyDown);
	setInterval(createObstacle, 2000);
	moveObstacles();
});

onUnmounted(() => {
	window.removeEventListener("keydown", handleKeyDown);
});

// --- RESTART GAME ---
const restartGame = () => {
  // Återställ dinosauriens position
  position.value = 0; // Tillbaka till botten

  // Återställ hinder
  obstacles.value = [];

  // Återställ spelets status
  gameOver.value = false;
  isPaused.value = false; // Spela vidare när man startar om

  // Starta om hindermovementen
  moveObstacles(); // Eller ett annat sätt att starta om hindermovementen
};

</script>

<template>
	<div class="game-container">
		<div class="dino" :style="{ bottom: position + 'px' }">Dino</div>
		<div v-for="(obstacle, index) in obstacles" :key="index" class="obstacle" :style="{ left: obstacle.left + 'px' }"></div>
    <button v-if="gameOver" @click="restartGame" class="restart-button">Starta om</button>
	</div>
</template>

<style scoped>
.game-container {
	width: 100vw;
	height: 200px;
	background-color: lightgray;
	position: relative;
	overflow: hidden;
	border-bottom: 2px solid black;
}

.dino {
	width: 40px;
	height: 40px;
	background-color: green;
	position: absolute;
	bottom: 0;
	left: 50px;
	border-radius: 5px;
}

.obstacle {
	width: 25px;
	height: 30px;
	background-color: red;
	position: absolute;
	bottom: 0;
	border-radius: 5px;
}

.restart-button {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 10px 20px;
  background-color: #4caf50;
  color: white;
  font-size: 18px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.restart-button:hover {
  background-color: #45a049;
}

</style>
