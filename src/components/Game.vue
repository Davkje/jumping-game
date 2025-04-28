<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

// GAME STATES
const gameStart = ref(true);
const gameRunning = ref(false);
const gameOver = ref(true);
const score = ref(0);

const dinoWidth = 68;
const dinoHeight = 68;

const obstacleWidth = 42;
const obstacleHeight = 52;

// JUMP
const isJumping = ref(false);
const position = ref(0);
const gravity = 6;
const jumpHeight = 220;
const jumpSpeed = 25;

// GAME WIDTH
const gameContainer = ref<HTMLElement | null>(null); // ref to hold the DOM element
let backgroundX = 0;
const backgroundSpeed = 2; // Pixlar per frame

// ----- AUDIO -----

const music = new Audio("./sounds/music.wav");
music.volume = 0.4;
const deathSound = new Audio("./sounds/marimba/marimba_death.wav");
deathSound.volume = 0.3;
const clickSounds = [
	new Audio("sounds/clicks/click_1.wav"),
	new Audio("sounds/clicks/click_2.wav"),
	new Audio("sounds/clicks/click_3.wav"),
	new Audio("sounds/clicks/click_4.wav"),
	new Audio("sounds/clicks/click_5.wav"),
];
const jumpSounds = [new Audio("sounds/jump_2.wav"), new Audio("sounds/jump_1.wav")];

const landSounds = [
	new Audio("sounds/lands/land_1.wav"),
	new Audio("sounds/lands/land_2.wav"),
	new Audio("sounds/lands/land_3.wav"),
	new Audio("sounds/lands/land_4.wav"),
	new Audio("sounds/lands/land_5.wav"),
	new Audio("sounds/lands/land_6.wav"),
	new Audio("sounds/lands/land_7.wav"),
	new Audio("sounds/lands/land_8.wav"),
];

const marimbaSounds = [
	new Audio("sounds/marimba/jump_marimba_1.wav"),
	new Audio("sounds/marimba/jump_marimba_2.wav"),
	new Audio("sounds/marimba/jump_marimba_3.wav"),
	new Audio("sounds/marimba/jump_marimba_4.wav"),
	new Audio("sounds/marimba/jump_marimba_5.wav"),
	new Audio("sounds/marimba/jump_marimba_6.wav"),
	new Audio("sounds/marimba/jump_marimba_7.wav"),
	new Audio("sounds/marimba/jump_marimba_8.wav"),
];


// Wrapper function to play sound respecting the mute state
function playSound(sound: HTMLAudioElement) {
	sound.play();
}

function startMusic() {
	music.loop = true;
	playSound(music);
}

function playRandomClickSound() {
	const randomIndex = Math.floor(Math.random() * clickSounds.length);
	const sound = clickSounds[randomIndex];
	sound.currentTime = 0;
	sound.play();
}

function playRandomLandSound() {
	const randomIndex = Math.floor(Math.random() * landSounds.length);
	const sound = landSounds[randomIndex];
	sound.currentTime = 0;
	sound.play();
}

function playRandomJumpSound() {
  if (gameOver.value) return;

	const randomIndex = Math.floor(Math.random() * jumpSounds.length);
	const jumpSound = jumpSounds[randomIndex];
	jumpSound.currentTime = 0;
	jumpSound.play();
  const randomIndex2 = Math.floor(Math.random() * marimbaSounds.length);
	const marimbaSound = marimbaSounds[randomIndex2];
	marimbaSound.currentTime = 0;
	marimbaSound.play();
}

//----

const moveBackground = () => {
	if (!gameRunning.value || !gameContainer.value) return; // Kolla att elementet finns

	backgroundX -= backgroundSpeed;
	gameContainer.value.style.backgroundPosition = `${backgroundX}px 0`;

	requestAnimationFrame(moveBackground);
};

const gameWidth = ref(window.innerWidth); // Set initial game width to window width
// Uppdatera `gameWidth` vid resize
const handleResize = () => {
	gameWidth.value = window.innerWidth;
};

// OBSTACLES
let obstacleInterval: ReturnType<typeof setInterval> | null = null; // Store the interval for obstacle generation
const obstacleSpeed = 4;
let obstacleGenerationActive = false; // Flag to control obstacle generation

type Obstacle = {
	left: number;
	bottom: number;
	scored?: boolean; // valfri, om du vill markera om hindret har gett poäng
};

