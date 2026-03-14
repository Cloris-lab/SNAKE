<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>像素之森 · 星露谷农场版</title>
    <style>
        :root {
            --bg-grass: #72B155;     /* 农场外围草地绿 */
            --wood-base: #D28C47;    /* 木板主色 */
            --wood-light: #F5B041;   /* 木板亮边 */
            --wood-dark: #873600;    /* 木板暗边 */
            --wood-border: #3E2723;  /* 木板外框线 */
            --text-color: #FFF;
        }

        * { box-sizing: border-box; margin: 0; padding: 0; font-family: 'Courier New', Courier, monospace; user-select: none; }

        body {
            background-color: var(--bg-grass);
            /* 简单的草地像素纹理 */
            background-image: radial-gradient(#609945 10%, transparent 10%);
            background-size: 20px 20px;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            min-height: 100vh; overflow: hidden;
        }

        /* 核心：星露谷式斜切木板立体感（亮边 + 暗底边） */
        .wood-panel {
            background-color: var(--wood-base);
            border: 4px solid var(--wood-border);
            /* 极具像素游戏风格的内发光/内阴影立体感 */
            box-shadow: 
                inset 4px 4px 0 var(--wood-light), 
                inset -4px -4px 0 var(--wood-dark),
                5px 5px 15px rgba(0,0,0,0.3);
            color: var(--text-color);
            text-shadow: 2px 2px 0 var(--wood-border);
            border-radius: 2px;
        }

        #ui-board {
            width: 600px;
            display: flex; justify-content: space-between;
            padding: 12px 20px; margin-bottom: 15px;
            font-size: 22px; font-weight: bold;
        }

        #game-container { position: relative; line-height: 0; }

        /* 画布：禁用抗锯齿，保持像素锐利 */
        canvas {
            display: block;
            image-rendering: pixelated;
        }

        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            background: rgba(0, 0, 0, 0.4); backdrop-filter: blur(2px);
            z-index: 10; text-align: center;
        }

        .hidden { display: none !important; }

        .btn {
            background-color: #E67E22;
            border: 4px solid #78281F;
            box-shadow: inset 3px 3px 0 #F39C12, inset -3px -3px 0 #935116;
            color: white; padding: 12px 30px; font-size: 24px; font-weight: bold;
            cursor: pointer; margin-top: 25px; text-shadow: 2px 2px 0 #78281F;
        }
        .btn:active {
            box-shadow: inset 4px 4px 0 #935116, inset -2px -2px 0 #F39C12;
            transform: translate(2px, 2px);
        }

        .tip { font-size: 14px; margin-top: 10px; color: #FFF3E0; text-shadow: 1px 1px 0 #3E2723; }
        
        #controls { margin-top: 15px; display: none; }
        @media (max-width: 768px) {
            #ui-board { width: 100%; max-width: 400px; font-size: 18px; }
            canvas { width: 100%; max-width: 400px; height: auto; }
            #controls { display: block; text-align: center; color: white; text-shadow: 1px 1px black; }
        }
    </style>
</head>
<body>

    <div id="ui-board" class="wood-panel">
        <span>🍎 收成: <span id="current-score">0</span></span>
        <span>🏆 最高: <span id="high-score">0</span></span>
    </div>

    <div id="game-container" class="wood-panel" style="padding: 6px;">
        
        <div id="screen-start" class="overlay">
            <h1 style="font-size: 42px; margin-bottom: 10px;">像素农场</h1>
            <p class="tip">电脑: WASD/方向键 | 手机: 滑动屏幕</p>
            <button class="btn" onclick="game.start()">开始劳作</button>
        </div>

        <div id="screen-gameover" class="overlay hidden">
            <h2 style="font-size: 36px;">今天辛苦了！</h2>
            <p id="final-score" style="margin-top: 15px; font-size: 20px;"></p>
            <button class="btn" onclick="game.start()">明天继续</button>
        </div>

        <canvas id="gameCanvas" width="600" height="600"></canvas>
    </div>

    <div id="controls"><p>滑动屏幕控制小蛇移动</p></div>

<script>
const game = {
    ctx: null,
    gridCount: 20,
    tileSize: 30, // 放大至 30px 每格
    
    // 水果池 (6种)
    fruitTypes: ['apple', 'strawberry', 'banana', 'orange', 'grape', 'pineapple'],
    
    state: {
        snake: [],
        food: { x: 0, y: 0, type: 'apple' },
        dir: 'RIGHT',
        nextDir: 'RIGHT',
        score: 0,
        running: false,
        speed: 160
    },

    init() {
        const cvs = document.getElementById('gameCanvas');
        this.ctx = cvs.getContext('2d');
        
        // 加载最高分
        const hs = localStorage.getItem('stardewSnakeHS') || 0;
        document.getElementById('high-score').innerText = hs;
        
        // 键盘控制
        window.addEventListener('keydown', e => {
            const keys = { 37:'LEFT', 38:'UP', 39:'RIGHT', 40:'DOWN', 87:'UP', 83:'DOWN', 65:'LEFT', 68:'RIGHT' };
            if (keys[e.keyCode]) {
                e.preventDefault();
                this.setDirection(keys[e.keyCode]);
            }
        });

        // 触摸控制
        let startX, startY;
        document.addEventListener('touchstart', e => {
            startX = e.touches[0].clientX; startY = e.touches[0].clientY;
        }, {passive: false});
        document.addEventListener('touchmove', e => e.preventDefault(), {passive: false}); // 防止滚动
        document.addEventListener('touchend', e => {
            let dx = e.changedTouches[0].clientX - startX;
            let dy = e.changedTouches[0].clientY - startY;
            if (Math.abs(dx) > Math.abs(dy)) {
                if (Math.abs(dx) > 30) this.setDirection(dx > 0 ? 'RIGHT' : 'LEFT');
            } else {
                if (Math.abs(dy) > 30) this.setDirection(dy > 0 ? 'DOWN' : 'UP');
            }
        });

        this.drawGrid(); // 绘制初始背景
    },

    setDirection(newDir) {
        const opp = { 'UP':'DOWN', 'DOWN':'UP', 'LEFT':'RIGHT', 'RIGHT':'LEFT' };
        if (newDir !== opp[this.state.dir]) this.state.nextDir = newDir;
    },

    start() {
        this.state.snake = [{x:5, y:10}, {x:4, y:10}, {x:3, y:10}];
        this.state.dir = this.state.nextDir = 'RIGHT';
        this.state.score = 0;
        this.state.speed = 160;
        this.state.running = true;
        this.spawnFood();
        
        document.getElementById('screen-start').classList.add('hidden');
        document.getElementById('screen-gameover').classList.add('hidden');
        document.getElementById('current-score').innerText = '0';
        
        this.loop();
    },

    spawnFood() {
        this.state.food = {
            x: Math.floor(Math.random() * this.gridCount),
            y: Math.floor(Math.random() * this.gridCount),
            type: this.fruitTypes[Math.floor(Math.random() * this.fruitTypes.length)]
        };
        // 避免生成在蛇身上
        if (this.state.snake.some(p => p.x === this.state.food.x && p.y === this.state.food.y)) {
            this.spawnFood();
        }
    },

    loop() {
        if (!this.state.running) return;
        this.update();
        this.draw();
        setTimeout(() => requestAnimationFrame(() => this.loop()), this.state.speed);
    },

    update() {
        this.state.dir = this.state.nextDir;
        const head = { ...this.state.snake[0] };

        if (this.state.dir === 'UP') head.y--;
        if (this.state.dir === 'DOWN') head.y++;
        if (this.state.dir === 'LEFT') head.x--;
        if (this.state.dir === 'RIGHT') head.x++;

        // 撞墙或咬到自己
        if (head.x < 0 || head.x >= this.gridCount || head.y < 0 || head.y >= this.gridCount ||
            this.state.snake.some(p => p.x === head.x && p.y === head.y)) {
            this.gameOver();
            return;
        }

        this.state.snake.unshift(head);
        
        // 吃食物
        if (head.x === this.state.food.x && head.y === this.state.food.y) {
            this.state.score += 10;
            // 提速逻辑
            this.state.speed = Math.max(70, 160 - Math.floor(this.state.score / 50) * 10);
            this.spawnFood();
            document.getElementById('current-score').innerText = this.state.score;
        } else {
            this.state.snake.pop();
        }
    },

    draw() {
        this.drawGrid();
        this.drawLargeFruit(this.state.food);
        this.drawSnake();
    },

    // --- 暖色农场格子 ---
    drawGrid() {
        const ctx = this.ctx;
        const ts = this.tileSize;
        for (let i = 0; i < this.gridCount; i++) {
            for (let j = 0; j < this.gridCount; j++) {
                // 沙棕色交替方格
                ctx.fillStyle = (i + j) % 2 === 0 ? '#E8D5A6' : '#DDC794';
                ctx.fillRect(i * ts, j * ts, ts, ts);
                
                // 细木纹分隔线 (只画右边和下边，形成网格)
                ctx.fillStyle = 'rgba(166, 124, 82, 0.3)'; 
                ctx.fillRect(i * ts + ts - 1, j * ts, 1, ts);
                ctx.fillRect(i * ts, j * ts + ts - 1, ts, 1);
            }
        }
    },

    // --- 治愈青碧蛇 ---
    drawSnake() {
        const ctx = this.ctx;
        const ts = this.tileSize;

        this.state.snake.forEach((p, index) => {
            const px = p.x * ts;
            const py = p.y * ts;
            const isHead = index === 0;

            // 基础底色 - 青碧色
            ctx.fillStyle = '#40E0D0';
            // 蛇身稍微内缩，留出间隙
            ctx.fillRect(px + 1, py + 1, ts - 2, ts - 2);

            // 亮边高光 (左上角)
            ctx.fillStyle = '#A3E4D7';
            ctx.fillRect(px + 1, py + 1, ts - 2, 4); // 上边
            ctx.fillRect(px + 1, py + 1, 4, ts - 2); // 左边

            // 暗角阴影 (右下角)
            ctx.fillStyle = '#17A589';
            ctx.fillRect(px + 1, py + ts - 5, ts - 2, 4); // 下边
            ctx.fillRect(px + ts - 5, py + 1, 4, ts - 2); // 右边

            // 闪亮小眼睛 (仅头部，跟随方向)
            if (isHead) {
                const eyeColor = '#FFF';
                const pupilColor = '#2C3E50';
                const eyeSize = 6, pupilSize = 3;
                let e1x, e1y, e2x, e2y;

                // 根据方向计算两只眼睛的偏移量
                if (this.state.dir === 'RIGHT') {
                    e1x = 18; e1y = 6; e2x = 18; e2y = 18;
                } else if (this.state.dir === 'LEFT') {
                    e1x = 6; e1y = 6; e2x = 6; e2y = 18;
                } else if (this.state.dir === 'UP') {
                    e1x = 6; e1y = 6; e2x = 18; e2y = 6;
                } else if (this.state.dir === 'DOWN') {
                    e1x = 6; e1y = 18; e2x = 18; e2y = 18;
                }

                // 画眼白
                ctx.fillStyle = eyeColor;
                ctx.fillRect(px + e1x, py + e1y, eyeSize, eyeSize);
                ctx.fillRect(px + e2x, py + e2y, eyeSize, eyeSize);
                
                // 画瞳孔 (偏向移动方向)
                ctx.fillStyle = pupilColor;
                const pOff = (this.state.dir === 'RIGHT' || this.state.dir === 'DOWN') ? 3 : 0;
                let p1x = (this.state.dir === 'RIGHT' || this.state.dir === 'LEFT') ? px + e1x + pOff : px + e1x + 1.5;
                let p1y = (this.state.dir === 'UP' || this.state.dir === 'DOWN') ? py + e1y + pOff : py + e1y + 1.5;
                let p2x = (this.state.dir === 'RIGHT' || this.state.dir === 'LEFT') ? px + e2x + pOff : px + e2x + 1.5;
                let p2y = (this.state.dir === 'UP' || this.state.dir === 'DOWN') ? py + e2y + pOff : py + e2y + 1.5;

                ctx.fillRect(p1x, p1y, pupilSize, pupilSize);
                ctx.fillRect(p2x, p2y, pupilSize, pupilSize);
            }
        });
    },

    // --- 6种超大水果 ---
    drawLargeFruit(food) {
        const ctx = this.ctx;
        // 网格内偏移，留出边距，使得水果绘制在中心区域
        const x = food.x * this.tileSize;
        const y = food.y * this.tileSize;
        const u = 3; // 基础像素块大小：将30px细分为10x10的画布，每块3px

        // 在格子内部画像素点的辅助函数
        const dot = (ox, oy, color, w=1, h=1) => {
            ctx.fillStyle = color;
            ctx.fillRect(x + ox*u, y + oy*u, w*u, h*u);
        };

        const { type } = food;

        if (type === 'apple') {
            dot(3,3,'#E74C3C',4,5); dot(2,4,'#E74C3C',6,3); // 红主体
            dot(3,3,'#FF9999',1,1); // 高光
            dot(4,2,'#5D4037',1,2); // 蒂
            dot(5,1,'#2ECC71',2,1); // 叶
        } 
        else if (type === 'strawberry') {
            dot(2,3,'#E74C3C',6,2); dot(3,5,'#E74C3C',4,2); dot(4,7,'#E74C3C',2,2); // 主体
            dot(3,4,'#F1C40F'); dot(6,4,'#F1C40F'); dot(4,6,'#F1C40F'); // 籽
            dot(3,2,'#2ECC71',4,1); dot(4,1,'#2ECC71',2,1); // 叶冠
        }
        else if (type === 'banana') {
            dot(2,6,'#F1C40F',4,2); dot(5,5,'#F1C40F',2,3); dot(6,3,'#F1C40F',2,3); // 黄主体
            dot(2,7,'#873600',1,1); // 尾部黑点
            dot(7,2,'#2ECC71',1,2); // 蒂
        }
        else if (type === 'orange') {
            dot(3,3,'#E67E22',4,5); dot(2,4,'#E67E22',6,3); // 橙主体
            dot(3,4,'#FAD7A1',1,1); // 高光
            dot(6,5,'#D35400',1,1); dot(5,7,'#D35400',1,1); // 暗纹
            dot(4,2,'#2ECC71',2,1); // 叶
        }
        else if (type === 'grape') {
            const purp = '#8E44AD', light = '#D2B4DE';
            dot(3,3,purp,2,2); dot(5,3,purp,2,2); dot(7,3,purp,2,2);
            dot(4,5,purp,2,2); dot(6,5,purp,2,2);
            dot(5,7,purp,2,2);
            dot(3,3,light); dot(5,3,light); dot(7,3,light); // 高光
            dot(5,1,'#873600',1,2); dot(6,1,'#2ECC71',1,1); // 藤与叶
        }
        else if (type === 'pineapple') {
            // 菠萝主体
            dot(3,4,'#F1C40F',4,4); dot(4,8,'#F1C40F',2,1); 
            dot(3,5,'#D4AC0D'); dot(5,7,'#D4AC0D'); dot(6,4,'#D4AC0D'); // 格子纹
            // 绿头
            dot(4,2,'#2ECC71',2,2); dot(3,1,'#2ECC71'); dot(6,1,'#2ECC71'); dot(4,0,'#2ECC71',2,1); 
        }
    },

    gameOver() {
        this.state.running = false;
        const hs = localStorage.getItem('stardewSnakeHS') || 0;
        if (this.state.score > hs) {
            localStorage.setItem('stardewSnakeHS', this.state.score);
        }
        
        document.getElementById('final-score').innerText = `本次收成：${this.state.score} 个水果`;
        document.getElementById('screen-gameover').classList.remove('hidden');
        document.getElementById('high-score').innerText = localStorage.getItem('stardewSnakeHS');
    }
};

// 页面加载完毕后初始化
window.onload = () => game.init();
</script>
</body>
</html>
