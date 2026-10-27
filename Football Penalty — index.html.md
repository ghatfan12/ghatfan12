```html
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Football Penalty</title>
<style>
*{box-sizing:border-box}
body{
  margin:0;background:#071b15;color:white;
  font-family:Arial,sans-serif;text-align:center;
}
header{padding:18px 10px 8px}
h1{margin:0;color:#ffe45c;font-size:clamp(25px,5vw,38px)}
p{margin:8px 0;color:#d4e9dc}
.game{
  width:min(94vw,760px);margin:12px auto;
  background:#123d29;border:2px solid #397651;
  border-radius:18px;padding:10px;
}
canvas{
  display:block;width:100%;height:auto;
  border-radius:10px;background:#278849;
  touch-action:manipulation;
}
.stats{
  display:flex;justify-content:space-around;gap:8px;
  padding:12px 2px;font-weight:bold;flex-wrap:wrap
}
.stats span{color:#ffe45c}
button{
  border:0;border-radius:12px;padding:14px 24px;
  margin:5px;background:#ffe45c;color:#172318;
  font-weight:bold;font-size:17px;cursor:pointer;
}
button:active{transform:scale(.97)}
#message{min-height:26px;font-weight:bold;color:#ffe45c}
.hint{font-size:13px;color:#c7e3d2;padding-bottom:8px}
</style>
</head>
<body>
<header>
  <h1>⚽ FOOTBALL PENALTY</h1>
  <p>Bisakah kamu mengalahkan sang kiper?</p>
</header>

<main class="game">
  <canvas id="field" width="700" height="440"></canvas>

  <div class="stats">
    <div>GOL: <span id="goals">0</span></div>
    <div>TENDANGAN: <span id="shots">0/5</span></div>
    <div>SAVE KIPER: <span id="saves">0</span></div>
  </div>

  <div id="message">Pilih arah tendangan, lalu tekan TENDANG!</div>
  <button id="kick">⚽ TENDANG!</button>
  <button id="restart">↻ MAIN LAGI</button>
  <div class="hint">Klik bagian gawang untuk membidik. Bisa juga sentuh layar HP.</div>
</main>

<script>
const canvas = document.getElementById('field');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

const goal = {x:150,y:55,w:400,h:155};
let aim = {x:350,y:100};
let ball = {x:350,y:350,r:12};
let keeper = {x:350,y:165};
let goals = 0, shots = 0, saves = 0;
let moving = false, ended = false;
let kickData = null;
let resultTimer = 0;
let message = 'Pilih arah tendangan, lalu tekan TENDANG!';

function updateUI(){
  document.getElementById('goals').textContent=goals;
  document.getElementById('shots').textContent=shots+'/5';
  document.getElementById('saves').textContent=saves;
  document.getElementById('message').textContent=message;
}

function drawField(){
  ctx.clearRect(0,0,W,H);
  for(let i=0;i<8;i++){
    ctx.fillStyle=i%2?'#278849':'#2e9650';
    ctx.fillRect(i*W/8,0,W/8,H);
  }

  // Garis lapangan
  ctx.strokeStyle='rgba(255,255,255,.7)';
  ctx.lineWidth=3;
  ctx.strokeRect(25,20,W-50,H-40);

  // Gawang dan jaring
  ctx.fillStyle='#eaf4ed';
  ctx.fillRect(goal.x,goal.y,goal.w,goal.h);
  ctx.fillStyle='#b8d1c1';
  ctx.fillRect(goal.x+7,goal.y+7,goal.w-14,goal.h-7);

  ctx.strokeStyle='#fff';
  ctx.lineWidth=5;
  ctx.strokeRect(goal.x,goal.y,goal.w,goal.h);

  ctx.strokeStyle='#789d85';
  ctx.lineWidth=1;
  for(let x=goal.x+20;x<goal.x+goal.w;x+=25){
    ctx.beginPath();ctx.moveTo(x,goal.y);ctx.lineTo(x,goal.y+goal.h);ctx.stroke();
  }
  for(let y=goal.y+20;y<goal.y+goal.h;y+=25){
    ctx.beginPath();ctx.moveTo(goal.x,y);ctx.lineTo(goal.x+goal.w,y);ctx.stroke();
  }

  // Kiper
  ctx.save();
  ctx.translate(keeper.x,keeper.y);
  ctx.fillStyle='#172bda';
  ctx.fillRect(-22,-12,44,52);
  ctx.fillStyle='#f2c6a0';
  ctx.beginPath();ctx.arc(0,-25,16,0,Math.PI*2);ctx.fill();

  ctx.strokeStyle='#f2c6a0';
  ctx.lineWidth=11;ctx.lineCap='round';
  ctx.beginPath();
  ctx.moveTo(-17,0);ctx.lineTo(-39,12);
  ctx.moveTo(17,0);ctx.lineTo(39,12);
  ctx.stroke();

  ctx.strokeStyle='#202b37';
  ctx.lineWidth=10;
  ctx.beginPath();
  ctx.moveTo(-10,38);ctx.lineTo(-14,55);
  ctx.moveTo(10,38);ctx.lineTo(14,55);
  ctx.stroke();

  // Sarung tangan
  ctx.fillStyle='#ffe45c';
  ctx.beginPath();ctx.arc(-40,13,9,0,Math.PI*2);ctx.arc(40,13,9,0,Math.PI*2);ctx.fill();
  ctx.restore();

  // Titik penalti
  ctx.fillStyle='white';
  ctx.beginPath();ctx.arc(350,350,4,0,Math.PI*2);ctx.fill();

  // Target bidikan
  if(!moving&&!ended){
    ctx.strokeStyle='#ffe45c';
    ctx.lineWidth=3;
    ctx.beginPath();ctx.arc(aim.x,aim.y,13,0,Math.PI*2);ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(aim.x-20,aim.y);ctx.lineTo(aim.x+20,aim.y);
    ctx.moveTo(aim.x,aim.y-20);ctx.lineTo(aim.x,aim.y+20);
    ctx.stroke();
  }

  // Bola
  ctx.fillStyle='#fff';
  ctx.beginPath();ctx.arc(ball.x,ball.y,ball.r,0,Math.PI*2);ctx.fill();
  ctx.strokeStyle='#17231c';ctx.lineWidth=2;ctx.stroke();
  ctx.fillStyle='#17231c';
  ctx.beginPath();ctx.arc(ball.x,ball.y,4,0,Math.PI*2);ctx.fill();
}

function resetBall(){
  ball={x:350,y:350,r:12};
  keeper={x:350,y:165};
  aim={x:350,y:100};
}

function setAim(e){
  if(moving||ended)return;
  const rect=canvas.getBoundingClientRect();
  const x=(e.clientX-rect.left)*W/rect.width;
  const y=(e.clientY-rect.top)*H/rect.height;

  // Bidikan diarahkan ke area gawang.
  aim.x=Math.max(goal.x+18,Math.min(goal.x+goal.w-18,x));
  aim.y=Math.max(goal.y+18,Math.min(goal.y+goal.h-18,y));
  drawField();
}

canvas.addEventListener('pointerdown',setAim);

function kick(){
  if(moving||ended)return;

  shots++;
  moving=true;
  resultTimer=0;

  // Kiper menebak arah tendangan dengan sedikit variasi.
  const keeperTargetX=350+(Math.random()-.5)*270;
  const keeperTargetY=165+(Math.random()-.5)*35;

  kickData={
    startX:ball.x,startY:ball.y,
    targetX:aim.x,targetY:aim.y,
    keeperStartX:keeper.x,keeperStartY:keeper.y,
    keeperTargetX,keeperTargetY,
    t:0, saved:false
  };
  message='TENDANGAN DILEPASKAN!';
  updateUI();
}

function finishShot(){
  const k=kickData;
  const distance=Math.hypot(k.targetX-k.keeper.x,k.targetY-k.keeper.y);
  const onTarget=
    k.targetX>=goal.x+10&&k.targetX<=goal.x+goal.w-10&&
    k.targetY>=goal.y+10&&k.targetY<=goal.y+goal.h-10;

  if(onTarget&&distance<52){
    saves++;
    message='🧤 DITEPIS KIPER!';
  }else if(onTarget){
    goals++;
    message='⚽ GOOOL! Tendangan hebat!';
  }else{
    message='❌ MELEBAR! Coba arah lain.';
  }

  moving=false;
  kickData=null;
  resultTimer=65;
  updateUI();
}

function animate(){
  if(moving&&kickData){
    const k=kickData;
    k.t=Math.min(1,k.t+.035);
    const t=k.t;
    const ease=1-Math.pow(1-t,2);

    ball.x=k.startX+(k.targetX-k.startX)*ease;
    ball.y=k.startY+(k.targetY-k.startY)*ease;

    keeper.x=k.keeperStartX+(k.keeperTargetX-k.keeperStartX)*Math.min(1,t*1.6);
    keeper.y=k.keeperStartY+(k.keeperTargetY-k.keeperStartY)*Math.min(1,t*1.6);

    if(t>=1)finishShot();
  }else if(resultTimer>0){
    resultTimer--;
    if(resultTimer===0){
      if(shots>=5){
        ended=true;
        message='🏆 SELESAI! Kamu mencetak '+goals+' dari 5 penalti!';
        updateUI();
      }else{
        resetBall();
        message='Siap untuk tendangan berikutnya!';
        updateUI();
      }
    }
  }

  drawField();
  requestAnimationFrame(animate);
}

document.getElementById('kick').addEventListener('click',kick);

document.getElementById('restart').addEventListener('click',()=>{
  goals=0;shots=0;saves=0;
  moving=false;ended=false;kickData=null;resultTimer=0;
  resetBall();
  message='Pilih arah tendangan, lalu tekan TENDANG!';
  updateUI();
  drawField();
});

updateUI();
drawField();
animate();
</script>
</body>
</html>
```