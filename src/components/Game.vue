<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

// GAME STATES
const gameStart = ref(true);
const gameRunning = ref(false);
const gameOver = ref(true);
const score = ref(0);

// JUMP
const isJumping = ref(false);
const position = ref(0);
const gravity = 6;
const jumpHeight = 120;
const jumpSpeed = 15;

// GAME WIDTH
const gameWidth = ref(window.innerWidth); // Set initial game width to window width
// Update gameWidth on window resize
const handleResize = () => {
	gameWidth.value = window.innerWidth;
};

// OBSTACLES
let obstacleInterval: ReturnType<typeof setInterval> | null = null; // Store the interval for obstacle generation
const obstacleSpeed = 4;
let obstacleGenerationActive = false; // Flag to control obstacle generation

const obstacles = ref([]);
const obstacleWidth = 25;
const obstacleHeight = 30;

const generateObstacles = () => {
	if (obstacleGenerationActive) return; // Prevent generating if it's already active
	obstacleGenerationActive = true;

	const minInterval = 1500; // Minimum interval time (1.5 seconds)
	const maxInterval = 3000; // Maximum interval time (3 seconds)

	const createObstacle = () => {
		if (gameOver.value) {
			obstacleGenerationActive = false; // Stop generating obstacles if the game is over
			return;
		}

		const newObstacle = {
			left: gameWidth.value, // Use dynamic game width to position obstacles
			bottom: 0,
		};
		obstacles.value.push(newObstacle);

		// Set a random interval for the next obstacle generation
		const nextInterval = Math.floor(Math.random() * (maxInterval - minInterval + 1)) + minInterval;
		obstacleInterval = setTimeout(createObstacle, nextInterval); // Create next obstacle after a random time
	};

	createObstacle(); // Start generating the first obstacle
};

// MOVE OBSTACLES
const moveObstacles = (deltaTime: number) => {
	if (gameOver.value) return; // Stop obstacles if game is over

	obstacles.value = obstacles.value.filter((obstacle) => {
		obstacle.left -= (obstacleSpeed * deltaTime) / 16.67;

		// Only give a score once when the obstacle fully passes the dino
		if (!obstacle.scored && obstacle.left + obstacleWidth <= 50) {
			score.value++;
			obstacle.scored = true; // Mark as scored so it doesn’t count again
		}

		// Keep only obstacles that are still on the screen
		return obstacle.left >= -obstacleWidth;
	});
};

// Check for collisions
const checkCollisions = () => {
	obstacles.value.forEach((obstacle: any) => {
		if (
			obstacle.left < 50 + 40 && // Dino's right side
			obstacle.left + obstacleWidth > 50 && // Dino's left side
			obstacle.bottom < position.value + 40 && // Dino's top side
			obstacle.bottom + obstacleHeight > position.value // Dino's bottom side
		) {
			gameOver.value = true; // Game over if collision happens
		}
	});
};

// Game loop with requestAnimationFrame
let lastTime = 0;
const gameLoop = (timestamp: number) => {
	if (gameOver.value) {
		gameRunning.value = false;
		obstacleGenerationActive = false; // Reset flag for obstacle generation
		console.log(`gameOver: ${gameOver.value}`);
		return; // Stop the game loop if the game is over
	}

	// Calculate deltaTime to adjust for different frame rates
	const deltaTime = timestamp - lastTime;
	lastTime = timestamp;

	moveObstacles(deltaTime);
	checkCollisions();
	requestAnimationFrame(gameLoop); // Recursively call the game loop for smooth animation
};

// Start game logic
const startGame = (): void => {
	obstacles.value = []; // Clear obstacles
	gameOver.value = false;
	gameStart.value = false;
	score.value = 0; // Reset score
	console.log(`gameOver: ${gameOver.value}`);
	position.value = 0;
	obstacleGenerationActive = false; // Reset flag for obstacle generation

	// Clear any existing obstacle generation intervals
	if (obstacleInterval) {
		clearTimeout(obstacleInterval); // Stop any ongoing obstacle generation
		obstacleInterval = null;
	}

	// Start generating obstacles
	generateObstacles();
	gameRunning.value = true; // Set gameRunning to true when the game starts
	requestAnimationFrame(gameLoop); // Start the game loop with requestAnimationFrame
};

// KEYS
const handleKeyDown = (event: KeyboardEvent) => {
	if (event.repeat) return;
	if (event.code === "Space") jump();
};

const rotation = ref(0); // Track rotation angle
// JUMP
const jump = () => {
	if (gameOver.value || isJumping.value) return; // Prevent jumping if game is over
	isJumping.value = true;

	rotation.value += 90; // Increase rotation angle by 90° each jump

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

// Initialiseras när sidan är laddad
onMounted(() => {
	window.addEventListener("keydown", handleKeyDown);
	window.addEventListener("resize", handleResize); // Add event listener on mount
});

// Ta bort event listeners när komponenten tas bort
onUnmounted(() => {
	window.removeEventListener("keydown", handleKeyDown);
	window.removeEventListener("resize", handleResize); // Remove event listener on unmount
});
</script>

<template>
	<div v-if="!gameStart" class="score">Score: {{ score }}</div>

	<h1 v-if="gameStart">Welcome</h1>
	<h1 v-if="gameOver && !gameStart">Game Over</h1>

	<div class="game-container">
		<div class="dino" :style="{ bottom: position + 'px', transform: 'rotate(' + rotation + 'deg)' }"></div>

		<div v-for="(obstacle, index) in obstacles" :key="index" class="obstacle" :style="{ left: obstacle.left + 'px', bottom: obstacle.bottom + 'px' }"></div>
	</div>

	<button v-if="gameStart" @click="startGame" class="start-button">Play</button>
	<button v-if="gameOver && !gameStart" @click="startGame" class="restart-button">Play Again</button>
</template>

<style scoped>
.game-container {
	width: 95vw;
	height: 250px;
	background-color: lightgray;
	position: relative;
	border-top-right-radius: 10px;
	border-top-left-radius: 10px;
	overflow: hidden;
	border-bottom: 8px solid #292929;
}

.dino {
	width: 40px;
	height: 40px;
	background-image: linear-gradient(rgb(80, 97, 175), rgb(86, 121, 234));
	position: absolute;
	bottom: 0;
	left: 50px;
	border-radius: 5px;
	transition: transform 0.5s ease-out;
}

.obstacle {
	width: 25px;
	height: 30px;
	background-color: #292929;
	position: absolute;
	bottom: 0;
	border-top-right-radius: 5px;
	border-top-left-radius: 5px;
}

.start-button {
	position: absolute;
	bottom: 15%;
	left: 50%;
	transform: translate(-50%, -50%);
	padding: 10px 20px;
	background-color: rgb(80, 97, 175);
	color: white;
	font-size: 18px;
	border: none;
	border-radius: 5px;
	cursor: pointer;
}

.restart-button {
	position: absolute;
	bottom: 15%;
	left: 50%;
	transform: translate(-50%, -50%);
	padding: 10px 20px;
	background-color: rgb(80, 97, 175);
	color: white;
	font-size: 18px;
	border: none;
	border-radius: 5px;
	cursor: pointer;
}

.restart-button:hover,
.start-button:hover {
	background-color: rgb(84, 103, 198);
}

.score {
	position: absolute;
	top: 10px;
	left: 50%;
	transform: translateX(-50%);
	font-size: 24px;
	font-weight: bold;
	color: white;
}

h1 {
	position: absolute;
	top: 15%;
	left: 50%;
	transform: translateX(-50%);
	color: white;
}
</style>
