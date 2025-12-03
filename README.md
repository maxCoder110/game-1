<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Pixel Plane Shooter</title>
    <style>
        body {
            background-color: #222;
            display: flex;
            justify-content: center;
            align-items: center;
            color: white;
            font-family: 'Courier New', Courier, monospace;
            height: 100vh;
            margin: 0;
            overflow: hidden;
        }
        #gameContainer {
            position: relative;
            /* Scale up the small canvas to look chunky and pixelated */
            transform: scale(3); 
            transform-origin: center center;
            image-rendering: pixelated; /* Key for the retro look */
            box-shadow: 0 0 20px rgba(0,0,0,0.5);
        }
        canvas {
            background-color: #2a9d2a; /* Base grass color */
            display: block;
            border: 2px solid #1a6d1a;
        }
        #uiLayer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            padding: 5px;
            box-sizing: border-box;
            display: flex;
            justify-content: space-between;
            text-shadow: 1px 1px 0 #000;
            font-size: 10px;
            pointer-events: none;
        }
        #gameOverScreen {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%) scale(0.5); /* Counteract container scale */
            text-align: center;
            display: none;
            z-index: 10;
            background: rgba(0,0,0,0.8);
            padding: 20px;
            border: 2px solid white;
        }
    </style>
</head>
<body>

<div id="gameContainer">
    <canvas id="gameCanvas" width="240" height="320"></canvas>
    <div id="uiLayer">
        <div>Score: <span id="score">0</span></div>
        <div>Level: <span id="level">1</span></div>
    </div>
    <div id="gameOverScreen">
        <h1 id="goTitle">GAME OVER</h1>
        <p>Final Score: <span id="finalScore">0</span></p>
        <p>Press ENTER to restart</p>
    </div>
</div>

<script>
/** setup **/
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const levelEl = document.getElementById('level');
const goScreen = document.getElementById('gameOverScreen');
const finalScoreEl = document.getElementById('finalScore');
const goTitle = document.getElementById('goTitle');

let gameState = 'playing'; // playing, gameover, bossEntrance
let score = 0;
let level = 1;
let frames = 0;
let gameSpeed = 1.5; // Base scrolling speed

// Input handling
const keys = {};
document.addEventListener('keydown', e => keys[e.code] = true);
document.addEventListener('keyup', e => keys[e.code] = false);
document.addEventListener('keydown', e => {
    if (gameState === 'gameover' && e.code === 'Enter') resetGame();
});

/** Classes **/

// Helper for drawing health bars
function drawHealthBar(ctx, x, y, width, health, maxHealth) {
    if (health >= maxHealth) return; // Don't show if full health
    const barWidth = width;
    const barHeight = 3;
    const yOffset = -5;
    const healthPct = Math.max(0, health / maxHealth);
    
    ctx.fillStyle = '#555';
    ctx.fillRect(x, y + yOffset, barWidth, barHeight);
    ctx.fillStyle = healthPct > 0.5 ? '#0f0' : healthPct > 0.2 ? '#ff0' : '#f00';
    ctx.fillRect(x, y + yOffset, barWidth * healthPct, barHeight);
}


class Player {
    constructor() {
        this.width = 16;
        this.height = 16;
        this.x = canvas.width / 2 - this.width / 2;
        this.y = canvas.height - 40;
        this.speed = 3;
        this.color = '#777'; // Grey plane
        this.bullets = [];
        this.lastShot = 0;
        this.maxHealth = 100;
        this.health = this.maxHealth;
        this.powerUpActive = false;
        this.powerUpTimer = 0;
    }

    update() {
        // Movement
        if ((keys['ArrowLeft'] || keys['KeyA']) && this.x > 0) this.x -= this.speed;
        if ((keys['ArrowRight'] || keys['KeyD']) && this.x + this.width < canvas.width) this.x += this.speed;
        if ((keys['ArrowUp'] || keys['KeyW']) && this.y > 0) this.y -= this.speed;
        if ((keys['ArrowDown'] || keys['KeyS']) && this.y + this.height < canvas.height) this.y += this.speed;

        // Shooting
        if (keys['Space']) {
            const now = Date.now();
            if (now - this.lastShot > 250) { // Fire rate limitation
                if(this.powerUpActive) {
                    // Spread shot
                     this.bullets.push(new Bullet(this.x + this.width/2, this.y, 0, -5, '#ff0'));
                     this.bullets.push(new Bullet(this.x, this.y + 4, -1.5, -4.5, '#ff0'));
                     this.bullets.push(new Bullet(this.x + this.width, this.y + 4, 1.5, -4.5, '#ff0'));
                } else {
                    // Normal shot
                    this.bullets.push(new Bullet(this.x + this.width / 2, this.y, 0, -5, '#fff'));
                }
                this.lastShot = now;
            }
        }

        // Powerup timer
        if(this.powerUpActive && Date.now() > this.powerUpTimer) {
            this.powerUpActive = false;
        }

        // Update bullets
        this.bullets.forEach(b => b.update());
        this.bullets = this.bullets.filter(b => !b.markedForDeletion);
    }

