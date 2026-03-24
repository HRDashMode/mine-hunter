<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Abir's Mine Hunt</title>
    <style>
        body { background-color: #0d0d0d; color: white; font-family: 'Arial Black', sans-serif; display: flex; flex-direction: column; align-items: center; margin: 0; overflow: hidden; touch-action: manipulation; }
        #header { text-align: center; padding: 10px; }
        #ui { display: flex; gap: 15px; margin-bottom: 10px; }
        .stat-box { padding: 8px 15px; border-radius: 10px; font-size: 14px; text-align: center; min-width: 80px; }
        .safe-box { border: 2px solid #ff00ff; color: #ff00ff; box-shadow: 0 0 10px #ff00ff; }
        .bag-box { border: 2px solid #00f3ff; color: #00f3ff; box-shadow: 0 0 10px #00f3ff; }
        #game-area { position: relative; touch-action: none; }
        canvas { background-color: #2b1d12; border: 4px solid #444; border-radius: 5px; display: block; max-width: 95vw; max-height: 50vh; }
        #controls { display: grid; grid-template-columns: repeat(3, 1fr); gap: 15px; margin-top: 20px; width: 250px; }
        .btn { width: 70px; height: 70px; background: #1a1a1a; border: 2px solid #39ff14; color: #39ff14; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 30px; box-shadow: 0 0 15px rgba(57, 255, 20, 0.3); user-select: none; -webkit-tap-highlight-color: transparent; }
        .btn:active { background: #39ff14; color: black; }
        #up { grid-column: 2; }
        #left { grid-column: 1; }
        #down { grid-column: 2; }
        #right { grid-column: 3; }
        .overlay { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: rgba(0,0,0,0.9); padding: 20px; border-radius: 20px; display: none; text-align: center; z-index: 100; width: 85%; box-sizing: border-box; }
        #win-msg { border: 6px solid #39ff14; color: #39ff14; box-shadow: 0 0 30px #39ff14; }
        #lose-msg { border: 6px solid #ff0000; color: #ff0000; box-shadow: 0 0 30px #ff0000; }
        button.retry-btn { background: white; border: none; padding: 12px 25px; font-weight: bold; border-radius: 25px; margin-top: 15px; font-size: 1rem; }
    </style>
</head>
<body>
    <div id="header">
        <h2 style="color: #39ff14; margin: 5px; text-shadow: 0 0 10px #39ff14;">ABIR'S MINE HUNT</h2>
        <div id="ui">
            <div class="stat-box safe-box">💎 SAFE<br><span id="safe-count">0</span></div>
            <div class="stat-box bag-box">🎒 BAG<br><span id="bag-count">0 / 15</span></div>
        </div>
    </div>
    <div id="game-area">
        <canvas id="gameCanvas" width="400" height="400"></canvas>
        <div id="win-msg" class="overlay">
            <h1 style="font-size: 2rem;">Congrats Abir,<br>you WON !!</h1>
            <button class="retry-btn" onclick="location.reload()">PLAY AGAIN</button>
        </div>
        <div id="lose-msg" class="overlay">
            <h1 style="font-size: 3rem;">BOOM!</h1>
            <p>You ate a Red Mine!</p>
            <button class="retry-btn" onclick="location.reload()">TRY AGAIN</button>
        </div>
    </div>
    <div id="controls">
        <div class="btn" id="up">▲</div>
        <div class="btn" id="left">◀</div>
        <div class="btn" id="down">▼</div>
        <div class="btn" id="right">▶</div>
    </div>
<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    let audioCtx;
    const tileSize = 40, rows = 10, cols = 10;
    let world = [], gameOver = false, particles = [];
    let player = { x: 0, y: 0, rotation: 0, mouth: 0.25 };
    let inventory = { safe: 0, bag: 0 };
    const EMPTY = 0, DIRT = 1, GOLD = 2, DIAMOND = 3, STONE = 4, WOOD = 5, MINE = 6;
    const colors = { [DIRT]: '#4e342e', [GOLD]: '#ffee00', [DIAMOND]: '#00ffff', [STONE]: '#9e9e9e', [WOOD]: '#ff9100', [MINE]: '#ff0000', player: '#39ff14' };
    function initAudio() { if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)(); }
    function playBurp() { if (!audioCtx) return; const osc = audioCtx.createOscillator(); const g = audioCtx.createGain(); osc.type = 'sawtooth'; osc.frequency.setValueAtTime(100, audioCtx.currentTime); osc.frequency.exponentialRampToValueAtTime(40, audioCtx.currentTime + 0.3); g.gain.setValueAtTime(0.1, audioCtx.currentTime); g.gain.linearRampToValueAtTime(0, audioCtx.currentTime + 0.3); osc.connect(g); g.connect(audioCtx.destination); osc.start(); osc.stop(audioCtx.currentTime + 0.3); }
    function playBlast() { if (!audioCtx) return; const osc = audioCtx.createOscillator(); const g = audioCtx.createGain(); osc.type = 'triangle'; osc.frequency.setValueAtTime(120, audioCtx.currentTime); osc.frequency.exponentialRampToValueAtTime(10, audioCtx.currentTime + 0.8); g.gain.setValueAtTime(0.4, audioCtx.currentTime); osc.connect(g); g.connect(audioCtx.destination); osc.start(); osc.stop(audioCtx.currentTime + 0.8); }
    function init() {
        for (let r = 0; r < rows; r++) {
            world[r] = [];
            for (let c = 0; c < cols; c++) {
                if (r === 0 && c === 0) world[r][c] = EMPTY;
                else {
                    let rand = Math.random();
                    if (rand < 0.08) world[r][c] = MINE;
                    else if (rand < 0.12) world[r][c] = GOLD;
                    else if (rand < 0.35) world[r][c] = STONE;
                    else world[r][c] = DIRT;
                }
            }
        }
        requestAnimationFrame(gameLoop);
    }
    function move(dx, dy, rot) {
        if (gameOver) return;
        initAudio();
        let nX = player.x + dx, nY = player.y + dy;
        if (nX >= 0 && nX < cols && nY >= 0 && nY < rows) {
            let tile = world[nY][nX];
            player.rotation = rot; player.mouth = 0;
            if (tile === MINE) {
                playBlast(); gameOver = true;
                for(let i=0; i<15; i++) particles.push({ x: nX*tileSize+20, y: nY*tileSize+20, vx: (Math.random()-0.5)*12, vy: (Math.random()-0.5)*12, life: 1 });
                setTimeout(() => document.getElementById('lose-msg').style.display='block', 800);
            } else {
                if (tile !== EMPTY) {
                    playBurp();
                    if (tile === GOLD || tile === DIAMOND) inventory.safe++;
                    else if (tile !== DIRT) inventory.bag++;
                    world[nY][nX] = EMPTY;
                }
                player.x = nX; player.y = nY;
                setTimeout(() => player.mouth = 0.25, 100);
            }
            document.getElementById('safe-count').innerText = inventory.safe;
            document.getElementById('bag-count').innerText = inventory.bag + " / 15";
            if (inventory.bag >= 15) { gameOver = true; document.getElementById('win-msg').style.display='block'; }
        }
    }
    function gameLoop() {
        ctx.clearRect(0,0,400,400);
        for(let r=0; r<rows; r++) {
            for(let c=0; c<cols; c++) {
                if(world[r][c] !== EMPTY) {
                    ctx.fillStyle = colors[world[r][c]];
                    ctx.shadowBlur = (world[r][c] === MINE || world[r][c] === GOLD) ? 10 : 0;
                    ctx.shadowColor = colors[world[r][c]];
                    ctx.fillRect(c*tileSize+2, r*tileSize+2, tileSize-4, tileSize-4);
                }
            }
        }
        if(!gameOver) {
            ctx.fillStyle = colors.player; ctx.shadowBlur = 15; ctx.shadowColor = colors.player;
            ctx.beginPath();
            let px = player.x*tileSize+20, py = player.y*tileSize+20;
            ctx.moveTo(px, py);
            ctx.arc(px, py, 16, (player.rotation+player.mouth)*Math.PI, (player.rotation+(2-player.mouth))*Math.PI);
            ctx.lineTo(px, py);
            ctx.fill();
        }
        particles.forEach((p, i) => {
            ctx.fillStyle = `rgba(57, 255, 20, ${p.life})`;
            ctx.fillRect(p.x, p.y, 4, 4);
            p.x += p.vx; p.y += p.vy; p.life -= 0.03;
        });
        requestAnimationFrame(gameLoop);
    }
    document.getElementById('up').onclick = (e) => { e.preventDefault(); move(0, -1, 1.5); };
    document.getElementById('down').onclick = (e) => { e.preventDefault(); move(0, 1, 0.5); };
    document.getElementById('left').onclick = (e) => { e.preventDefault(); move(-1, 0, 1); };
    document.getElementById('right').onclick = (e) => { e.preventDefault(); move(1, 0, 0); };
    window.onkeydown = (e) => {
        if(e.key === 'ArrowUp') move(0,-1,1.5);
        if(e.key === 'ArrowDown') move(0,1,0.5);
        if(e.key === 'ArrowLeft') move(-1,0,1);
        if(e.key === 'ArrowRight') move(1,0,0);
    };
    init();
</script>
</body>
</html>
