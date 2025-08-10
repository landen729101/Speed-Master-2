<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
<title>Speed Master - Traffic Racer</title>
<style>
  body, html {
    margin: 0; padding: 0; overflow: hidden; background: #111;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    color: white;
  }
  #gameCanvas {
    display: block;
    margin: 0 auto;
    background: linear-gradient(to top, #222, #555);
    border: 3px solid #eee;
    border-radius: 10px;
    touch-action: none;
  }
  #startScreen, #gameOverScreen {
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0,0,0,0.9);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 10;
  }
  #startScreen input, #gameOverScreen button {
    font-size: 1.2em;
    padding: 10px 20px;
    margin: 10px;
    border-radius: 8px;
    border: none;
  }
  #leaderboard {
    margin-top: 20px;
    max-height: 180px;
    overflow-y: auto;
    width: 280px;
    background: #222;
    border-radius: 8px;
    padding: 8px;
    text-align: left;
  }
  #leaderboard h3 {
    margin: 5px 0;
    font-size: 1.1em;
    text-align: center;
  }
  #leaderboard ol {
    margin: 0; padding-left: 20px;
  }
  #leaderboard li {
    margin: 2px 0;
  }
  #scoreDisplay {
    position: fixed;
    top: 5px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 1.4em;
    font-weight: bold;
    user-select: none;
    z-index: 5;
  }
  #instructions {
    position: fixed;
    bottom: 5px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 0.9em;
    color: #bbb;
    user-select: none;
  }
</style>
</head>
<body>

<div id="startScreen">
  <h1>Speed Master</h1>
  <label for="playerName">Enter your name:</label><br />
  <input id="playerName" type="text" maxlength="12" placeholder="Your name" autocomplete="off" /><br />
  <button id="startBtn">Start Race</button>
  <div id="leaderboard">
    <h3>Leaderboard</h3>
    <ol id="leaderboardList"></ol>
  </div>
</div>

<canvas id="gameCanvas" width="320" height="480" tabindex="0"></canvas>

<div id="scoreDisplay" style="display:none;">Score: 0</div>
<div id="instructions">Tap left or right side to move</div>

<div id="gameOverScreen" style="display:none;">
  <h2>Game Over!</h2>
  <p id="finalScore"></p>
  <button id="restartBtn">Play Again</button>
  <div id="leaderboardGameOver" style="margin-top:20px;"></div>
</div>