    draw() {
        // Draw Player Plane (simple shape meant to look like the grey plane)
        ctx.fillStyle = this.color;
        // Fuselage
        ctx.fillRect(this.x + 6, this.y, 4, 16);
        // Wings
        ctx.fillRect(this.x, this.y + 6, 16, 4);
        // Tail
        ctx.fillRect(this.x + 4, this.y + 14, 8, 2);
        
        drawHealthBar(ctx, this.x, this.y, this.width, this.health, this.maxHealth);
        this.bullets.forEach(b => b.draw());
    }
}

class Bullet {
    constructor(x, y, vx, vy, color) {
        this.width = 3;
        this.height = 6;
        this.x = x - this.width/2;
        this.y = y;
        this.vx = vx;
        this.vy = vy;
        this.color = color;
        this.markedForDeletion = false;
    }
    update() {
        this.x += this.vx;
        this.y += this.vy;
        if (this.y < 0 || this.y > canvas.height || this.x < 0 || this.x > canvas.width) {
            this.markedForDeletion = true;
        }
    }
    draw() {
        ctx.fillStyle = this.color;
        ctx.fillRect(Math.round(this.x), Math.round(this.y), this.width, this.height);
    }
}


// Base Enemy Class
class Enemy {
    constructor(x, y, width, height, speed, health, points) {
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
        this.speed = speed;
        this.maxHealth = health;
        this.health = health;
        this.points = points;
        this.markedForDeletion = false;
    }
    takeDamage(amount) {
        this.health -= amount;
        if(this.health <= 0) {
            this.markedForDeletion = true;
            score += this.points;
            checkLevelUp();
            // Chance to drop powerup
            if(Math.random() < 0.15) {
                powerUps.push(new PowerUp(this.x + this.width/2, this.y + this.height/2));
            }
        }
    }
    update() { this.y += this.speed; }
    drawHealth() { drawHealthBar(ctx, this.x, this.y, this.width, this.health, this.maxHealth); }
}

class EnemyPlane extends Enemy {
    constructor(x, y, speedMultiplier) {
        super(x, y, 16, 14, (1.5 + Math.random()) * speedMultiplier, 20, 100);
        this.color = '#c00'; // Red plane
    }
    draw() {
        ctx.fillStyle = this.color;
        // Fuselage
        ctx.fillRect(this.x + 6, this.y, 4, 14);
        // Wings
        ctx.fillRect(this.x, this.y + 4, 16, 4);
        // Tail
        ctx.fillRect(this.x + 4, this.y, 8, 2);
        this.drawHealth();
    }
}

class EnemyTank extends Enemy {
    constructor(x, y) {
        // Tanks move slower, have more health, stay on "ground"
        super(x, y, 18, 18, gameSpeed, 60, 250);
        this.color = '#555'; // Dark grey tank body
        this.turretColor = '#ccc'; // Lighter turret
    }
    draw() {
        // Draw tank body
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x, this.y, this.width, this.height);
        // Draw treads stripes
        ctx.fillStyle = '#333';
        ctx.fillRect(this.x, this.y, 4, this.height);
        ctx.fillRect(this.x + this.width - 4, this.y, 4, this.height);
        // Draw turret
        ctx.fillStyle = this.turretColor;
        ctx.fillRect(this.x + 5, this.y + 5, 8, 8);
        // Draw barrel pointing down
        ctx.fillRect(this.x + 7, this.y + 10, 4, 6);
        
        this.drawHealth();
    }
}

