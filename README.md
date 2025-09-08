# ai-smart-wearable
AI Smart Wearable — audio + directional haptic prototype
# AI Smart Wearable — Hear & Feel

**Prototype** that provides audio + direction-aware haptic alerts for visually impaired users.

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0" />
  <title>Hear & Feel — Prototype</title>
  <style>
    body { font-family: system-ui, -apple-system, Arial; margin:0; background:#111; color:#eaeaea; display:flex; flex-direction:column; height:100vh; }
    header { padding:10px 14px; background:#0b0b0b; display:flex; align-items:center; gap:10px; box-shadow: 0 2px 6px rgba(0,0,0,0.6); }
    header h1 { font-size:1rem; margin:0; }
    #main { flex:1; display:flex; gap:8px; padding:8px; box-sizing:border-box; }
    #videoWrap { flex:1; position:relative; border-radius:10px; overflow:hidden; background:#222; display:flex; align-items:center; justify-content:center; }
    video { width:100%; height:100%; object-fit:cover; transform: scaleX(-1); } /* mirror for natural feel */
    canvas { position:absolute; left:0; top:0; width:100%; height:100%; pointer-events:none; }
    aside { width:340px; max-width:40%; background:#0f1720; padding:12px; border-radius:10px; display:flex; flex-direction:column; gap:8px; }
    button { padding:10px; border-radius:8px; border:0; background:#2563eb; color:white; font-weight:600; }
    .small { font-size:0.9rem; opacity:0.9; }
    .status { background:#081129; padding:8px; border-radius:8px; font-size:0.9rem; min-height:48px; }
    label { display:flex; gap:8px; align-items:center; }
    .row { display:flex; gap:8px; margin-top:6px; }
  </style>
</head>
<body>
  <header>
    <h1>Hear & Feel — Hackathon Prototype</h1>
    <div class="small">Dual-sense alerts: audio + vibration. Say "Help me" to send location.</div>
  </header>

  <div id="main">
    <div id="videoWrap">
      <video id="video" autoplay playsinline muted></video>
      <canvas id="overlay"></canvas>
    </div>

    <aside>
      <div class="status" id="status">Model not loaded.</div>

      <label><input id="toggleDetect" type="checkbox" checked> Detection enabled</label>
      <label><input id="toggleAudio" type="checkbox" checked> Audio enabled</label>
      <label><input id="toggleVibe" type="checkbox" checked> Vibration enabled</label>

      <div class="row">
        <button id="startBtn">Start Camera</button>
        <button id="stopBtn">Stop</button>
      </div>

      <div style="margin-top:8px;">
        <div class="small"><strong>Detection Log</strong></div>
        <div id="log" style="height:180px; overflow:auto; background:#061022; padding:8px; border-radius:6px; font-size:0.9rem;"></div>
      </div>
    </aside>
  </div>

  <!-- TensorFlow and coco-ssd via CDN -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.13.0/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd@2.2.2/dist/coco-ssd.min.js"></script>

  <script>
  // ====== Settings ======
  const MIN_SCORE = 0.5; // confidence threshold
  const MAX_VOICE_RATE = 1.0;
  const DANGER_CLASSES = new Set(["car","truck","bus","bicycle","motorbike","person","chair","stairs","bench"]); // sample
  const SUPPRESS_REPEAT_MS = 1200; // don't repeat same exact notification immediately

  // ====== DOM ======
  const video = document.getElementById('video');
  const overlay = document.getElementById('overlay');
  const ctx = overlay.getContext('2d');
  const statusEl = document.getElementById('status');
  const logEl = document.getElementById('log');

  const toggleDetect = document.getElementById('toggleDetect');
  const toggleAudio = document.getElementById('toggleAudio');
  const toggleVibe = document.getElementById('toggleVibe');

  const startBtn = document.getElementById('startBtn');
  const stopBtn = document.getElementById('stopBtn');

  let model = null;
  let stream = null;
  let detectLoop = null;
  let lastAnnounce = 0;
  let lastKey = null;

  // Helpful small logger
  function addLog(s){
    const d = new Date().toLocaleTimeString();
    logEl.innerHTML = `<div>[${d}] ${s}</div>` + logEl.innerHTML;
  }

  // Utility: speak text (audio)
  function speak(text, urgent=false){
    if(!toggleAudio.checked) return;
    const u = new SpeechSynthesisUtterance(text);
    u.rate = urgent ? Math.max(0.9, MAX_VOICE_RATE) : 0.9;
    window.speechSynthesis.cancel();
    window.speechSynthesis.speak(u);
  }

  // Utility: vibrate with a pattern mapped to danger and direction
  // We'll use patterns (ms on, ms off). Phone vibration can't localize left/right,
  // but pattern differences + short/long combination will stand in for direction.
  function vibrateFor(level, direction){
    if(!navigator.vibrate || !toggleVibe.checked) return;
    // level: 'low' | 'medium' | 'high'
    // direction: 'left' | 'right' | 'center'
    let pattern;
    if(level === 'high'){
      pattern = [250,100,250]; // urgent
    } else if(level === 'medium'){
      pattern = [140,80,140];
    } else {
      pattern = [70];
    }

    // Add a directional prefix/suffix so user can learn patterns:
    if(direction === 'left'){
      pattern = [60,30].concat(pattern);
    } else if(direction === 'right'){
      pattern = [30,60].concat(pattern);
    } else {
      // center: no extra
    }
    navigator.vibrate(pattern);
  }

  // Determine danger level heuristically from bbox size + class
  function mapToDanger(obj){
    // obj has .score, .bbox = [x,y,w,h], .class
    const area = obj.bbox[2] * obj.bbox[3];
    let level = 'low';
    if(obj.class && (obj.class === 'car' || obj.class === 'truck' || obj.class === 'bus' || obj.class === 'motorbike')){
      level = 'high';
    } else if(area > 0.15 * video.videoWidth * video.videoHeight){
      level = 'high';
    } else if(area > 0.05 * video.videoWidth * video.videoHeight){
      level = 'medium';
    }
    return level;
  }

  // Decide left/center/right from bbox center relative to frame
  function mapToDirection(obj){
    const cx = obj.bbox[0] + obj.bbox[2]/2;
    const rel = (cx / video.videoWidth) - 0.5; // -0.5..+0.5
    if(rel < -0.18) return 'left';
    if(rel > 0.18) return 'right';
    return 'center';
  }

  // Generate a short unique key for deduplication
  function detectionKey(obj){
    return `${obj.class}|${Math.round(obj.bbox[0])}|${Math.round(obj.bbox[1])}|${Math.round(obj.bbox[2])}`;
  }

  // Draw boxes
  function drawDetections(dets){
    overlay.width = video.videoWidth;
    overlay.height = video.videoHeight;
    ctx.clearRect(0,0,overlay.width,overlay.height);
    ctx.lineWidth = 3;
    ctx.font = "14px system-ui";
    dets.forEach(d=>{
      const [x,y,w,h] = d.bbox;
      ctx.strokeStyle = 'rgba(0,200,150,0.9)';
      ctx.fillStyle = 'rgba(0,200,150,0.15)';
      ctx.fillRect(x,y,w,h);
      ctx.strokeRect(x,y,w,h);
      const txt = `${d.class} ${(d.score*100|0)}%`;
      ctx.fillStyle = '#fff';
      ctx.fillText(txt, x+4, y+16);
    });
    // center guide line
    ctx.strokeStyle = 'rgba(255,255,255,0.12)';
    ctx.beginPath();
    ctx.moveTo(overlay.width/2, 0);
    ctx.lineTo(overlay.width/2, overlay.height);
    ctx.stroke();
  }

  // Main detection loop
  async function runDetect(){
    if(!model || !toggleDetect.checked) return;
    if (video.readyState < 2) {
      setTimeout(runDetect, 200);
      return;
    }
    try {
      const preds = await model.detect(video);
      const now = Date.now();

      // Filter by score
      const good = preds.filter(p => p.score >= MIN_SCORE);

      // Draw
      drawDetections(good);

      // Choose top-priority object: prefer highest danger then score
      if(good.length){
        // sort by danger then area then score
        good.sort((a,b)=>{
          const da = mapToDanger(a), db = mapToDanger(b);
          const imp = ({low:0, medium:1, high:2});
          if (imp[da] !== imp[db]) return imp[db]-imp[da];
          const areaA = a.bbox[2]*a.bbox[3], areaB = b.bbox[2]*b.bbox[3];
          if (areaA !== areaB) return areaB-areaA;
          return b.score - a.score;
        });
        const top = good[0];
        const dkey = detectionKey(top);

        // Deduplicate repeated announcements quickly
        if (dkey !== lastKey || (now - lastAnnounce) > SUPPRESS_REPEAT_MS){
          lastKey = dkey;
          lastAnnounce = now;

          const dir = mapToDirection(top);
          const level = mapToDanger(top);
          const txt = `${top.class} ${Math.round(top.score*100)}% ${dir === 'center' ? 'ahead' : 'to the ' + dir}`;
          addLog(`${txt} (level: ${level})`);
          // Audio and vibration
          if(toggleAudio.checked){
            const urgent = (level === 'high');
            speak(`${top.class} ${dir === 'center' ? 'ahead' : dir}`, urgent);
          }
          if(toggleVibe.checked){
            vibrateFor(level, dir);
          }
        }
      }
    } catch(e){
      console.error(e);
      addLog("Detect error: " + (e.message||e));
    } finally {
      detectLoop = requestAnimationFrame(runDetect);
    }
  }

  // Camera start/stop
  async function startCamera(){
    if(stream) return;
    try {
      stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" }, audio: false});
      video.srcObject = stream;
      await video.play();
      overlay.width = video.videoWidth;
      overlay.height = video.videoHeight;
      statusEl.innerText = 'Camera running. Loading model...';
      if(!model){
        model = await cocoSsd.load();
        statusEl.innerText = 'Model loaded. Detecting...';
      } else {
        statusEl.innerText = 'Detecting...';
      }
      runDetect();
      addLog('Camera started');
    } catch(err){
      console.error(err);
      alert('Camera permission denied or not available. Try in Chrome on Android/iOS Safari with HTTPS.');
      statusEl.innerText = 'Camera not available';
    }
  }

  function stopCamera(){
    if(stream){
      stream.getTracks().forEach(t => t.stop());
      stream = null;
      video.srcObject = null;
      cancelAnimationFrame(detectLoop);
      detectLoop = null;
      drawDetections([]); // clear
      statusEl.innerText = 'Stopped';
      addLog('Camera stopped');
    }
  }

  // ========== Voice command ("Help me") ===========
  function setupVoiceCommands(){
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    if(!SpeechRecognition){
      addLog('Voice recognition not supported in this browser.');
      return;
    }
    const recog = new SpeechRecognition();
    recog.continuous = true;
    recog.interimResults = false;
    recog.lang = 'en-US';
    recog.onresult = (ev) => {
      const last = ev.results[ev.results.length-1];
      const txt = last[0].transcript.trim().toLowerCase();
      addLog('[VOICE] ' + txt);
      if(txt.includes('help me') || txt.includes('help')){
        triggerHelp();
      } else if (txt.includes("what's around me") || txt.includes("what is around me")){
        // immediate manual query
        speak('Scanning environment. Please wait.');
        addLog('Manual voice query triggered scanning.');
      }
    };
    recog.onerror = (e)=> addLog('Voice error: '+ e.error);
    recog.onend = ()=> {
      // restart so continuous listening persists
      try { recog.start(); } catch(e){}
    };
    try { recog.start(); addLog('Voice listening started (say "Help me")'); } catch(e){ addLog('Voice start error'); }
  }

  // Trigger help: grab GPS and POST to /help
  async function triggerHelp(){
    addLog('Help triggered — getting location...');
    if(!navigator.geolocation){
      addLog('Geolocation unavailable.');
      speak('Unable to get location.');
      return;
    }
    navigator.geolocation.getCurrentPosition(async (pos)=>{
      const payload = {
        lat: pos.coords.latitude,
        lon: pos.coords.longitude,
        accuracy: pos.coords.accuracy,
        ts: new Date().toISOString()
      };
      addLog('Location: ' + payload.lat.toFixed(5) + ',' + payload.lon.toFixed(5));
      speak('Help message sent. Sending location to your emergency contact.');
      // POST to server endpoint (you need to provide a backend)
      try {
        await fetch('/help', {
          method:'POST',
          headers:{'Content-Type':'application/json'},
          body: JSON.stringify(payload)
        });
        addLog('Help POSTed to server /help');
      } catch(e){
        addLog('Failed to post to /help: ' + e.message);
      }
    }, (err)=>{
      addLog('Geolocation error: ' + err.message);
      speak('Unable to get location.');
    }, { enableHighAccuracy: true, timeout: 8000 });
  }

  // ====== UI wiring ======
  startBtn.onclick = startCamera;
  stopBtn.onclick = stopCamera;

  // Start immediately if allowed
  window.addEventListener('load', async () => {
    statusEl.innerText = 'Initializing...';
    // Try to pre-load model to speed things up
    try {
      model = await cocoSsd.load();
      statusEl.innerText = 'Model preloaded. Press Start Camera.';
      addLog('Model loaded (preload).');
    } catch(e){
      statusEl.innerText = 'Model load failed.';
      addLog('Model load failed: ' + e.message);
    }
    setupVoiceCommands();
  });

  // Prevent screen from sleeping on some browsers (optional)
  if ('wakeLock' in navigator) {
    let wakelock = null;
    try {
      navigator.wakeLock.request('screen').then(l => { wakelock = l; addLog('WakeLock active'); }).catch(()=>{});
    } catch(e){}
  }

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
