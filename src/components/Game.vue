<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

// GAME STATES
const isPaused = ref(false);
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

let obstacleInterval: ReturnType<typeof setInterval> | null = null;

// Lyssnar på tangentbordet
const handleKeyDown = (event: KeyboardEvent) => {
	console.log("Tangent nedtryckt:", event.code); // ✅ Kolla om "P" registreras i konsolen
	if (event.repeat) return;
	if (event.code === "Space") jump();
	if (event.code === "KeyP") togglePause(); // Tryck "P" för att pausa
};

// JUMP
const jump = () => {
	if (isPaused.value || gameOver.value || isJumping.value) return; // Hoppa inte om spelet är pausat
	isJumping.value = true;

	let jumpInterval = setInterval(() => {
		if (isPaused.value) {
			clearInterval(jumpInterval);
			return;
		}

		if (position.value < jumpHeight) {
			position.value += jumpSpeed;
		} else {
			clearInterval(jumpInterval);
			let fallInterval = setInterval(() => {
				if (isPaused.value) {
					clearInterval(fallInterval);
					return;
				}

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

// Funktion för att växla pausläge
const togglePause = () => {
	isPaused.value = !isPaused.value;
	console.log("Paus status:", isPaused.value);

	if (isPaused.value) {
		clearInterval(obstacleInterval!); // Pausa hindren
	} else {
		moveObstacles(); // Starta om hindren
		if (isJumping.value) resumeJump(); // Fortsätt hoppet om det pausades
	}
};

const resumeJump = () => {
	let fallInterval = setInterval(() => {
		if (isPaused.value) {
			clearInterval(fallInterval);
			return;
		}

		if (position.value > 0) {
			position.value -= gravity;
		} else {
			clearInterval(fallInterval);
			isJumping.value = false;
		}
	}, 20);
};

const createObstacle = () => {
	obstacles.value.push({ left: window.innerWidth });
};
let lastTimestamp = 0; // En enda variabel för att lagra senaste tidsstämpeln

const moveObstacles = (timestamp?: number) => {
	if (isPaused.value || gameOver.value) {
		// Om spelet är pausat, spara senaste tidsstämpeln men gör inget mer
		lastTimestamp = timestamp || lastTimestamp;
		return;
	}

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

	// Anropa nästa frame
	requestAnimationFrame(moveObstacles);
};

// --- RESTART GAME ---
const restartGame = () => {
	position.value = 0;
	obstacles.value = [];

	gameOver.value = false;
	isPaused.value = false;
	isJumping.value = false;
	console.log("Paus status:", isPaused.value);

	requestAnimationFrame(moveObstacles); // 🏃 Starta smooth hinder
};

onMounted(() => {
	window.addEventListener("keydown", handleKeyDown);
	setInterval(createObstacle, 2000);
	requestAnimationFrame(moveObstacles); // Starta smooth rörelse
});

onUnmounted(() => {
	window.removeEventListener("keydown", handleKeyDown);
});
</script>

<template>
	<div class="game-container">
		<div class="dino" :style="{ bottom: position + 'px' }"></div>
		<div v-for="(obstacle, index) in obstacles" :key="index" class="obstacle" :style="{ left: obstacle.left + 'px' }"></div>
		<button v-if="gameOver" @click="restartGame" class="restart-button">Starta om</button>
		<button class="pause-button" @click="togglePause">P</button>
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

.pause-button {
	position: absolute;
	top: 10px;
	right: 10px;
	padding: 10px 15px;
	background-color: #ffcc00;
	border: none;
	cursor: pointer;
	font-size: 16px;
	font-weight: bold;
	border-radius: 5px;
}

.pause-button:hover {
	background-color: #ffdb4d;
}
</style>