const obstacles = ref<Obstacle[]>([]);

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

	const dinoX = gameWidth.value / 2 - dinoWidth / 2; // Hämta aktuellt X-värde för dinon

	obstacles.value = obstacles.value.filter((obstacle) => {
		obstacle.left -= (obstacleSpeed * deltaTime) / 16.67;

		// Ge poäng när hindret har passerat helt och dinon har hoppat över det
		if (!obstacle.scored && obstacle.left + obstacleWidth <= dinoX && position.value > 50) {
			score.value++;
			obstacle.scored = true; // Markera som räknat så vi inte får dubbla poäng
		}

		// Behåll bara hinder som fortfarande syns på skärmen
		return obstacle.left >= -obstacleWidth;
	});
};

// Check for collisions
const checkCollisions = () => {
	const dinoX = gameWidth.value / 2 - dinoWidth / 2; // Beräkna dinoX varje gång

	obstacles.value.forEach((obstacle: any) => {
		if (
			obstacle.left < dinoX + dinoWidth && // Dino's right side
			obstacle.left + obstacleWidth > dinoX && // Dino's left side
			obstacle.bottom < position.value + dinoHeight && // Dino's top side
			obstacle.bottom + obstacleHeight > position.value // Dino's bottom side
		) {
			gameOver.value = true;
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
		playSound(deathSound);
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
	playRandomClickSound();
	startMusic();
	backgroundX = 0; // Återställ position
	requestAnimationFrame(moveBackground);

	// Rensa hinder innan spelet startar igen
	obstacles.value = [];
	setTimeout(() => {
		obstacles.value = [];
	}, 0);

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

// JUMP
const jump = () => {
	if (gameOver.value || isJumping.value) return; // Prevent jumping if game is over
	isJumping.value = true;
	playRandomJumpSound();

	let jumpInterval = setInterval(() => {
		if (position.value < jumpHeight) {
			position.value += jumpSpeed;
		} else {
			clearInterval(jumpInterval);
			let fallInterval = setInterval(() => {
				if (position.value > 0) {
					position.value -= gravity;
					if (position.value < 0) {
						position.value = 0; // Se till att den aldrig går under marken!
					}
				} else {
					clearInterval(fallInterval);
					isJumping.value = false;
					playRandomLandSound();
				}
			}, 20);
		}
	}, 20);
	playRandomLandSound();
};

const handleTouch = () => {
	jump();
};

// Initialiseras när sidan är laddad
onMounted(() => {
	window.addEventListener("keydown", handleKeyDown);
	window.addEventListener("resize", handleResize);
	window.addEventListener("touchstart", handleTouch); // Lägg till touchlyssnare
});

// Ta bort event listeners när komponenten tas bort
onUnmounted(() => {
	window.removeEventListener("keydown", handleKeyDown);
	window.removeEventListener("resize", handleResize);
	window.removeEventListener("touchstart", handleTouch); // Ta bort touchlyssnare
});
</script>

<template>
	<h1 v-if="gameStart">Davids Dino Game</h1>
  <h3 v-if="gameStart">Get to know me and my project by playing!</h3>
	<h1 v-if="gameOver && !gameStart">Game Over</h1>
  <h3 v-if="gameOver && !gameStart">You got {{score}} points!</h3>

  <!-- Messages -->
  <div class="message" v-if="score === 0 && !gameOver">Jump over the rocks with [SPACE]</div>
  <div class="message" v-if="score === 1 && !gameOver">Good job! Lets go!</div>
  <div class="message" v-if="(score === 2 || score === 3) && !gameOver">This game was made with TypeScript and Vue!</div>
  <div class="message" v-if="(score === 4 || score === 5) && !gameOver">I've made and implemented all the sounds and music!</div>
  <div class="message" v-if="(score === 6 || score === 7) && !gameOver">Have you noticed the randomized sound effects? No big deal!</div>
  <div class="message" v-if="(score === 8 || score === 9) && !gameOver">The Dino is just a gif that I made in Procreate! Neat huh?</div>
  <div class="message" v-if="score === 10 && !gameOver">10 points! Oh You're good!</div>
  <div class="message" v-if="(score === 11 || score === 12) && !gameOver">Thanks for playing! Now see how far you can get!</div>
  <div class="message" v-if="score === 20 && !gameOver">20 points! Now you got it!</div>
  <div class="message" v-if="score === 30 && !gameOver">30 points!! You're crushing it!</div>
  <div class="message" v-if="score === 40 && !gameOver">40 points!!! OMG!!</div>
  <div class="message" v-if="score === 50 && !gameOver">50 points!!! Go for 100! COME ON!!!"</div>
  <div class="message" v-if="score === 100 && !gameOver">100 POINTS!!!!! I NEVER THOUGHT ANYONE WOULD GET THIS FAR!!!!! YOU BEAT THE GAME!!!</div>

	<div class="game-container" ref="gameContainer">
		<div class="dino" :class="{ jumping: isJumping, dead: gameOver && !gameStart }" :style="{ bottom: position + 'px' }"></div>

		<div v-for="(obstacle, index) in obstacles" :key="index" class="obstacle" :style="{ left: obstacle.left + 'px', bottom: obstacle.bottom + 'px' }"></div>
	</div>

	<button v-if="gameStart" @click="startGame" class="start-button">Start</button>
	<button v-if="gameOver && !gameStart" @click="startGame" class="restart-button">Play Again</button>
	<div v-if="!gameOver && !gameStart" class="score">Score: {{ score }}</div>
</template>

<style scoped>
.game-container {
	width: 98dvw;
	height: 400px;
	background-image: url(/src/assets/images/bg.png);
	background-size: auto 100%; /* Låser höjden men låter bredden flöda */
	background-repeat: repeat-x;
	position: relative;
	overflow: hidden;
	border-radius: 10px;
	border-bottom: solid black 2px;
}

.dino {
	width: 68px;
	height: 68px;
	background-image: url(/src/assets/images/dino_walk.gif);
	background-size: cover; /* Ensures the entire image fits inside */
	background-repeat: no-repeat; /* Prevents tiling */
	background-position: center; /* Centers the image */
	position: absolute;
	bottom: 0;
	left: calc(50% - 34px); /* 50% av skärmen minus halva bredden (68px / 2) */
	border-radius: 5px;
	transition: transform 0.5s ease-out;
	z-index: 10;
}

.dino.jumping {
	background-image: url(/src/assets/images/dino_jump.gif);
}
.dino.dead {
	background-image: url(/src/assets/images/dino_dead.png);
}

.obstacle {
	width: 42px;
	height: 52px;
	background-image: url(/src/assets/images/obstacle.png);
	background-size: 100%; /* Ensures the entire image fits inside */
	background-repeat: no-repeat; /* Prevents tiling */
	background-position: bottom; /* Centers the image */
	position: absolute;
	bottom: 0;
	/* border-top-right-radius: 5px;
	border-top-left-radius: 5px; */
	/* border: solid red; */
}

.start-button {
	position: absolute;
	bottom: 8%;
	left: 50%;
	transform: translate(-50%, -50%);
	padding: 10px 20px;
	background-color: rgb(212, 212, 212);
	color: black;
	font-size: clamp(1.2rem, 2vw, 1.7rem);
	border: none;
	border-radius: 5px;
	cursor: pointer;
}

.restart-button {
	position: absolute;
	bottom: 8%;
	left: 50%;
	transform: translate(-50%, -50%);
	padding: 10px 20px;
	background-color: rgb(212, 212, 212);
	color: black;
	font-size: clamp(1.2rem, 2vw, 1.7rem);
	border: none;
	border-radius: 5px;
	cursor: pointer;
}

.restart-button:hover,
.start-button:hover {
	background-color: white;
}

.score {
	position: absolute;
	bottom: 12%;
	left: 50%;
	transform: translateX(-50%);
	font-size: clamp(1.2rem, 3vw, 1.7rem);
	color: white;
  cursor: default;
}

h1 {
	position: absolute;
  font-size: clamp(2rem, 7vw, 3rem);
  width: 100%;
  margin: 0;
  text-align: center;
	top: 6%;
	left: 50%;
	transform: translateX(-50%);
	color: white;
}

h3 {
	position: absolute;
  font-size: clamp(1.2rem, 3vw, 1.7rem);
	top: 17%;
	left: 50%;
  margin: 0;
  width: 100%;
  text-align: center;
	transform: translateX(-50%);
	color: white;
  z-index: 999;
}

.message {
  position: absolute;
  display: flex;
  justify-content: center;
  align-items: center;
  border: solid rgb(0, 0, 0) 3px;
  border-radius: 5rem 5rem 5rem 0;
  color: black;
  background-color: rgb(243, 243, 243);
  bottom: 62%;
  left: 55%;
  margin: 0;
  width: clamp(10vw, 500px, 30vw);
  min-height: 10vh;
  font-style: italic;
  font-size: clamp(1rem, 3vw, 1.7rem);
  border-radius: 2rem 2rem 2rem 0;
  text-align: center;
  padding: 1.2rem;
  z-index: 99;
}

@media (max-width: 668px) {
  h1 {
    /* top: 8%; */
  }

  h3 {
    top: 14%;
    z-index: 999;
    font-size: clamp(1rem, 3vw, 1.5rem);
  }

    .restart-button,
    .start-button {
    bottom: 4%;
  }

  .score {
    bottom: 8%;
  }
}

</style>
