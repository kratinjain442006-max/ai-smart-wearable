# ai-smart-wearable
AI Smart Wearable — audio + directional haptic prototype
# AI Smart Wearable — Hear & Feel

**Prototype** that provides audio + direction-aware haptic alerts for visually impaired users.

import React, { useRef, useState } from "react";
import * as cocoSsd from "@tensorflow-models/coco-ssd";
import "@tensorflow/tfjs";
import useSpeechSynthesis from "react-speech-kit/dist/useSpeechSynthesis";

export default function App() {
  const videoRef = useRef(null);
  const canvasRef = useRef(null);
  const modelRef = useRef(null);
  const [running, setRunning] = useState(false);
  const [log, setLog] = useState([]);
  const { speak } = useSpeechSynthesis();

  // Track last spoken objects (avoid repetition)
  const lastSpoken = useRef({});
  const COOLDOWN_MS = 5000; // 5 sec

  // Start Camera + Load Model
  const startCamera = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    videoRef.current.srcObject = stream;
    await videoRef.current.play();

    modelRef.current = await cocoSsd.load();
    setRunning(true);
    detectFrame();
  };

  const stopCamera = () => {
    setRunning(false);
    let tracks = videoRef.current.srcObject.getTracks();
    tracks.forEach((track) => track.stop());
  };

  // Get Direction of Object
  const getDirection = (bbox, vw) => {
    const [x, , w] = bbox;
    const centerX = x + w / 2;
    if (centerX < vw / 3) return "on your left";
    if (centerX > (2 * vw) / 3) return "on your right";
    return "ahead";
  };

  // Get Danger Level (size = closeness)
  const getDangerLevel = (p, vw, vh) => {
    const [x, y, w, h] = p.bbox;
    const area = (w * h) / (vw * vh);
    if (area > 0.2) return "high";
    if (area > 0.05) return "medium";
    return "low";
  };

  // Speak & Vibrate
  const speakAndVibrate = (msg, dir, danger, obj) => {
    const now = Date.now();
    if (!lastSpoken.current[obj] || now - lastSpoken.current[obj] > COOLDOWN_MS) {
      speak({ text: msg });
      vibrate(dir, danger);
      lastSpoken.current[obj] = now;
      setLog((prev) => [...prev, `${new Date().toLocaleTimeString()} ${msg}`]);
    }
  };

  // Vibration Feedback
  const vibrate = (dir, danger) => {
    if (!navigator.vibrate) return;
    let pattern;
    if (danger === "high") pattern = [400];
    else if (danger === "medium") pattern = [200];
    else pattern = [100];

    if (dir.includes("left")) navigator.vibrate([...pattern, 100, ...pattern]);
    else if (dir.includes("right")) navigator.vibrate([100, ...pattern, 100]);
    else navigator.vibrate(pattern);
  };

  // Detection Loop
  const detectFrame = async () => {
    if (!modelRef.current || !running) return;

    const preds = await modelRef.current.detect(videoRef.current);
    const vw = videoRef.current.videoWidth;
    const vh = videoRef.current.videoHeight;

    const ctx = canvasRef.current.getContext("2d");
    canvasRef.current.width = vw;
    canvasRef.current.height = vh;
    ctx.clearRect(0, 0, vw, vh);

    preds
      .filter((p) => p.score > 0.6)
      .forEach((p) => {
        const [x, y, w, h] = p.bbox;
        ctx.strokeStyle = "lime";
        ctx.lineWidth = 2;
        ctx.strokeRect(x, y, w, h);
        ctx.fillStyle = "lime";
        ctx.fillText(`${p.class} ${Math.round(p.score * 100)}%`, x, y > 10 ? y - 5 : 10);

        const dir = getDirection(p.bbox, vw);
        const danger = getDangerLevel(p, vw, vh);
        const msg = `${p.class} ${dir}`;

        speakAndVibrate(msg, dir, danger, p.class);
      });

    requestAnimationFrame(detectFrame);
  };

  // SOS Emergency
  const sendSOS = () => {
    speak({ text: "Emergency help requested. Sending location." });
    alert("🚨 SOS sent with location!");
  };

  // Voice Command Simulation
  const handleVoiceCommand = (cmd) => {
    if (cmd.toLowerCase().includes("around")) {
      speak({ text: "Scanning surroundings" });
      if (log.length > 0) speak({ text: log.slice(-3).join(", ") });
    } else if (cmd.toLowerCase().includes("help")) {
      sendSOS();
    }
  };

  return (
    <div className="min-h-screen bg-gray-900 text-white flex flex-col items-center p-4">
      <h1 className="text-2xl font-bold mb-2">Hear & Feel — AI Assistant</h1>
      <div className="relative">
        <video ref={videoRef} className="rounded-2xl shadow-lg" width="480" height="360" />
        <canvas ref={canvasRef} className="absolute top-0 left-0" />
      </div>

      <div className="flex gap-3 mt-4">
        <button onClick={startCamera} className="bg-green-500 px-4 py-2 rounded-lg">Start</button>
        <button onClick={stopCamera} className="bg-yellow-500 px-4 py-2 rounded-lg">Stop</button>
        <button onClick={sendSOS} className="bg-red-600 px-4 py-2 rounded-lg">🚨 SOS</button>
      </div>

      <div className="mt-4 space-x-2">
        <button onClick={() => handleVoiceCommand("what's around me")} className="bg-blue-500 px-4 py-2 rounded-lg">
          "What's around me?"
        </button>
        <button onClick={() => handleVoiceCommand("help me")} className="bg-pink-500 px-4 py-2 rounded-lg">
          "Help me"
        </button>
      </div>

      <div className="mt-4 w-full max-w-md bg-gray-800 p-3 rounded-xl h-40 overflow-y-scroll text-sm">
        {log.map((entry, i) => (
          <p key={i}>{entry}</p>
        ))}
      </div>
    </div>
  );
}








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