<audio id="bgMusic" loop src="https://cdn.pixabay.com/download/audio/2022/03/29/audio_ebff882d41.mp3?filename=retro-game-loop-10870.mp3" crossorigin="anonymous"></audio>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  const startScreen = document.getElementById('startScreen');
  const gameOverScreen = document.getElementById('gameOverScreen');
  const leaderboardList = document.getElementById('leaderboardList');
  const leaderboardGameOver = document.getElementById('leaderboardGameOver');
  const playerNameInput = document.getElementById('playerName');
  const startBtn = document.getElementById('startBtn');
  const restartBtn = document.getElementById('restartBtn');
  const scoreDisplay = document.getElementById('scoreDisplay');
  const finalScoreText = document.getElementById('finalScore');
  const bgMusic = document.getElementById('bgMusic');

  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;

  let playerName = '';
  let gameStarted = false;
  let gameOver = false;

  // Game state
  let score = 0;
  let speed = 2; // base speed of cars
  let playerX = WIDTH / 2 - 20;
  const playerWidth = 40;
  const playerHeight = 70;
  const laneWidth = WIDTH / 3;
  const lanesX = [laneWidth / 2 - playerWidth / 2, laneWidth * 1.5 - playerWidth / 2, laneWidth * 2.5 - playerWidth / 2];

  // Controls
  let targetLane = 1; // 0,1,2 lanes
  let currentLane = 1;

  // Cars on screen
  const traffic = [];

  // Car colors - for player and traffic cars
  const carColors = ['#E63946', '#F1FAEE', '#A8DADC', '#457B9D', '#1D3557'];

  // Explosion frames
  let explosionFrames = 0;

  // Background music volume
  bgMusic.volume = 0.3;

  // Add traffic cars randomly
  function spawnTraffic() {
    // Avoid spawning in same lane as last car just above top to avoid immediate collisions
    const lane = Math.floor(Math.random() * 3);
    traffic.push({
      x: lanesX[lane],
      y: -100,
      width: playerWidth,
      height: playerHeight,
      color: carColors[Math.floor(Math.random() * carColors.length)],
      passed: false,
    });
  }

  // Draw rounded rect for cars
  function drawCar(x, y, width, height, color, isPlayer = false) {
    ctx.fillStyle = color;
    ctx.beginPath();
    const radius = 10;
    ctx.moveTo(x + radius, y);
    ctx.lineTo(x + width - radius, y);
    ctx.quadraticCurveTo(x + width, y, x + width, y + radius);
    ctx.lineTo(x + width, y + height - radius);
    ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
    ctx.lineTo(x + radius, y + height);
    ctx.quadraticCurveTo(x, y + height, x, y + height - radius);
    ctx.lineTo(x, y + radius);
    ctx.quadraticCurveTo(x, y, x + radius, y);
    ctx.closePath();
    ctx.fill();

    // Add details for player car - windows and stripes
    if (isPlayer) {
      ctx.fillStyle = 'rgba(255,255,255,0.6)';
      ctx.fillRect(x + width * 0.15, y + height * 0.15, width * 0.7, height * 0.25);
      ctx.fillRect(x + width * 0.35, y + height * 0.5, width * 0.3, height * 0.15);
    }
  }

  // Draw explosion animation on player's car
  function drawExplosion(x, y, frame) {
    const maxRadius = 40;
    const step = maxRadius / 10;
    ctx.save();
    ctx.globalAlpha = 1 - frame / 10;
    ctx.fillStyle = 'orange';
    ctx.beginPath();
    ctx.arc(x + playerWidth/2, y + playerHeight/2, frame * step, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }

  // Draw the road with lanes
  function drawRoad() {
    ctx.fillStyle = '#444';
    ctx.fillRect(0, 0, WIDTH, HEIGHT);

    // Lane dividers
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 4;
    ctx.setLineDash([20, 15]);
    for(let i = 1; i < 3; i++) {
      ctx.beginPath();
      ctx.moveTo(laneWidth * i, 0);
      ctx.lineTo(laneWidth * i, HEIGHT);
      ctx.stroke();
    }
    ctx.setLineDash([]);
  }

  // Check collision between player and car
  function checkCollision(rect1, rect2) {
    return !(
      rect1.x > rect2.x + rect2.width ||
      rect1.x + rect1.width < rect2.x ||
      rect1.y > rect2.y + rect2.height ||
      rect1.y + rect1.height < rect2.y
    );
  }

  // Update game state
  function update() {
    if (gameOver) return;

    // Move player lane smoothly to target lane
    const targetX = lanesX[targetLane];
    if (Math.abs(playerX - targetX) > 5) {
      playerX += (targetX - playerX) * 0.3;
    } else {
      playerX = targetX;
      currentLane = targetLane;
    }

    // Move traffic down
    for (let car of traffic) {
      car.y += speed;

      // Check if player passed car for score
      if (!car.passed && car.y > HEIGHT) {
        car.passed = true;
        score++;
        speed += 0.05; // speed up slightly each car passed
      }
    }

    // Remove cars that are offscreen
    while (traffic.length > 0 && traffic[0].y > HEIGHT + 100) {
      traffic.shift();
    }

    // Spawn new traffic cars randomly
    if (traffic.length < 5 && Math.random() < 0.04 + speed * 0.005) {
      spawnTraffic();
    }

    // Check collision with traffic
    const playerRect = {x: playerX, y: HEIGHT - playerHeight - 10, width: playerWidth, height: playerHeight};
    for (let car of traffic) {
      const carRect = {x: car.x, y: car.y, width: car.width, height: car.height};
      if (checkCollision(playerRect, carRect)) {
        gameOver = true;
        explosionFrames = 0;
        bgMusic.pause();
        break;
      }
    }

    scoreDisplay.textContent = `Score: ${score}`;
  }

  // Draw everything
  function draw() {
    drawRoad();

    // Draw traffic cars
    for (let car of traffic) {
      drawCar(car.x, car.y, car.width, car.height, car.color);
    }

    // Draw player car
    const playerY = HEIGHT - playerHeight - 10;
    if (!gameOver) {
      drawCar(playerX, playerY, playerWidth, playerHeight, carColors[0], true);
    } else {
      if (explosionFrames < 10) {
        drawExplosion(playerX, playerY, explosionFrames);
        explosionFrames++;
      } else {
        // Show game over screen after explosion
        showGameOver();
      }
    }
  }

  // Main game loop
  function gameLoop() {
    update();
    draw();
    if (!gameOver || explosionFrames < 10) {
      requestAnimationFrame(gameLoop);
    }
  }

  // Leaderboard helper functions
  function loadLeaderboard() {
    const data = localStorage.getItem('speedMasterLeaderboard');
    if (!data) return [];
    try {
      return JSON.parse(data);
    } catch {
      return [];
    }
  }
  function saveLeaderboard(list) {
    localStorage.setItem('speedMasterLeaderboard', JSON.stringify(list));
  }
  function addScoreToLeaderboard(name, score) {
    const list = loadLeaderboard();
    list.push({name, score});
    list.sort((a,b) => b.score - a.score);
    if (list.length > 10) list.length = 10;
    saveLeaderboard(list);
    return list;
  }
  function updateLeaderboardDisplay() {
    const list = loadLeaderboard();
    leaderboardList.innerHTML = list.map(item => `<li><strong>${item.name}</strong>: ${item.score}</li>`).join('');
  }
  function showLeaderboardGameOver() {
    const list = loadLeaderboard();
    leaderboardGameOver.innerHTML = `<h3>Leaderboard</h3><ol>${list.map(item => `<li><strong>${item.name}</strong>: ${item.score}</li>`).join('')}</ol>`;
  }

  // Show game over UI
  function showGameOver() {
    finalScoreText.textContent = `${playerName}, your score: ${score}`;
    startScreen.style.display = 'none';
    scoreDisplay.style.display = 'none';
    gameOverScreen.style.display = 'flex';
    showLeaderboardGameOver();
  }

  // Start game function
  function startGame() {
    playerName = playerNameInput.value.trim();
    if (!playerName) {
      alert('Please enter your name.');
      playerNameInput.focus();
      return;
    }
    startScreen.style.display = 'none';
    gameOverScreen.style.display = 'none';
    scoreDisplay.style.display = 'block';

    // Reset game state
    score = 0;
    speed = 2;
    targetLane = 1;
    currentLane = 1;
    playerX = lanesX[1];
    traffic.length = 0;
    gameOver = false;

    bgMusic.currentTime = 0;
    bgMusic.play().catch(() => { /* auto play may block */ });

    updateLeaderboardDisplay();

    canvas.focus();
    gameLoop();
  }

  // Restart game after game over
  restartBtn.addEventListener('click', () => {
    addScoreToLeaderboard(playerName, score);
    startGame();
  });

  startBtn.addEventListener('click', startGame);

  // Touch controls - tap left/right side of canvas to move lanes
  canvas.addEventListener('touchstart', e => {
    if (gameOver) return;
    const touchX = e.touches[0].clientX;
    const rect = canvas.getBoundingClientRect();
    const x = touchX - rect.left;

    if (x < WIDTH / 2) {
      // Move left
      if (targetLane > 0) targetLane--;
    } else {
      // Move right
      if (targetLane < 2) targetLane++;
    }
    e.preventDefault();
  });

  // Keyboard controls for desktop testing
  window.addEventListener('keydown', e => {
    if (gameOver) return;
    if (e.key === 'ArrowLeft' || e.key === 'a') {
      if (targetLane > 0) targetLane--;
    }
    if (e.key === 'ArrowRight' || e.key === 'd') {
      if (targetLane < 2) targetLane++;
    }
  });

  // Initial leaderboard display
  updateLeaderboardDisplay();
})();
</script>

</body>
</html>
