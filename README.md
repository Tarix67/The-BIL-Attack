<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Survivor Prototype</title>
<style>
  html, body {
    margin: 0; padding: 0; height: 100%;
    background: #1a1a2e;
    display: flex; align-items: center; justify-content: center;
    font-family: 'Courier New', monospace;
    color: #eee;
    overflow: hidden;
  }
  #ui {
    position: fixed; top: 10px; left: 10px;
    font-size: 18px;
    text-shadow: 2px 2px 0 #000;
    pointer-events: none;
  }
  canvas {
    image-rendering: pixelated;
    border: 3px solid #444;
    background: #0d1117;
  }
  #gameover {
    position: fixed; top: 50%; left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;
    display: none;
    text-shadow: 2px 2px 0 #000;
  }
  #gameover h1 { color: #e94560; font-size: 40px; margin-bottom: 10px; }
  #gameover button {
    background: #e94560; color: white; border: none;
    padding: 10px 20px; font-size: 16px; cursor: pointer;
    font-family: inherit;
  }
</style>
</head>
<body>

<div id="ui">
  HP: <span id="hp">100</span> | Счёт: <span id="score">0</span> | Волна: <span id="wave">1</span>
</div>

<canvas id="game" width="800" height="600"></canvas>

<div id="gameover">
  <h1>ТЫ ПОЙМАН</h1>
  <p>Счёт: <span id="finalScore">0</span></p>
  <button onclick="restart()">Заново</button>
</div>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

// ---------- Состояние игры ----------
let keys = {};
let player, bullets, enemies, particles;
let score, wave, spawnTimer, waveTimer, gameOver;

const PLAYER_SPEED = 3;
const PLAYER_SIZE = 20;
const BULLET_SPEED = 7;
const FIRE_RATE = 15; // кадров между выстрелами
const ENEMY_BASE_SPEED = 1.2;
const ENEMY_SIZE = 18;

function init() {
  player = { x: W/2, y: H/2, hp: 100, maxHp: 100, fireCooldown: 0 };
  bullets = [];
  enemies = [];
  particles = [];
  score = 0;
  wave = 1;
  spawnTimer = 0;
  waveTimer = 0;
  gameOver = false;
  document.getElementById('gameover').style.display = 'none';
}

// ---------- Ввод ----------
window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

function restart() { init(); requestAnimationFrame(loop); }

// ---------- Спавн врагов ----------
function spawnEnemy() {
  // спавним за пределами экрана, с случайной стороны
  const side = Math.floor(Math.random() * 4);
  let x, y;
  if (side === 0) { x = -30; y = Math.random() * H; }
  else if (side === 1) { x = W + 30; y = Math.random() * H; }
  else if (side === 2) { x = Math.random() * W; y = -30; }
  else { x = Math.random() * W; y = H + 30; }

  const speedMult = 1 + (wave - 1) * 0.12;
  enemies.push({
    x, y,
    hp: 2 + Math.floor(wave / 3),
    speed: ENEMY_BASE_SPEED * speedMult,
    color: `hsl(${350 - wave * 5}, 70%, 55%)`
  });
}

