# my-shooting-game
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <title>看你有多猛</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #menu, #leaderboard, #gameOver { 
            position: absolute; 
            top: 50%; 
            left: 50%; 
            transform: translate(-50%, -50%);
            background-color: rgba(0,0,0,0.7);
            padding: 20px;
            border-radius: 10px;
            text-align: center;
            color: white;
        }
        #menu button, #leaderboard button, #gameOver button {
            display: block;
            width: 200px;
            margin: 10px auto;
            padding: 10px;
            font-size: 18px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        #leaderboard, #gameOver {
            display: none;
        }
        #leaderboard ul, #gameOver ul {
            list-style-type: none;
            padding: 0;
        }
        #leaderboard li, #gameOver li {
            margin: 5px 0;
        }
        #playerName {
            display: block;
            width: 180px;
            margin: 10px auto;
            padding: 10px;
            font-size: 16px;
            text-align: center;
        }
        .leaderboard-container {
            display: flex;
            justify-content: space-around;
        }
        .leaderboard-column {
            width: 45%;
        }
        .game-running {
            cursor: none;
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>
    <div id="menu">
        <button id="startButton">開始新遊戲</button>
        <button id="leaderboardButton">排行榜</button>
    </div>
    <div id="leaderboard">
        <h2>排行榜</h2>
        <div class="leaderboard-container">
            <div class="leaderboard-column">
                <h3>得分排行</h3>
                <ul id="scoreLeaderboard"></ul>
            </div>
            <div class="leaderboard-column">
                <h3>生存時間排行</h3>
                <ul id="timeLeaderboard"></ul>
            </div>
        </div>
        <button id="backButton">返回主菜單</button>
    </div>
    <div id="gameOver">
        <h2>遊戲結束</h2>
        <ul id="gameOverStats"></ul>
        <input type="text" id="playerName" placeholder="輸入您的名字" maxlength="10">
        <button id="submitScoreButton">提交分數</button>
        <button id="restartButton">重新開始</button>
        <button id="menuButton">返回主菜單</button>
    </div>
    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        let player = {
            x: canvas.width / 2,
            y: canvas.height / 2,
            width: 12.5,
            height: 12.5,
            health: 1
        };

        let bullets = [];
        let enemies = [];
        let score = 0;
        let mouseX = 0, mouseY = 0;
        let gameRunning = false;
        let scoreLeaderboard = [];
        let timeLeaderboard = [];
        let gameStartTime;
        let enemiesDefeated = 0;
        let survivalTime = 0;
        let enemySpawnRate = 0.02;
        let enemySpeed = 2;
        let playerSpeed = 5;
        let enemyCount = 10;
        let lastDifficultyIncrease = 0;

        let keys = {
            w: false,
            a: false,
            s: false,
            d: false
        };

        function drawPlayer() {
            ctx.fillStyle = 'blue';
            ctx.fillRect(player.x - player.width / 2, player.y - player.height / 2, player.width, player.height);
        }

        function drawBullets() {
            ctx.fillStyle = 'red';
            bullets.forEach(bullet => {
                ctx.beginPath();
                ctx.arc(bullet.x, bullet.y, 3, 0, Math.PI * 2);
                ctx.fill();
            });
        }

        function drawEnemies() {
            ctx.fillStyle = 'green';
            enemies.forEach(enemy => {
                ctx.fillRect(enemy.x - 5, enemy.y - 5, 10, 10);
            });
        }

        function drawCrosshair() {
            ctx.strokeStyle = 'black';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.moveTo(mouseX - 10, mouseY);
            ctx.lineTo(mouseX + 10, mouseY);
            ctx.moveTo(mouseX, mouseY - 10);
            ctx.lineTo(mouseX, mouseY + 10);
            ctx.stroke();

            ctx.beginPath();
            ctx.arc(mouseX, mouseY, 3, 0, Math.PI * 2);
            ctx.stroke();
        }

        function moveBullets() {
            bullets.forEach(bullet => {
                let dx = bullet.targetX - bullet.startX;
                let dy = bullet.targetY - bullet.startY;
                let distance = Math.sqrt(dx * dx + dy * dy);
                bullet.x += (dx / distance) * 10;
                bullet.y += (dy / distance) * 10;
            });
            bullets = bullets.filter(bullet => 
                bullet.x > 0 && bullet.x < canvas.width && 
                bullet.y > 0 && bullet.y < canvas.height
            );
        }

        function moveEnemies() {
            enemies.forEach(enemy => {
                let dx = player.x - enemy.x;
                let dy = player.y - enemy.y;
                let distance = Math.sqrt(dx * dx + dy * dy);
                enemy.x += (dx / distance) * enemySpeed;
                enemy.y += (dy / distance) * enemySpeed;

                if (distance < (player.width + 10) / 2) {
                    endGame();
                }
            });
            if (Math.random() < enemySpawnRate && enemies.length < enemyCount) {
                let spawnSide = Math.floor(Math.random() * 4);
                let newEnemy = {};
                switch(spawnSide) {
                    case 0: // 上
                        newEnemy = {x: Math.random() * canvas.width, y: 0};
                        break;
                    case 1: // 右
                        newEnemy = {x: canvas.width, y: Math.random() * canvas.height};
                        break;
                    case 2: // 下
                        newEnemy = {x: Math.random() * canvas.width, y: canvas.height};
                        break;
                    case 3: // 左
                        newEnemy = {x: 0, y: Math.random() * canvas.height};
                        break;
                }
                enemies.push(newEnemy);
            }
        }

        function checkCollisions() {
            bullets.forEach((bullet, bulletIndex) => {
                enemies.forEach((enemy, enemyIndex) => {
                    let dx = bullet.x - enemy.x;
                    let dy = bullet.y - enemy.y;
                    let distance = Math.sqrt(dx * dx + dy * dy);
                    if (distance < 8) {  // 5 (enemy size / 2) + 3 (bullet radius)
                        enemies.splice(enemyIndex, 1);
                        bullets.splice(bulletIndex, 1);
                        score += 10;
                        enemiesDefeated++;
                    }
                });
            });
        }

        function movePlayer() {
            if (keys.w && player.y > player.height / 2) player.y -= playerSpeed;
            if (keys.s && player.y < canvas.height - player.height / 2) player.y += playerSpeed;
            if (keys.a && player.x > player.width / 2) player.x -= playerSpeed;
            if (keys.d && player.x < canvas.width - player.width / 2) player.x += playerSpeed;
        }

        function updateDifficulty() {
            let currentTime = Math.floor((new Date() - gameStartTime) / 1000);
            if (currentTime - lastDifficultyIncrease >= 20) {
                // 增加敵人數量和速度
                enemyCount *= 1.2;
                enemySpeed *= 1.2;
                
                // 降低玩家速度
                playerSpeed *= 0.8;
                
                lastDifficultyIncrease = currentTime;
            }
        }

        function gameLoop() {
            if (!gameRunning) return;
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            movePlayer();
            drawPlayer();
            drawBullets();
            drawEnemies();
            drawCrosshair();
            moveBullets();
            moveEnemies();
            checkCollisions();
            updateDifficulty();
            survivalTime = Math.floor((new Date() - gameStartTime) / 1000);
            requestAnimationFrame(gameLoop);
        }

        function startGame() {
            gameRunning = true;
            player.health = 1;
            score = 0;
            enemiesDefeated = 0;
            bullets = [];
            enemies = [];
            gameStartTime = new Date();
            survivalTime = 0;
            enemySpawnRate = 0.02;
            enemySpeed = 2;
            playerSpeed = 5;
            enemyCount = 10;
            lastDifficultyIncrease = 0;
            document.getElementById('menu').style.display = 'none';
            document.body.classList.add('game-running');
            gameLoop();
        }

        function endGame() {
            gameRunning = false;
            document.body.classList.remove('game-running');
            
            const gameOverStats = document.getElementById('gameOverStats');
            gameOverStats.innerHTML = `
                <li>遊玩時間: ${survivalTime} 秒</li>
                <li>擊倒敵人數: ${enemiesDefeated}</li>
                <li>得分: ${score}</li>
            `;
            
            document.getElementById('gameOver').style.display = 'block';
        }

        function submitScore() {
            let playerName = document.getElementById('playerName').value.trim();
            if (playerName === "") {
                playerName = "匿名玩家";
            }
            scoreLeaderboard.push({name: playerName, score: score});
            timeLeaderboard.push({name: playerName, time: survivalTime});
            
            scoreLeaderboard.sort((a, b) => b.score - a.score);
            timeLeaderboard.sort((a, b) => b.time - a.time);
            
            scoreLeaderboard = scoreLeaderboard.slice(0, 5);
            timeLeaderboard = timeLeaderboard.slice(0, 5);
            
            updateLeaderboards();
            document.getElementById('gameOver').style.display = 'none';
            document.getElementById('leaderboard').style.display = 'block';
        }

        function updateLeaderboards() {
            const scoreLeaderboardElement = document.getElementById('scoreLeaderboard');
            const timeLeaderboardElement = document.getElementById('timeLeaderboard');
            
            scoreLeaderboardElement.innerHTML = '';
            timeLeaderboardElement.innerHTML = '';
            
            scoreLeaderboard.forEach((entry, index) => {
                const li = document.createElement('li');
                li.textContent = `第 ${index + 1} 名: ${entry.name} - ${entry.score} 分`;
                scoreLeaderboardElement.appendChild(li);
            });
            
            timeLeaderboard.forEach((entry, index) => {
                const li = document.createElement('li');
                li.textContent = `第 ${index + 1} 名: ${entry.name} - ${entry.time} 秒`;
                timeLeaderboardElement.appendChild(li);
            });
        }

        canvas.addEventListener('mousemove', (event) => {
            mouseX = event.clientX;
            mouseY = event.clientY;
        });

        canvas.addEventListener('mousedown', (event) => {
            if (!gameRunning) return;
            if (event.button === 0) {  // 只有左鍵 (button 0)
                bullets.push({
                    x: player.x, 
                    y: player.y, 
                    startX: player.x, 
                    startY: player.y,
                    targetX: event.clientX, 
                    targetY: event.clientY
                });
            }
        });

        document.addEventListener('keydown', (event) => {
            if (event.key.toLowerCase() in keys) {
                keys[event.key.toLowerCase()] = true;
            }
        });

        document.addEventListener('keyup', (event) => {
            if (event.key.toLowerCase() in keys) {
                keys[event.key.toLowerCase()] = false;
            }
        });

        canvas.addEventListener('contextmenu', (event) => {
            event.preventDefault();  // 防止右鍵菜單出現
        });

        document.getElementById('startButton').addEventListener('click', startGame);
        document.getElementById('leaderboardButton').addEventListener('click', () => {
            document.getElementById('menu').style.display = 'none';
            document.getElementById('leaderboard').style.display = 'block';
        });
        document.getElementById('backButton').addEventListener('click', () => {
            document.getElementById('leaderboard').style.display = 'none';
            document.getElementById('menu').style.display = 'block';
        });
        document.getElementById('submitScoreButton').addEventListener('click', submitScore);
        document.getElementById('restartButton').addEventListener('click', () => {
            document.getElementById('gameOver').style.display = 'none';
            startGame();
        });
        document.getElementById('menuButton').addEventListener('click', () => {
            document.getElementById('gameOver').style.display = 'none';
            document.getElementById('menu').style.display = 'block';
        });

        updateLeaderboards();
    </script>
</body>
</html>
