<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pixel Sky Striker</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

        body {
            margin: 0;
            overflow: hidden;
            background-color: #222;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Press Start 2P', cursive;
        }

        #game-container {
            position: relative;
            box-shadow: 0 0 20px rgba(0,0,0,0.5);
            border: 4px solid #333;
        }

        canvas {
            display: block;
            background-color: #008f11; /* Base grass color */
            image-rendering: pixelated; /* Keeps it sharp */
        }

        /* UI Overlays */
        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            padding: 20px;
            box-sizing: border-box;
            display: flex;
            justify-content: space-between;
        }

        .hud-text {
            color: #fff;
            text-shadow: 2px 2px 0 #000;
            font-size: 16px;
            z-index: 10;
        }

        #game-over-screen, #start-screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.7);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            color: white;
            text-align: center;
            z-index: 20;
            backdrop-filter: blur(2px);
        }

        h1 {
            font-size: 30px;
            color: #bd9aff; /* Light purple */
            text-shadow: 4px 4px 0 #4b0082; /* Dark purple shadow */
            margin-bottom: 20px;
            line-height: 1.5;
        }

        button {
            font-family: 'Press Start 2P', cursive;
            background: #ffcc00;
            border: 4px solid #fff;
            padding: 15px 20px;
            font-size: 14px;
            cursor: pointer;
            color: #333;
            text-transform: uppercase;
            box-shadow: 0 4px 0 #b38f00;
            transition: transform 0.1s;
        }

        button:active {
            transform: translateY(4px);
            box-shadow: 0 0 0 #b38f00;
        }

        .hidden { display: none !important; }
    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="gameCanvas"></canvas>

        <!-- HUD -->
        <div id="ui-layer">
            <div class="hud-text">SCORE: <span id="score">0</span></div>
            <div class="hud-text">HP: <span id="hp">100</span>%</div>
        </div>

        <!-- Start Screen -->
        <div id="start-screen">
            <h1>PIXEL SKY<br>STRIKER</h1>
            <p style="font-size: 10px; margin-bottom: 30px; line-height: 20px;">
                WASD / ARROWS to Move<br>
                SPACE to Shoot
            </p>
            <button id="start-btn">TAKEOFF</button>
        </div>

        <!-- Game Over Screen -->
        <div id="game-over-screen" class="hidden">
            <h1>MISSION<br>FAILED</h1>
            <p style="margin-bottom: 20px;">FINAL SCORE: <span id="final-score">0</span></p>
            <button id="restart-btn">RETRY</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        
        // Game Settings
        const GAME_WIDTH = 600;
        const GAME_HEIGHT = 800;
        canvas.width = GAME_WIDTH;
        canvas.height = GAME_HEIGHT;

        // Colors from your image
        const COLORS = {
            grass: "#1a8c26",
            treeTop: "#32CD32",
            treeTrunk: "#8b4513",
            player: "#4b0082", // Indigo/Purple
            playerLight: "#6a0dad",
            enemy: "#cc0000",
            bullet: "#ffff00",
            powerupSpread: "#00ffff",
            powerupSpeed: "#ff00ff"
        };

        // Game State
        let gameActive = false;
        let score = 0;
        let frame = 0;
        let keys = {};
        let touchX = null;
        let touchY = null;

        // Entities
        let player;
        let bullets = [];
        let enemies = [];
        let particles = [];
        let trees = [];
        let powerups = [];

        // Inputs
        window.addEventListener('keydown', e => keys[e.code] = true);
        window.addEventListener('keyup', e => keys[e.code] = false);
        
        // Touch controls
        canvas.addEventListener('touchstart', e => {
            const rect = canvas.getBoundingClientRect();
            touchX = e.touches[0].clientX - rect.left;
            touchY = e.touches[0].clientY - rect.top;
            keys['Space'] = true; // Auto shoot on mobile
        });
        canvas.addEventListener('touchmove', e => {
            e.preventDefault();
            const rect = canvas.getBoundingClientRect();
            touchX = e.touches[0].clientX - rect.left;
            touchY = e.touches[0].clientY - rect.top;
        });
        canvas.addEventListener('touchend', () => {
            touchX = null;
            keys['Space'] = false;
        });

        document.getElementById('start-btn').addEventListener('click', startGame);
        document.getElementById('restart-btn').addEventListener('click', startGame);

        // --- CLASSES ---

        class Player {
            constructor() {
                this.w = 40;
                this.h = 40;
                this.x = GAME_WIDTH / 2 - this.w / 2;
                this.y = GAME_HEIGHT - 100;
                this.speed = 5;
                this.hp = 100;
                this.cooldown = 0;
                this.powerup = 'normal'; // normal, spread
                this.powerupTimer = 0;
            }

            update() {
                // Keyboard Movement
                if (keys['ArrowLeft'] || keys['KeyA']) this.x -= this.speed;
                if (keys['ArrowRight'] || keys['KeyD']) this.x += this.speed;
                if (keys['ArrowUp'] || keys['KeyW']) this.y -= this.speed;
                if (keys['ArrowDown'] || keys['KeyS']) this.y += this.speed;

                // Touch Movement (Follow finger)
                if (touchX !== null) {
                    const dx = touchX - (this.x + this.w/2);
                    const dy = touchY - (this.y + this.h/2);
                    this.x += dx * 0.1;
                    this.y += dy * 0.1;
                }

                // Boundaries
                if (this.x < 0) this.x = 0;
                if (this.x > GAME_WIDTH - this.w) this.x = GAME_WIDTH - this.w;
                if (this.y < 0) this.y = 0;
                if (this.y > GAME_HEIGHT - this.h) this.y = GAME_HEIGHT - this.h;

                // Shooting
                if (keys['Space']) {
                    this.shoot();
                }

                if (this.cooldown > 0) this.cooldown--;
                if (this.powerupTimer > 0) {
                    this.powerupTimer--;
                    if (this.powerupTimer <= 0) this.powerup = 'normal';
                }
            }

            shoot() {
                if (this.cooldown > 0) return;

                if (this.powerup === 'spread') {
                    bullets.push(new Bullet(this.x + this.w/2, this.y, 0, -10));
                    bullets.push(new Bullet(this.x + this.w/2, this.y, -3, -9));
                    bullets.push(new Bullet(this.x + this.w/2, this.y, 3, -9));
                    this.cooldown = 15;
                } else if (this.powerup === 'rapid') {
                    bullets.push(new Bullet(this.x + this.w/2, this.y, 0, -12));
                    this.cooldown = 5;
                } else {
                    bullets.push(new Bullet(this.x + this.w/2, this.y, 0, -10));
                    this.cooldown = 12;
                }
            }

            draw() {
                drawPixelPlane(this.x, this.y, COLORS.player, COLORS.playerLight);
            }
        }

        class Bullet {
            constructor(x, y, vx, vy, isEnemy = false) {
                this.x = x;
                this.y = y;
                this.vx = vx;
                this.vy = vy;
                this.r = 4;
                this.isEnemy = isEnemy;
                this.markedForDeletion = false;
            }

            update() {
                this.x += this.vx;
                this.y += this.vy;

                if (this.y < -10 || this.y > GAME_HEIGHT + 10) this.markedForDeletion = true;
            }

            draw() {
                ctx.fillStyle = this.isEnemy ? '#fff' : COLORS.bullet;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.r, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        class Enemy {
            constructor() {
                this.w = 40;
                this.h = 40;
                this.x = Math.random() * (GAME_WIDTH - this.w);
                this.y = -50;
                this.speed = Math.random() * 2 + 2;
                this.markedForDeletion = false;
                this.shootTimer = Math.random() * 100;
            }

            update() {
                this.y += this.speed;
                if (this.y > GAME_HEIGHT) this.markedForDeletion = true;

                this.shootTimer--;
                if (this.shootTimer <= 0) {
                    bullets.push(new Bullet(this.x + this.w/2, this.y + this.h, 0, 6, true));
                    this.shootTimer = 100 + Math.random() * 100;
                }
            }

            draw() {
                drawPixelPlane(this.x, this.y, COLORS.enemy, "#ff6666", true);
            }
        }

        class Tree {
            constructor() {
                this.x = Math.random() * GAME_WIDTH;
                this.y = -100;
                this.speed = 2; // Scroll speed
                this.markedForDeletion = false;
                this.scale = 0.8 + Math.random() * 0.4;
            }

            update() {
                this.y += this.speed;
                if (this.y > GAME_HEIGHT) this.markedForDeletion = true;
            }

            draw() {
                drawPixelTree(this.x, this.y, this.scale);
            }
        }

        class PowerUp {
            constructor(type) {
                this.type = type; // 'spread', 'rapid'
                this.x = Math.random() * (GAME_WIDTH - 30);
                this.y = -30;
                this.w = 30;
                this.h = 30;
                this.speed = 3;
                this.markedForDeletion = false;
            }

            update() {
                this.y += this.speed;
                if (this.y > GAME_HEIGHT) this.markedForDeletion = true;
            }

            draw() {
                ctx.fillStyle = this.type === 'spread' ? COLORS.powerupSpread : COLORS.powerupSpeed;
                ctx.fillRect(this.x, this.y, this.w, this.h);
                ctx.strokeStyle = '#fff';
                ctx.lineWidth = 2;
                ctx.strokeRect(this.x, this.y, this.w, this.h);
                
                ctx.fillStyle = "#000";
                ctx.font = "10px 'Press Start 2P'";
                ctx.fillText(this.type === 'spread' ? "S" : "R", this.x + 8, this.y + 20);
            }
        }

        class Particle {
            constructor(x, y, color) {
                this.x = x;
                this.y = y;
                this.vx = (Math.random() - 0.5) * 6;
                this.vy = (Math.random() - 0.5) * 6;
                this.life = 1;
                this.color = color;
                this.markedForDeletion = false;
            }
            update() {
                this.x += this.vx;
                this.y += this.vy;
                this.life -= 0.05;
                if (this.life <= 0) this.markedForDeletion = true;
            }
            draw() {
                ctx.globalAlpha = this.life;
                ctx.fillStyle = this.color;
                ctx.fillRect(this.x, this.y, 6, 6);
                ctx.globalAlpha = 1;
            }
        }

        // --- DRAWING HELPERS (Procedural Pixel Art) ---

        function drawPixelPlane(x, y, mainColor, lightColor, isEnemy = false) {
            ctx.save();
            ctx.translate(x, y);
            if (isEnemy) {
                ctx.translate(20, 20);
                ctx.rotate(Math.PI);
                ctx.translate(-20, -20);
            }

            // Body
            ctx.fillStyle = mainColor;
            ctx.fillRect(15, 5, 10, 30);
            
            // Wings
            ctx.fillStyle = lightColor;
            ctx.fillRect(0, 15, 40, 10);
            
            // Tail
            ctx.fillRect(10, 30, 20, 5);
            
            // Cockpit
            ctx.fillStyle = "#87CEEB"; // Sky blue
            ctx.fillRect(18, 10, 4, 8);

            // Propeller
            if (frame % 4 === 0) {
                ctx.fillStyle = "#ccc";
                ctx.fillRect(10, 0, 20, 2);
            }

            ctx.restore();
        }

        function drawPixelTree(x, y, scale) {
            ctx.save();
            ctx.translate(x, y);
            ctx.scale(scale, scale);

            // Shadow
            ctx.fillStyle = "rgba(0,0,0,0.2)";
            ctx.fillRect(10, 50, 20, 10);

            // Trunk (Chunky brown pixels)
            ctx.fillStyle = COLORS.treeTrunk;
            ctx.fillRect(15, 30, 10, 20); // Main trunk
            ctx.fillRect(10, 45, 5, 5);   // Root left
            ctx.fillRect(25, 45, 5, 5);   // Root right

            // Leaves (Chunky green circles/blocks)
            ctx.fillStyle = COLORS.treeTop;
            ctx.fillRect(5, 5, 30, 30);   // Main block
            ctx.fillRect(0, 10, 40, 20);  // Wide block
            ctx.fillRect(10, 0, 20, 40);  // Tall block
            
            // Highlight
            ctx.fillStyle = "#5cdb5c";
            ctx.fillRect(10, 5, 10, 10);

            ctx.restore();
        }

        // --- GAME LOGIC ---

        function startGame() {
            document.getElementById('start-screen').classList.add('hidden');
            document.getElementById('game-over-screen').classList.add('hidden');
            
            score = 0;
            gameActive = true;
            player = new Player();
            bullets = [];
            enemies = [];
            trees = [];
            powerups = [];
            particles = [];
            
            // Pre-populate some trees
            for(let i=0; i<10; i++) {
                let t = new Tree();
                t.y = Math.random() * GAME_HEIGHT;
                trees.push(t);
            }

            document.getElementById('score').innerText = score;
            document.getElementById('hp').innerText = player.hp;
            
            animate();
        }

        function spawnEntities() {
            // Spawn Enemy
            if (Math.random() < 0.02 + (score * 0.0001)) { // Gets harder over time
                enemies.push(new Enemy());
            }
            
            // Spawn Tree
            if (Math.random() < 0.03) {
                trees.push(new Tree());
            }

            // Spawn Powerup
            if (Math.random() < 0.002) {
                let type = Math.random() > 0.5 ? 'spread' : 'rapid';
                powerups.push(new PowerUp(type));
            }
        }

        function checkCollisions() {
            // Bullets hitting Enemies
            bullets.forEach(b => {
                if (b.isEnemy) {
                    // Enemy bullet hitting Player
                    if (
                        b.x > player.x && b.x < player.x + player.w &&
                        b.y > player.y && b.y < player.y + player.h
                    ) {
                        b.markedForDeletion = true;
                        player.hp -= 10;
                        createExplosion(player.x + 20, player.y + 20, COLORS.player);
                        if (player.hp <= 0) gameOver();
                    }
                } else {
                    // Player bullet hitting Enemy
                    enemies.forEach(e => {
                        if (
                            b.x > e.x && b.x < e.x + e.w &&
                            b.y > e.y && b.y < e.y + e.h
                        ) {
                            b.markedForDeletion = true;
                            e.markedForDeletion = true;
                            createExplosion(e.x + 20, e.y + 20, "#ff9900");
                            score += 100;
                        }
                    });
                }
            });

            // Player hitting Enemy (Crash)
            enemies.forEach(e => {
                if (
                    player.x < e.x + e.w && player.x + player.w > e.x &&
                    player.y < e.y + e.h && player.y + player.h > e.y
                ) {
                    e.markedForDeletion = true;
                    player.hp -= 30;
                    createExplosion(e.x + 20, e.y + 20, "#ff0000");
                    if (player.hp <= 0) gameOver();
                }
            });

            // Player hitting Powerup
            powerups.forEach(p => {
                if (
                    player.x < p.x + p.w && player.x + player.w > p.x &&
                    player.y < p.y + p.h && player.y + player.h > p.y
                ) {
                    p.markedForDeletion = true;
                    player.powerup = p.type;
                    player.powerupTimer = 600; // 10 seconds approx
                    score += 50;
                }
            });
        }

        function createExplosion(x, y, color) {
            for (let i = 0; i < 10; i++) {
                particles.push(new Particle(x, y, color));
            }
        }

        function gameOver() {
            gameActive = false;
            document.getElementById('final-score').innerText = score;
            document.getElementById('game-over-screen').classList.remove('hidden');
        }

        function animate() {
            if (!gameActive) return;
            
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw Background (Grass)
            ctx.fillStyle = COLORS.grass;
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            spawnEntities();

            // --- LAYERING ---
            // 1. Trees (Below planes)
            trees.forEach(t => { t.update(); t.draw(); });
            trees = trees.filter(t => !t.markedForDeletion);
            // Sort trees by Y so lower trees overlap higher ones (2.5D effect)
            trees.sort((a, b) => a.y - b.y);

            // 2. Powerups
            powerups.forEach(p => { p.update(); p.draw(); });
            powerups = powerups.filter(p => !p.markedForDeletion);

            // 3. Player
            player.update();
            player.draw();

            // 4. Enemies
            enemies.forEach(e => { e.update(); e.draw(); });
            enemies = enemies.filter(e => !e.markedForDeletion);

            // 5. Bullets
            bullets.forEach(b => { b.update(); b.draw(); });
            bullets = bullets.filter(b => !b.markedForDeletion);

            // 6. Particles
            particles.forEach(p => { p.update(); p.draw(); });
            particles = particles.filter(p => !p.markedForDeletion);

            // UI Update
            document.getElementById('score').innerText = score;
            document.getElementById('hp').innerText = Math.max(0, player.hp);
            
            checkCollisions();

            frame++;
            requestAnimationFrame(animate);
        }

    </script>
</body>
</html>
