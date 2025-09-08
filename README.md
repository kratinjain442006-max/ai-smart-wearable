# ai-smart-wearable
AI Smart Wearable — audio + directional haptic prototype
# AI Smart Wearable — Hear & Feel

**Prototype** that provides audio + direction-aware haptic alerts for visually impaired users.

<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1.0" />
<title>Hear & Feel — Prototype (Improved)</title>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap">
<style>
  :root{
    --bg:#071023; --card:#0f1724; --muted:#9aa7b2;
    --accent:#2563eb; --success:#10b981;
  }
  *{box-sizing:border-box;font-family:Inter,system-ui,Segoe UI,Roboto,Arial;}
  body{margin:0;background:linear-gradient(180deg,#05060a, #071023 40%); color:#e6eef6; min-height:100vh; padding:18px;}
  header{display:flex;align-items:center;gap:16px;margin-bottom:18px}
  h1{margin:0;font-size:20px}
  .wrapper{display:grid;grid-template-columns: 1fr 360px; gap:18px; align-items:start;}
  .viewer{background:var(--card); padding:12px; border-radius:12px; box-shadow:0 8px 30px rgba(2,6,23,0.6);}
  #videoWrap{position:relative; border-radius:10px; overflow:hidden; background:#000; width:100%; aspect-ratio:4/3;}
  video{width:100%; height:100%; object-fit:cover; transform:scaleX(-1);} /* mirrored */
  canvas{position:absolute; left:0; top:0; width:100%; height:100%; pointer-events:none;}
  .controls{display:flex; gap:8px; margin-top:10px; flex-wrap:wrap;}
  button{background:var(--accent); color:white; border:0; padding:10px 12px; border-radius:8px; font-weight:600; cursor:pointer;}
  button.ghost{background:transparent; border:1px solid rgba(255,255,255,0.06);}
  .panel{background:rgba(255,255,255,0.02); padding:12px; border-radius:10px;}
  .side{display:flex; flex-direction:column; gap:12px;}
  .toggles label{display:flex; align-items:center; gap:8px; font-size:14px; color:var(--muted)}
  .log{height:160px; overflow:auto; font-size:13px; background:rgba(0,0,0,0.15); padding:8px; border-radius:8px;}
  .summary{display:flex; gap:8px; flex-wrap:wrap;}
  .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01)); padding:10px; border-radius:8px; min-width:120px;}
  .card h3{margin:0;font-size:13px}
  .card p{margin:6px 0 0 0;color:var(--muted);font-size:13px}
  .big-help{background:#ef4444; color:white; border-radius:10px; padding:12px; text-align:center; font-weight:700; cursor:pointer;}
  footer{margin-top:18px;color:var(--muted); font-size:13px}
  .status{font-size:13px;color:var(--muted)}
  .confidence{font-weight:700}
  .danger-high{color:#ff6b6b}
  .danger-med{color:#f59e0b}
  .danger-low{color:#94a3b8}
  .controls .range{width:180px}
  @media(max-width:900px){
    .wrapper{grid-template-columns:1fr; }
    .side{order:2}
  }
</style>
</head>
<body>
  <header>
    <h1>Hear & Feel — Hackathon Prototype</h1>
    <div class="status">Dual-sense alerts: audio + vibration • Voice: "What's around me" / "Help me"</div>
  </header>

  <div class="wrapper">
    <div class="viewer panel">
      <div id="videoWrap">
        <video id="video" autoplay playsinline muted></video>
        <canvas id="overlay"></canvas>
      </div>

      <div class="controls">
        <button id="startBtn">Start Camera</button>
        <button id="stopBtn" class="ghost">Stop</button>
        <button id="summaryBtn" class="ghost">Read Summary</button>
        <div style="flex:1"></div>
        <label style="display:flex; align-items:center; gap:8px">
          Sensitivity
          <input id="sensitivity" class="range" type="range" min="0.2" max="0.9" step="0.05" value="0.5">
        </label>
      </div>

      <div style="height:12px"></div>

      <div class="summary" id="summaryArea"></div>
      <div style="height:8px"></div>
      <div class="log" id="log"></div>
    </div>

    <div class="side">
      <div class="panel">
        <div style="display:flex; justify-content:space-between; align-items:center;">
          <div><strong>Controls & Feedback</strong></div>
          <div class="status" id="modelStatus">Model: loading...</div>
        </div>

        <div style="height:8px"></div>
        <div class="toggles">
          <label><input id="toggleDetect" type="checkbox" checked> Detection enabled</label>
          <label><input id="toggleAudio" type="checkbox" checked> Audio enabled</label>
          <label><input id="toggleVibe" type="checkbox" checked> Vibration enabled</label>
        </div>

        <div style="height:10px"></div>

        <div style="display:flex; gap:8px;">
          <button id="helpBtn" style="flex:1" class="big-help">EMERGENCY: HELP ME</button>
        </div>

        <div style="height:10px"></div>

        <div>
          <div style="font-weight:700;margin-bottom:6px">Voice / Commands</div>
          <div style="color:var(--muted); font-size:13px">
            Say <em>"What's around me"</em> to get a verbal summary. Say <em>"Help me"</em> to send location & alert.
          </div>
        </div>
      </div>

      <div class="panel" style="display:flex; flex-direction:column; gap:8px;">
        <div style="font-weight:700">Detection Settings</div>
        <div style="display:flex; gap:8px; flex-direction:column;">
          <label style="font-size:13px">Min confidence: <span id="minScoreLabel">50%</span></label>
          <input id="minScore" type="range" min="0.2" max="0.9" step="0.01" value="0.5">
          <div style="font-size:13px; color:var(--muted)">Vibration patterns indicate left/right/center + danger level.</div>
        </div>
      </div>
    </div>
  </div>

<footer>
  Tip: for left/right haptic on a wristband you’ll need a BLE wearable. This demo uses phone vibration as a prototype.
</footer>

<!-- TF.js & COCO-SSD -->
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.13.0/dist/tf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd@2.2.2/dist/coco-ssd.min.js"></script>

<script>
/*
 Improved single-file prototype:
 - Multi-object detection (using coco-ssd).
 - Summarization and voice commands.
 - Better UI + confidence controls + improved dedupe + distance heuristic.
 Notes: coco-ssd is easy to run in browser; accuracy for stairs may be limited.
*/

const video = document.getElementById('video');
const overlay = document.getElementById('overlay');
const ctx = overlay.getContext('2d');
const startBtn = document.getElementById('startBtn');
const stopBtn = document.getElementById('stopBtn');
const helpBtn = document.getElementById('helpBtn');
const summaryBtn = document.getElementById('summaryBtn');
const logEl = document.getElementById('log');
const summaryArea = document.getElementById('summaryArea');
const minScoreEl = document.getElementById('minScore');
const minScoreLabel = document.getElementById('minScoreLabel');
const sensitivity = document.getElementById('sensitivity');

const toggleAudio = document.getElementById('toggleAudio');
const toggleVibe = document.getElementById('toggleVibe');
const toggleDetect = document.getElementById('toggleDetect');
const modelStatus = document.getElementById('modelStatus');

let model = null;
let stream = null;
let raf = null;
let lastAnnounceT = 0;
let lastAnnouncedKeys = new Map(); // key -> timestamp
const REPEAT_MS = 1400;

const STATE = {
  minScore: parseFloat(minScoreEl.value),
  sensitivity: parseFloat(sensitivity.value)
};

minScoreEl.addEventListener('input', () => {
  STATE.minScore = parseFloat(minScoreEl.value);
  minScoreLabel.innerText = Math.round(STATE.minScore * 100) + '%';
});

sensitivity.addEventListener('input', () => {
  STATE.sensitivity = parseFloat(sensitivity.value);
});

function addLog(txt){
  const t = new Date().toLocaleTimeString();
  logEl.innerHTML = `<div>[${t}] ${txt}</div>` + logEl.innerHTML;
}

// Speak helper
function speak(text, urgent=false){
  if(!toggleAudio.checked) return;
  try{
    window.speechSynthesis.cancel(); // avoid overlapping
    const u = new SpeechSynthesisUtterance(text);
    u.rate = urgent ? 1.05 : 0.95;
    u.pitch = urgent ? 1.1 : 1.0;
    u.lang = 'en-US';
    window.speechSynthesis.speak(u);
  }catch(e){ console.warn(e); }
}

// Vibration helper: different patterns for left/right/center & danger
function vibrate(direction, level){
  if(!toggleVibe.checked) return;
  if(!navigator.vibrate) return;
  // direction: left/right/center
  // level: 'low'|'medium'|'high'
  let base;
  if(level === 'high') base = [250,80,250,80,200];
  else if(level === 'medium') base = [160,60,160];
  else base = [80];
  // prefix to hint direction
  if(direction === 'left') base = [60,30].concat(base);
  if(direction === 'right') base = [30,60].concat(base);
  navigator.vibrate(base);
}

// Map bbox -> direction relative to center
function bboxDirection(bbox, vw){
  const cx = bbox[0] + bbox[2]/2;
  const rel = (cx / vw) - 0.5;
  if(rel < -0.22) return 'left';
  if(rel > 0.22) return 'right';
  return 'center';
}

// Map to danger using class & area heuristics
function mapDanger(obj, vw, vh){
  const area = (obj.bbox[2]*obj.bbox[3]) / (vw*vh);
  const cls = obj.class;
  // classes we deem inherently higher risk
  if(['car','truck','bus','motorbike','bicycle'].includes(cls)) return 'high';
  if(['person'].includes(cls) && area > 0.08) return 'medium';
  if(area > 0.12) return 'high';
  if(area > 0.04) return 'medium';
  return 'low';
}

// Create stable key
function detKey(obj){
  return `${obj.class}|${Math.round(obj.bbox[0])}|${Math.round(obj.bbox[1])}|${Math.round(obj.bbox[2])}`;
}

// Draw boxes & labels
function draw(dets){
  overlay.width = video.videoWidth;
  overlay.height = video.videoHeight;
  ctx.clearRect(0,0,overlay.width,overlay.height);
  ctx.lineWidth = 3;
  ctx.font = "14px Inter, system-ui";
  dets.forEach(d=>{
    const [x,y,w,h] = d.bbox;
    ctx.fillStyle = "rgba(0,140,120,0.12)";
    ctx.fillRect(x,y,w,h);
    ctx.strokeStyle = "rgba(0,200,160,0.9)";
    ctx.strokeRect(x,y,w,h);
    const text = `${d.class} ${(d.score*100|0)}%`;
    ctx.fillStyle = '#ffffff';
    ctx.fillText(text, x+6, y+16);
  });
  // center guide
  ctx.strokeStyle = 'rgba(255,255,255,0.08)';
  ctx.beginPath();
  ctx.moveTo(overlay.width/2,0);
  ctx.lineTo(overlay.width/2,overlay.height);
  ctx.stroke();
}

// Produce a human summary from top detections
function produceSummary(dets, vw, vh){
  if(!dets.length) return 'No important objects detected nearby.';
  // Group by direction & class; pick top by danger->area->score
  const buckets = {left:[], center:[], right:[]};
  dets.forEach(d=>{
    const dir = bboxDirection(d.bbox, vw);
    buckets[dir].push(d);
  });
  const parts = [];
  ['left','center','right'].forEach(dir=>{
    if(buckets[dir].length){
      // sort by danger+area
      buckets[dir].sort((a,b)=>{
        const da = mapDanger(a,vw,vh), db = mapDanger(b,vw,vh);
        const scoreMap = {low:0, medium:1, high:2};
        if(scoreMap[da] !== scoreMap[db]) return scoreMap[db] - scoreMap[da];
        const areaA = a.bbox[2]*a.bbox[3], areaB = b.bbox[2]*b.bbox[3];
        if(areaA !== areaB) return areaB - areaA;
        return b.score - a.score;
      });
      const top = buckets[dir][0];
      parts.push(`${top.class} ${dir === 'center' ? 'ahead' : 'to the ' + dir}`);
    }
  });
  return parts.join(', ') + '.';
}

// Main detection loop
async function detectLoop(){
  if(!model || !toggleDetect.checked) { raf = requestAnimationFrame(detectLoop); return; }
  if(video.readyState < 2){ raf = requestAnimationFrame(detectLoop); return; }

  try{
    const predictions = await model.detect(video);
    const vw = video.videoWidth, vh = video.videoHeight;
    const filtered = predictions.filter(p => p.score >= STATE.minScore);

    draw(filtered);

    // Update summary cards UI
    updateSummaryCards(filtered, vw, vh);

    // Announce top immediate danger if any
    if(filtered.length){
      // choose top by danger->area->score
      filtered.sort((a,b)=>{
        const ia = mapDanger(a,vw,vh), ib = mapDanger(b,vw,vh);
        const map = {low:0, medium:1, high:2};
        if(map[ia] !== map[ib]) return map[ib]-map[ia];
        const areaA = a.bbox[2]*a.bbox[3], areaB = b.bbox[2]*b.bbox[3];
        if(areaA !== areaB) return areaB-areaA;
        return b.score - a.score;
      });
      const top = filtered[0];
      const key = detKey(top);
      const now = Date.now();
      const lastT = lastAnnouncedKeys.get(key) || 0;
      // Only announce if new or enough time passed
      if(now - lastT > REPEAT_MS){
        lastAnnouncedKeys.set(key, now);
        const dir = bboxDirection(top.bbox, vw);
        const danger = mapDanger(top, vw, vh);
        const msg = `${top.class} ${dir === 'center' ? 'ahead' : (dir + ' side')}`;
        addLog(`${top.class} ${(top.score*100|0)}% ${dir} (level: ${danger})`);
        speak(msg, danger === 'high');
        vibrate(dir, danger);
      }
    }
  }catch(e){
    console.error('Detect error', e);
    addLog('Detection error: ' + (e.message||e));
  } finally {
    raf = requestAnimationFrame(detectLoop);
  }
}

// Update summary cards area
function updateSummaryCards(dets, vw, vh){
  // create map: class -> best det
  const map = new Map();
  dets.forEach(d=>{
    const existing = map.get(d.class);
    const area = d.bbox[2]*d.bbox[3];
    if(!existing || area > (existing.bbox[2]*existing.bbox[3])) map.set(d.class, d);
  });

  summaryArea.innerHTML = '';
  if(map.size === 0){
    summaryArea.innerHTML = `<div class="card"><h3>No objects</h3><p class="muted">Nothing detected</p></div>`;
    return;
  }
  for(const [cls, obj] of map){
    const dir = bboxDirection(obj.bbox, vw);
    const danger = mapDanger(obj, vw, vh);
    const conf = Math.round(obj.score*100);
    const area = Math.round((obj.bbox[2]*obj.bbox[3]) / (vw*vh) * 100);
    const div = document.createElement('div');
    div.className = 'card';
    div.innerHTML = `<h3>${cls} <span style="float:right" class="confidence">${conf}%</span></h3>
      <p>${dir === 'center' ? 'Ahead' : dir}</p>
      <p class="${danger === 'high' ? 'danger-high' : danger === 'medium' ? 'danger-med' : 'danger-low'}">Danger: ${danger} • size: ${area}%</p>`;
    summaryArea.appendChild(div);
  }
}

// Start / Stop camera
async function startCamera(){
  if(stream) return;
  try{
    stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" }, audio: false });
    video.srcObject = stream;
    await video.play();
    overlay.width = video.videoWidth;
    overlay.height = video.videoHeight;
    addLog('Camera started');
    if(!model){
      modelStatus.innerText = 'Loading model...';
      model = await cocoSsd.load();
      modelStatus.innerText = 'Model loaded';
      addLog('Model loaded (coco-ssd).');
    } else {
      modelStatus.innerText = 'Model ready';
    }
    // kickstart loop
    if(!raf) detectLoop();
  }catch(e){
    console.error(e);
    alert('Camera unavailable. Ensure permissions, and try Chrome on Android or Safari on iOS (HTTPS).');
  }
}

function stopCamera(){
  if(stream){
    stream.getTracks().forEach(t => t.stop());
    stream = null;
    video.srcObject = null;
    addLog('Camera stopped');
  }
  if(raf) cancelAnimationFrame(raf);
  raf = null;
  ctx.clearRect(0,0,overlay.width,overlay.height);
}

// Voice commands: continuous recognition for "help me" or "what's around me"
function setupVoiceCommands(){
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if(!SpeechRecognition){
    addLog('Voice recognition unsupported in this browser.');
    return;
  }
  const recog = new SpeechRecognition();
  recog.continuous = true;
  recog.interimResults = false;
  recog.lang = 'en-US';
  recog.onresult = (ev) => {
    const phrase = ev.results[ev.results.length-1][0].transcript.trim().toLowerCase();
    addLog('[VOICE] ' + phrase);
    if(phrase.includes('help me') || phrase.includes('help')){
      triggerHelp();
    } else if(phrase.includes("what's around me") || phrase.includes('what is around me') || phrase.includes('what around me')){
      // read current visible summary
      const s = produceSummary(/* we need last predictions: */ lastPredictions || [], video.videoWidth, video.videoHeight);
      speak(s, false);
      addLog('Voice summary read');
    }
  };
  recog.onerror = (e) => addLog('Voice error: ' + e.error);
  recog.onend = () => { try{ recog.start(); } catch(e){} };
  try{ recog.start(); addLog('Voice listening (continuous) started'); } catch(e){ addLog('Voice start failed'); }
}

// Keep a copy of last predictions for summary command
let lastPredictions = [];

// small wrapper to intercept model.detect for storing predictions
(function interceptDetect(){
  const orig = (window.cocoSsd && window.cocoSsd.load) ? null : null;
  // We'll capture predictions inside detectLoop directly by assigning lastPredictions
})();

// Hook summaryBtn to read a summary
summaryBtn.addEventListener('click', () => {
  const s = produceSummary(lastPredictions || [], video.videoWidth, video.videoHeight);
  speak(s, false);
  addLog('Manual summary read');
});

// Emergency help: get geolocation and POST to /help (server must exist)
async function triggerHelp(){
  addLog('Help triggered — getting location...');
  if(!navigator.geolocation){
    addLog('Geolocation not supported');
    speak('Unable to get location');
    return;
  }
  navigator.geolocation.getCurrentPosition(async pos => {
    const payload = {
      lat: pos.coords.latitude,
      lon: pos.coords.longitude,
      accuracy: pos.coords.accuracy,
      ts: new Date().toISOString()
    };
    addLog('Location: ' + payload.lat.toFixed(6) + ',' + payload.lon.toFixed(6));
    speak('Help message sent to emergency contact', true);
    // POST to server endpoint (you need a backend to forward SMS/email)
    try{
      await fetch('/help', {
        method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(payload)
      });
      addLog('Help posted to server /help (if available)');
    }catch(e){
      addLog('Failed to post help to server: ' + e.message);
    }
    // Add an SOS vibration pattern and audio
    for(let i=0;i<2;i++){ navigator.vibrate([300,100,300]); await new Promise(r=>setTimeout(r,400)); }
  }, err => {
    addLog('Geolocation error: ' + err.message);
    speak('Unable to get location', true);
  }, {enableHighAccuracy:true, timeout:8000});
}

helpBtn.addEventListener('click', triggerHelp);

// Wire start/stop + detection toggles
startBtn.onclick = startCamera;
stopBtn.onclick = stopCamera;

// When model detection runs we also update lastPredictions
// We'll wrap model.detect by replacing detectLoop's call to set lastPredictions
// So modify detectLoop above: fetch predictions into lastPredictions
// To do that, override model.detect after model loads is tricky; instead update lastPredictions inside detectLoop by assignment.
// Modify detectLoop to set lastPredictions; we already have that variable referenced earlier.
// Let's re-define detectLoop to include lastPredictions assignment (so overwrite old function)
async function detectLoop(){
  if(!model || !toggleDetect.checked) { raf = requestAnimationFrame(detectLoop); return; }
  if(video.readyState < 2){ raf = requestAnimationFrame(detectLoop); return; }

  try{
    const predictions = await model.detect(video);
    lastPredictions = predictions; // store for summary voice command
    const vw = video.videoWidth, vh = video.videoHeight;
    const filtered = predictions.filter(p => p.score >= STATE.minScore);

    draw(filtered);
    updateSummaryCards(filtered, vw, vh);

    if(filtered.length){
      filtered.sort((a,b)=>{
        const ia = mapDanger(a,vw,vh), ib = mapDanger(b,vw,vh);
        const map = {low:0, medium:1, high:2};
        if(map[ia] !== map[ib]) return map[ib]-map[ia];
        const areaA = a.bbox[2]*a.bbox[3], areaB = b.bbox[2]*b.bbox[3];
        if(areaA !== areaB) return areaB-areaA;
        return b.score - a.score;
      });
      const top = filtered[0];
      const key = detKey(top);
      const now = Date.now();
      const lastT = lastAnnouncedKeys.get(key) || 0;
      if(now - lastT > REPEAT_MS){
        lastAnnouncedKeys.set(key, now);
        const dir = bboxDirection(top.bbox, vw);
        const danger = mapDanger(top, vw, vh);
        const msg = `${top.class} ${dir === 'center' ? 'ahead' : (dir + ' side')}`;
        addLog(`${top.class} ${(top.score*100|0)}% ${dir} (level: ${danger})`);
        speak(msg, danger === 'high');
        vibrate(dir, danger);
      }
    }
  }catch(e){
    console.error('Detect error', e);
    addLog('Detection error: ' + (e.message||e));
  } finally {
    raf = requestAnimationFrame(detectLoop);
  }
}

// Initialize: preload model and voice commands
(async function init(){
  modelStatus.innerText = 'Loading model...';
  try{
    model = await cocoSsd.load();
    modelStatus.innerText = 'Model loaded';
    addLog('Model loaded (coco-ssd).');
  }catch(e){
    modelStatus.innerText = 'Model failed';
    addLog('Model failed to load: ' + e.message);
  }
  setupVoiceCommands();
})();

// Keep UI reactive to toggles
minScoreEl.addEventListener('change', (e)=> STATE.minScore = parseFloat(e.target.value));
toggleDetect.addEventListener('change', ()=> {
  if(!toggleDetect.checked) addLog('Detection paused by user');
});
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
