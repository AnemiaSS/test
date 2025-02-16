<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Eye Detection with Camera Controls</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils"></script>
    <style>
        body {
            font-family: 'Arial', sans-serif;
            background: #faf8f8;
            text-align: center;
            margin: 0;
            padding: 0;
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            flex-direction: column;
        }
        .container {
            background-color: rgba(255, 255, 255, 0.9);
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            width: 100%;
            max-width: 720px;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
        }
        .main-canvas {
            border-radius: 10px;
            max-width: 100%;
            max-height: 400px;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
            margin: 15px 0;
        }
        .camera-controls {
            display: flex;
            gap: 10px;
            margin-bottom: 15px;
        }
        .camera-btn {
            background-color: #f5b62a;
            color: white;
            padding: 8px 15px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: background-color 0.3s ease;
        }
        .camera-btn:hover {
            background-color: #f07325;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>ถ่ายภาพเพื่อวิเคราะห์</h1>
        <div class="camera-controls">
            <button id="flipCamera" class="camera-btn">สลับกล้อง</button>
        </div>
        <video id="video" autoplay playsinline></video>
        <canvas id="outputCanvas" class="main-canvas"></canvas>
        <button id="captureButton" class="camera-btn">📸 ถ่ายภาพ</button>
    </div>

    <script>
        const videoElement = document.getElementById('video');
        const canvasElement = document.getElementById('outputCanvas');
        const canvasCtx = canvasElement.getContext('2d');
        let currentCamera = 'user';
        let faceMesh;
        let camera;
        let currentLandmarks = null;

        async function startCamera() {
            try {
                if (videoElement.srcObject) {
                    videoElement.srcObject.getTracks().forEach(track => track.stop());
                }
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { facingMode: currentCamera }
                });
                videoElement.srcObject = stream;
                await videoElement.play();
                if (!faceMesh) {
                    faceMesh = new FaceMesh({
                        locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`,
                    });
                    faceMesh.setOptions({
                        maxNumFaces: 1,
                        refineLandmarks: true,
                        minDetectionConfidence: 0.5,
                        minTrackingConfidence: 0.5,
                    });
                    faceMesh.onResults(onResults);
                }
                if (camera) {
                    camera.stop();
                }
                camera = new Camera(videoElement, {
                    onFrame: async () => {
                        await faceMesh.send({ image: videoElement });
                    },
                    width: 640,
                    height: 480,
                });
                camera.start();
            } catch (err) {
                console.error('Error starting camera:', err);
                alert('ไม่สามารถเปิดกล้องได้ กรุณาตรวจสอบการอนุญาตการใช้กล้อง');
            }
        }

        function switchCamera() {
            currentCamera = currentCamera === 'user' ? 'environment' : 'user';
            startCamera();
        }

        function onResults(results) {
            canvasElement.width = videoElement.videoWidth;
            canvasElement.height = videoElement.videoHeight;

            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);

            if (results.multiFaceLandmarks && results.multiFaceLandmarks.length > 0) {
                currentLandmarks = results.multiFaceLandmarks[0];
            }

            canvasCtx.restore();
        }

        document.getElementById('flipCamera').addEventListener('click', switchCamera);
        document.getElementById('captureButton').addEventListener('click', () => {
            if (!currentLandmarks) return;
            
            const leftEyeIndices = [33, 160, 158, 133, 153, 144];
            const rightEyeIndices = [362, 385, 387, 263, 373, 380];
            const scaleFactor = 2.5;

            function captureEyeRegion(landmarks, indices, scaleFactor) {
                const xCoords = indices.map((idx) => landmarks[idx].x * canvasElement.width);
                const yCoords = indices.map((idx) => landmarks[idx].y * canvasElement.height);

                const xMin = Math.min(...xCoords);
                const xMax = Math.max(...xCoords);
                const yMin = Math.min(...yCoords);
                const yMax = Math.max(...yCoords);

                const width = xMax - xMin;
                const height = yMax - yMin;
                const size = Math.max(width, height) * scaleFactor;

                const centerX = (xMin + xMax) / 2;
                const centerY = (yMin + yMax) / 2;

                return { x: centerX - size / 2, y: centerY - size / 2, size: size };
            }

            const leftRegion = captureEyeRegion(currentLandmarks, leftEyeIndices, scaleFactor);
            const rightRegion = captureEyeRegion(currentLandmarks, rightEyeIndices, scaleFactor);

            function storeCapturedImages(leftRegion, rightRegion) {
                const leftCanvas = document.createElement('canvas');
                const rightCanvas = document.createElement('canvas');
                leftCanvas.width = rightCanvas.width = leftRegion.size;
                leftCanvas.height = rightCanvas.height = rightRegion.size;

                const leftCtx = leftCanvas.getContext('2d');
                const rightCtx = rightCanvas.getContext('2d');

                leftCtx.drawImage(canvasElement, leftRegion.x, leftRegion.y, leftRegion.size, leftRegion.size, 0, 0, leftRegion.size, leftRegion.size);
                rightCtx.drawImage(canvasElement, rightRegion.x, rightRegion.y, rightRegion.size, rightRegion.size, 0, 0, rightRegion.size, rightRegion.size);

                localStorage.setItem('leftEyeImage', leftCanvas.toDataURL('image/png'));
                localStorage.setItem('rightEyeImage', rightCanvas.toDataURL('image/png'));

                window.location.href = "page5.html";
            }

            storeCapturedImages(leftRegion, rightRegion);
        });

        startCamera();
    </script>
</body>
</html>
