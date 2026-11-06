<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>フォーム比較＆骨格解析アプリ</title>
    
    <!-- 骨格検出用のAIライブラリ -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/pose-detection"></script>

    <style>
        * { box-sizing: border-box; }
        body {
            margin: 0; padding: 0;
            background-color: #1a1a1a; color: #fff;
            font-family: sans-serif;
            display: flex; flex-direction: column;
            height: 100vh; overflow: hidden;
            user-select: none; /* 誤動作防止 */
        }

        /* 全体コントロール（上部） */
        .global-controls {
            display: flex; flex-direction: column;
            padding: 15px 20px; background-color: #222;
            gap: 15px; border-bottom: 2px solid #444;
        }
        
        .controls-row {
            display: flex; justify-content: center; align-items: center; gap: 25px;
            width: 100%;
        }

        #playBtn {
            padding: 12px 30px; font-size: 18px; font-weight: bold;
            cursor: pointer; background-color: #4CAF50; color: white;
            border: none; border-radius: 6px; min-width: 140px;
        }
        #playBtn.paused { background-color: #f44336; }

        .toggle-group {
            display: flex; gap: 15px; font-size: 16px;
        }
        input[type="checkbox"] {
            transform: scale(1.5); margin-right: 8px; vertical-align: middle;
        }

        /* スライダーをタブレットで操作しやすくするカスタムデザイン */
        input[type="range"] {
            -webkit-appearance: none; appearance: none;
            background: transparent; outline: none; margin: 0;
        }
        input[type="range"]::-webkit-slider-runnable-track {
            width: 100%; height: 12px; background: #555; border-radius: 6px;
        }
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none; appearance: none;
            height: 34px; width: 34px; background: #4CAF50;
            border-radius: 50%; cursor: pointer;
            margin-top: -11px; /* トラックの中央に配置 */
            box-shadow: 0 2px 5px rgba(0,0,0,0.5);
        }
        
        /* 同時進行スライダー専用（より大きく） */
        .master-slider-container {
            display: flex; align-items: center; gap: 15px; width: 100%; font-size: 18px;
        }
        #masterSlider { flex: 1; }
        #masterSlider::-webkit-slider-runnable-track { height: 16px; background: #444; }
        #masterSlider::-webkit-slider-thumb { height: 44px; width: 44px; background: #2196F3; margin-top: -14px; }

        /* 左右の動画エリア */
        .video-area {
            display: flex; flex: 1; width: 100%; gap: 5px; padding: 5px;
        }
        .video-section {
            flex: 1; display: flex; flex-direction: column; gap: 5px;
        }
        .video-wrapper {
            position: relative; flex: 1;
            border: 2px solid #555; background: #000; overflow: hidden;
        }
        video { width: 100%; height: 100%; object-fit: contain; }
        canvas {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%; object-fit: contain; pointer-events: none; z-index: 10;
        }
        
        /* グリッド線 */
        .grid-overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background-image: 
                linear-gradient(to right, rgba(255,255,255,0.25) 1px, transparent 1px),
                linear-gradient(to bottom, rgba(255,255,255,0.25) 1px, transparent 1px);
            background-size: 10% 10%; pointer-events: none; z-index: 5;
        }

        /* 個別コントロール（下部） */
        .local-controls {
            background-color: #333; padding: 12px; border-radius: 6px;
            display: flex; flex-direction: column; gap: 12px; font-size: 15px;
        }
        .local-controls-row { display: flex; justify-content: space-between; align-items: center; }
        .start-slider { width: 65%; }
        input[type="file"] { max-width: 220px; font-size: 14px; }
    </style>
