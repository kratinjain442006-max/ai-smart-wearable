# ai-smart-wearable
AI Smart Wearable — audio + directional haptic prototype
# AI Smart Wearable — Hear & Feel

**Prototype** that provides audio + direction-aware haptic alerts for visually impaired users.

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Smart Wearable Assistant Prototype</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #1a1a1a;
            color: #e0e0e0;
        }
        .container {
            max-width: 90%;
            width: 100%;
            padding: 2rem;
            background-color: #2a2a2a;
            border-radius: 1rem;
            box-shadow: 0 10px 20px rgba(0, 0, 0, 0.4);
            text-align: center;
        }
        .video-container {
            position: relative;
            width: 100%;
            max-width: 600px;
            margin: 0 auto 1.5rem;
            border-radius: 0.75rem;
            overflow: hidden;
            border: 2px solid #5a5a5a;
        }
        video {
            width: 100%;
            height: auto;
            border-radius: 0.75rem;
        }
        #feedback-box {
            background-color: #333;
            border-radius: 0.5rem;
            padding: 1rem;
            min-height: 4rem;
            display: flex;
            justify-content: center;
            align-items: center;
            font-size: 1.125rem;
            font-weight: 600;
            transition: background-color 0.3s ease;
        }
        .alert-low {
            background-color: #ffcc00;
            color: #333;
        }
        .alert-high {
            background-color: #e53e3e;
            color: white;
        }
        .button-group {
            margin-top: 1.5rem;
            display: flex;
            justify-content: center;
            gap: 1rem;
            flex-wrap: wrap;
        }
        button {
            padding: 0.75rem 1.5rem;
            border-radius: 0.5rem;
            font-weight: 700;
            color: white;
            transition: transform 0.2s ease, box-shadow 0.2s ease;
            cursor: pointer;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
        }
        button:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 8px rgba(0, 0, 0, 0.3);
        }
        .btn-left {
            background-color: #3498db;
        }
        .btn-right {
            background-color: #e74c3c;
        }
        .btn-center {
            background-color: #2ecc71;
        }
    </style>
</head>
<body class="bg-gray-900 text-white">
    <div class="container flex flex-col items-center">
        <h1 class="text-3xl sm:text-4xl font-bold mb-2 text-cyan-400">Hear & Feel Technology</h1>
        <p class="text-lg sm:text-xl font-medium text-gray-400 mb-6">Prototype for Visually Impaired Assistance</p>

        <div class="video-container">
            <video id="webcam-feed" autoplay playsinline></video>
        </div>

        <div id="feedback-box" class="mb-6">
            <p id="feedback-text" class="text-gray-400">Waiting for detection...</p>
        </div>

        <div class="button-group">
            <button id="left-alert" class="btn-left">Simulate Obstacle on Left</button>
            <button id="right-alert" class="btn-right">Simulate Obstacle on Right</button>
            <button id="danger-alert" class="btn-center">Simulate High Danger</button>
        </div>
    </div>

    <script>
        const video = document.getElementById('webcam-feed');
        const feedbackBox = document.getElementById('feedback-box');
        const feedbackText = document.getElementById('feedback-text');
        const leftAlertBtn = document.getElementById('left-alert');
        const rightAlertBtn = document.getElementById('right-alert');
        const dangerAlertBtn = document.getElementById('danger-alert');

        let isVibrating = false;

        // Request access to the webcam
        async function setupWebcam() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ video: true });
                video.srcObject = stream;
            } catch (err) {
                feedbackText.textContent = 'Error: Could not access webcam. Please check your permissions.';
                console.error('Error accessing webcam: ', err);
            }
        }

        function playAudio(message) {
            const utterance = new SpeechSynthesisUtterance(message);
            utterance.pitch = 1;
            utterance.rate = 1.2;
            utterance.volume = 1;
            speechSynthesis.speak(utterance);
        }

        function vibrate(pattern) {
            if ("vibrate" in navigator) {
                if (isVibrating) return; // Prevent multiple vibrations at once
                isVibrating = true;
                navigator.vibrate(pattern);
                // Reset after the vibration pattern ends
                const totalDuration = pattern.reduce((sum, val) => sum + val, 0);
                setTimeout(() => {
                    isVibrating = false;
                }, totalDuration);
            } else {
                console.log("Vibration API not supported on this device.");
            }
        }

        function setFeedback(message, intensity) {
            feedbackText.textContent = message;
            feedbackBox.classList.remove('alert-low', 'alert-high');
            if (intensity === 'low') {
                feedbackBox.classList.add('alert-low');
                playAudio(message);
                vibrate([200, 100, 200]); // A light buzz
            } else if (intensity === 'high') {
                feedbackBox.classList.add('alert-high');
                playAudio(message);
                vibrate([500, 100, 500, 100, 500]); // A strong vibration
            }
        }

        leftAlertBtn.addEventListener('click', () => {
            setFeedback("Obstacle coming from the left!", 'low');
        });

        rightAlertBtn.addEventListener('click', () => {
            setFeedback("Obstacle coming from the right!", 'low');
        });

        dangerAlertBtn.addEventListener('click', () => {
            setFeedback("High danger ahead! Stop!", 'high');
        });

        window.onload = setupWebcam;
    </script>
</body>
</html>





## Demo
Short demo: (add GIF or link to demo video)

## Repo structure
ai-smart-wearable/
├─ detection/
│  ├─ app.py
│  ├─ detector.py
│  ├─ utils.py
│  └─ requirements.txt
├─ esp32/
│  └─ vib_wristband.ino
├─ assets/
│  └─ sample_frame.png
├─ README.md
└─ LICENSE
## Features
- Object detection (YOLOv8)
- Direction-aware vibration (left/right/center)
- Danger-level alerts (short/long vibration + urgent voice)
- Offline TTS (pyttsx3) and BLE haptic commands

## Quick start (local)
1. Clone:
```bash
git clone https://github.com/<your-username>/ai-smart-wearable.git
cd ai-smart-wearable/detection
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\Activate.ps1
pip install -r requirements.txt
python app.py
