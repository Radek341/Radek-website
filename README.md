# Radek-website
<!doctype html>
<html lang="cs">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Šachy — demo</title>
<meta name="description" content="Interaktivní šachovnice - klikni pro tahy. Základní pravidla pohybu, proměna pěšce, tahy a historie." />
<style>
  :root{
    --light:#f0d9b5;
    --dark:#b58863;
    --accent:#2b7a78;
    --bg:#0b1020;
    --panel:#0f1724;
    font-family: Inter, system-ui, -apple-system, 'Segoe UI', Roboto, Arial;
  }
  html,body{height:100%;margin:0;background:linear-gradient(180deg,#071026,#081427);color:#e6eef6;display:flex;align-items:center;justify-content:center;padding:28px}
  .container{width:980px;max-width:96vw;display:grid;grid-template-columns:1fr 320px;gap:18px;align-items:start}
  .boardCard{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));padding:12px;border-radius:12px}
  h1{margin:6px 0;font-size:20px;color:#eaf6f6}
  .board{width:640px;height:640px;display:grid;grid-template-columns:repeat(8,1fr);border-radius:8px;overflow:hidden;box-shadow:0 14px 40px rgba(2,6,23,0.6)}
  .square{position:relative;width:100%;height:100%;display:flex;align-items:center;justify-content:center;font-size:48px;cursor:pointer;user-select:none}
  .sq-light{background:var(--light);color:#222}
  .sq-dark{background:var(--dark);color:#111}
  .coords{font-size:12px;position:absolute;left:6px;bottom:6px;color:rgba(0,0,0,0.35)}
  .square.highlight{outline:4px solid rgba(43,122,120,0.35);box-sizing:border-box}
  .square.capture{outline:4px solid rgba(195,23,45,0.35)}
  .controls{margin-top:10px;display:flex;gap:8px;align-items:center}
  .btn{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;font-weight:700;color:#fff;cursor:pointer}
  .btn.ghost{background:transparent;border:1px solid rgba(255,255,255,0.06);color:var(--light)}
  .sidebar{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));padding:12px;border-radius:12px;color:#e6eef6}
  .stat{display:flex;justify-content:space-between;margin-bottom:8px;color:var(--light)}
  .moves{height:420px;overflow:auto;background:rgba(255,255,255,0.02);padding:8px;border-radius:8px}
  .small{font-size:13px;color:rgba(255,255,255,0.7)}
  footer{margin-top:10px;color:var(--light);font-size:13px}
  @media (max-width:920px){ .container{grid-template-columns:1fr} .board{width:100%;height:480px} }
</style>
</head>
<body>
  <div class="container">
    <div class="boardCard">
      <h1>Šachovnice — demo</h1>
      <div id="board" class="board" aria-label="Šachovnice" role="application"></div>
      <div class="controls">
        <button id="undoBtn" class="btn ghost">Zpět</button>
        <button id="resetBtn" class="btn">Reset</button>
        <div style="margin-left:auto" class="small">Klikni na figurku → klikni na pole pro tah</div>
      </div>
      <footer>Legenda: ♔♕♖♗♘♙ (bílý) / ♚♛♜♝♞♟ (černý)</footer>
    </div>

    <aside class="sidebar">
      <div style="font-weight:700;margin-bottom:8px">Stav hry</div>
      <div class="stat"><span>Na tahu</span><span id="turn">Bílý</span></div>
      <div class="stat"><span>Poslední tah</span><span id="lastMove">-</span></div>
      <div class="stat"><span>Material (bílý - černý)</span><span id="material">-</span></div>
      <div style="height:1px;background:rgba(255,255,255,0.03);margin:10px 0"></div>
      <div style="font-weight:700;margin-bottom:6px">Historie tahů</div>
      <div id="moves" class="moves"></div>
      <div style="height:1px;background:rgba(255,255,255,0.03);margin:10px 0"></div>
      <div style="font-weight:700;margin-bottom:6px">Možnosti</div>
      <button id="dumpFen" class="btn ghost" style="width:100%;margin-top:6px">Zobraz FEN (konzolově)</button>
      <div style="height:6px"></div>
      <div class="small">Poznámka: en-passant a rošáda nejsou implementovány. Pěšec promotuje na dámu automaticky.</div>
    </aside>
  </div>

<script>
/*
  Jednoduchá šachová logika:
  - reprezentace board jako 8x8 pole s objekty {type:'p','r','n','b','q','k', color:'w'|'b'}
  - klikni na figurku -> ukáž legalní tahy (základní pohyb)
  - tahy provádí přesun a pokud dojde k zajetí, figurka zmizí
  - ukládá historii tahů (algebrafické označení je zjednodušené)
  - proměna pěšce v dámu při dosažení poslední řady
  - bez rošády, bez en-passant, bez úplné kontroly šachu
*/

// Unicode symboly pro zobrazení
const UNICODE = {
  w: {k:'♔', q:'♕', r:'♖', b:'♗', n:'♘', p:'♙'},
  b: {k:'♚', q:'♛', r:'♜', b:'♝', n:'♞', p:'♟'}
};

// board state
let board = []; // 8x8
let selected = null; // {r,c}
let legalMoves = []; // array of {r,c,capture}
let turn = 'w'; // 'w' or 'b'
let movesHistory = []; // list of moves
const boardEl = document.getElementById('board');
const turnEl = document.getElementById('turn');
const lastMoveEl = document.getElementById('lastMove');
const movesEl = document.getElementById('moves');
const materialEl = document.getElementById('material');

function coordToAlgebraic(r,c){
  const file = String.fromCharCode('a'.charCodeAt(0) + c);
  const rank = 8 - r;
  return file + rank;
}
function algebraicToCoord(s){
  const c = s.charCodeAt(0) - 'a'.charCodeAt(0);
  const r = 8 - Number(s[1]);
  return {r,c};
}

// initialize starting position
function initBoard(){
  board = Array.from({length:8},()=>Array(8).fill(null));
  const back = ['r','n','b','q','k','b','n','r'];
  for(let c=0;c<8;c++){
    board[0][c] = {type: back[c], color:'b'}; // black back rank (top)
    board[1][c] = {type:'p', color:'b'};
    board[6][c] = {type:'p', color:'w'};
    board[7][c] = {type: back[c], color:'w'};
  }
  selected = null; legalMoves = []; turn='w'; movesHistory = [];
  updateUI();
  renderBoard();
}

// render board DOM
function renderBoard(){
  boardEl.innerHTML = '';
  for(let r=0;r<8;r++){
    for(let c=0;c<8;c++){
      const sq = document.createElement('div');
      sq.className = 'square ' + (((r + c) % 2 === 0) ? 'sq-light' : 'sq-dark');
      sq.dataset.r = r; sq.dataset.c = c;
      // piece
      const piece = board[r][c];
      if(piece){
        const span = document.createElement('div');
        span.textContent = UNICODE[piece.color][piece.type];
        span.style.pointerEvents = 'none';
        sq.appendChild(span);
      }
      // coords (small)
      const coord = document.createElement('div');
      coord.className = 'coords';
      coord.textContent = coordToAlgebraic(r,c);
      sq.appendChild(coord);

      // highlight if selected
      if(selected && selected.r==r && selected.c==c) sq.classList.add('highlight');

      // highlight legal moves
      const lm = legalMoves.find(m=>m.r==r && m.c==c);
      if(lm){
        if(lm.capture) sq.classList.add('capture'); else sq.classList.add('highlight');
      }

      sq.addEventListener('click', onSquareClick);
      boardEl.appendChild(sq);
    }
  }
}

// click handler
function onSquareClick(e){
  const r = Number(this.dataset.r), c = Number(this.dataset.c);
  const piece = board[r][c];

  // if there's a selected piece and this square is a legal move -> move
  const move = legalMoves.find(m=>m.r==r && m.c==c);
  if(selected && move){
    makeMove(selected.r, selected.c, r, c);
    selected = null; legalMoves = [];
    renderBoard(); updateUI();
    return;
  }

  // otherwise if clicked own piece -> select & show moves
  if(piece && piece.color === turn){
    selected = {r,c};
    legalMoves = generateLegalMoves(r,c);
    renderBoard();
    return;
  }

  // clicked empty or opponent piece not a legal move -> clear selection
  selected = null; legalMoves = [];
  renderBoard();
}

// generate basic legal moves for piece at r,c
function generateLegalMoves(r,c){
  const p = board[r][c];
  if(!p) return [];
  const dir = p.color === 'w' ? -1 : 1;
  const moves = [];
  const onBoard = (rr,cc)=> rr>=0 && rr<8 && cc>=0 && cc<8;

  if(p.type === 'p'){
    // one step forward
    const fr = r + dir;
    if(onBoard(fr,c) && !board[fr][c]){
      moves.push({r:fr,c, capture:false});
      // two steps from starting rank
      const startRank = (p.color==='w' ? 6 : 1);
      const fr2 = r + dir*2;
      if(r === startRank && !board[fr2][c]) moves.push({r:fr2,c, capture:false});
    }
    // captures
    for(const dc of [-1,1]){
      const cr = r + dir, cc = c + dc;
      if(onBoard(cr,cc) && board[cr][cc] && board[cr][cc].color !== p.color){
        moves.push({r:cr,c:cc, capture:true});
      }
    }
    // NOTE: no en-passant implemented
  }

  if(p.type === 'n'){ // knight
    const del = [[-2,-1],[-2,1],[-1,-2],[-1,2],[1,-2],[1,2],[2,-1],[2,1]];
    for(const d of del){
      const rr = r + d[0], cc = c + d[1];
      if(onBoard(rr,cc) && (!board[rr][cc] || board[rr][cc].color !== p.color)){
        moves.push({r:rr,c:cc, capture: !!board[rr][cc]});
      }
    }
  }

  if(p.type === 'b' || p.type === 'r' || p.type === 'q'){
    const del = [];
    if(p.type === 'b' || p.type === 'q'){
      del.push([-1,-1],[-1,1],[1,-1],[1,1]);
    }
    if(p.type === 'r' || p.type === 'q'){
      del.push([-1,0],[1,0],[0,-1],[0,1]);
    }
    for(const d of del){
      let rr = r + d[0], cc = c + d[1];
      while(onBoard(rr,cc)){
        if(!board[rr][cc]) { moves.push({r:rr,c:cc, capture:false}); }
        else {
          if(board[rr][cc].color !== p.color) moves.push({r:rr,c:cc, capture:true});
          break;
        }
        rr += d[0]; cc += d[1];
      }
    }
  }

  if(p.type === 'k'){
    const del = [[-1,-1],[-1,0],[-1,1],[0,-1],[0,1],[1,-1],[1,0],[1,1]];
    for(const d of del){
      const rr = r + d[0], cc = c + d[1];
      if(onBoard(rr,cc) && (!board[rr][cc] || board[rr][cc].color !== p.color)){
        moves.push({r:rr,c:cc, capture: !!board[rr][cc]});
      }
    }
    // NOTE: no castling implemented
  }

  return moves;
}

// perform move from (sr,sc) to (tr,tc)
function makeMove(sr,sc,tr,tc){
  const piece = board[sr][sc];
  const target = board[tr][tc];
  // simple move
  board[tr][tc] = piece;
  board[sr][sc] = null;

  // pawn promotion: auto promote to queen/dáma
  if(piece.type === 'p'){
    if((piece.color === 'w' && tr === 0) || (piece.color === 'b' && tr === 7)){
      board[tr][tc] = {type:'q', color: piece.color};
    }
  }

  // record move (simple algebraic-ish)
  const mv = `${piece.color==='w' ? '' : ''}${piece.type.toUpperCase()} ${coordToAlgebraic(sr,sc)}→${coordToAlgebraic(tr,tc)}${target ? ' x' + target.type.toUpperCase() : ''}`;
  movesHistory.push(mv);
  lastMoveEl.textContent = mv;
  // switch turn
  turn = (turn === 'w' ? 'b' : 'w');
  updateUI();
  renderBoard();
}

// UI updates
function updateUI(){
  turnEl.textContent = (turn === 'w' ? 'Bílý' : 'Černý');
  // render moves
  movesEl.innerHTML = '';
  for(let i=0;i<movesHistory.length;i++){
    const div = document.createElement('div');
    div.textContent = (i+1) + '. ' + movesHistory[i];
    movesEl.appendChild(div);
  }
  // simple material count
  const mat = countMaterial();
  materialEl.textContent = `${mat.w} - ${mat.b}`;
}

// simple material count: sum piece values
function countMaterial(){
  const value = {p:1, n:3, b:3, r:5, q:9, k:0};
  let w=0,b=0;
  for(let r=0;r<8;r++) for(let c=0;c<8;c++){
    const p = board[r][c];
    if(!p) continue;
    if(p.color==='w') w += value[p.type]||0;
    else b += value[p.type]||0;
  }
  return {w,b};
}

// undo last move (very simple: reconstruct by popping history is not enough; implement by keeping snapshots)
let snapshots = [];
function snapshot(){
  snapshots.push({
    board: board.map(row => row.map(cell => cell ? {...cell} : null)),
    turn, movesHistory: movesHistory.slice()
  });
  if(snapshots.length > 200) snapshots.shift();
}

function undo(){
  if(snapshots.length === 0) return;
  const s = snapshots.pop();
  board = s.board.map(row => row.map(cell => cell ? {...cell} : null));
  turn = s.turn;
  movesHistory = s.movesHistory.slice();
  lastMoveEl.textContent = movesHistory.length ? movesHistory[movesHistory.length-1] : '-';
  selected = null; legalMoves = [];
  updateUI(); renderBoard();
}

// event handlers for reset / undo / dumpFEN
document.getElementById('resetBtn').addEventListener('click', ()=>{ initBoard(); snapshots = []; });
document.getElementById('undoBtn').addEventListener('click', ()=>{ undo(); });

document.getElementById('dumpFen').addEventListener('click', ()=> {
  console.log(boardToFEN());
  alert('FEN vypsán do konzole (devtools).');
});

// convert board to simple FEN (without move counters)
function boardToFEN(){
  let fen = '';
  for(let r=0;r<8;r++){
    let empty = 0;
    for(let c=0;c<8;c++){
      const p = board[r][c];
      if(!p){ empty++; }
      else {
        if(empty){ fen += empty; empty = 0; }
        const ch = p.type;
        fen += (p.color === 'w' ? ch.toUpperCase() : ch.toLowerCase());
      }
    }
    if(empty) fen += empty;
    if(r<7) fen += '/';
  }
  fen += ' ' + (turn === 'w' ? 'w' : 'b') + ' - - 0 1';
  return fen;
}

// capture snapshot before every move
function makeMoveWithSnapshot(sr,sc,tr,tc){
  snapshot();
  makeMove(sr,sc,tr,tc);
  // clear snapshots older than 200 handled in snapshot()
}

// override makeMove usage in click flow
function makeMove(sr,sc,tr,tc){
  const piece = board[sr][sc];
  const target = board[tr][tc];
  // perform move
  board[tr][tc] = piece;
  board[sr][sc] = null;
  // pawn promotion
  if(piece.type === 'p'){
    if((piece.color === 'w' && tr === 0) || (piece.color === 'b' && tr === 7)){
      board[tr][tc] = {type:'q', color: piece.color};
    }
  }
  const mv = `${(piece.color==='w' ? '' : '')}${piece.type.toUpperCase()} ${coordToAlgebraic(sr,sc)}→${coordToAlgebraic(tr,tc)}${target ? ' x' + target.type.toUpperCase() : ''}`;
  movesHistory.push(mv);
  lastMoveEl.textContent = mv;
  turn = (turn === 'w' ? 'b' : 'w');
  updateUI();
}

// But we used makeMoveWithSnapshot in click flow earlier; adjust onSquareClick to call snapshot-based
// To keep things consistent, modify onSquareClick to call makeMoveWithSnapshot
function onSquareClick(e){
  const r = Number(this.dataset.r), c = Number(this.dataset.c);
  const piece = board[r][c];

  // if there's a selected piece and this square is a legal move -> move
  const move = legalMoves.find(m=>m.r==r && m.c==c);
  if(selected && move){
    makeMoveWithSnapshot(selected.r, selected.c, r, c);
    selected = null; legalMoves = [];
    renderBoard(); updateUI();
    return;
  }

  // otherwise if clicked own piece -> select & show moves
  if(piece && piece.color === turn){
    selected = {r,c};
    legalMoves = generateLegalMoves(r,c);
    renderBoard();
    return;
  }

  // clicked empty or opponent piece not a legal move -> clear selection
  selected = null; legalMoves = [];
  renderBoard();
}

// attach updated handler to squares dynamically by re-rendering; initial init below

// initialize
initBoard();

</script>
</body>
</html>
