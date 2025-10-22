# Radek-website
<!doctype html>
<html lang="cs">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>MiniCraft — 2D blokový sandbox</title>
<meta name="description" content="Jednoduchý 2D Minecraft-ish sandbox: levý klik přidá blok, pravý klik smaže. HTML/CSS/JS single-file."/>
<style>
  :root{
    --bg:#7ec0ee; /* sky */
    --panel:#0f1724;
    --ui:#ffffffcc;
    --muted:#09304a;
    --accent:#ffd166;
    font-family: Inter, system-ui, -apple-system, 'Segoe UI', Roboto, Arial;
  }
  html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg),#cfeef9);display:flex;align-items:center;justify-content:center;padding:20px}
  .wrap{width:1000px;max-width:98vw;display:grid;grid-template-columns:720px 1fr;gap:18px}
  .boardCard{background:linear-gradient(180deg,rgba(255,255,255,0.03),rgba(255,255,255,0.01));padding:12px;border-radius:12px;box-shadow:0 12px 34px rgba(2,6,23,0.25)}
  canvas{display:block;width:100%;height:560px;border-radius:8px;background:linear-gradient(180deg,#9ad6ff,#7ec0ee);cursor:crosshair}
  h1{margin:6px 0;font-size:20px;color:#04293a}
  .sidebar{background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));padding:12px;border-radius:12px}
  .palette{display:flex;flex-wrap:wrap;gap:8px}
  .block-btn{width:84px;height:56px;border-radius:8px;border:2px solid rgba(0,0,0,0.06);display:flex;flex-direction:column;align-items:center;justify-content:center;cursor:pointer;user-select:none}
  .block-btn.active{outline:3px solid rgba(255,209,102,0.9)}
  .controls{display:flex;gap:8px;margin-top:12px}
  .btn{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;font-weight:700;cursor:pointer}
  .btn.ghost{background:transparent;border:1px solid rgba(0,0,0,0.06)}
  .hint{color:var(--muted);font-size:13px;margin-top:6px}
  .small{font-size:13px;color:var(--muted)}
  footer{margin-top:10px;color:var(--muted)}
  @media(max-width:920px){.wrap{grid-template-columns:1fr}canvas{height:420px}}
</style>
</head>
<body>
  <div class="wrap">
    <div class="boardCard">
      <h1>MiniCraft — 2D blokový sandbox</h1>
      <p class="small">Levý klik = přidat blok. Pravý klik = smazat blok. Klávesy 1–5 pro výběr bloku. Ukládej/Načti svět.</p>

      <canvas id="world" width="720" height="560" aria-label="2D blokový svět"></canvas>

      <div class="controls">
        <button id="saveBtn" class="btn">Uložit</button>
        <button id="loadBtn" class="btn ghost">Načíst</button>
        <button id="clearBtn" class="btn ghost">Vyčistit</button>
        <div style="margin-left:auto" class="small">Soubor: <span id="worldName">svet1</span></div>
      </div>

      <div class="hint">Tip: stiskni a drž mezerník a táhni myší pro rychlé kácení/kladění.</div>
      <footer>Stačí nahrát `index.html` na GitHub Pages.</footer>
    </div>

    <aside class="sidebar">
      <div style="font-weight:700;margin-bottom:8px">Paleta bloků</div>
      <div class="palette" id="palette"></div>

      <div style="height:1px;background:rgba(0,0,0,0.06);margin:10px 0"></div>
      <div style="font-weight:700;margin-bottom:8px">Nástroje</div>
      <div style="display:flex;gap:8px;margin-bottom:8px">
        <button id="eraserBtn" class="btn ghost">Eraser (pravý klik)</button>
        <button id="fillBtn" class="btn ghost">Vyplnit</button>
      </div>

      <div style="height:1px;background:rgba(0,0,0,0.06);margin:10px 0"></div>
      <div style="font-weight:700;margin-bottom:8px">Export / Import</div>
      <button id="exportBtn" class="btn">Export JSON</button>
      <input id="importFile" type="file" accept="application/json" style="margin-top:8px" />

      <div style="height:1px;background:rgba(0,0,0,0.06);margin:10px 0"></div>
      <div class="small">Klávesové zkratky: 1–5 výběr bloku, C = vyčistit, S = uložit.</div>
    </aside>
  </div>

<script>
// --- Config ---
const canvas = document.getElementById('world');
const ctx = canvas.getContext('2d');
let W = canvas.width, H = canvas.height;
const COLS = 36; // number of columns
const ROWS = 28; // number of rows
const CELL_W = Math.floor(W / COLS);
const CELL_H = Math.floor(H / ROWS);

// block definitions
const BLOCKS = [
  {id:0, name:'Prázdno', color:'#00000000'},
  {id:1, name:'Tráva', color:'#49a942', top:'#6bde6b'},
  {id:2, name:'Hlína', color:'#8b5a2b', top:'#b07645'},
  {id:3, name:'Kámen', color:'#7d7d7d', top:'#a6a6a6'},
  {id:4, name:'Dřevo', color:'#9b6b3a', top:'#c28a5c'},
  {id:5, name:'Listí', color:'#3da34d', top:'#6bd26b'}
];

// world grid
let grid = Array.from({length:ROWS}, ()=> Array(COLS).fill(0));
let currentBlock = 1; // default grass
let mouseDown = false;
let placing = true; // true = place, false = erase
let fillMode = false;

// UI elements
const paletteEl = document.getElementById('palette');
const saveBtn = document.getElementById('saveBtn');
const loadBtn = document.getElementById('loadBtn');
const clearBtn = document.getElementById('clearBtn');
const eraserBtn = document.getElementById('eraserBtn');
const fillBtn = document.getElementById('fillBtn');
const exportBtn = document.getElementById('exportBtn');
const importFile = document.getElementById('importFile');
const worldNameEl = document.getElementById('worldName');

// build palette
function buildPalette(){
  paletteEl.innerHTML = '';
  BLOCKS.forEach((b, i)=>{
    if(b.id===0) return; // skip empty
    const btn = document.createElement('div');
    btn.className = 'block-btn';
    btn.dataset.id = b.id;
    btn.innerHTML = `<div style="font-weight:700">${b.name}</div>`;
    btn.style.background = `linear-gradient(180deg, ${b.top || b.color}, ${b.color})`;
    if(b.id === currentBlock) btn.classList.add('active');
    btn.addEventListener('click', ()=>{ currentBlock = b.id; updatePalette(); });
    paletteEl.appendChild(btn);
  });
}
buildPalette();
function updatePalette(){
  document.querySelectorAll('.block-btn').forEach(el=> el.classList.toggle('active', Number(el.dataset.id)===currentBlock));
}

// draw grid
function drawGrid(){
  ctx.clearRect(0,0,W,H);
  // sky background gradient already via CSS, draw some sun/clouds
  drawClouds();

  for(let r=0;r<ROWS;r++){
    for(let c=0;c<COLS;c++){
      const id = grid[r][c];
      if(id === 0) continue;
      const b = BLOCKS.find(x=>x.id===id);
      const x = c*CELL_W, y = r*CELL_H;
      // draw main block
      ctx.fillStyle = b.color;
      ctx.fillRect(x, y, CELL_W, CELL_H);
      // simple top highlight
      if(b.top){ ctx.fillStyle = b.top; ctx.fillRect(x, y, CELL_W, Math.max(3, CELL_H*0.18)); }
      // subtle cell border
      ctx.strokeStyle = 'rgba(0,0,0,0.06)'; ctx.strokeRect(x+0.5, y+0.5, CELL_W-1, CELL_H-1);
    }
  }
  // grid lines (optional subtle)
  ctx.strokeStyle = 'rgba(0,0,0,0.04)';
  for(let r=0;r<=ROWS;r++) ctx.strokeRect(0, r*CELL_H, W, 0);
}

function drawClouds(){
  // decorative: some moving clouds
  const t = Date.now()/800;
  ctx.save();
  for(let i=0;i<6;i++){
    const cx = (i*200 + (t*30)) % (W+200) - 100;
    const cy = 30 + (i%3)*30;
    ctx.beginPath(); ctx.fillStyle = 'rgba(255,255,255,0.85)';
    ctx.arc(cx, cy, 28, 0, Math.PI*2);
    ctx.arc(cx+30, cy+6, 20, 0, Math.PI*2);
    ctx.arc(cx-25, cy+8, 18, 0, Math.PI*2);
    ctx.fill();
  }
  ctx.restore();
}

// coordinate helpers
function pxToCell(x,y){
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width; const scaleY = canvas.height / rect.height;
  const cx = (x - rect.left) * scaleX; const cy = (y - rect.top) * scaleY;
  const col = Math.floor(cx / CELL_W); const row = Math.floor(cy / CELL_H);
  return {row, col};
}

// place / remove
function placeAt(row,col){
  if(row<0||row>=ROWS||col<0||col>=COLS) return;
  if(placing) grid[row][col] = currentBlock; else grid[row][col] = 0;
}

function floodFill(r,c, targetId, replacementId){
  if(targetId === replacementId) return;
  const stack = [[r,c]];
  while(stack.length){
    const [rr,cc] = stack.pop();
    if(rr<0||rr>=ROWS||cc<0||cc>=COLS) continue;
    if(grid[rr][cc] !== targetId) continue;
    grid[rr][cc] = replacementId;
    stack.push([rr+1,cc],[rr-1,cc],[rr,cc+1],[rr,cc-1]);
  }
}

// events
canvas.addEventListener('contextmenu', e=> e.preventDefault());
canvas.addEventListener('pointerdown', e=>{
  mouseDown = true;
  const isRight = (e.button === 2) || e.ctrlKey; // right click or ctrl for Mac trackpad
  placing = !isRight;
  const {row,col} = pxToCell(e.clientX, e.clientY);
  if(fillMode && placing){ floodFill(row,col, grid[row][col], currentBlock); }
  else placeAt(row,col);
  drawGrid();
});
canvas.addEventListener('pointermove', e=>{
  if(!mouseDown) return;
  const {row,col} = pxToCell(e.clientX, e.clientY);
  placeAt(row,col);
  drawGrid();
});
window.addEventListener('pointerup', e=>{ mouseDown = false; });

// toolbar
saveBtn.addEventListener('click', ()=>{
  const name = prompt('Název světa (uloží do localStorage):', worldNameEl.textContent) || worldNameEl.textContent;
  localStorage.setItem('minicraft_world_' + name, JSON.stringify(grid));
  worldNameEl.textContent = name; alert('Uloženo!');
});
loadBtn.addEventListener('click', ()=>{
  const name = prompt('Název světa k načtení:', worldNameEl.textContent);
  if(!name) return;
  const raw = localStorage.getItem('minicraft_world_' + name);
  if(!raw){ alert('Svět nenalezen'); return; }
  grid = JSON.parse(raw);
  worldNameEl.textContent = name; drawGrid();
});
clearBtn.addEventListener('click', ()=>{ if(confirm('Vyčistit svět?')){ grid = Array.from({length:ROWS}, ()=> Array(COLS).fill(0)); drawGrid(); }});
eraserBtn.addEventListener('click', ()=>{ placing = false; alert('Eraser aktivní: pravý klik nebo drž pravé tlačítko'); });
fillBtn.addEventListener('click', ()=>{ fillMode = !fillMode; fillBtn.classList.toggle('active', fillMode); alert('Fill mode ' + (fillMode ? 'ON' : 'OFF')); });
exportBtn.addEventListener('click', ()=>{ const data = JSON.stringify({cols:COLS,rows:ROWS,grid}); const blob = new Blob([data], {type:'application/json'}); const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href = url; a.download = (worldNameEl.textContent || 'svet') + '.json'; a.click(); URL.revokeObjectURL(url); });
importFile.addEventListener('change', (e)=>{
  const f = e.target.files[0]; if(!f) return; const r = new FileReader(); r.onload = ()=>{ try{ const obj = JSON.parse(r.result); if(obj.grid) { grid = obj.grid; drawGrid(); alert('Načteno'); } else alert('Neplatný soubor'); }catch(err){ alert('Chyba: ' + err.message); } }; r.readAsText(f);
});

// keyboard shortcuts
window.addEventListener('keydown', e=>{
  if(e.key >= '1' && e.key <= '5'){ currentBlock = Number(e.key); updatePalette(); drawGrid(); }
  if(e.key === 'c' || e.key === 'C'){ if(confirm('Vyčistit svět?')){ grid = Array.from({length:ROWS}, ()=> Array(COLS).fill(0)); drawGrid(); }}
  if(e.key === 's' || e.key === 'S'){ saveBtn.click(); }
});

// responsive redraw animation for clouds
function animate(){ drawGrid(); requestAnimationFrame(animate); }
requestAnimationFrame(animate);

// initial demo: place a ground layer
(function initDemo(){
  for(let r=ROWS-6; r<ROWS; r++){
    for(let c=0;c<COLS;c++){
      grid[r][c] = (r === ROWS-6) ? 1 : (r < ROWS-2 ? 2 : 3);
    }
  }
  // some trees
  grid[ROWS-7][5] = 4; grid[ROWS-8][5] = 4; grid[ROWS-9][5] = 5; grid[ROWS-9][4] = 5; grid[ROWS-9][6] = 5;
  grid[ROWS-7][12] = 4; grid[ROWS-8][12] = 4; grid[ROWS-9][12] = 5; grid[ROWS-9][11] = 5; grid[ROWS-9][13] = 5;
  drawGrid();
})();

// fit canvas visually to container
function fitCanvas(){
  const rect = canvas.getBoundingClientRect(); canvas.style.width = rect.width + 'px'; canvas.style.height = rect.height + 'px';
}
window.addEventListener('resize', fitCanvas); fitCanvas();
</script>
</body>
</html>
