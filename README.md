<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>VondeeWorld — Offline Jigsaw (3×5)</title>
<style>
  /* --- Brand / layout --- */
  :root{
    --brand-pink:#ff4fa2;
    --bg:#fff4f7;
    --paper:#fff;
    --ink:#0b0b0b;
    --muted:rgba(11,11,11,0.6);
    --edge:rgba(15,15,15,0.55);
    --max-width:940px;
  }
  *{box-sizing:border-box}
  html,body{height:100%}
  body{
    margin:12px;
    font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial;
    background:linear-gradient(180deg,var(--bg),#fff);
    color:var(--ink);
    -webkit-tap-highlight-color: transparent;
  }
  header{display:flex;justify-content:space-between;align-items:center;margin-bottom:10px}
  .brand{display:flex;gap:12px;align-items:center}
  .logo{width:56px;height:56px;border-radius:10px;background:linear-gradient(135deg,var(--brand-pink),#ff9acb);display:flex;align-items:center;justify-content:center;color:#fff;font-weight:700}
  .title{font-weight:700;font-size:18px}
  .slogan{font-size:13px;color:var(--muted);font-style:italic}
  .controls{display:flex;gap:8px;flex-wrap:wrap;align-items:center}
  .btn{background:var(--paper);border:1px solid rgba(0,0,0,0.06);padding:8px 10px;border-radius:8px;cursor:pointer}
  .btn:disabled{opacity:.5;cursor:not-allowed}
  .panel{background:var(--paper);padding:10px;border-radius:12px;box-shadow:0 8px 22px rgba(2,6,23,0.06)}
  .badges{display:flex;gap:8px;align-items:center}
  .badge{padding:6px 10px;border-radius:999px;font-weight:600;background:#fff}
  .main{display:flex;gap:12px;flex-wrap:wrap}
  .left{flex:1 1 640px;min-width:260px}
  .right{width:320px;min-width:220px}
  .board-wrap{width:100%;max-width:var(--max-width);margin:auto}
  /* board aspect ratio: landscape by default, portrait on small screens */
  .board{width:100%;background:var(--paper);border-radius:12px;overflow:hidden;position:relative;box-shadow:0 12px 30px rgba(2,6,23,0.06);display:grid}
  .board{aspect-ratio:16/9}
  @media (max-width:640px){ .board{aspect-ratio:9/16} }
  .slot{position:relative}
  /* piece container rules */
  .piece-svg{position:absolute;touch-action:none;cursor:grab;will-change:transform,left,top}
  .pool{display:flex;flex-wrap:wrap;gap:8px;margin-top:8px}
  .pool > div{display:flex;align-items:center;justify-content:center;padding:6px;border-radius:8px;background:#fff;box-shadow:0 6px 18px rgba(2,6,23,0.04)}
  .small{font-size:13px;color:var(--muted)}
  .overlayHint{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;z-index:999;background:rgba(0,0,0,0.45)}
  .fullImage{max-width:96%;max-height:96%;object-fit:contain;border-radius:8px;box-shadow:0 12px 36px rgba(2,6,23,0.35)}
  footer{margin-top:12px;font-size:13px;color:var(--muted)}
  /* snap animation (silent bounce) */
  .snap{ animation: snap-bounce 360ms cubic-bezier(.2,.9,.2,1); transform-origin:center center; }
  @keyframes snap-bounce {
    0% { transform: scale(0.92) translateY(-6px); }
    60% { transform: scale(1.06) translateY(4px); }
    100% { transform: scale(1) translateY(0); }
  }
  /* dragging visual */
  .dragging { opacity:.96; transform:scale(1.02); transition: transform .08s linear; z-index:999; }
</style>
</head>
<body>
  <header>
    <div class="brand">
      <div class="logo">VW</div>
      <div>
        <div class="title">VondeeWorld</div>
        <div class="slogan">confidence in elegance</div>
      </div>
    </div>

    <div class="controls">
      <div style="display:flex;gap:8px;align-items:center">
        <button id="startBtn" class="btn">Start</button>
        <button id="hintBtn" class="btn">Hint</button>
        <button id="retryBtn" class="btn">Retry</button>
      </div>
    </div>
  </header>

  <div class="controls panel" style="margin-bottom:10px;gap:10px">
    <label class="small">Image file: <input id="imgFile" type="file" accept="image/*"></label>
    <label class="small">or URL: <input id="imgUrl" type="url" placeholder="https://..." style="width:220px"></label>
    <button id="loadBtn" class="btn">Load</button>
    <div style="flex:1"></div>
    <div class="badges">
      <div class="badge" id="attemptsBadge">Attempts: 0 / 2</div>
      <div class="badge" id="timerBadge">Time: 60s</div>
    </div>
  </div>

  <div class="main">
    <div class="left">
      <div class="board-wrap panel">
        <div id="board" class="board"></div>
      </div>
    </div>

    <div class="right panel">
      <div class="small" style="font-weight:700;margin-bottom:8px">Pieces pool</div>
      <div id="pool" class="pool"></div>
      <div style="height:10px"></div>
      <div class="small" id="message">Load an image and press Start.</div>
    </div>
  </div>

  <footer>Offline, mobile-friendly. Save this file and open on your phone. If Safari/iOS asks for permission to access the file, allow it to run fully.</footer>

<script>
/* Offline universal jigsaw (pointer + touch) - 3x5, 2 attempts, 60s, SVG shapes
   - Works on modern Android & iOS browsers using pointer events or touch fallback
   - Save locally and open file directly (no external dependencies)
*/

/* ---------- Config ---------- */
const ROWS = 3, COLS = 5, TOTAL = ROWS*COLS;
const MAX_ATTEMPTS = 2, ATTEMPT_TIME = 60;

/* ---------- DOM ---------- */
const board = document.getElementById('board');
const pool = document.getElementById('pool');
const startBtn = document.getElementById('startBtn');
const hintBtn = document.getElementById('hintBtn');
const retryBtn = document.getElementById('retryBtn');
const loadBtn = document.getElementById('loadBtn');
const imgFile = document.getElementById('imgFile');
const imgUrl = document.getElementById('imgUrl');
const msgEl = document.getElementById('message');
const attemptsBadge = document.getElementById('attemptsBadge');
const timerBadge = document.getElementById('timerBadge');

/* ---------- State ---------- */
let imgSrc = '';
let attempt = 0;
let timeLeft = ATTEMPT_TIME;
let timerId = null;
let edgeMap = [];
let mapping = [];
let placedMap = {};       // slotIdx -> pieceId
let pieceMap = {};        // pieceId -> piece object {svg,origIndex,wrap}
let dragging = null;      // {pieceId, startX, startY, offsetX, offsetY, origLeft, origTop}
let poolOrder = [];

/* ---------- Helpers ---------- */
function show(msg){ msgEl.textContent = msg; }
function updateBadges(){ attemptsBadge.textContent = `Attempts: ${attempt} / ${MAX_ATTEMPTS}`; timerBadge.textContent = `Time: ${timeLeft}s`; }

/* ---------- Edge map (tabs/blanks) ---------- */
function generateEdgeMap(){
  edgeMap = new Array(TOTAL).fill(null).map(()=>({top:0,right:0,bottom:0,left:0}));
  for(let r=0;r<ROWS;r++){
    for(let c=0;c<COLS-1;c++){
      const idx = r*COLS + c;
      const t = Math.random()>0.5?1:-1;
      edgeMap[idx].right = t;
      edgeMap[idx+1].left = -t;
    }
  }
  for(let r=0;r<ROWS-1;r++){
    for(let c=0;c<COLS;c++){
      const idx = r*COLS + c;
      const t = Math.random()>0.5?1:-1;
      edgeMap[idx].bottom = t;
      edgeMap[idx+COLS].top = -t;
    }
  }
}

/* ---------- Geometry ---------- */
function getBoardGeometry(){
  const rect = board.getBoundingClientRect();
  const boardW = rect.width, boardH = rect.height;
  const pieceW = boardW / COLS, pieceH = boardH / ROWS;
  return {boardW, boardH, pieceW, pieceH};
}

/* ---------- SVG path generator (tabs/blanks) ---------- */
function makePiecePath(w,h,edges){
  const ts = Math.min(w,h) * 0.22;
  const td = Math.min(w,h) * 0.14;
  function horiz(x0,x1,y,dir){
    if(dir===0) return `L ${x1} ${y} `;
    const mid=(x0+x1)/2, left=mid-ts/2, right=mid+ts/2, depth=dir*td;
    return `L ${left} ${y} C ${left+ts*0.08} ${y} ${mid-ts*0.02} ${y+depth} ${mid} ${y+depth} C ${mid+ts*0.02} ${y+depth} ${right-ts*0.08} ${y} ${right} ${y} L ${x1} ${y} `;
  }
  function vert(y0,y1,x,dir){
    if(dir===0) return `L ${x} ${y1} `;
    const mid=(y0+y1)/2, top=mid-ts/2, bottom=mid+ts/2, depth=dir*td;
    return `L ${x} ${top} C ${x} ${top+ts*0.08} ${x+depth} ${mid-ts*0.02} ${x+depth} ${mid} C ${x+depth} ${mid+ts*0.02} ${x} ${bottom-ts*0.08} ${x} ${bottom} L ${x} ${y1} `;
  }
  function horizRtoL(x0,x1,y,dir){
    if(dir===0) return `L ${x1} ${y} `;
    const mid=(x0+x1)/2, left=mid-ts/2, right=mid+ts/2, depth=-dir*td;
    return `L ${right} ${y} C ${right-ts*0.08} ${y} ${mid+ts*0.02} ${y+depth} ${mid} ${y+depth} C ${mid-ts*0.02} ${y+depth} ${left+ts*0.08} ${y} ${left} ${y} L ${x1} ${y} `;
  }
  function vertBtoT(y0,y1,x,dir){
    if(dir===0) return `L ${x} ${y1} `;
    const mid=(y0+y1)/2, top=mid-ts/2, bottom=mid+ts/2, depth=-dir*td;
    return `L ${x} ${bottom} C ${x} ${bottom-ts*0.08} ${x+depth} ${mid+ts*0.02} ${x+depth} ${mid} C ${x+depth} ${mid-ts*0.02} ${x} ${top+ts*0.08} ${x} ${top} L ${x} ${y1} `;
  }

  let d = `M 0 0 `;
  d += horiz(0,w,0, edges.top);
  d += vert(0,h,w, edges.right);
  d += horizRtoL(w,0,h, edges.bottom);
  d += vertBtoT(h,0,0, edges.left);
  d += 'Z';
  return d;
}

/* ---------- Build pieces (SVG) ---------- */
function buildPieceSVG(r,c,boardW,boardH,pw,ph,edges,pid){
  const svgns = "http://www.w3.org/2000/svg";
  const svg = document.createElementNS(svgns,'svg');
  svg.setAttribute('width', pw); svg.setAttribute('height', ph);
  svg.setAttribute('viewBox', `0 0 ${pw} ${ph}`);
  svg.classList.add('piece-svg');
  svg.dataset.pieceId = pid;

  const clipId = `clip-${pid}`;
  const defs = document.createElementNS(svgns,'defs');
  const clip = document.createElementNS(svgns,'clipPath'); clip.setAttribute('id', clipId);
  const path = document.createElementNS(svgns,'path');
  const d = makePiecePath(pw, ph, edges);
  path.setAttribute('d', d); path.setAttribute('fill','white'); path.setAttribute('stroke','none');
  clip.appendChild(path); defs.appendChild(clip); svg.appendChild(defs);

  const image = document.createElementNS(svgns,'image');
  const xOff = -c * pw, yOff = -r * ph;
  image.setAttributeNS(null,'href', imgSrc);
  image.setAttribute('x', xOff); image.setAttribute('y', yOff);
  image.setAttribute('width', boardW); image.setAttribute('height', boardH);
  image.setAttribute('clip-path', `url(#${clipId})`);
  svg.appendChild(image);

  const outline = document.createElementNS(svgns,'path');
  outline.setAttribute('d', d);
  outline.setAttribute('fill','none');
  outline.setAttribute('stroke', 'rgba(15,15,15,0.55)');
  outline.setAttribute('stroke-width', Math.max(1, Math.min(pw,ph)*0.02));
  outline.setAttribute('vector-effect','non-scaling-stroke');
  svg.appendChild(outline);

  return svg;
}

/* ---------- Build board layout & pool ---------- */
function shuffle(arr){ for(let i=arr.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [arr[i],arr[j]]=[arr[j],arr[i]]; } return arr; }

function buildBoardAndPool(){
  board.innerHTML=''; pool.innerHTML='';
  placedMap = {}; pieceMap = {}; poolOrder = [];

  const geom = getBoardGeometry();
  const {boardW,boardH,pieceW,pieceH} = geom;
  board.style.position='relative';
  board.style.gridTemplateColumns = `repeat(${COLS},1fr)`;
  board.style.gridTemplateRows = `repeat(${ROWS},1fr)`;

  // create invisible slots with left/top coords
  for(let r=0;r<ROWS;r++){
    for(let c=0;c<COLS;c++){
      const s = document.createElement('div');
      s.className='slot';
      s.dataset.index = r*COLS + c;
      const left = c*pieceW, top = r*pieceH;
      s.dataset.left = left; s.dataset.top = top;
      s.style.width='100%'; s.style.height='100%';
      board.appendChild(s);
    }
  }

  // create randomized visual order
  mapping = Array.from({length:TOTAL}, (_,i)=>i);
  shuffle(mapping);

  // create piece SVGs and small wrappers in pool
  mapping.forEach((visualIndex,i) => {
    const r = Math.floor(visualIndex/COLS), c = visualIndex % COLS;
    const pid = `p-${visualIndex}-${Date.now()}-${i}`;
    const svg = buildPieceSVG(r,c,boardW,boardH,pieceW,pieceH,edgeMap[visualIndex], pid);
    // wrapper for pool appearance
    const wrap = document.createElement('div');
    wrap.style.width = Math.max(48, Math.min(120, pieceW*0.9)) + 'px';
    wrap.style.height = Math.max(36, Math.min(90, pieceH*0.9)) + 'px';
    wrap.appendChild(svg);
    pool.appendChild(wrap);

    // store piece info
    pieceMap[pid] = {svg, origIndex: visualIndex, wrap};
    poolOrder.push(pid);

    // pointer/touch events
    bindDragHandlers(svg, pid);
  });
}

/* ---------- Pointer/touch drag system (unified) ---------- */
function bindDragHandlers(svg, pid){
  // pointer events preferred
  svg.style.touchAction = 'none';
  svg.addEventListener('pointerdown', startPointerDrag);
  svg.addEventListener('touchstart', startTouchDrag, {passive:false});
}

function startPointerDrag(e){
  e.preventDefault();
  const svg = e.currentTarget;
  const pid = svg.dataset.pieceId;
  const rect = svg.getBoundingClientRect();
  // if svg currently in pool, record its initial wrapper position center for return
  const wrap = pieceMap[pid].wrap;
  const wrapRect = wrap.getBoundingClientRect();
  const origLeft = svg.style.left || '';
  const origTop = svg.style.top || '';
  const {x:clientX, y:clientY} = e;
  // compute offset between pointer and svg top-left
  const offsetX = clientX - rect.left, offsetY = clientY - rect.top;
  dragging = {pid, offsetX, offsetY, origParent: svg.parentElement, origLeft, origTop};
  // move svg to board (absolute) so it follows finger
  svg.classList.add('dragging');
  // take it out of wrapper and attach directly to board
  if(svg.parentElement !== board){
    // set absolute size based on full piece size
    const geom = getBoardGeometry();
    svg.style.width = geom.pieceW + 'px'; svg.style.height = geom.pieceH + 'px';
    svg.style.position = 'absolute';
    // place center roughly where finger is
    svg.style.left = (clientX - offsetX - board.getBoundingClientRect().left) + 'px';
    svg.style.top = (clientY - offsetY - board.getBoundingClientRect().top) + 'px';
    board.appendChild(svg);
  }
  // capture pointer moves
  svg.setPointerCapture(e.pointerId);
  svg.addEventListener('pointermove', onPointerMove);
  svg.addEventListener('pointerup', onPointerUp);
  svg.addEventListener('pointercancel', onPointerCancel);
}

function onPointerMove(e){
  if(!dragging) return;
  const svg = e.currentTarget;
  const {offsetX, offsetY} = dragging;
  const boardRect = board.getBoundingClientRect();
  const left = e.clientX - boardRect.left - offsetX;
  const top = e.clientY - boardRect.top - offsetY;
  svg.style.left = left + 'px';
  svg.style.top  = top + 'px';
}

function onPointerUp(e){
  if(!dragging) return;
  const svg = e.currentTarget;
  svg.classList.remove('dragging');
  svg.releasePointerCapture(e.pointerId);
  svg.removeEventListener('pointermove', onPointerMove);
  svg.removeEventListener('pointerup', onPointerUp);
  svg.removeEventListener('pointercancel', onPointerCancel);
  handleDrop(svg, e.clientX, e.clientY);
  dragging = null;
}

function onPointerCancel(e){
  if(!dragging) return;
  const svg = e.currentTarget;
  svg.classList.remove('dragging');
  svg.removeEventListener('pointermove', onPointerMove);
  svg.removeEventListener('pointerup', onPointerUp);
  svg.removeEventListener('pointercancel', onPointerCancel);
  returnToPool(svg);
  dragging = null;
}

/* Touch fallback (for older Safari that may not fully support pointer events) */
function startTouchDrag(e){
  e.preventDefault();
  const t = e.changedTouches[0];
  const svg = e.currentTarget;
  const pid = svg.dataset.pieceId;
  const rect = svg.getBoundingClientRect();
  const offsetX = t.clientX - rect.left, offsetY = t.clientY - rect.top;
  dragging = {pid, offsetX, offsetY, origParent: svg.parentElement, origLeft: svg.style.left || '', origTop: svg.style.top || ''};
  svg.classList.add('dragging');
  if(svg.parentElement !== board){
    const geom = getBoardGeometry();
    svg.style.width = geom.pieceW + 'px'; svg.style.height = geom.pieceH + 'px';
    svg.style.position = 'absolute';
    svg.style.left = (t.clientX - offsetX - board.getBoundingClientRect().left) + 'px';
    svg.style.top = (t.clientY - offsetY - board.getBoundingClientRect().top) + 'px';
    board.appendChild(svg);
  }
  // attach touchmove/up listeners on document (global)
  function touchMoveHandler(ev){
    ev.preventDefault();
    const touch = ev.changedTouches[0];
    const left = touch.clientX - board.getBoundingClientRect().left - offsetX;
    const top  = touch.clientY - board.getBoundingClientRect().top - offsetY;
    svg.style.left = left + 'px'; svg.style.top = top + 'px';
  }
  function touchEndHandler(ev){
    ev.preventDefault();
    const touch = ev.changedTouches[0];
    document.removeEventListener('touchmove', touchMoveHandler);
    document.removeEventListener('touchend', touchEndHandler);
    svg.classList.remove('dragging');
    handleDrop(svg, touch.clientX, touch.clientY);
    dragging = null;
  }
  document.addEventListener('touchmove', touchMoveHandler, {passive:false});
  document.addEventListener('touchend', touchEndHandler, {passive:false});
}

/* ---------- Drop logic & snapping ---------- */
function handleDrop(svg, clientX, clientY){
  const boardRect = board.getBoundingClientRect();
  // find element at pointer (ignoring the dragged svg itself)
  let el = document.elementFromPoint(clientX, clientY);
  // if element is inside board, try find nearest slot
  if(!el) { returnToPool(svg); return; }
  // find nearest slot by distance to its center
  const slots = Array.from(board.querySelectorAll('.slot'));
  let nearest = null, bestDist = Infinity;
  const pointerX = clientX - boardRect.left, pointerY = clientY - boardRect.top;
  slots.forEach(s=>{
    const left = parseFloat(s.dataset.left), top = parseFloat(s.dataset.top);
    const centerX = left + (getBoardGeometry().pieceW/2), centerY = top + (getBoardGeometry().pieceH/2);
    const d = Math.hypot(centerX - pointerX, centerY - pointerY);
    if(d < bestDist){ bestDist = d; nearest = s; }
  });
  // if nearest within threshold (one piece diagonal), snap; else return to pool
  const threshold = Math.max(getBoardGeometry().pieceW, getBoardGeometry().pieceH) * 0.6;
  if(nearest && bestDist <= threshold){
    const slotIdx = parseInt(nearest.dataset.index,10);
    // if slot occupied, return existing piece to pool
    if(placedMap[slotIdx]){
      const oldId = placedMap[slotIdx];
      const old = pieceMap[oldId];
      if(old){
        returnPieceToPool(old);
      }
      delete placedMap[slotIdx];
    }
    // snap svg to slot position
    const left = parseFloat(nearest.dataset.left), top = parseFloat(nearest.dataset.top);
    svg.style.left = left + 'px'; svg.style.top = top + 'px';
    svg.style.position = 'absolute';
    board.appendChild(svg);
    const pid = svg.dataset.pieceId;
    placedMap[slotIdx] = pid;
    // mark as placed (disable further dragging)
    svg.setAttribute('data-placed','true');
    svg.setAttribute('draggable','false');
    // if correct piece original index matches slot -> animate snap
    const orig = Number(svg.dataset.origIndex);
    if(orig === slotIdx){
      svg.classList.add('snap');
      setTimeout(()=> svg.classList.remove('snap'), 420);
    }
    checkSolved();
    return;
  } else {
    // not close enough -> return to pool
    returnToPool(svg);
  }
}

function returnToPool(svg){
  const pid = svg.dataset.pieceId;
  const obj = pieceMap[pid];
  if(!obj) return;
  // reattach to its wrapper
  const wrap = obj.wrap;
  wrap.appendChild(svg);
  svg.style.position=''; svg.style.left=''; svg.style.top=''; svg.style.width=''; svg.style.height='';
  svg.removeAttribute('data-placed');
  svg.setAttribute('draggable','true');
}

/* when removing a placed piece: return wrapper with svg */
function returnPieceToPool(obj){
  const {svg,wrap} = obj;
  if(svg.parentElement === board){
    wrap.appendChild(svg);
    svg.style.position=''; svg.style.left=''; svg.style.top=''; svg.style.width=''; svg.style.height='';
    svg.removeAttribute('data-placed');
    svg.setAttribute('draggable','true');
  }
}

/* ---------- Game logic: start, timer, attempts ---------- */
function startTimer(){
  clearInterval(timerId);
  timeLeft = ATTEMPT_TIME; updateBadges();
  timerId = setInterval(()=>{
    timeLeft--; updateBadges();
    if(timeLeft<=0){ clearInterval(timerId); onTimeout(); }
  },1000);
}
function onTimeout(){
  show('Time up for this attempt.');
  attempt++;
  updateBadges();
  if(attempt < MAX_ATTEMPTS){
    show(`Attempt ${attempt} ended. Preparing next attempt...`);
    setTimeout(()=> prepareNewAttempt(), 900);
  } else {
    lockGame();
  }
}
function prepareNewAttempt(){
  // clear placed pieces back to pool
  Object.keys(pieceMap).forEach(pid=>{
    const obj = pieceMap[pid];
    returnPieceToPool(obj);
  });
  placedMap = {};
  createMappingAndShuffle();
  buildBoardAndPool();
  startTimer();
  hintBtn.disabled = attempt >= MAX_ATTEMPTS;
  retryBtn.disabled = attempt >= MAX_ATTEMPTS;
  show(`Attempt ${attempt+1} started.`);
}

function checkSolved(){
  for(let i=0;i<TOTAL;i++){
    if(!placedMap[i]) return false;
    const pid = placedMap[i];
    const p = pieceMap[pid];
    if(!p) return false;
    if(Number(p.origIndex) !== i) return false;
  }
  // solved
  clearInterval(timerId);
  show(`Solved on attempt ${attempt}! ${timeLeft}s remaining.`);
  hintBtn.disabled = true; retryBtn.disabled = true; startBtn.disabled = false;
  // lock drag
  Object.values(pieceMap).forEach(p=> p.svg.setAttribute('draggable','false'));
  return true;
}

function lockGame(){
  clearInterval(timerId);
  show('No attempts left — game locked.');
  hintBtn.disabled = true; retryBtn.disabled = true; startBtn.disabled = false;
  Object.values(pieceMap).forEach(p=> p.svg.setAttribute('draggable','false'));
}

function createMappingAndShuffle(){
  mapping = Array.from({length:TOTAL}, (_,i)=>i);
  shuffle(mapping);
}

/* ---------- Build sequence for fresh game ---------- */
function startGame(){
  if(!imgSrc) { alert('Load an image first (file or URL).'); return; }
  generateEdgeMap();
  attempt = 1;
  updateBadges();
  createMappingAndShuffle();
  buildBoardAndPool();
  startTimer();
  hintBtn.disabled = false; retryBtn.disabled = false;
  show('Game started. Good luck!');
  startBtn.disabled = true;
}

function retryAttempt(){
  if(attempt >= MAX_ATTEMPTS) return;
  clearInterval(timerId);
  // restart same attempt (do not increment attempt)
  clearPlacedToPool();
  createMappingAndShuffle();
  buildBoardAndPool();
  timeLeft = ATTEMPT_TIME; updateBadges();
  startTimer();
  show(`Attempt ${attempt} restarted.`);
}

function clearPlacedToPool(){
  Object.keys(placedMap).forEach(idx=>{
    const pid = placedMap[idx];
    const obj = pieceMap[pid];
    if(obj) returnPieceToPool(obj);
  });
  placedMap = {};
}

/* ---------- Hint (2s) ---------- */
function showHint(){
  if(attempt >= MAX_ATTEMPTS) return;
  const overlay = document.createElement('div'); overlay.className='overlayHint';
  const img = document.createElement('img'); img.className='fullImage'; img.src = imgSrc;
  overlay.appendChild(img); document.body.appendChild(overlay);
  setTimeout(()=> overlay.remove(), 2000);
}

/* ---------- Load image handling ---------- */
loadBtn.addEventListener('click', ()=>{
  if(imgFile.files && imgFile.files[0]){
    const r = new FileReader();
    r.onload = e => { imgSrc = e.target.result; show('Loaded image file. Press Start.'); };
    r.readAsDataURL(imgFile.files[0]);
  } else if(imgUrl.value.trim()){
    imgSrc = imgUrl.value.trim();
    const t = new Image();
    t.onload = ()=> show('Loaded image URL. Press Start.');
    t.onerror = ()=> show('Failed to load image URL.');
    t.src = imgSrc;
  } else {
    alert('Choose a file or enter a URL.');
    return;
  }
  attempt = 0; timeLeft = ATTEMPT_TIME; updateBadges();
  hintBtn.disabled = true; retryBtn.disabled = true; startBtn.disabled = false;
});

/* ---------- Wire UI ---------- */
startBtn.addEventListener('click', startGame);
hintBtn.addEventListener('click', showHint);
retryBtn.addEventListener('click', retryAttempt);

/* ---------- Resize handling: rebuild pieces to match pixel sizes ---------- */
let resizeTimer = null;
window.addEventListener('resize', ()=>{
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(()=>{
    if(!imgSrc) return;
    // preserve placed pieces origIndex if any
    const prevPlaced = {...placedMap};
    if(attempt>0 && attempt <= MAX_ATTEMPTS){
      buildBoardAndPool();
      // re-place any previously placed pieces where possible (by matching origIndex)
      Object.entries(prevPlaced).forEach(([slotIdx, pid])=>{
        const obj = pieceMap[pid];
        if(obj){
          // find slot element
          const slot = board.querySelector(`.slot[data-index='${slotIdx}']`);
          if(slot){
            const left = parseFloat(slot.dataset.left), top = parseFloat(slot.dataset.top);
            obj.svg.style.position='absolute'; obj.svg.style.left = left + 'px'; obj.svg.style.top = top + 'px';
            board.appendChild(obj.svg);
            placedMap[slotIdx] = pid;
            obj.svg.setAttribute('draggable','false');
          }
        }
      });
    } else {
      buildBoardAndPool();
    }
  }, 180);
});

/* ---------- Initialization ---------- */
(function init(){
  show('Load an image and press Start.');
  updateBadges();
  hintBtn.disabled = true; retryBtn.disabled = true;
  // some mobile browsers need a small delay before accurate getBoundingClientRect - reflow safely
  setTimeout(()=>{}, 50);
})();
</script>
</body>
</html>
