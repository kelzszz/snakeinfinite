<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Infinite Snake: Hardcore Edition</title>
    <style>
        body {
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            min-height: 100vh; margin: 0; background-color: #050505; color: #ecf0f1;
            font-family: sans-serif; overflow: hidden; touch-action: none;
        }
        #game-container { position: relative; }
        canvas { border: 4px solid #333; background-color: #000; max-width: 95vw; max-height: 45vh; display: block; }
        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.9); display: none; flex-direction: column;
            align-items: center; justify-content: center; z-index: 10;
        }
        #overlay h2 { color: #e74c3c; font-size: 42px; margin: 0; }
        #overlay button {
            margin-top: 20px; padding: 12px 25px; font-size: 18px; 
            background: #2ecc71; border: none; color: white; border-radius: 5px;
        }
        .stats { display: flex; flex-direction: column; align-items: center; gap: 5px; font-size: 18px; margin: 5px 0; font-weight: bold; }
        .weapon-info { color: #f1c40f; font-weight: bold; margin-bottom: 5px; text-align: center; }
        #ally-status { font-size: 14px; margin-top: 2px; }

        .controls-wrapper { background: rgba(255,255,255,0.05); padding: 10px; border-radius: 20px; margin-top: 10px; }
        .controls { display: grid; grid-template-areas: ". up ." "left . right" ". down ."; gap: 8px; }
        .btn { 
            width: 60px; height: 60px; background: #34495e; border: none; border-radius: 12px; 
            color: white; font-size: 24px; display: flex; align-items: center; justify-content: center; 
            user-select: none; -webkit-tap-highlight-color: transparent;
            box-shadow: 0 4px #1a252f; cursor: pointer;
        }
        .btn:active { background: #27ae60; transform: translateY(2px); box-shadow: 0 2px #1a252f; }
        .up { grid-area: up; } .left { grid-area: left; } .right { grid-area: right; } .down { grid-area: down; }
    </style>
</head>
<body>

    <div class="stats">
        <div id="scoreBoard">Score: 0</div>
        <div id="ally-status" style="color: #95a5a6;">Ally: Non Active</div>
    </div>
    <div class="weapon-info">
        <div id="weaponDisplay">Weapon: Loading...</div>
    </div>

    <div id="game-container">
        <canvas id="snakeGame" width="600" height="600"></canvas>
        <div id="overlay">
            <h2>GAME OVER</h2>
            <p id="finalScore">Score: 0</p>
            <button onmousedown="location.reload()" ontouchstart="location.reload()">RESTART</button>
        </div>
    </div>

    <div class="controls-wrapper">
        <div class="controls">
            <div class="btn up" id="ctrl-UP">▲</div>
            <div class="btn left" id="ctrl-LEFT">◀</div>
            <div class="btn right" id="ctrl-RIGHT">▶</div>
            <div class="btn down" id="ctrl-DOWN">▼</div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById("snakeGame");
        const ctx = canvas.getContext("2d");
        const box = 20;
        const viewSize = 600;

        let score = 0, foodCounter = 0;
        let snake = [{x: 300, y: 300}, {x: 280, y: 300}, {x: 260, y: 300}];
        let ally = null, allyTimer = 0;
        let foods = [], enemies = [], bullets = [];
        let d = "RIGHT", nextD = "RIGHT", isGameOver = false;
        let camX = 0, camY = 0;

        const Weapons = {
            SNIPER: { name: "SNIPER", rate: 800, dmg: 2, speed: 18, color: "#3498db", pierce: 2 },
            MACHINEGUN: { name: "MACHINE GUN", rate: 250, dmg: 1.2, speed: 22, color: "#2ecc71", pierce: 0 },
            ROCKET: { name: "HOMING ROCKET", rate: 2000, dmg: 3, speed: 8, color: "#f1c40f", isAoe: true, pierce: 0 },
            SPREAD: { name: "SPREADSHOT", rate: 1000, dmg: 1.8, speed: 14, color: "#9b59b6", isSpread: true, pierce: 2 }
        };
        let currentWeapon = Weapons.SNIPER;
        let weaponTimer = 25, fireInterval;

        const starLayers = [
            { count: 40, size: 1, speed: 0.2, color: "#444", stars: [] },
            { count: 20, size: 2, speed: 0.5, color: "#666", stars: [] }
        ];
        starLayers.forEach(l => { for(let i=0; i<l.count; i++) l.stars.push({x:Math.random()*600, y:Math.random()*600}); });

        function setDirection(dir) {
            if (isGameOver) return;
            if(dir == "LEFT" && d != "RIGHT") nextD = "LEFT";
            else if(dir == "UP" && d != "DOWN") nextD = "UP";
            else if(dir == "RIGHT" && d != "LEFT") nextD = "RIGHT";
            else if(dir == "DOWN" && d != "UP") nextD = "DOWN";
        }

        const controlIds = ["UP", "DOWN", "LEFT", "RIGHT"];
        controlIds.forEach(id => {
            const el = document.getElementById("ctrl-" + id);
            const handler = (e) => { e.preventDefault(); setDirection(id); };
            el.addEventListener("mousedown", handler);
            el.addEventListener("touchstart", handler);
        });

        function spawnWorldEntity(type) {
            const angle = Math.random() * Math.PI * 2;
            const dist = 600 + Math.random() * 200;
            const spawnX = Math.floor((snake[0].x + Math.cos(angle) * dist) / box) * box;
            const spawnY = Math.floor((snake[0].y + Math.sin(angle) * dist) / box) * box;
            if (type === 'food') {
                foods.push({ x: spawnX, y: spawnY, hp: Math.floor(Math.random() * 5) + 1 });
            } else {
                // Tier progression: White Gray > White > Red
                // HP +5 buff applied, Speed +50% applied
                const Tiers = [
                    {color:"#bdc3c7", hp:6, speed:7.5},  // White Gray (5+1 hp, 5*1.5 spd)
                    {color:"#ffffff", hp:8, speed:9},    // White (5+3 hp, 6*1.5 spd)
                    {color:"#e74c3c", hp:10, speed:10.5} // Red (5+5 hp, 7*1.5 spd)
                ];
                let t = Tiers[Math.floor(Math.random() * Tiers.length)];
                enemies.push({ x: spawnX, y: spawnY, ...t });
            }
        }

        function rollRandomWeapon() {
            const keys = Object.keys(Weapons);
            currentWeapon = Weapons[keys[Math.floor(Math.random() * keys.length)]];
            weaponTimer = 25;
            clearInterval(fireInterval);
            fireInterval = setInterval(() => { if(!isGameOver) fire(snake[0], currentWeapon); }, currentWeapon.rate);
        }

        function fire(origin, weapon, isAlly = false) {
            let targets = [...enemies, ...foods];
            if (targets.length === 0) return;
            let closest = targets.reduce((p, c) => Math.hypot(c.x-origin.x, c.y-origin.y) < Math.hypot(p.x-origin.x, p.y-origin.y) ? c : p);
            let baseAngle = Math.atan2(closest.y - origin.y, closest.x - origin.x);
            let count = (weapon.isSpread && !isAlly) ? 5 : 1;
            for(let i=0; i<count; i++) {
                let angle = baseAngle + (weapon.isSpread ? (i - 2) * 0.2 : 0);
                bullets.push({ 
                    x: origin.x + 10, y: origin.y + 10, 
                    vx: Math.cos(angle) * weapon.speed, vy: Math.sin(angle) * weapon.speed, 
                    dmg: (isAlly ? 0.3 : weapon.dmg), color: isAlly ? "#3498db" : weapon.color,
                    isAoe: !isAlly && weapon.isAoe, 
                    pierce: (isAlly ? 0 : weapon.pierce || 0),
                    hitList: [], 
                    life: 0 
                });
            }
        }

        function triggerAoe(x, y, dmg) {
            enemies.forEach(e => { if(Math.hypot(x-e.x, y-e.y) < 120) e.hp -= dmg; });
            foods.forEach(f => { if(Math.hypot(x-f.x, y-f.y) < 120) f.hp -= dmg; });
        }

        function update() {
            if(isGameOver) return;
            d = nextD;
            let head = { ...snake[0] };
            if(d == "LEFT") head.x -= box; if(d == "UP") head.y -= box;
            if(d == "RIGHT") head.x += box; if(d == "DOWN") head.y += box;
            for(let p of snake) if(head.x == p.x && head.y == p.y) return endGame();
            camX = head.x - viewSize / 2; camY = head.y - viewSize / 2;

            foods.forEach((f, i) => {
                if(head.x == f.x && head.y == f.y) {
                    score++; foodCounter++; foods.splice(i, 1); spawnWorldEntity('food');
                    snake.push({...snake[snake.length-1]});
                    if(foodCounter >= 15) { 
                        ally = {x: head.x, y: head.y}; 
                        allyTimer = 45;
                        foodCounter = 0; 
                    }
                }
            });

            snake.unshift(head); snake.pop();

            if(ally) {
                let t = enemies[0] || foods[0];
                if(t) { let a = Math.atan2(t.y - ally.y, t.x - ally.x); ally.x += Math.cos(a)*6.5; ally.y += Math.sin(a)*6.5; }
                if(allyTimer <= 0) ally = null;
            }

            enemies.forEach(en => {
                let a = Math.atan2(head.y - en.y, head.x - en.x);
                en.x += Math.cos(a) * en.speed; en.y += Math.sin(a) * en.speed;
                if(Math.hypot(en.x - head.x, en.y - head.y) < box) endGame();
            });

            bullets.forEach((b, bi) => {
                b.life += 100;
                b.x += b.vx; b.y += b.vy;
                let hitOccurred = false;

                [...enemies, ...foods].forEach(ent => {
                    if(!b.hitList.includes(ent) && Math.hypot(b.x - ent.x, b.y - ent.y) < (ent.size || box)) {
                        if(b.isAoe) {
                            triggerAoe(b.x, b.y, b.dmg);
                            b.pierce = 0;
                        } else {
                            ent.hp -= b.dmg;
                            b.hitList.push(ent);
                        }
                        
                        if(b.pierce > 0) b.pierce--;
                        else hitOccurred = true;
                    }
                });

                if(hitOccurred || b.life > 5000) bullets.splice(bi, 1);
            });

            enemies = enemies.filter(en => en.hp > 0);
            foods = foods.filter(f => {
                if(f.hp <= 0) { score++; foodCounter++; spawnWorldEntity('food'); snake.push({...snake[snake.length-1]}); return false; }
                return true;
            });

            // POPULATION CONTROLLER: Maintain 10-15 enemies
            if(enemies.length < 12) {
                spawnWorldEntity('enemy');
            }
        }

        function endGame() { isGameOver = true; document.getElementById("overlay").style.display = "flex"; document.getElementById("finalScore").innerText = "Final Score: " + score; }

        function draw() {
            ctx.fillStyle = "#050505"; ctx.fillRect(0, 0, viewSize, viewSize);
            starLayers.forEach(layer => {
                ctx.fillStyle = layer.color;
                layer.stars.forEach(s => {
                    let sx = (s.x - camX * layer.speed) % viewSize, sy = (s.y - camY * layer.speed) % viewSize;
                    if(sx < 0) sx += viewSize; if(sy < 0) sy += viewSize;
                    ctx.fillRect(sx, sy, layer.size, layer.size);
                });
            });
            foods.forEach(f => { ctx.fillStyle = "#f1c40f"; ctx.fillRect(f.x-camX, f.y-camY, box, box); ctx.fillStyle = "black"; ctx.font = "bold 12px Arial"; ctx.fillText(Math.ceil(f.hp), f.x-camX+6, f.y-camY+15); });
            enemies.forEach(en => { ctx.fillStyle = en.color; ctx.fillRect(en.x-camX, en.y-camY, box, box); });
            bullets.forEach(b => { ctx.fillStyle = b.color; ctx.fillRect(b.x-camX, b.y-camY, b.isAoe?10:5, b.isAoe?10:5); });
            if(ally) { ctx.fillStyle = "#3498db"; ctx.fillRect(ally.x-camX, ally.y-camY, box, box); ctx.strokeStyle="white"; ctx.strokeRect(ally.x-camX, ally.y-camY, box, box); }
            snake.forEach((p, i) => { ctx.fillStyle = (i == 0) ? "#2ecc71" : "#27ae60"; ctx.fillRect(p.x-camX, p.y-camY, box-1, box-1); if(i==0) { ctx.fillStyle = "white"; ctx.fillRect(p.x-camX+5, p.y-camY+5, 10, 10); } });
            requestAnimationFrame(draw);
        }

        document.addEventListener("keydown", (e) => { const keys = {37: "LEFT", 38: "UP", 39: "RIGHT", 40: "DOWN"}; if(keys[e.keyCode]) setDirection(keys[e.keyCode]); });
        setInterval(update, 100);
        setInterval(() => { if(!isGameOver && ally) fire(ally, Weapons.MACHINEGUN, true); }, 500);
        setInterval(() => { if(!isGameOver) { 
            weaponTimer--; 
            if(weaponTimer <= 0) rollRandomWeapon(); 
            
            const allyEl = document.getElementById("ally-status");
            if(ally) { 
                allyTimer--; 
                allyEl.innerText = `Ally: Active (${allyTimer}s)`;
                allyEl.style.color = "#3498db";
            } else {
                allyEl.innerText = `Ally: Non Active (${foodCounter}/15 food)`;
                allyEl.style.color = "#95a5a6";
            }
            
            document.getElementById("weaponDisplay").innerText = `Weapon: ${currentWeapon.name} (${weaponTimer}s)`; 
            document.getElementById("scoreBoard").innerText = "Score: " + score; 
        } }, 1000);
        rollRandomWeapon(); 
        for(let i=0; i<20; i++) spawnWorldEntity('food');
        draw();
    </script>
</body>
</html>        .kills { color: #e74c3c; }
        .ally-timer { color: #3498db; display: none; animation: blink 1s infinite; }
        @keyframes blink { 50% { opacity: 0.5; } }
        .controls { display: grid; grid-template-areas: ". up ." "left . right" ". down ."; gap: 10px; margin: 10px; }
        .btn {
            width: 60px; height: 60px; background: #2c3e50; border: none;
            border-radius: 50%; color: white; font-size: 24px;
            display: flex; align-items: center; justify-content: center;
            box-shadow: 0 4px #1a252f;
        }
        .btn:active { background: #27ae60; transform: translateY(2px); }
        .up { grid-area: up; } .left { grid-area: left; } .right { grid-area: right; } .down { grid-area: down; }
    </style>
</head>
<body>

    <div class="stats">
        <div class="score" id="scoreBoard">Score: 0</div>
        <div class="kills" id="killBoard">Kills: 0</div>
        <div id="allyStatus" class="ally-timer">ALLY ACTIVE!</div>
    </div>
    <canvas id="snakeGame" width="600" height="600"></canvas>

    <div class="controls">
        <button class="btn up" onclick="setDirection('UP')">▲</button>
        <button class="btn left" onclick="setDirection('LEFT')">◀</button>
        <button class="btn right" onclick="setDirection('RIGHT')">▶</button>
        <button class="btn down" onclick="setDirection('DOWN')">▼</button>
    </div>

    <script>
        const canvas = document.getElementById("snakeGame");
        const ctx = canvas.getContext("2d");
        const box = 20;
        const viewSize = 600;

        let score = 0, killCount = 0, foodCounter = 0;
        let snake = [{x: 300, y: 300}, {x: 280, y: 300}, {x: 260, y: 300}];
        let ally = null, allyTimeout = null;
        let foods = [], enemies = [], bullets = [];
        let d = "RIGHT", nextD = "RIGHT", isBossActive = false;
        let camX = 0, camY = 0;

        // Parallax Layers (Stars/Dust)
        const starLayers = [
            { count: 50, size: 1, speed: 0.2, color: "#444", stars: [] }, // Far
            { count: 30, size: 2, speed: 0.5, color: "#666", stars: [] }, // Mid
            { count: 15, size: 2, speed: 0.8, color: "#888", stars: [] }  // Near
        ];

        // Initialize stars
        starLayers.forEach(layer => {
            for(let i=0; i<layer.count; i++) {
                layer.stars.push({ x: Math.random() * viewSize, y: Math.random() * viewSize });
            }
        });

        const Tiers = [
            { color: "#2ecc71", hp: 1, speed: 4 },
            { color: "#e74c3c", hp: 2, speed: 5 },
            { color: "#8b0000", hp: 3, speed: 6 }
        ];

        function spawnWorldEntity(type, isMinion = false, bx = 0, by = 0) {
            if (isMinion) {
                enemies.push({ x: bx, y: by, hp: 1, speed: 10, color: "#d980fa", type: 'minion' });
                return;
            }
            const angle = Math.random() * Math.PI * 2;
            const dist = 400 + Math.random() * 300;
            const spawnX = Math.floor((snake[0].x + Math.cos(angle) * dist) / box) * box;
            const spawnY = Math.floor((snake[0].y + Math.sin(angle) * dist) / box) * box;

            if (type === 'food') {
                foods.push({ x: spawnX, y: spawnY, hp: Math.floor(Math.random() * 5) + 1 });
            } else if (type === 'enemy' && !isBossActive) {
                let t = Tiers[Math.floor(Math.random() * Tiers.length)];
                enemies.push({ x: spawnX, y: spawnY, ...t, type: 'mob' });
            }
        }

        for(let i=0; i<15; i++) spawnWorldEntity('food');

        function spawnAlly() {
            document.getElementById("allyStatus").style.display = "block";
            ally = { x: snake[0].x, y: snake[0].y };
            if(allyTimeout) clearTimeout(allyTimeout);
            allyTimeout = setTimeout(() => {
                ally = null;
                document.getElementById("allyStatus").style.display = "none";
            }, 20000);
        }

        function spawnBoss() {
            isBossActive = true;
            enemies = [{ 
                x: snake[0].x, y: snake[0].y - 400, hp: 50, maxHp: 50, speed: 7, 
                color: "#9b59b6", type: 'boss', size: 40, spawnTimer: 0 
            }];
        }

        function setDirection(dir) {
            if(dir == "LEFT" && d != "RIGHT") nextD = "LEFT";
            else if(dir == "UP" && d != "DOWN") nextD = "UP";
            else if(dir == "RIGHT" && d != "LEFT") nextD = "RIGHT";
            else if(dir == "DOWN" && d != "UP") nextD = "DOWN";
        }

        function getDamage() {
            if(Math.random() < 0.25) return Math.random() < 0.2 ? 4 : 2;
            return 1;
        }

        function fireFrom(origin, color = "#3498db") {
            let targets = [...enemies, ...foods];
            if (targets.length === 0) return;
            let closest = targets.reduce((p, c) => Math.hypot(c.x-origin.x, c.y-origin.y) < Math.hypot(p.x-origin.x, p.y-origin.y) ? c : p);
            let angle = Math.atan2(closest.y - origin.y, closest.x - origin.x);
            let dmg = getDamage();
            bullets.push({ 
                x: origin.x + 10, y: origin.y + 10, 
                vx: Math.cos(angle) * 14, vy: Math.sin(angle) * 14, 
                dmg: dmg, color: dmg === 4 ? "#ff0000" : (dmg === 2 ? "#f39c12" : color)
            });
        }

        function update() {
            d = nextD;
            let head = { ...snake[0] };
            if(d == "LEFT") head.x -= box; if(d == "UP") head.y -= box;
            if(d == "RIGHT") head.x += box; if(d == "DOWN") head.y += box;

            for(let p of snake) if(head.x == p.x && head.y == p.y) return resetGame();

            camX = head.x - viewSize / 2;
            camY = head.y - viewSize / 2;

            foods.forEach((f, i) => {
                if(head.x == f.x && head.y == f.y) {
                    score++; foodCounter++; foods.splice(i, 1); spawnWorldEntity('food');
                    snake.push({...snake[snake.length-1]});
                    if(foodCounter >= 15) { spawnAlly(); foodCounter = 0; }
                }
            });

            snake.unshift(head);
            snake.pop();

            if(ally) {
                let target = enemies.length > 0 ? enemies[0] : foods[0];
                if(target) {
                    let angle = Math.atan2(target.y - ally.y, target.x - ally.x);
                    ally.x += Math.cos(angle) * 5; ally.y += Math.sin(angle) * 5;
                }
            }

            enemies.forEach((en, i) => {
                let target = head;
                if(en.type === 'boss' && en.hp < en.maxHp && foods.length > 0) target = foods[0];
                let angle = Math.atan2(target.y - en.y, target.x - en.x);
                en.x += Math.cos(angle) * en.speed;
                en.y += Math.sin(angle) * en.speed;

                if(en.type === 'boss') {
                    en.spawnTimer += 100;
                    if(en.spawnTimer >= 5000) { spawnWorldEntity('enemy', true, en.x, en.y); en.spawnTimer = 0; }
                    foods.forEach((f, fi) => {
                        if(Math.hypot(en.x - f.x, en.y - f.y) < 30) {
                            en.hp = Math.min(en.maxHp, en.hp + 2);
                            foods.splice(fi, 1); spawnWorldEntity('food');
                        }
                    });
                }
                if(Math.hypot(en.x - head.x, en.y - head.y) < (en.size || box)) resetGame();
            });

            bullets.forEach((b, bi) => {
                b.x += b.vx; b.y += b.vy;
                enemies.forEach((en, ei) => {
                    if(Math.hypot(b.x - en.x, b.y - en.y) < (en.size || box)) {
                        en.hp -= b.dmg; bullets.splice(bi, 1);
                        if(en.hp <= 0) {
                            if(en.type === 'boss') isBossActive = false;
                            enemies.splice(ei, 1); killCount++;
                            if(killCount % 20 === 0) spawnBoss();
                        }
                    }
                });
                foods.forEach((f, fi) => {
                    if(Math.hypot(b.x - f.x, b.y - f.y) < box) {
                        f.hp -= b.dmg; bullets.splice(bi, 1);
                        if(f.hp <= 0) { 
                            score++; foodCounter++; foods.splice(fi, 1); spawnWorldEntity('food'); 
                            snake.push({...snake[snake.length-1]});
                            if(foodCounter >= 15) { spawnAlly(); foodCounter = 0; }
                        }
                    }
                });
                if(Math.hypot(b.x - head.x, b.y - head.y) > 800) bullets.splice(bi, 1);
            });

            if(Math.random() < 0.03 && enemies.length < 12) spawnWorldEntity('enemy');
            document.getElementById("scoreBoard").innerText = "Score: " + score;
            document.getElementById("killBoard").innerText = "Kills: " + killCount;
        }

        function resetGame() { alert("Killed! Score: " + score); location.reload(); }

        function draw() {
            ctx.fillStyle = "#050505"; ctx.fillRect(0, 0, viewSize, viewSize);

            // DRAW PARALLAX LAYERS
            starLayers.forEach(layer => {
                ctx.fillStyle = layer.color;
                layer.stars.forEach(s => {
                    // Parallax math: position minus (camera * speed factor)
                    let sx = (s.x - camX * layer.speed) % viewSize;
                    let sy = (s.y - camY * layer.speed) % viewSize;
                    if(sx < 0) sx += viewSize;
                    if(sy < 0) sy += viewSize;
                    ctx.fillRect(sx, sy, layer.size, layer.size);
                });
            });

            // Grid (Now subtler and moves with camera)
            ctx.strokeStyle = "rgba(40, 40, 40, 0.3)"; ctx.beginPath();
            for (let x = -camX % box; x < viewSize; x += box) { ctx.moveTo(x, 0); ctx.lineTo(x, viewSize); }
            for (let y = -camY % box; y < viewSize; y += box) { ctx.moveTo(0, y); ctx.lineTo(viewSize, y); }
            ctx.stroke();

            foods.forEach(f => {
                ctx.fillStyle = "#f1c40f"; ctx.fillRect(f.x - camX, f.y - camY, box, box);
                ctx.fillStyle = "black"; ctx.font = "12px Arial";
                ctx.fillText(f.hp, f.x - camX + 6, f.y - camY + 15);
            });

            enemies.forEach(en => {
                ctx.fillStyle = en.color;
                let s = en.size || box;
                ctx.fillRect(en.x - camX, en.y - camY, s, s);
                if(en.type === 'boss') {
                    ctx.fillStyle = "red"; ctx.fillRect(en.x - camX, en.y - camY - 12, s, 5);
                    ctx.fillStyle = "green"; ctx.fillRect(en.x - camX, en.y - camY - 12, s * (en.hp/en.maxHp), 5);
                }
            });

            bullets.forEach(b => { ctx.fillStyle = b.color; ctx.fillRect(b.x - camX, b.y - camY, 5, 5); });
            if(ally) { 
                ctx.fillStyle = "#3498db"; ctx.fillRect(ally.x - camX, ally.y - camY, box, box);
                ctx.strokeStyle = "white"; ctx.strokeRect(ally.x - camX, ally.y - camY, box, box);
            }

            snake.forEach((p, i) => {
                ctx.fillStyle = (i == 0) ? "#2ecc71" : "#27ae60";
                ctx.fillRect(p.x - camX, p.y - camY, box-1, box-1);
                if(i==0) { ctx.fillStyle = "white"; ctx.fillRect(p.x - camX + 5, p.y - camY + 5, 10, 10); }
            });

            requestAnimationFrame(draw);
        }

        document.addEventListener("keydown", (e) => {
            const keys = {37: "LEFT", 38: "UP", 39: "RIGHT", 40: "DOWN"};
            if(keys[e.keyCode]) setDirection(keys[e.keyCode]);
        });

        setInterval(update, 100);
        setInterval(() => {
            fireFrom(snake[0]);
            if(ally) fireFrom(ally, "#e67e22");
        }, 800);
        draw();
    </script>
</body>
</html>
