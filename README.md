<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>Mi Granja</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&family=Nunito:wght@400;700;900&display=swap');
  * { margin:0; padding:0; box-sizing:border-box; }
  body {
    background: #1a0a2e;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    min-height: 100vh; font-family: 'Nunito', sans-serif;
    overflow: hidden;
  }
  #game-title {
    font-family: 'Press Start 2P', monospace;
    color: #f9e04b; font-size: 12px;
    text-shadow: 3px 3px 0 #c47c00, 0 0 20px #f9e04b88;
    margin-bottom: 8px; letter-spacing: 2px;
  }
  #game-wrapper {
    position: relative;
    border: 4px solid #f9e04b;
    box-shadow: 0 0 0 4px #c47c00, 0 0 40px #f9e04b44;
    border-radius: 4px; overflow: hidden;
  }
  canvas { display: block; image-rendering: pixelated; }
  #hud {
    display: flex; justify-content: space-between; align-items: center;
    background: #0d1b0f; border-top: 3px solid #3d8b3d;
    padding: 6px 10px; gap: 8px; flex-wrap: wrap;
  }
  .hud-item { display: flex; align-items: center; gap: 4px; color: #a8e063; font-size: 11px; font-weight: 700; }
  .hud-icon { font-size: 14px; }
  #controls-hint { color: #556b55; font-size: 9px; text-align: center; margin-top: 5px; font-family: 'Press Start 2P', monospace; letter-spacing: 1px; }
  #action-btn {
    position: absolute; bottom: 80px; right: 16px;
    width: 58px; height: 58px; border-radius: 50%;
    background: linear-gradient(135deg, #f9e04b, #c47c00);
    border: 3px solid #fff3; box-shadow: 0 4px 16px #f9e04b66;
    color: #1a0a2e; font-size: 24px; cursor: pointer;
    display: none; align-items: center; justify-content: center;
    font-weight: 900; transition: transform 0.1s; z-index: 10;
  }
  #action-btn:active { transform: scale(0.9); }
  #toast {
    position: absolute; top: 10px; left: 50%; transform: translateX(-50%);
    background: #0d1b0fee; border: 2px solid #a8e063;
    color: #a8e063; font-size: 10px; font-weight: 700;
    padding: 5px 12px; border-radius: 20px; pointer-events: none;
    opacity: 0; transition: opacity 0.3s; white-space: nowrap; z-index: 20;
  }
  #dpad { position: absolute; bottom: 65px; left: 12px; display: none; z-index: 10; }
  .dpad-row { display: flex; justify-content: center; gap: 2px; margin: 2px 0; }
  .dpad-btn {
    width: 42px; height: 42px; background: #ffffff22;
    border: 2px solid #ffffff33; border-radius: 8px;
    color: #fff; font-size: 16px; cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    user-select: none; -webkit-user-select: none; touch-action: none;
  }
  .dpad-btn:active { background: #ffffff44; }
</style>
</head>
<body>
<div id="game-title">🌾 MI GRANJA</div>
<div id="game-wrapper">
  <canvas id="gameCanvas" width="544" height="352"></canvas>
  <div id="hud">
    <div class="hud-item"><span class="hud-icon">🌱</span><span id="hud-seeds">Semillas: 10</span></div>
    <div class="hud-item"><span class="hud-icon">🌾</span><span id="hud-crops">Cosecha: 0</span></div>
    <div class="hud-item"><span class="hud-icon">🐄</span><span id="hud-milk">Leche: 0</span></div>
    <div class="hud-item"><span class="hud-icon">💰</span><span id="hud-gold">Oro: 0</span></div>
    <div class="hud-item"><span class="hud-icon">☀️</span><span id="hud-day">Día 1</span></div>
  </div>
  <div id="toast"></div>
  <div id="dpad">
    <div class="dpad-row"><div class="dpad-btn" id="btn-up">▲</div></div>
    <div class="dpad-row">
      <div class="dpad-btn" id="btn-left">◀</div>
      <div style="width:42px"></div>
      <div class="dpad-btn" id="btn-right">▶</div>
    </div>
    <div class="dpad-row"><div class="dpad-btn" id="btn-down">▼</div></div>
  </div>
  <button id="action-btn">⚡</button>
</div>
<div id="controls-hint">WASD/Flechas · E acción · Ordeña las vacas 🐄</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const TILE=32, COLS=17, ROWS=11;

const T={GRASS:0,DIRT:1,WATER:2,PATH:3,CROP_PLANTED:4,CROP_GROWING:5,CROP_READY:6,BARN:7,SHOP:8,MOUNTAIN:9,ROAD:10,BRIDGE:11,FENCE:12,PASTURE:13,TREE:14,SNOW:15};

// 17x11 map
const mapLayout=[
  [9,9,9,15,15,9,9,  0,0,0,0,0,  2,2,2,0,0],
  [9,9,15,15,9,9,0,  0,0,0,0,0,  2,0,0,0,0],
  [9,9,9,0,0,0,8,    3,3,3,3,3,  2,0,0,7,0],
  [9,0,0,0,13,13,3,  1,1,1,1,3,  2,3,3,3,0],
  [0,12,13,13,13,12, 3,1,1,1,3,  2,3,0,0,0],
  [0,12,13,13,13,12, 3,1,1,1,3, 11,3,0,0,0],
  [0,12,13,13,13,12, 3,1,1,1,3,  2,3,0,0,0],
  [0,0,0,0,12,12,3,  1,1,1,1,3,  2,3,3,3,0],
  [0,14,0,0,0,0,3,   0,3,0,0,0,  2,0,0,0,0],
  [10,10,10,10,10,10,10,10,10,10,10,10,10,10,10,10,10],
  [0,14,0,0,0,0,0,   0,0,0,0,0,  0,0,0,0,0],
];

const cropTimers={};
const player={x:7.5,y:5.5,dir:'down',step:0,stepTimer:0};
const state={seeds:10,crops:0,milk:0,gold:0,day:1};
let dayTimer=0;
const DAY_DURATION=60;

const cattle=[
  {x:2.2,y:4.3,vx:0.4,vy:0.1,dir:'right',step:0,stepTimer:0,milkReady:false,milkTimer:0},
  {x:3.8,y:5.6,vx:-0.35,vy:0.2,dir:'left',step:0,stepTimer:0,milkReady:true,milkTimer:28},
  {x:4.2,y:4.8,vx:0.25,vy:-0.3,dir:'right',step:0,stepTimer:0,milkReady:false,milkTimer:10},
];

const keys={};
document.addEventListener('keydown',e=>{keys[e.key]=true;});
document.addEventListener('keyup',e=>{keys[e.key]=false;});

function tileAt(x,y){const c=Math.floor(x),r=Math.floor(y);if(r<0||r>=ROWS||c<0||c>=COLS)return T.MOUNTAIN;return mapLayout[r][c];}
function setTile(c,r,t){if(r>=0&&r<ROWS&&c>=0&&c<COLS)mapLayout[r][c]=t;}
function isWalkable(x,y){const t=tileAt(x,y);return![T.MOUNTAIN,T.WATER,T.BARN,T.SHOP,T.SNOW,T.TREE,T.FENCE].includes(t);}
function isCowWalkable(x,y){const t=tileAt(x,y);return[T.GRASS,T.PASTURE,T.PATH].includes(t);}

function showToast(msg,color='#a8e063'){const el=document.getElementById('toast');el.textContent=msg;el.style.borderColor=color;el.style.color=color;el.style.opacity='1';clearTimeout(el._t);el._t=setTimeout(()=>el.style.opacity='0',2200);}
function updateHUD(){document.getElementById('hud-seeds').textContent=`Semillas: ${state.seeds}`;document.getElementById('hud-crops').textContent=`Cosecha: ${state.crops}`;document.getElementById('hud-milk').textContent=`Leche: ${state.milk}`;document.getElementById('hud-gold').textContent=`Oro: ${state.gold}`;document.getElementById('hud-day').textContent=`Día ${state.day}`;}

function doAction(){
  const px=Math.floor(player.x),py=Math.floor(player.y);
  const off={up:[0,-1],down:[0,1],left:[-1,0],right:[1,0]};
  const [dx,dy]=off[player.dir];
  const tx=px+dx,ty=py+dy,t=tileAt(tx,ty);
  if(t===T.SHOP){if(state.gold>=5){state.gold-=5;state.seeds+=5;showToast('🛒 ¡Compraste 5 semillas!');}else showToast('💸 Sin suficiente oro!','#e74c3c');updateHUD();return;}
  if(t===T.BARN){const e=state.crops*3+state.milk*4;if(e>0){showToast(`🏚 Vendiste todo por ${e} oro!`);state.gold+=e;state.crops=0;state.milk=0;}else showToast('📦 Nada que vender!','#e7c34c');updateHUD();return;}
  if(t===T.DIRT&&state.seeds>0){setTile(tx,ty,T.CROP_PLANTED);cropTimers[`${tx},${ty}`]=0;state.seeds--;showToast('🌱 Plantaste una semilla!');updateHUD();return;}
  if(t===T.CROP_READY){setTile(tx,ty,T.DIRT);delete cropTimers[`${tx},${ty}`];state.crops++;showToast('🌾 ¡Cosechaste!');updateHUD();return;}
  if(t===T.CROP_GROWING){showToast('⏳ Aún creciendo...','#f9e04b');return;}
  if(t===T.CROP_PLANTED){showToast('🌱 Recién plantado...');return;}
  // Milk
  for(let c of cattle){if(Math.hypot(c.x-player.x,c.y-player.y)<1.8&&c.milkReady){c.milkReady=false;c.milkTimer=0;state.milk++;showToast('🥛 ¡Ordeñaste la vaca!');updateHUD();return;}}
}

document.addEventListener('keydown',e=>{if(e.key==='e'||e.key==='E')doAction();});
document.getElementById('action-btn').addEventListener('click',doAction);

const isMobile='ontouchstart' in window||navigator.maxTouchPoints>0;
if(isMobile){document.getElementById('dpad').style.display='block';document.getElementById('action-btn').style.display='flex';document.getElementById('controls-hint').style.display='none';}
const dpadMap={'btn-up':'ArrowUp','btn-down':'ArrowDown','btn-left':'ArrowLeft','btn-right':'ArrowRight'};
Object.entries(dpadMap).forEach(([id,key])=>{const btn=document.getElementById(id);btn.addEventListener('touchstart',e=>{e.preventDefault();keys[key]=true;});btn.addEventListener('touchend',e=>{e.preventDefault();keys[key]=false;});});

// ===== DRAW =====
function drawGrass(x,y,col,row){
  ctx.fillStyle='#56a832';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#4a9428';
  for(let i=0;i<5;i++)ctx.fillRect(x+(col*7+row*3+i*11)%28,y+(row*5+i*7)%28,4,4);
  ctx.fillStyle='#62be3a';
  ctx.fillRect(x+6,y+8,2,6);ctx.fillRect(x+18,y+14,2,5);ctx.fillRect(x+26,y+6,2,7);
}

function drawMtn(x,y,snow){
  ctx.fillStyle='#7a8e96';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#5c7080';
  ctx.beginPath();ctx.moveTo(x,y+TILE);ctx.lineTo(x+TILE/2,y+5);ctx.lineTo(x+TILE,y+TILE);ctx.fill();
  ctx.fillStyle='#4a5e6a';
  ctx.beginPath();ctx.moveTo(x+TILE/2,y+5);ctx.lineTo(x+TILE,y+TILE);ctx.lineTo(x+TILE/2,y+TILE);ctx.fill();
  if(snow){ctx.fillStyle='#eef6ff';ctx.beginPath();ctx.moveTo(x+TILE/2,y+5);ctx.lineTo(x+TILE/2-9,y+19);ctx.lineTo(x+TILE/2+9,y+19);ctx.fill();}
}

function drawWater(x,y){
  ctx.fillStyle='#1a70bb';ctx.fillRect(x,y,TILE,TILE);
  const t=(Date.now()/700)%1;
  ctx.fillStyle='#2e90db';
  ctx.fillRect(x+2+t*10,y+8,10,3);ctx.fillRect(x+16+(1-t)*8,y+18,8,3);ctx.fillRect(x+4+t*6,y+24,12,3);
}

function drawBridge(x,y){
  drawWater(x,y);
  ctx.fillStyle='#6a3a10';ctx.fillRect(x,y+6,TILE,4);ctx.fillRect(x,y+22,TILE,4);
  ctx.fillStyle='#8B5e3c';ctx.fillRect(x,y+10,TILE,12);
  ctx.fillStyle='#a87040';
  for(let i=0;i<4;i++)ctx.fillRect(x+i*9+1,y+10,7,12);
  ctx.fillStyle='#6b3a20';ctx.fillRect(x,y+10,TILE,2);ctx.fillRect(x,y+20,TILE,2);
}

function drawRoad(x,y){
  ctx.fillStyle='#4a4a4a';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#3a3a3a';ctx.fillRect(x,y,TILE,5);ctx.fillRect(x,y+TILE-5,TILE,5);
  ctx.fillStyle='#606060';ctx.fillRect(x,y+5,TILE,3);ctx.fillRect(x,y+TILE-8,TILE,3);
  // Animated dash
  const off=Math.floor(Date.now()/60)%32;
  ctx.fillStyle='#f9e04b';
  ctx.fillRect(x+((off)%32)-6,y+14,10,4);
  ctx.fillRect(x+((off+16)%32)-6,y+14,10,4);
}

function drawTree(x,y){
  ctx.fillStyle='#56a832';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#3a2a18';ctx.fillRect(x+13,y+18,6,14);
  ctx.fillStyle='#1e5e10';ctx.beginPath();ctx.moveTo(x+16,y+1);ctx.lineTo(x+4,y+18);ctx.lineTo(x+28,y+18);ctx.fill();
  ctx.fillStyle='#2e8020';ctx.beginPath();ctx.moveTo(x+16,y+5);ctx.lineTo(x+5,y+20);ctx.lineTo(x+27,y+20);ctx.fill();
  ctx.fillStyle='#3d9a28';ctx.beginPath();ctx.moveTo(x+16,y+9);ctx.lineTo(x+7,y+22);ctx.lineTo(x+25,y+22);ctx.fill();
}

function drawFence(x,y){
  ctx.fillStyle='#5ebe3c';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#c8a058';
  ctx.fillRect(x,y+10,TILE,4);ctx.fillRect(x,y+20,TILE,4);
  ctx.fillRect(x+2,y+6,5,20);ctx.fillRect(x+TILE-7,y+6,5,20);
}

function drawPasture(x,y,col,row){
  ctx.fillStyle='#5ebe3c';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#4aaa2c';
  for(let i=0;i<6;i++)ctx.fillRect(x+(col*7+row*5+i*9)%26+2,y+(row*7+i*5)%26+2,3,3);
  ctx.fillStyle='#70d840';
  ctx.fillRect(x+5,y+8,2,5);ctx.fillRect(x+15,y+14,2,4);ctx.fillRect(x+24,y+6,2,6);
}

function drawCropTile(x,y,type){
  ctx.fillStyle='#8B5E3C';ctx.fillRect(x,y,TILE,TILE);
  ctx.fillStyle='#7a5232';
  for(let i=0;i<4;i++)ctx.fillRect(x+(Math.floor(x/TILE)*11+i*9)%24+2,y+(Math.floor(y/TILE)*7+i*5)%24+2,5,3);
  if(type===T.CROP_PLANTED){
    ctx.fillStyle='#5a9228';ctx.fillRect(x+14,y+20,4,8);
    ctx.fillStyle='#76c240';ctx.fillRect(x+10,y+14,12,8);
  }else if(type===T.CROP_GROWING){
    ctx.fillStyle='#3a7a18';ctx.fillRect(x+12,y+12,4,16);
    ctx.fillStyle='#56a832';ctx.fillRect(x+6,y+10,8,10);ctx.fillRect(x+18,y+14,8,8);
    ctx.fillStyle='#76c240';ctx.fillRect(x+10,y+6,12,8);
  }else if(type===T.CROP_READY){
    ctx.fillStyle='#3a7a18';ctx.fillRect(x+12,y+8,4,20);
    ctx.fillStyle='#f9e04b';ctx.fillRect(x+8,y+4,16,10);
    ctx.fillStyle='#c47c00';ctx.fillRect(x+10,y+6,12,6);
    ctx.fillStyle='#f9e04b';ctx.fillRect(x+4,y+8,8,8);ctx.fillRect(x+20,y+8,8,8);
  }
}

function drawBuilding(x,y,type){
  if(type===T.BARN){
    ctx.fillStyle='#56a832';ctx.fillRect(x,y,TILE,TILE);
    ctx.fillStyle='#c0392b';ctx.fillRect(x+2,y+9,28,22);
    ctx.fillStyle='#8B1a14';
    ctx.beginPath();ctx.moveTo(x,y+9);ctx.lineTo(x+16,y+0);ctx.lineTo(x+32,y+9);ctx.fill();
    ctx.fillStyle='#e74c3c';ctx.fillRect(x+2,y+9,28,6);
    ctx.fillStyle='#3a0e0a';ctx.fillRect(x+11,y+18,10,13);
    ctx.fillStyle='#f9e04b';ctx.font='6px "Press Start 2P"';ctx.fillText('VENTA',x+0,y+8);
  }else if(type===T.SHOP){
    ctx.fillStyle='#56a832';ctx.fillRect(x,y,TILE,TILE);
    ctx.fillStyle='#8e44ad';ctx.fillRect(x+2,y+9,28,22);
    ctx.fillStyle='#5a1e80';
    ctx.beginPath();ctx.moveTo(x,y+9);ctx.lineTo(x+16,y+0);ctx.lineTo(x+32,y+9);ctx.fill();
    ctx.fillStyle='#9b59b6';ctx.fillRect(x+2,y+9,28,6);
    ctx.fillStyle='#ecf0f1';ctx.fillRect(x+6,y+16,7,8);ctx.fillRect(x+19,y+16,7,8);
    ctx.fillStyle='#f9e04b';ctx.font='6px "Press Start 2P"';ctx.fillText('TIENDA',x+0,y+8);
  }
}

function drawTile(col,row,type){
  const x=col*TILE,y=row*TILE;
  if(type===T.MOUNTAIN)drawMtn(x,y,false);
  else if(type===T.SNOW)drawMtn(x,y,true);
  else if(type===T.WATER)drawWater(x,y);
  else if(type===T.BRIDGE)drawBridge(x,y);
  else if(type===T.ROAD)drawRoad(x,y);
  else if(type===T.TREE)drawTree(x,y);
  else if(type===T.FENCE)drawFence(x,y);
  else if(type===T.PASTURE)drawPasture(x,y,col,row);
  else if(type===T.GRASS)drawGrass(x,y,col,row);
  else if(type===T.DIRT){ctx.fillStyle='#8B5E3C';ctx.fillRect(x,y,TILE,TILE);ctx.fillStyle='#7a5232';for(let i=0;i<4;i++)ctx.fillRect(x+(col*11+i*9)%24+2,y+(row*7+i*5)%24+2,5,3);}
  else if(type===T.PATH){ctx.fillStyle='#c4a265';ctx.fillRect(x,y,TILE,TILE);ctx.fillStyle='#b8956a';ctx.fillRect(x+1,y+1,TILE-2,3);ctx.fillRect(x+1,y+TILE-4,TILE-2,3);}
  else if(type===T.BARN)drawBuilding(x,y,T.BARN);
  else if(type===T.SHOP)drawBuilding(x,y,T.SHOP);
  else if([T.CROP_PLANTED,T.CROP_GROWING,T.CROP_READY].includes(type))drawCropTile(x,y,type);
}

function drawCow(c){
  const x=c.x*TILE-14,y=c.y*TILE-16;
  const b=Math.sin(c.step*0.35)*1.5;
  ctx.fillStyle='#00000033';
  ctx.beginPath();ctx.ellipse(c.x*TILE,c.y*TILE+6,12,5,0,0,Math.PI*2);ctx.fill();
  // Body
  ctx.fillStyle='#f0ede0';ctx.fillRect(x+4,y+8+b,22,14);
  ctx.fillStyle='#3a2a1a';ctx.fillRect(x+6,y+9+b,8,5);ctx.fillRect(x+18,y+13+b,5,4);
  // Legs
  ctx.fillStyle='#d4c8a8';
  ctx.fillRect(x+6,y+20+b,4,8);ctx.fillRect(x+12,y+20+b,4,8);ctx.fillRect(x+18,y+20+b,4,8);ctx.fillRect(x+22,y+20+b,4,8);
  ctx.fillStyle='#4a3728';
  ctx.fillRect(x+6,y+26+b,4,3);ctx.fillRect(x+12,y+26+b,4,3);ctx.fillRect(x+18,y+26+b,4,3);ctx.fillRect(x+22,y+26+b,4,3);
  // Head
  ctx.fillStyle='#f0ede0';
  if(c.dir==='right'){
    ctx.fillRect(x+24,y+6+b,10,10);
    ctx.fillStyle='#e8b4a0';ctx.fillRect(x+30,y+10+b,6,5);
    ctx.fillStyle='#222';ctx.fillRect(x+26,y+8+b,2,2);
    ctx.fillStyle='#e8a090';ctx.fillRect(x+24,y+5+b,4,4);
    ctx.fillStyle='#d4c080';ctx.fillRect(x+26,y+3+b,2,5);
  }else{
    ctx.fillRect(x-2,y+6+b,10,10);
    ctx.fillStyle='#e8b4a0';ctx.fillRect(x-4,y+10+b,6,5);
    ctx.fillStyle='#222';ctx.fillRect(x+2,y+8+b,2,2);
    ctx.fillStyle='#e8a090';ctx.fillRect(x+2,y+5+b,4,4);
    ctx.fillStyle='#d4c080';ctx.fillRect(x+2,y+3+b,2,5);
  }
  // Tail
  ctx.fillStyle='#c8bca0';ctx.fillRect(c.dir==='right'?x+2:x+22,y+8+b,4,3);
  // Milk indicator
  if(c.milkReady){
    ctx.fillStyle='#f9e04b';ctx.font='bold 11px Nunito';ctx.textAlign='center';
    ctx.fillText('🥛',c.x*TILE,c.y*TILE-20+b);
  }
}

function drawPlayer(){
  const x=player.x*TILE-8,y=player.y*TILE-20;
  ctx.fillStyle='#00000044';ctx.beginPath();ctx.ellipse(player.x*TILE,player.y*TILE+4,8,4,0,0,Math.PI*2);ctx.fill();
  const b=Math.sin(player.step*0.3)*2;
  ctx.fillStyle='#e74c3c';ctx.fillRect(x+4,y+10+b,10,12);
  ctx.fillStyle='#2980b9';ctx.fillRect(x+4,y+18+b,4,8);ctx.fillRect(x+8,y+18+b,4,8);
  ctx.fillStyle='#4a3728';ctx.fillRect(x+3,y+24+b,5,4);ctx.fillRect(x+8,y+24+b,5,4);
  ctx.fillStyle='#f5cba7';ctx.fillRect(x+4,y+2+b,10,10);
  ctx.fillStyle='#f39c12';ctx.fillRect(x+2,y+4+b,14,4);ctx.fillRect(x+5,y+0+b,8,5);
  if(player.dir==='down'){ctx.fillStyle='#2c1810';ctx.fillRect(x+6,y+6+b,2,2);ctx.fillRect(x+10,y+6+b,2,2);ctx.fillRect(x+7,y+10+b,4,1);}
  ctx.fillStyle='#8B6914';ctx.fillRect(x+14,y+10+b,2,14);
  ctx.fillStyle='#c0392b';ctx.fillRect(x+12,y+8+b,6,4);
}

function drawDayBar(){
  const pct=dayTimer/DAY_DURATION,w=canvas.width;
  ctx.fillStyle='#0d1b0f88';ctx.fillRect(w-96,6,90,10);
  const g=ctx.createLinearGradient(w-95,0,w-9,0);
  g.addColorStop(0,'#f9e04b');g.addColorStop(1,'#e67e22');
  ctx.fillStyle=g;ctx.fillRect(w-95,7,Math.max(0,88*(1-pct)),8);
  ctx.fillStyle='#fff';ctx.font='7px "Press Start 2P"';ctx.fillText('DÍA',w-96,5);
}

// ===== UPDATE =====
let lastTime=0;

function updateCattle(dt){
  for(let c of cattle){
    c.milkTimer+=dt;
    if(c.milkTimer>=30&&!c.milkReady)c.milkReady=true;
    const nx=c.x+c.vx*dt,ny=c.y+c.vy*dt;
    const m=0.3;
    const okX=nx>=0.8&&nx<=5.4&&isCowWalkable(nx+m,c.y+m)&&isCowWalkable(nx+1-m,c.y+1-m);
    const okY=ny>=2.8&&ny<=7.4&&isCowWalkable(c.x+m,ny+m)&&isCowWalkable(c.x+1-m,ny+1-m);
    if(okX)c.x=nx;else{c.vx=-c.vx+(Math.random()-0.5)*0.3;}
    if(okY)c.y=ny;else{c.vy=-c.vy+(Math.random()-0.5)*0.3;}
    const spd=Math.hypot(c.vx,c.vy);
    if(spd>0.8){c.vx=c.vx/spd*0.8;c.vy=c.vy/spd*0.8;}
    if(spd<0.15){c.vx+=(Math.random()-0.5)*0.4;c.vy+=(Math.random()-0.5)*0.4;}
    c.dir=c.vx>=0?'right':'left';
    c.stepTimer+=dt;if(c.stepTimer>0.18){c.step++;c.stepTimer=0;}
  }
}

function gameLoop(timestamp){
  const dt=Math.min((timestamp-lastTime)/1000,0.1);
  lastTime=timestamp;

  dayTimer+=dt;
  if(dayTimer>=DAY_DURATION){
    dayTimer=0;state.day++;
    Object.keys(cropTimers).forEach(key=>{
      cropTimers[key]++;
      const [col,row]=key.split(',').map(Number);
      if(cropTimers[key]===1)setTile(col,row,T.CROP_GROWING);
      if(cropTimers[key]>=2)setTile(col,row,T.CROP_READY);
    });
    showToast(`🌙 ¡Día ${state.day}! Las plantas crecieron.`);
    updateHUD();
  }

  const SPEED=4;let dx=0,dy=0;
  if(keys['ArrowUp']||keys['w']||keys['W']){dy=-SPEED*dt;player.dir='up';}
  if(keys['ArrowDown']||keys['s']||keys['S']){dy=SPEED*dt;player.dir='down';}
  if(keys['ArrowLeft']||keys['a']||keys['A']){dx=-SPEED*dt;player.dir='left';}
  if(keys['ArrowRight']||keys['d']||keys['D']){dx=SPEED*dt;player.dir='right';}
  const nx=player.x+dx,ny=player.y+dy,m=0.35;
  if(isWalkable(nx+m,player.y+m)&&isWalkable(nx+1-m,player.y+m)&&isWalkable(nx+m,player.y+1-m)&&isWalkable(nx+1-m,player.y+1-m)&&nx>=0&&nx<COLS-1)player.x=nx;
  if(isWalkable(player.x+m,ny+m)&&isWalkable(player.x+1-m,ny+m)&&isWalkable(player.x+m,ny+1-m)&&isWalkable(player.x+1-m,ny+1-m)&&ny>=0&&ny<ROWS-1)player.y=ny;
  if(dx!==0||dy!==0){player.stepTimer+=dt;if(player.stepTimer>0.15){player.step++;player.stepTimer=0;}}

  updateCattle(dt);

  ctx.clearRect(0,0,canvas.width,canvas.height);
  for(let r=0;r<ROWS;r++)for(let c=0;c<COLS;c++)drawTile(c,r,mapLayout[r][c]);

  // Grid on dirt
  ctx.strokeStyle='#ffffff11';ctx.lineWidth=0.5;
  for(let r=0;r<ROWS;r++)for(let c=0;c<COLS;c++){const t=mapLayout[r][c];if(t>=T.DIRT&&t<=T.CROP_READY)ctx.strokeRect(c*TILE,r*TILE,TILE,TILE);}

  // Sort by Y
  const entities=[...cattle.map(c=>({type:'cow',obj:c,sy:c.y})),{type:'player',obj:player,sy:player.y}].sort((a,b)=>a.sy-b.sy);
  ctx.textAlign='left';
  for(let e of entities){if(e.type==='cow')drawCow(e.obj);else drawPlayer();}
  ctx.textAlign='left';

  drawDayBar();

  // Action hints
  const px=Math.floor(player.x),py=Math.floor(player.y);
  const offs={up:[0,-1],down:[0,1],left:[-1,0],right:[1,0]};
  const [odx,ody]=offs[player.dir];
  const ft=tileAt(px+odx,py+ody);
  if([T.DIRT,T.CROP_PLANTED,T.CROP_GROWING,T.CROP_READY,T.BARN,T.SHOP].includes(ft)){
    const hx=(px+odx)*TILE+TILE/2,hy=(py+ody)*TILE+TILE/2;
    ctx.fillStyle='#f9e04b';ctx.font='bold 10px Nunito';ctx.textAlign='center';ctx.fillText('[E]',hx,hy-4);ctx.textAlign='left';
  }
  for(let c of cattle){if(Math.hypot(c.x-player.x,c.y-player.y)<1.8&&c.milkReady){ctx.fillStyle='#f9e04b';ctx.font='bold 9px Nunito';ctx.textAlign='center';ctx.fillText('[E] Ordeñar',c.x*TILE,c.y*TILE-24);ctx.textAlign='left';}}

  requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
</script>
</body>
</html>