</head>
<body>
    <!-- 上部の全体操作パネル -->
    <div class="global-controls">
        <div class="controls-row">
            <button id="playBtn">再生</button>
            <label>速度: <input type="range" id="speed" min="0.1" max="1" step="0.1" value="1" style="width: 100px;"> <span id="speedVal">1.0</span>x</label>
            <div class="toggle-group">
                <label><input type="checkbox" id="toggleGrid" checked> グリッド</label>
                <label><input type="checkbox" id="toggleSkeleton" checked> 骨格AI</label>
            </div>
        </div>
        <!-- 同時進行スライダーを独立させて画面幅いっぱいに -->
        <div class="master-slider-container">
            <label>全体スライド:</label>
            <input type="range" id="masterSlider" min="0" max="100" value="0" step="0.001">
        </div>
    </div>

    <!-- 左右の動画と個別操作エリア -->
    <div class="video-area">
        <!-- 動画1 -->
        <div class="video-section">
            <div class="video-wrapper">
                <video id="video1" playsinline muted></video>
                <div class="grid-overlay" id="grid1"></div>
                <canvas id="canvas1"></canvas>
            </div>
            <div class="local-controls">
                <div class="local-controls-row">
                    <input type="file" id="upload1" accept="video/*">
                </div>
                <div class="local-controls-row">
                    <label>開始位置:</label>
                    <!-- stepを0.001にして極細かく動かせるように -->
                    <input type="range" id="startSlider1" class="start-slider" min="0" max="0" step="0.001" value="0">
                    <span id="startText1" style="min-width: 50px; text-align: right;">0.00秒</span>
                </div>
            </div>
        </div>

        <!-- 動画2 -->
        <div class="video-section">
            <div class="video-wrapper">
                <video id="video2" playsinline muted></video>
                <div class="grid-overlay" id="grid2"></div>
                <canvas id="canvas2"></canvas>
            </div>
            <div class="local-controls">
                <div class="local-controls-row">
                    <input type="file" id="upload2" accept="video/*">
                </div>
                <div class="local-controls-row">
                    <label>開始位置:</label>
                    <input type="range" id="startSlider2" class="start-slider" min="0" max="0" step="0.001" value="0">
                    <span id="startText2" style="min-width: 50px; text-align: right;">0.00秒</span>
                </div>
            </div>
        </div>
    </div>

    <script>
        const v1 = document.getElementById('video1');
        const v2 = document.getElementById('video2');
        const c1 = document.getElementById('canvas1');
        const c2 = document.getElementById('canvas2');
        const ctx1 = c1.getContext('2d');
        const ctx2 = c2.getContext('2d');

        const playBtn = document.getElementById('playBtn');
        const masterSlider = document.getElementById('masterSlider');
        const startSlider1 = document.getElementById('startSlider1');
        const startSlider2 = document.getElementById('startSlider2');
        
        let detector;
        let isPlaying = false;
        let offset1 = 0;
        let offset2 = 0;
        let isMasterDragging = false;
        let showSkeleton = true;

        // AIモデル初期化
        async function initAI() {
            const detectorConfig = { modelType: poseDetection.movenet.modelType.SINGLEPOSE_LIGHTNING };
            detector = await poseDetection.createDetector(poseDetection.SupportedModels.MoveNet, detectorConfig);
        }
        initAI();

        // 表示オンオフ機能
        document.getElementById('toggleGrid').addEventListener('change', e => {
            const display = e.target.checked ? 'block' : 'none';
            document.getElementById('grid1').style.display = display;
            document.getElementById('grid2').style.display = display;
        });

        document.getElementById('toggleSkeleton').addEventListener('change', e => {
            showSkeleton = e.target.checked;
            if (!isPlaying) {
                // 停止中にオンオフを切り替えたら即座に画面に反映させる
                if(showSkeleton) {
                    processFrame(v1, c1, ctx1);
                    processFrame(v2, c2, ctx2);
                } else {
                    ctx1.clearRect(0, 0, c1.width, c1.height);
                    ctx2.clearRect(0, 0, c2.width, c2.height);
                }
            }
        });

        // ファイル読み込み処理
        document.getElementById('upload1').addEventListener('change', e => loadVideo(e, v1, startSlider1));
        document.getElementById('upload2').addEventListener('change', e => loadVideo(e, v2, startSlider2));

        function loadVideo(event, videoEl, sliderEl) {
            const file = event.target.files[0];
            if (file) {
                videoEl.src = URL.createObjectURL(file);
                videoEl.load();
                videoEl.onloadedmetadata = () => {
                    sliderEl.max = videoEl.duration; 
                    updateMasterSliderMax();
                };
            }
        }

        // 開始位置スライダー（個別）を動かした時の処理
        startSlider1.addEventListener('input', e => {
            offset1 = parseFloat(e.target.value);
            document.getElementById('startText1').textContent = offset1.toFixed(2) + '秒';
            v1.currentTime = offset1;
            processFrame(v1, c1, ctx1); 
            updateMasterSliderMax();
            resetToStart();
        });

        startSlider2.addEventListener('input', e => {
            offset2 = parseFloat(e.target.value);
            document.getElementById('startText2').textContent = offset2.toFixed(2) + '秒';
            v2.currentTime = offset2;
            processFrame(v2, c2, ctx2);
            updateMasterSliderMax();
            resetToStart();
        });

        function resetToStart() {
            if(!isPlaying) {
                masterSlider.value = 0;
                v1.currentTime = offset1;
                v2.currentTime = offset2;
            }
        }

        function updateMasterSliderMax() {
            const max1 = v1.duration ? v1.duration - offset1 : 0;
            const max2 = v2.duration ? v2.duration - offset2 : 0;
            masterSlider.max = Math.max(max1, max2, 0.1);
        }

        // 全体スライド（同時進行）の操作
        masterSlider.addEventListener('mousedown', () => { isMasterDragging = true; pauseVideos(); });
        masterSlider.addEventListener('touchstart', () => { isMasterDragging = true; pauseVideos(); });
        masterSlider.addEventListener('mouseup', () => isMasterDragging = false);
        masterSlider.addEventListener('touchend', () => isMasterDragging = false);

        masterSlider.addEventListener('input', e => {
            const elapsed = parseFloat(e.target.value);
            v1.currentTime = offset1 + elapsed;
            v2.currentTime = offset2 + elapsed;
            
            if (!isPlaying) {
                processFrame(v1, c1, ctx1);
                processFrame(v2, c2, ctx2);
            }
        });

        v1.addEventListener('timeupdate', () => {
            if (isPlaying && !isMasterDragging) {
                masterSlider.value = Math.max(0, v1.currentTime - offset1);
            }
        });

        // 速度調整
        document.getElementById('speed').addEventListener('input', e => {
            const speed = parseFloat(e.target.value);
            v1.playbackRate = speed;
            v2.playbackRate = speed;
            document.getElementById('speedVal').textContent = speed.toFixed(1);
        });

        // 再生・一時停止の処理
        function pauseVideos() {
            if(isPlaying) {
                v1.pause(); v2.pause();
                playBtn.textContent = '再生'; playBtn.classList.remove('paused');
                isPlaying = false;
            }
        }

        playBtn.addEventListener('click', () => {
            if (isPlaying) {
                pauseVideos();
            } else {
                if (parseFloat(masterSlider.value) >= parseFloat(masterSlider.max)) {
                    masterSlider.value = 0;
                    v1.currentTime = offset1;
                    v2.currentTime = offset2;
                }
                v1.play(); v2.play();
                playBtn.textContent = '一時停止'; playBtn.classList.add('paused');
                isPlaying = true;
                detectAndDraw();
            }
        });

        async function detectAndDraw() {
            if (!isPlaying) return;
            await processFrame(v1, c1, ctx1);
            await processFrame(v2, c2, ctx2);
            requestAnimationFrame(detectAndDraw);
        }

        async function processFrame(video, canvas, ctx) {
            if (video.readyState < 2 || !detector) return;
            
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // チェックボックスがオフの場合はAI解析・描画をスキップ
            if (!showSkeleton) return;

            const poses = await detector.estimatePoses(video);
            if (poses.length > 0) drawSkeleton(poses[0].keypoints, ctx);
        }

        function drawSkeleton(keypoints, ctx) {
            const threshold = 0.3;
            const keypointMap = {};

            keypoints.forEach(p => {
                keypointMap[p.name] = p;
                if (p.score > threshold) {
                    ctx.beginPath();
                    ctx.arc(p.x, p.y, 6, 0, 2 * Math.PI);
                    ctx.fillStyle = '#00FF00';
                    ctx.fill();
                }
            });

            const edges = [
                ['left_shoulder', 'right_shoulder'],
                ['left_shoulder', 'left_elbow'], ['left_elbow', 'left_wrist'],
                ['right_shoulder', 'right_elbow'], ['right_elbow', 'right_wrist'],
                ['left_shoulder', 'left_hip'], ['right_shoulder', 'right_hip'],
                ['left_hip', 'right_hip'],
                ['left_hip', 'left_knee'], ['left_knee', 'left_ankle'],
                ['right_hip', 'right_knee'], ['right_knee', 'right_ankle']
            ];

            ctx.lineWidth = 4;
            ctx.strokeStyle = '#FF0000';
            edges.forEach(([p1, p2]) => {
                const kp1 = keypointMap[p1];
                const kp2 = keypointMap[p2];
                if (kp1 && kp2 && kp1.score > threshold && kp2.score > threshold) {
                    ctx.beginPath();
                    ctx.moveTo(kp1.x, kp1.y);
                    ctx.lineTo(kp2.x, kp2.y);
                    ctx.stroke();
                }
            });
        }
    </script>
</body>
</html>
