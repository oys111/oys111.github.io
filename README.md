[Upl<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tetris Game</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #FDF6E3;
            font-family: Arial, sans-serif;
            overflow: hidden;
        }
        #gameContainer {
            position: relative;
        }
        #gameCanvas {
            border: 4px solid #586E75;
            border-radius: 10px;
            box-shadow: 0 0 20px rgba(88, 110, 117, 0.3);
        }
        #startScreen, #gameOverScreen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(253, 246, 227, 0.9);
            color: #586E75;
            font-size: 24px;
            text-align: center;
        }
        #score {
            position: absolute;
            top: 10px;
            left: 10px;
            color: #586E75;
            font-size: 20px;
        }
        .key {
            display: inline-block;
            background-color: #93A1A1;
            color: #FDF6E3;
            padding: 5px 10px;
            border-radius: 5px;
            margin: 0 5px;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <canvas id="gameCanvas" width="300" height="600"></canvas>
        <div id="score">Score: 0</div>
        <div id="startScreen">
            Press <span class="key">Space</span> to Start Game<br><br>
            Controls:<br>
            <span class="key">←</span> <span class="key">→</span> Move<br>
            <span class="key">↓</span> Drop<br>
            <span class="key">W</span> Rotate
        </div>
        <div id="gameOverScreen" style="display:none;">
            Game Over<br>
            Press <span class="key">Space</span> to Restart
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const blockSize = 30;
        const boardWidth = 10;
        const boardHeight = 20;
        let board = Array(boardHeight).fill().map(() => Array(boardWidth).fill(0));
        let currentPiece;
        let gameLoop;
        let gameOver = false;
        let score = 0;
        const colors = ['#DC322F', '#CB4B16', '#B58900', '#859900', '#268BD2', '#6C71C4', '#D33682'];

        const pieces = [
            [[1, 1, 1, 1]],
            [[1, 1], [1, 1]],
            [[1, 1, 1], [0, 1, 0]],
            [[1, 1, 1], [1, 0, 0]],
            [[1, 1, 1], [0, 0, 1]],
            [[1, 1, 0], [0, 1, 1]],
            [[0, 1, 1], [1, 1, 0]]
        ];

        function createPiece() {
            const pieceType = Math.floor(Math.random() * pieces.length);
            const color = colors[Math.floor(Math.random() * colors.length)];
            return {
                shape: pieces[pieceType],
                color: color,
                x: Math.floor(boardWidth / 2) - Math.floor(pieces[pieceType][0].length / 2),
                y: 0,
                ghostY: 0
            };
        }

        function drawBoard() {
            ctx.fillStyle = '#FDF6E3';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            for (let y = 0; y < boardHeight; y++) {
                for (let x = 0; x < boardWidth; x++) {
                    if (board[y][x]) {
                        ctx.fillStyle = board[y][x];
                        ctx.fillRect(x * blockSize, y * blockSize, blockSize - 1, blockSize - 1);
                        ctx.strokeStyle = '#EEE8D5';
                        ctx.strokeRect(x * blockSize, y * blockSize, blockSize - 1, blockSize - 1);
                    }
                }
            }
        }

        function drawPiece(piece, isGhost = false) {
            ctx.fillStyle = isGhost ? piece.color + '40' : piece.color;
            for (let y = 0; y < piece.shape.length; y++) {
                for (let x = 0; x < piece.shape[y].length; x++) {
                    if (piece.shape[y][x]) {
                        const drawX = (piece.x + x) * blockSize;
                        const drawY = (isGhost ? piece.ghostY + y : piece.y + y) * blockSize;
                        ctx.fillRect(drawX, drawY, blockSize - 1, blockSize - 1);
                        if (!isGhost) {
                            ctx.strokeStyle = '#EEE8D5';
                            ctx.strokeRect(drawX, drawY, blockSize - 1, blockSize - 1);
                        }
                    }
                }
            }
        }

        function updateGhostPiece() {
            currentPiece.ghostY = currentPiece.y;
            while (!isCollision(currentPiece.shape, currentPiece.x, currentPiece.ghostY + 1)) {
                currentPiece.ghostY++;
            }
        }

        function moveDown() {
            currentPiece.y++;
            if (isCollision(currentPiece.shape, currentPiece.x, currentPiece.y)) {
                currentPiece.y--;
                mergePiece();
                clearLines();
                currentPiece = createPiece();
                updateGhostPiece();
                if (isCollision(currentPiece.shape, currentPiece.x, currentPiece.y)) {
                    gameOver = true;
                    cancelAnimationFrame(gameLoop);
                    document.getElementById('gameOverScreen').style.display = 'flex';
                }
            }
            updateGhostPiece();
        }

        function moveLeft() {
            currentPiece.x--;
            if (isCollision(currentPiece.shape, currentPiece.x, currentPiece.y)) {
                currentPiece.x++;
            }
            updateGhostPiece();
        }

        function moveRight() {
            currentPiece.x++;
            if (isCollision(currentPiece.shape, currentPiece.x, currentPiece.y)) {
                currentPiece.x--;
            }
            updateGhostPiece();
        }

        function rotate() {
            const rotated = currentPiece.shape[0].map((_, i) => 
                currentPiece.shape.map(row => row[i]).reverse()
            );
            if (!isCollision(rotated, currentPiece.x, currentPiece.y)) {
                currentPiece.shape = rotated;
                updateGhostPiece();
            }
        }

        function isCollision(shape, x, y) {
            for (let row = 0; row < shape.length; row++) {
                for (let col = 0; col < shape[row].length; col++) {
                    if (shape[row][col] &&
                        (x + col < 0 ||
                         x + col >= boardWidth ||
                         y + row >= boardHeight ||
                         (y + row >= 0 && board[y + row][x + col]))) {
                        return true;
                    }
                }
            }
            return false;
        }

        function mergePiece() {
            for (let y = 0; y < currentPiece.shape.length; y++) {
                for (let x = 0; x < currentPiece.shape[y].length; x++) {
                    if (currentPiece.shape[y][x]) {
                        board[currentPiece.y + y][currentPiece.x + x] = currentPiece.color;
                    }
                }
            }
        }

        function clearLines() {
            let linesCleared = 0;
            for (let y = boardHeight - 1; y >= 0; y--) {
                if (board[y].every(cell => cell !== 0)) {
                    board.splice(y, 1);
                    board.unshift(Array(boardWidth).fill(0));
                    linesCleared++;
                }
            }
            if (linesCleared > 0) {
                score += linesCleared * 100;
                document.getElementById('score').innerText = `Score: ${score}`;
            }
        }

        let lastTime = 0;
        const dropInterval = 1000;
        let dropCounter = 0;

        function update(time = 0) {
            const deltaTime = time - lastTime;
            lastTime = time;
            
            dropCounter += deltaTime;
            if (dropCounter > dropInterval) {
                moveDown();
                dropCounter = 0;
            }

            draw();
            gameLoop = requestAnimationFrame(update);
        }

        function draw() {
            drawBoard();
            drawPiece(currentPiece, true);  // Draw ghost piece
            drawPiece(currentPiece);
        }

        function gameStart() {
            board = Array(boardHeight).fill().map(() => Array(boardWidth).fill(0));
            currentPiece = createPiece();
            updateGhostPiece();
            gameOver = false;
            score = 0;
            document.getElementById('score').innerText = `Score: ${score}`;
            document.getElementById('startScreen').style.display = 'none';
            document.getElementById('gameOverScreen').style.display = 'none';

            if (gameLoop) cancelAnimationFrame(gameLoop);
            update();
        }

        document.addEventListener('keydown', (e) => {
            if (!gameOver) {
                switch(e.key) {
                    case 'ArrowLeft': moveLeft(); break;
                    case 'ArrowRight': moveRight(); break;
                    case 'ArrowDown': moveDown(); break;
                    case 'w': rotate(); break;
                }
            }
            if (e.key === ' ') {
                e.preventDefault();
                gameStart();
            }
        });

        drawBoard();
    </script>
</body>
</html>
oading Tetris.html…]()