class Boss extends Enemy {
    constructor() {
        super(canvas.width/2 - 32, -70, 64, 48, 1, 1500, 5000);
        this.color = '#300'; // Dark red boss
        this.vx = 1;
        this.state = 'entering'; // entering, fighting
    }
    update() {
        if(this.state === 'entering') {
            this.y += this.speed;
            if(this.y > 20) {
                this.state = 'fighting';
                gameState = 'bossFight';
            }
        } else {
            // Side to side movement
            this.x += this.vx;
            if(this.x <= 0 || this.x + this.width >= canvas.width) this.vx *= -1;
            
            // Boss shooting (simple random bursts)
            if(frames % 40 === 0 && Math.random() > 0.3) {
                 enemyBullets.push(new Bullet(this.x + this.width/2, this.y + this.height, 0, 4, '#f90'));
                 enemyBullets.push(new Bullet(this.x + 10, this.y + this.height - 10, -1.5, 3.5, '#f90'));
                 enemyBullets.push(new Bullet(this.x + this.width - 10, this.y + this.height - 10, 1.5, 3.5, '#f90'));
            }
        }
    }
    draw() {
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x, this.y, this.width, this.height);
        // Add some boss details
        ctx.fillStyle = '#700';
        ctx.fillRect(this.x, this.y + 20, this.width, 8); // Wing stripe
        ctx.fillRect(this.x + 24, this.y + 30, 16, 12); // Cockpit area

        // Boss health bar is bigger
        drawHealthBar(ctx, this.x, this.y - 5, this.width, this.health, this.maxHealth);
    }
}

class Tree {
    constructor(x, y) {
        this.x = x;
        this.y = y;
        this.width = 20;
        this.height = 24;
        this.markedForDeletion = false;
    }
    update() {
        this.y += gameSpeed;
        if(this.y > canvas.height) this.markedForDeletion = true;
    }
    draw() {
        // Trunk
        ctx.fillStyle = '#642';
        ctx.fillRect(this.x + 8, this.y + 16, 4, 8);
        // Leaves (circles-ish)
        ctx.fillStyle = '#2a2';
        ctx.beginPath(); ctx.arc(this.x + 10, this.y + 8, 8, 0, Math.PI*2); ctx.fill();
        ctx.beginPath(); ctx.arc(this.x + 4, this.y + 12, 6, 0, Math.PI*2); ctx.fill();
        ctx.beginPath(); ctx.arc(this.x + 16, this.y + 12, 6, 0, Math.PI*2); ctx.fill();
    }
}

class PowerUp {
    constructor(x, y) {
        this.x = x;
        this.y = y;
        this.width = 12;
        this.height = 12;
        this.markedForDeletion = false;
        this.color = '#0cf'; // Cyan
    }
    update() {
        this.y += 1; // Drifts slowly
        if(this.y > canvas.height) this.markedForDeletion = true;
    }
    draw() {
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x, this.y, this.width, this.height);
        // 'P' for powerup
        ctx.fillStyle = 'white';
        ctx.font = '10px monospace';
        ctx.fillText('P', this.x + 3, this.y + 10);
    }
}

/** Game State Variables **/
let player;
let enemies = [];
let enemyBullets = [];
let trees = [];
let powerUps = [];
let bossActive = false;

function init() {
    player = new Player();
    enemies = [];
    enemyBullets = [];
    trees = [];
    powerUps = [];
    score = 0;
    level = 1;
    frames = 0;
    bossActive = false;
    gameState = 'playing';
    goScreen.style.display = 'none';
    
    // Pre-seed some trees
    for(let i=0; i<10; i++) {
         trees.push(new Tree(Math.random() * (canvas.width - 20), Math.random() * canvas.height));
    }
}

function resetGame() {
    init();
}

function checkLevelUp() {
    const levelThresholds = [0, 1500, 4000, 8000, 15000]; // Score needed for levels 1, 2, 3, 4, 5
    if (level < 5 && score >= levelThresholds[level]) {
        level++;
        gameSpeed += 0.2; // Background scrolls faster
    }
}