// ---------- Обновление ----------
function update() {
  if (gameOver) return;

  // --- движение игрока ---
  let dx = 0, dy = 0;
  if (keys['w'] || keys['arrowup']) dy -= 1;
  if (keys['s'] || keys['arrowdown']) dy += 1;
  if (keys['a'] || keys['arrowleft']) dx -= 1;
  if (keys['d'] || keys['arrowright']) dx += 1;
  const len = Math.hypot(dx, dy) || 1;
  player.x += (dx/len) * PLAYER_SPEED;
  player.y += (dy/len) * PLAYER_SPEED;
  player.x = Math.max(PLAYER_SIZE, Math.min(W - PLAYER_SIZE, player.x));
  player.y = Math.max(PLAYER_SIZE, Math.min(H - PLAYER_SIZE, player.y));

  // --- автострельба по ближайшему врагу ---
  player.fireCooldown--;
  if (player.fireCooldown <= 0 && enemies.length > 0) {
    let nearest = enemies[0], nd = Infinity;
    for (const e of enemies) {
      const d = Math.hypot(e.x - player.x, e.y - player.y);
      if (d < nd) { nd = d; nearest = e; }
    }
    const angle = Math.atan2(nearest.y - player.y, nearest.x - player.x);
    bullets.push({
      x: player.x, y: player.y,
      vx: Math.cos(angle) * BULLET_SPEED,
      vy: Math.sin(angle) * BULLET_SPEED
    });
    player.fireCooldown = FIRE_RATE;
  }

  // --- пули ---
  bullets = bullets.filter(b => {
    b.x += b.vx; b.y += b.vy;
    return b.x > -10 && b.x < W+10 && b.y > -10 && b.y < H+10;
  });

  // --- враги: движение к игроку ---
  for (const e of enemies) {
    const angle = Math.atan2(player.y - e.y, player.x - e.x);
    e.x += Math.cos(angle) * e.speed;
    e.y += Math.sin(angle) * e.speed;
  }

  // --- коллизии: пуля-враг ---
  for (let i = enemies.length - 1; i >= 0; i--) {
    const e = enemies[i];
    for (let j = bullets.length - 1; j >= 0; j--) {
      const b = bullets[j];
      if (Math.hypot(e.x - b.x, e.y - b.y) < ENEMY_SIZE/2 + 4) {
        bullets.splice(j, 1);
        e.hp--;
        spawnParticles(e.x, e.y, e.color);
        if (e.hp <= 0) {
          enemies.splice(i, 1);
          score += 10;
        }
        break;
      }
    }
  }

  // --- коллизии: враг-игрок ---
  for (const e of enemies) {
    if (Math.hypot(e.x - player.x, e.y - player.y) < ENEMY_SIZE/2 + PLAYER_SIZE/2) {
      player.hp -= 0.5; // урон за кадр контакта
    }
  }
  if (player.hp <= 0) {
    gameOver = true;
    document.getElementById('finalScore').textContent = score;
    document.getElementById('gameover').style.display = 'block';
  }

  // --- частицы ---
  particles = particles.filter(p => {
    p.x += p.vx; p.y += p.vy; p.life--;
    return p.life > 0;
  });

  // --- спавн волнами ---
  spawnTimer++;
  const spawnInterval = Math.max(15, 60 - wave * 3);
  if (spawnTimer > spawnInterval) {
    spawnEnemy();
    spawnTimer = 0;
  }
  waveTimer++;
  if (waveTimer > 600) { // ~10 сек при 60fps на новую волну
    wave++;
    waveTimer = 0;
  }

  // --- UI ---
  document.getElementById('hp').textContent = Math.max(0, Math.ceil(player.hp));
  document.getElementById('score').textContent = score;
  document.getElementById('wave').textContent = wave;
}

function spawnParticles(x, y, color) {
  for (let i = 0; i < 5; i++) {
    const angle = Math.random() * Math.PI * 2;
    const speed = Math.random() * 2 + 1;
    particles.push({
      x, y, vx: Math.cos(angle)*speed, vy: Math.sin(angle)*speed,
      life: 20, color
    });
  }
}

// ---------- Отрисовка ----------
function draw() {
  ctx.clearRect(0, 0, W, H);

  // фон-сетка (заглушка вместо тайлов)
  ctx.strokeStyle = '#1e2030';
  ctx.lineWidth = 1;
  for (let x = 0; x < W; x += 40) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
  }
  for (let y = 0; y < H; y += 40) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
  }

  // частицы
  for (const p of particles) {
    ctx.fillStyle = p.color;
    ctx.fillRect(p.x-2, p.y-2, 4, 4);
  }

  // пули
  ctx.fillStyle = '#ffd93d';
  for (const b of bullets) {
    ctx.fillRect(b.x-3, b.y-3, 6, 6);
  }

  // враги
  for (const e of enemies) {
    ctx.fillStyle = e.color;
    ctx.fillRect(e.x - ENEMY_SIZE/2, e.y - ENEMY_SIZE/2, ENEMY_SIZE, ENEMY_SIZE);
  }

  // игрок
  ctx.fillStyle = '#4ecdc4';
  ctx.fillRect(player.x - PLAYER_SIZE/2, player.y - PLAYER_SIZE/2, PLAYER_SIZE, PLAYER_SIZE);
  // "глаза" чтобы было видно направление на ближайшую цель (заглушка вместо спрайта)
  ctx.fillStyle = '#0d1117';
  ctx.fillRect(player.x-6, player.y-4, 3, 3);
  ctx.fillRect(player.x+3, player.y-4, 3, 3);
}

function loop() {
  update();
  draw();
  if (!gameOver) requestAnimationFrame(loop);
}

init();
loop();
</script>
</body>
</html>
