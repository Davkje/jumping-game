<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";
// GAME STATES
const gameOver = ref(false);

// JUMP
const isJumping = ref(false);
const position = ref(0);
const gravity = 4;
const jumpHeight = 120;
const jumpSpeed = 15;

// OBSTACLES
const obstacles = ref<{ left: number }[]>([]);
const obstacleSpeed = 4;
let obstacleInterval: ReturnType<typeof setInterval> | null = null; // För hinderintervall
let animationFrameId: number | null = null; // För att lagra ID för requestAnimationFrame

// Lyssnar på tangentbordet
const handleKeyDown = (event: KeyboardEvent) => {
	console.log("Tangent nedtryckt:", event.code);
	if (event.repeat) return;
	if (event.code === "Space") jump();
};

// JUMP
const jump = () => {
	if (gameOver.value || isJumping.value) return; // Hoppa inte om spelet är över
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

// Skapa hinder
const createObstacle = () => {
	if (!gameOver.value) {
		// Hindren ska inte skapas om spelet är över
		obstacles.value.push({ left: window.innerWidth });
	}
};

// Flytta hinder
let lastTimestamp = 0; // En enda variabel för att lagra senaste tidsstämpeln

const moveObstacles = (timestamp?: number) => {
	if (gameOver.value) return; // Stanna hinder-rörelsen om spelet är över

	// Beräkna tidsdifferensen sedan senaste anrop
	const deltaTime = timestamp - lastTimestamp;
	lastTimestamp = timestamp; // Uppdatera senaste tidsstämpeln

	// Uppdatera hinderpositionen baserat på tidsdifferensen
	obstacles.value.forEach((obstacle, index) => {
		obstacle.left -= (obstacleSpeed * deltaTime) / 16; // Gör rörelsen jämn

		if (obstacle.left < -50) {
			obstacles.value.splice(index, 1); // Ta bort hinder som har passerat
		}

		// Kolla om dino kraschar i hindret
		if (obstacle.left + 20 > 50 && obstacle.left < 50 + 40 && position.value < 30) {
			gameOver.value = true;
		}
	});

	// Anropa nästa frame för att fortsätta rörelsen
	animationFrameId = requestAnimationFrame(moveObstacles);
};

// Starta spelet
const startGame = () => {
	gameOver.value = false;
	position.value = 0;
	obstacles.value = [];
	lastTimestamp = 0; // Reset lastTimestamp when the game starts
	// Starta hinder interval
	obstacleInterval = setInterval(createObstacle, 2000);
	// Starta animation för att flytta hindren
	animationFrameId = requestAnimationFrame(moveObstacles);
};

// Starta om spelet
const restartGame = () => {
	// Stoppa nuvarande intervall och animation och starta spelet på nytt
	if (obstacleInterval) clearInterval(obstacleInterval);
	if (animationFrameId) cancelAnimationFrame(animationFrameId);
	startGame();
};

// Initialiseras när sidan är laddad
onMounted(() => {
	window.addEventListener("keydown", handleKeyDown);
});

// Ta bort event listeners när komponenten tas bort
onUnmounted(() => {
	window.removeEventListener("keydown", handleKeyDown);
	if (obstacleInterval) clearInterval(obstacleInterval);
	if (animationFrameId) cancelAnimationFrame(animationFrameId);
});
</script>

<template>
	<div class="game-container">
		<div class="dino" :style="{ bottom: position + 'px' }"></div>
		<div v-for="(obstacle, index) in obstacles" :key="index" class="obstacle" :style="{ left: obstacle.left + 'px' }"></div>
		<!-- Starta om knappen och spela-knapp -->
		<button v-if="gameOver" @click="restartGame" class="restart-button">Starta om</button>
		<!-- Starta spelet (syns bara om spelet inte är igång) -->
		<button v-if="!gameOver && obstacles.length === 0" @click="startGame" class="start-button">Starta spelet</button>
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

.restart-button,
.start-button {
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

.restart-button:hover,
.start-button:hover {
	background-color: #45a049;
}
</style>