function spawn() {
    frames++;

    // Spawn Trees (Scenery)
    if (frames % Math.floor(60 / gameSpeed) === 0) {
         trees.push(new Tree(Math.random() * (canvas.width - 20), -30));
    }

    // Don't spawn regular enemies if boss is coming or active
    if (level === 5 && !bossActive) {
        bossActive = true;
        gameState = 'bossEntrance';
        enemies.push(new Boss());
        return;
    }

    if (bossActive) return;

    // Spawn Enemies based on level difficulty
    let spawnRate = Math.max(20, 80 - (level * 10)); // Spawn faster each level
    
    if (frames % spawnRate === 0) {
        const xPos = Math.random() * (canvas.width - 20);
        // Mix of planes and tanks depending on level
        if (Math.random() > 0.3 + (level * 0.1)) {
             enemies.push(new EnemyPlane(xPos, -20, 1 + (level * 0.2)));
        } else {
             // Tanks need to spawn further up to roll onto screen
             enemies.push(new EnemyTank(xPos, -30));
        }
    }
}

// Axis-Aligned Bounding Box Collision
function checkCollision(rect1, rect2) {
    return (
        rect1.x < rect2.x + rect2.width &&
        rect1.x + rect1.width > rect2.x &&
        rect1.y < rect2.y + rect2.height &&
        rect1.y + rect1.height > rect2.y
    );
}

function handleCollisions() {
    // Player Bullets hitting Enemies
    player.bullets.forEach(bullet => {
        enemies.forEach(enemy => {
            if (!bullet.markedForDeletion && !enemy.markedForDeletion && checkCollision(bullet, enemy)) {
                bullet.markedForDeletion = true;
                enemy.takeDamage(10); // Standard bullet damage
            }
        });
    });

    // Player hitting Enemies (crash)
    enemies.forEach(enemy => {
        if (!enemy.markedForDeletion && checkCollision(player, enemy)) {
            enemy.takeDamage(100); // Instantly destroy most non-boss enemies
            player.health -= 30;
        }
    });
    
    // Enemy Bullets hitting Player
    enemyBullets.forEach(bullet => {
         if (!bullet.markedForDeletion && checkCollision(bullet, player)) {
             bullet.markedForDeletion = true;
             player.health -= 15;
         }
    });

    // Player hitting Powerups
    powerUps.forEach(pu => {
        if(!pu.markedForDeletion && checkCollision(player, pu)) {
            pu.markedForDeletion = true;
            player.powerUpActive = true;
            player.powerUpTimer = Date.now() + 5000; // 5 seconds duration
            player.health = Math.min(player.maxHealth, player.health + 20); // Heal 20
            score += 50; // Bonus points
        }
    });

    // Check Player Death
    if(player.health <= 0) {
        gameState = 'gameover';
    }
}

function updateGame() {
    if (gameState === 'gameover') return;

    player.update();
    spawn();
    trees.forEach(t => t.update());
    enemies.forEach(e => e.update());
    enemyBullets.forEach(b => b.update());
    powerUps.forEach(p => p.update());

    if(gameState !== 'bossEntrance') {
        handleCollisions();
    }

    // Cleanup
    trees = trees.filter(t => !t.markedForDeletion);
    enemies = enemies.filter(e => !e.markedForDeletion);
    enemyBullets = enemyBullets.filter(b => !b.markedForDeletion);
    powerUps = powerUps.filter(p => !p.markedForDeletion);

    // Victory Condition (Defeated Boss)
    if (level === 5 && bossActive && enemies.length === 0 && gameState === 'bossFight') {
        gameState = 'gameover';
        goTitle.innerText = "MISSION COMPLETE!";
        score += 10000; // Big boss bonus
    }
}


function drawGame() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Draw order matters for layering
    trees.forEach(t => t.draw()); // Scenery on bottom
    enemies.filter(e => e instanceof EnemyTank).forEach(e => e.draw()); // Tanks on ground
    powerUps.forEach(p => p.draw()); // Items on ground level
    enemies.filter(e => !(e instanceof EnemyTank)).forEach(e => e.draw()); // Planes/Boss in air
    enemyBullets.forEach(b => b.draw());
    player.draw(); // Player on top

    // Update UI
    scoreEl.innerText = score;
    levelEl.innerText = level;

    if (gameState === 'gameover') {
        ctx.fillStyle = 'rgba(0,0,0,0.5)';
        ctx.fillRect(0,0,canvas.width, canvas.height);
        finalScoreEl.innerText = score;
        goScreen.style.display = 'block';
    }
}

// Main Game Loop
let lastTime = 0;
function animate(timeStamp) {
    const deltaTime = timeStamp - lastTime;
    lastTime = timeStamp;

    updateGame();
    drawGame();
    requestAnimationFrame(animate);
}

// Start
init();
animate(0);

</script>
</body>
</html>
