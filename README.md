# Morden--game
 🔫 Player Shooting: Fast-paced shooting mechanics.
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Waziri Modern Arcade — Top-Down Shooter</title>
  <style>
    /* Minimal reset */
    html,body { height:100%; margin:0; background:#0b0f1a; font-family: Inter, system-ui, sans-serif; -webkit-tap-highlight-color: transparent; }
    #game {
      display:flex; align-items:center; justify-content:center;
      height:100vh; width:100vw; overflow:hidden;
      background:
        radial-gradient(circle at 10% 10%, rgba(255,255,255,0.02), transparent 10%),
        linear-gradient(180deg, rgba(10,12,20,1) 0%, rgba(4,6,12,1) 100%);
    }
    canvas { display:block; border-radius:12px; box-shadow: 0 10px 40px rgba(0,0,0,0.6), 0 0 40px rgba(40,120,255,0.05) inset; max-width:100%; max-height:100%; }
    /* HUD */
    .hud {
      position: absolute; top:14px; left:16px; color:#cfe6ff; font-weight:600;
      text-shadow: 0 2px 8px rgba(0,0,0,0.6); pointer-events:none;
    }
    .controls {
      position: absolute; right:16px; bottom:16px; display:flex; gap:8px;
    }
    .btn {
      background: linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border:1px solid rgba(255,255,255,0.06); color:#eaf6ff;
      padding:10px 12px; border-radius:12px; font-weight:700;
      backdrop-filter: blur(6px); box-shadow: 0 6px 18px rgba(0,0,0,0.5);
      user-select:none;
    }
    @media (pointer:coarse) {
      .btn { padding:14px 16px; font-size:16px; border-radius:16px; }
    }
    /* Small note */
    .note { position:absolute; bottom:10px; left:10px; color:#7fa4d9; font-size:13px; opacity:0.9; }
  </style>
</head>
<body>
  <div id="game">
    <canvas id="canvas" width="1200" height="700" aria-label="Arcade shooter game"></canvas>
    <div class="hud" id="hud">Score: 0 &nbsp;&nbsp; Lives: 3 &nbsp;&nbsp; Wave: 1</div>

    <div class="controls" id="controls" aria-hidden="true">
      <div class="btn" id="restartBtn">Restart</div>
      <div class="btn" id="pauseBtn">Pause</div>
    </div>

    <div class="note">Touch + drag to move on mobile. Tap right side to fire.</div>
  </div>

<script>
/* Modern top-down shooter — Single-file
   Features:
   - Responsive canvas & high-DPI support
   - Player with smooth movement, shooting
   - Enemy spawns and waves
   - Particles & glow-like effects
   - Touch controls & keyboard
   - Modular, commented ES6 code
*/

/* ---------- Utilities ---------- */
const rand = (min, max) => Math.random() * (max - min) + min;
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const dist = (a,b) => Math.hypot(a.x-b.x, a.y-b.y);

/* ---------- Setup canvas ---------- */
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d', { alpha: false });
let DPR = Math.max(1, window.devicePixelRatio || 1);

function resizeCanvas() {
  // We'll scale the canvas while preserving aspect ratio
  const parent = canvas.parentElement;
  const w = Math.min(parent.clientWidth - 40, 1400);
  const h = Math.min(parent.clientHeight - 40, 800);
  const ratio = 16/9;
  let cw = w, ch = Math.round(w / ratio);
  if (ch > h) { ch = h; cw = Math.round(h * ratio); }
  canvas.style.width = cw + 'px';
  canvas.style.height = ch + 'px';
  // Set drawing buffer scaled for DPR
  canvas.width = cw * DPR;
  canvas.height = ch * DPR;
  ctx.setTransform(DPR,0,0,DPR,0,0);
}
window.addEventListener('resize', () => { DPR = Math.max(1, window.devicePixelRatio || 1); resizeCanvas(); });
resizeCanvas();

/* ---------- Game state ---------- */
let lastTime = 0;
let running = true;
let score = 0;
let wave = 1;
let lives = 3;

/* ---------- Input ---------- */
const input = {
  keys: {},
  pointer: { x: 0, y: 0, down: false },
  touchMode: false,
};
window.addEventListener('keydown', e => input.keys[e.key.toLowerCase()] = true);
window.addEventListener('keyup', e => input.keys[e.key.toLowerCase()] = false);

// Mouse/touch controls
canvas.addEventListener('pointerdown', (e) => {
  input.pointer.down = true;
  input.pointer.x = e.offsetX * (canvas.width/canvas.clientWidth) / DPR;
  input.pointer.y = e.offsetY * (canvas.height/canvas.clientHeight) / DPR;
  input.touchMode = e.pointerType !== 'mouse';
});
canvas.addEventListener('pointermove', (e) => {
  input.pointer.x = e.offsetX * (canvas.width/canvas.clientWidth) / DPR;
  input.pointer.y = e.offsetY * (canvas.height/canvas.clientHeight) / DPR;
});
canvas.addEventListener('pointerup', (e) => { input.pointer.down = false; });

/* ---------- Entities ---------- */
class Entity {
  constructor(x,y){ this.x=x; this.y=y; this.vx=0; this.vy=0; this.dead=false; }
  update(dt){}
  render(ctx){}
}
const entities = [];
function spawn(ent){ entities.push(ent); return ent; }

/* ---------- Player ---------- */
class Player extends Entity {
  constructor(x,y){
    super(x,y);
    this.radius = 18;
    this.speed = 420;
    this.cooldown = 0;
    this.color = '#91c5ff';
    this.hitTimer = 0;
  }
  update(dt){
    // Movement: keyboard or touch
    let dx=0, dy=0;
    if (input.keys['w'] || input.keys['arrowup']) dy = -1;
    if (input.keys['s'] || input.keys['arrowdown']) dy = 1;
    if (input.keys['a'] || input.keys['arrowleft']) dx = -1;
    if (input.keys['d'] || input.keys['arrowright']) dx = 1;

    if (input.pointer.down && input.touchMode) {
      // Smooth follow to pointer
      const tx = input.pointer.x, ty = input.pointer.y;
      dx = (tx - this.x) * 2; dy = (ty - this.y) * 2;
      const m = Math.hypot(dx,dy);
      if (m>1){ dx/=m; dy/=m; }
    }

    // normalize
    if (dx !== 0 || dy !== 0) {
      const m = Math.hypot(dx,dy);
      this.vx = (dx / m) * this.speed;
      this.vy = (dy / m) * this.speed;
    } else { this.vx *= 0.85; this.vy *= 0.85; }

    this.x += this.vx * dt;
    this.y += this.vy * dt;

    // Clamp inside canvas
    this.x = clamp(this.x, this.radius, canvas.width/DPR - this.radius);
    this.y = clamp(this.y, this.radius, canvas.height/DPR - this.radius);

    // Shooting: click or space
    this.cooldown -= dt;
    if ((input.keys[' '] || input.pointer.down) && this.cooldown <= 0) {
      this.shoot();
      this.cooldown = 0.12; // fire rate
    }
    this.hitTimer = Math.max(0, this.hitTimer - dt);
  }
  shoot(){
    const spread = 0.05;
    const tx = (input.pointer.x || canvas.width/(2*DPR));
    const ty = (input.pointer.y || this.y - 100);
    const angle = Math.atan2(ty - this.y, tx - this.x);
    // create 1-3 bullets
    for (let i=-1;i<=1;i++){
      const a = angle + (i*spread);
      spawn(new Bullet(this.x + Math.cos(a)*this.radius, this.y + Math.sin(a)*this.radius, Math.cos(a)*900, Math.sin(a)*900, 'player'));
    }
    // muzzle particle
    for (let i=0;i<6;i++) {
      const a = angle + rand(-0.6,0.6);
      spawn(new Particle(this.x, this.y, Math.cos(a)*rand(120,400), Math.sin(a)*rand(120,400), 0.2, '#cfe6ff', 12));
    }
  }
  render(ctx){
    // glow
    const g = ctx.createRadialGradient(this.x, this.y, 4, this.x, this.y, this.radius*3);
    g.addColorStop(0, 'rgba(145,197,255,0.28)');
    g.addColorStop(1, 'rgba(10,12,20,0)');
    ctx.fillStyle = g;
    ctx.beginPath(); ctx.arc(this.x,this.y,this.radius*2.6,0,Math.PI*2); ctx.fill();

    // hull
    ctx.save();
    ctx.translate(this.x,this.y);
    ctx.rotate(Math.atan2(input.pointer.y - this.y || -1, input.pointer.x - this.x || 0));
    ctx.fillStyle = this.hitTimer>0 ? '#ff9d9d' : this.color;
    // Triangular ship
    ctx.beginPath();
    ctx.moveTo(20,0);
    ctx.lineTo(-14,12);
    ctx.lineTo(-10,0);
    ctx.lineTo(-14,-12);
    ctx.closePath();
    ctx.fill();
    // cockpit
    ctx.fillStyle = 'rgba(10,14,25,0.6)';
    ctx.beginPath(); ctx.ellipse(2,0,6,5,0,0,Math.PI*2); ctx.fill();
    ctx.restore();
  }
}

/* ---------- Bullet ---------- */
class Bullet extends Entity {
  constructor(x,y,vx,vy,owner='enemy'){
    super(x,y); this.vx=vx; this.vy=vy; this.radius=4; this.life=2; this.owner=owner;
  }
  update(dt){
    this.x += this.vx * dt; this.y += this.vy * dt;
    this.life -= dt; if (this.life <= 0) this.dead = true;
    if (this.x < -50 || this.x > canvas.width/DPR + 50 || this.y < -50 || this.y > canvas.height/DPR + 50) this.dead = true;
  }
  render(ctx){
    ctx.beginPath();
    ctx.fillStyle = this.owner==='player' ? '#eaf6ff' : '#ffb3b3';
    ctx.arc(this.x,this.y,this.radius,0,Math.PI*2); ctx.fill();
    // trail
    ctx.globalAlpha = 0.6;
    ctx.beginPath();
    ctx.arc(this.x - this.vx*0.015, this.y - this.vy*0.015, this.radius*1.6,0,Math.PI*2); ctx.fill();
    ctx.globalAlpha = 1;
  }
}

/* ---------- Enemy ---------- */
class Enemy extends Entity {
  constructor(x,y, type='drone'){
    super(x,y);
    this.type = type;
    this.radius = type==='tank'?28:16;
    this.speed = type==='tank'?80:140;
    this.health = type==='tank'?6:2;
    this.color = type==='tank' ? '#ffb37a' : '#ffd6a6';
    this.fireTimer = rand(0.4,1.8);
  }
  update(dt){
    // simple homing towards player
    const target = player;
    const angle = Math.atan2(target.y - this.y, target.x - this.x);
    this.vx = Math.cos(angle) * this.speed;
    this.vy = Math.sin(angle) * this.speed;
    this.x += this.vx * dt; this.y += this.vy * dt;

    // shooting occasionally
    this.fireTimer -= dt;
    if (this.fireTimer <= 0) {
      this.fireTimer = rand(0.8, 2.2);
      // shoot a bullet toward player
      const a = Math.atan2(target.y - this.y, target.x - this.x) + rand(-0.12,0.12);
      spawn(new Bullet(this.x + Math.cos(a)*this.radius, this.y + Math.sin(a)*this.radius, Math.cos(a)*380, Math.sin(a)*380, 'enemy'));
    }

    // clamp
    if (this.x < -50 || this.x > canvas.width/DPR + 50 || this.y < -50 || this.y > canvas.height/DPR + 50) {
      // wrap around so enemies re-enter
      if (this.x < -50) this.x = canvas.width/DPR + 20;
      if (this.x > canvas.width/DPR + 50) this.x = -20;
      if (this.y < -50) this.y = canvas.height/DPR + 20;
      if (this.y > canvas.height/DPR + 50) this.y = -20;
    }
  }
  render(ctx){
    // shadow glow
    const g = ctx.createRadialGradient(this.x, this.y, 4, this.x, this.y, this.radius*2.4);
    g.addColorStop(0, 'rgba(255,190,110,0.14)');
    g.addColorStop(1, 'rgba(10,12,20,0)');
    ctx.fillStyle = g;
    ctx.beginPath(); ctx.arc(this.x,this.y,this.radius*2.2,0,Math.PI*2); ctx.fill();

    ctx.fillStyle = this.color;
    ctx.beginPath(); ctx.arc(this.x,this.y,this.radius,0,Math.PI*2); ctx.fill();
    // health bar
    const hpw = this.radius*2;
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fillRect(this.x-hpw/2, this.y - this.radius - 10, hpw, 4);
    ctx.fillStyle = 'rgba(0,180,120,0.9)';
    ctx.fillRect(this.x-hpw/2, this.y - this.radius - 10, hpw * (this.health / (this.type==='tank'?6:2)), 4);
  }
}

/* ---------- Particles ---------- */
class Particle extends Entity {
  constructor(x,y,vx,vy,life,color,radius=4){
    super(x,y); this.vx=vx; this.vy=vy; this.life=life; this.max=life; this.color=color; this.radius=radius;
  }
  update(dt){
    this.vx *= 0.99; this.vy *= 0.99; this.vy += 220 * dt; // gravity feel
    this.x += this.vx*dt; this.y += this.vy*dt;
    this.life -= dt; if (this.life <= 0) this.dead = true;
  }
  render(ctx){
    const t = this.life / this.max;
    ctx.globalAlpha = Math.max(0, t);
    ctx.beginPath();
    ctx.fillStyle = this.color;
    ctx.arc(this.x,this.y,this.radius * t,0,Math.PI*2); ctx.fill();
    ctx.globalAlpha = 1;
  }
}

/* ---------- Collision & Game rules ---------- */
function handleCollisions(dt){
  for (let i=entities.length-1;i>=0;i--){
    const a = entities[i];
    if (a.dead) continue;
    // bullets hit enemies
    if (a instanceof Bullet && a.owner === 'player') {
      for (let j=entities.length-1;j>=0;j--){
        const b = entities[j];
        if (b instanceof Enemy && !b.dead) {
          if (dist(a,b) < a.radius + b.radius) {
            // hit!
            a.dead = true;
            b.health -= 1;
            spawn(new Particle(a.x, a.y, rand(-120,120), rand(-120,120), 0.45, '#ffd6a6', 6));
            if (b.health <= 0) {
              b.dead = true;
              explode(b.x, b.y, Math.min(20, 6 + wave*0.6));
              score += (b.type==='tank'?50:20);
            }
            break;
          }
        }
      }
    }
    // enemy bullets hit player
    if (a instanceof Bullet && a.owner === 'enemy') {
      if (dist(a, player) < a.radius + player.radius) {
        a.dead = true;
        player.hitTimer = 0.35;
        lives -= 1;
        spawn(new Particle(player.x, player.y, rand(-240,240), rand(-240,240), 0.6, '#ff9d9d', 10));
        if (lives <= 0) {
          // player dead -> game over
          running = false;
          showGameOver();
        }
      }
    }
    // enemy touches player
    if (a instanceof Enemy) {
      if (dist(a, player) < a.radius + player.radius) {
        a.dead = true;
        lives -= 1;
        spawn(new Particle(player.x, player.y, rand(-240,240), rand(-240,240), 0.6, '#ff9d9d', 12));
        if (lives <= 0) { running = false; showGameOver(); }
      }
    }
  }
}

/* ---------- Explosion effect ---------- */
function explode(x,y,count=12){
  for (let i=0;i<count;i++){
    const a = rand(0,Math.PI*2), s = rand(80,420);
    spawn(new Particle(x,y, Math.cos(a)*s, Math.sin(a)*s, rand(0.4,1.0), '#ffd6a6', rand(3,10)));
  }
}

/* ---------- Wave spawn logic ---------- */
function spawnWave(n) {
  for (let i=0;i<n;i++){
    // spawn at edges randomly
    const side = Math.floor(rand(0,4));
    let x, y;
    if (side===0){ x = -20; y = rand(20, canvas.height/DPR-20); }
    else if (side===1){ x = canvas.width/DPR + 20; y = rand(20, canvas.height/DPR-20); }
    else if (side===2){ x = rand(20, canvas.width/DPR-20); y = -20; }
    else { x = rand(20, canvas.width/DPR-20); y = canvas.height/DPR + 20; }
    const type = Math.random() < Math.min(0.2 + wave*0.03, 0.6) ? 'tank' : 'drone';
    spawn(new Enemy(x,y, type));
  }
}

/* ---------- HUD & UI ---------- */
const hud = document.getElementById('hud');
function updateHUD(){ hud.innerHTML = `Score: ${score} &nbsp;&nbsp; Lives: ${lives} &nbsp;&nbsp; Wave: ${wave}`; }

function showGameOver(){
  // overlay
  ctx.save();
  ctx.fillStyle = 'rgba(3,6,12,0.7)';
  ctx.fillRect(0,0,canvas.width/DPR, canvas.height/DPR);
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 42px system-ui';
  ctx.textAlign = 'center';
  ctx.fillText('Game Over', canvas.width/DPR/2, canvas.height/DPR/2 - 20);
  ctx.font = '20px system-ui';
  ctx.fillText(`Score: ${score}`, canvas.width/DPR/2, canvas.height/DPR/2 + 20);
  ctx.restore();
}

/* ---------- Game instance ---------- */
let player = spawn(new Player(canvas.width/DPR/2, canvas.height/DPR - 120));
function resetGame(){
  entities.length = 0;
  player = spawn(new Player(canvas.width/DPR/2, canvas.height/DPR - 120));
  score = 0; wave = 1; lives = 3; running = true;
  spawnWave(4);
}

/* ---------- Main loop ---------- */
function step(ts) {
  if (!lastTime) lastTime = ts;
  const dt = Math.min(0.033, (ts - lastTime) / 1000); // clamp dt to avoid big jumps
  lastTime = ts;
  if (running) update(dt);
  render();
  window.requestAnimationFrame(step);
}

function update(dt){
  // spawn next wave if none left
  if (!entities.some(e=> e instanceof Enemy)) {
    wave += 1;
    spawnWave(3 + Math.floor(wave * 1.4));
  }

  // update entities
  for (let i=0;i<entities.length;i++){
    const e = entities[i];
    e.update(dt);
  }

  handleCollisions(dt);

  // cleanup
  for (let i=entities.length-1;i>=0;i--) if (entities[i].dead) entities.splice(i,1);

  updateHUD();
}

function clearScreen(){
  // Slight vignette + subtle noise-like background
  ctx.fillStyle = '#071021';
  ctx.fillRect(0,0,canvas.width/DPR, canvas.height/DPR);

  // soft gradient top light
  const g = ctx.createLinearGradient(0,0,0,canvas.height/DPR);
  g.addColorStop(0, 'rgba(30,40,80,0.12)');
  g.addColorStop(0.6, 'rgba(5,7,12,0.0)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,canvas.width/DPR, canvas.height/DPR);
}

function render(){
  clearScreen();

  // render entities in depth order
  entities.sort((a,b)=> (a.y || 0) - (b.y || 0));
  for (const e of entities) e.render(ctx);

  // UI overlay: simple crosshair where pointer is (desktop only)
  if (!input.touchMode) {
    const x = input.pointer.x || canvas.width/DPR/2, y = input.pointer.y || canvas.height/DPR/2;
    ctx.save();
    ctx.globalAlpha = 0.9;
    ctx.beginPath();
    ctx.strokeStyle = 'rgba(150,200,255,0.9)';
    ctx.moveTo(x-12,y); ctx.lineTo(x+12,y); ctx.moveTo(x,y-12); ctx.lineTo(x,y+12); ctx.stroke();
    ctx.restore();
  }

  // On-screen score text (small)
  ctx.save();
  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  ctx.fillRect(10, canvas.height/DPR - 44, 160, 36);
  ctx.fillStyle = '#cfe6ff';
  ctx.font = 'bold 16px system-ui';
  ctx.textAlign = 'left';
  ctx.fillText(`Score ${score}`, 18, canvas.height/DPR - 18);
  ctx.restore();
}

/* ---------- UI buttons ---------- */
document.getElementById('restartBtn').addEventListener('click', () => resetGame());
document.getElementById('pauseBtn').addEventListener('click', (e) => {
  running = !running;
  e.target.textContent = running ? 'Pause' : 'Resume';
});

/* ---------- Start ---------- */
resetGame();
window.requestAnimationFrame(step);

/* ---------- Extra: touch help (tap right to fire) ---------- */
// If user touches right half quickly, count as fire tap
let lastTap = 0;
canvas.addEventListener('touchstart', (ev) => {
  const t = ev.touches[0];
  const rect = canvas.getBoundingClientRect();
  const x = (t.clientX - rect.left);
  const now = Date.now();
  if (x > rect.width * 0.6 && now - lastTap > 90) {
    // fake pointer down to fire once
    input.pointer.down = true;
    setTimeout(()=> input.pointer.down = false, 120);
  }
  lastTap = now;
});
</script>
</body>
</html>
