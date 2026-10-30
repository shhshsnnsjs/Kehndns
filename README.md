
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
<meta name="theme-color" content="#101522">
<title>MINES</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<style>
*{box-sizing:border-box}
body{margin:0;background:#101522;color:white;font-family:Arial,sans-serif}
.app{max-width:460px;margin:auto;padding:15px}
header{display:flex;justify-content:space-between;align-items:center;margin-bottom:18px}
.logo{font-size:25px;font-weight:900;letter-spacing:2px}
.balance{background:#202b40;padding:10px 14px;border-radius:12px}
.card{background:#192235;border:1px solid #34435c;border-radius:18px;padding:15px}
.heading{display:flex;justify-content:space-between;align-items:center}
h1{font-size:26px;margin:5px 0}
p{color:#9cabc3;font-size:12px}
.stats{display:grid;grid-template-columns:repeat(3,1fr);gap:7px;margin:15px 0}
.stat{background:#253249;border-radius:10px;padding:11px 3px;text-align:center}
.stat span{display:block;color:#9cabc3;font-size:9px;margin-bottom:7px}
.stat strong{font-size:14px}
.board{display:grid;grid-template-columns:repeat(5,1fr);gap:7px;background:#111a29;padding:9px;border-radius:13px}
.cell{aspect-ratio:1;border:0;border-radius:9px;background:linear-gradient(145deg,#405a7e,#293d5a);box-shadow:0 4px #17263b;color:white;font-size:24px;cursor:pointer}
.cell:disabled{cursor:default}
.cell.star{background:#17694c;box-shadow:0 4px #10442f}
.cell.bomb{background:#a22d43;box-shadow:0 4px #601b2a}
.cell.empty{opacity:.5}
label{display:block;color:#aab7cc;font-size:12px;margin:15px 0 7px}
.control{display:flex;align-items:center;justify-content:space-between;background:#253249;padding:7px;border-radius:10px}
.control button{width:42px;height:38px;border:0;border-radius:8px;background:#405a7e;color:white;font-size:22px;cursor:pointer}
input{width:60%;background:none;color:white;border:0;text-align:center;font-size:19px;outline:0}
.actions{display:grid;gap:9px;margin-top:17px}
.action{height:49px;border:0;border-radius:10px;font-weight:bold;font-size:15px;cursor:pointer}
.start{background:#20c77e;color:#062c1d}
.cash{background:#f1b943;color:#382500}
.action:disabled{opacity:.4;cursor:not-allowed}
#message{text-align:center;color:#afbdd0;font-size:12px;min-height:34px;padding:12px 2px}
</style>
</head>
<body>
<div class="app">
<header>
<div class="logo">💣 MINES</div>
<div class="balance">⭐ <b id="balance">1000</b></div>
</header>

<div class="card">
<div class="heading">
<div><h1>MINES</h1><p>Открывай звёзды и забирай выигрыш</p></div>
<b>💣 <span id="bombLabel">3</span></b>
</div>

<div class="stats">
<div class="stat"><span>СТАВКА</span><strong id="betLabel">50 ⭐</strong></div>
<div class="stat"><span>КОЭФФИЦИЕНТ</span><strong id="mult">x1.00</strong></div>
<div class="stat"><span>ВЫИГРЫШ</span><strong id="win">0 ⭐</strong></div>
</div>

<div class="board" id="board"></div>

<label>РАЗМЕР СТАВКИ</label>
<div class="control">
<button id="betMinus">−</button>
<input id="bet" type="number" min="1" value="50">
<button id="betPlus">+</button>
</div>

<label>КОЛИЧЕСТВО БОМБ</label>
<div class="control">
<button id="mineMinus">−</button>
<b id="mines">3</b>
<button id="minePlus">+</button>
</div>

<div class="actions">
<button class="action start" id="start">ИГРАТЬ</button>
<button class="action cash" id="cash" disabled>ЗАБРАТЬ 0 ⭐</button>
</div>

<div id="message">Выбери ставку и количество бомб</div>
</div>
</div>

<script>
const tg=window.Telegram?.WebApp;
tg?.ready();
tg?.expand();

const $=id=>document.getElementById(id);

let balance=1000;
let bet=50;
let mines=3;
let bombs=[];
let opened=[];
let multiplier=1;
let win=0;
let status="ready";

function fmt(n){
 return Math.floor(n).toLocaleString("ru-RU");
}

function message(text){
 $("message").textContent=text;
}

function calculateMultiplier(mines,opened){
 let probability=1;
 for(let i=0;i<opened;i++){
  probability*=(25-mines-i)/(25-i);
 }
 return Math.floor((0.97/probability)*100)/100;
}

function render(){
 $("balance").textContent=fmt(balance);
 $("betLabel").textContent=fmt(bet)+" ⭐";
 $("bombLabel").textContent=status==="playing"?mines:mines;
 $("mines").textContent=mines;
 $("mult").textContent="x"+multiplier.toFixed(2);
 $("win").textContent=fmt(win)+" ⭐";
 $("cash").textContent="ЗАБРАТЬ "+fmt(win)+" ⭐";

 const playing=status==="playing";

 $("start").disabled=playing;
 $("start").textContent=playing?"ИГРА ИДЁТ":
  status==="ready"?"ИГРАТЬ":"НОВАЯ ИГРА";

 $("cash").disabled=!playing||opened.length===0;

 $("bet").disabled=playing;
 $("betMinus").disabled=playing||bet<=1;
 $("betPlus").disabled=playing;
 $("mineMinus").disabled=playing||mines<=1;
 $("minePlus").disabled=playing||mines>=24;

 $("board").innerHTML="";

 for(let i=0;i<25;i++){
  const cell=document.createElement("button");
  cell.className="cell";

  if(opened.includes(i)){
   cell.textContent="⭐";
   cell.classList.add("star");
   cell.disabled=true;
  }else if(status==="lost"&&bombs.includes(i)){
   cell.textContent="💣";
   cell.classList.add("bomb");
   cell.disabled=true;
  }else if(status==="lost"){
   cell.textContent="·";
   cell.classList.add("empty");
   cell.disabled=true;
  }else{
   cell.disabled=!playing;
   cell.onclick=()=>openCell(i);
  }

  $("board").appendChild(cell);
 }
}

function startGame(){
 bet=Math.floor(Number($("bet").value));

 if(!Number.isSafeInteger(bet)||bet<1){
  message("Введи правильную ставку");
  return;
 }

 if(bet>balance){
  message("Недостаточно кредитов");
  return;
 }

 bombs=[];

 while(bombs.length<mines){
  const n=Math.floor(Math.random()*25);
  if(!bombs.includes(n))bombs.push(n);
 }

 balance-=bet;
 opened=[];
 multiplier=1;
 win=0;
 status="playing";

 message("Игра началась! Найди звезду ⭐");
 render();
}

function openCell(i){
 if(status!=="playing"||opened.includes(i))return;

 if(bombs.includes(i)){
  status="lost";
  win=0;
  message("💣 БОМБА! Ставка и накопленный выигрыш сгорели.");
  render();
  return;
 }

 opened.push(i);
 multiplier=calculateMultiplier(mines,opened.length);
 win=Math.floor(bet*multiplier);

 message("⭐ Звезда найдена! Продолжай или забери выигрыш.");
 render();
}

function cashout(){
 if(status!=="playing"||opened.length===0)return;

 balance+=win;
 status="cashed";

 message("🎉 Ты забрал "+fmt(win)+" виртуальных кредитов!");
 render();
}

$("start").onclick=startGame;
$("cash").onclick=cashout;

$("betMinus").onclick=()=>{
 bet=Math.max(1,bet-10);
 $("bet").value=bet;
 render();
};

$("betPlus").onclick=()=>{
 bet=Math.min(1000000,bet+10);
 $("bet").value=bet;
 render();
};

$("bet").onchange=()=>{
 bet=Math.max(1,Math.floor(Number($("bet").value)||1));
 $("bet").value=bet;
 render();
};

$("mineMinus").onclick=()=>{
 mines=Math.max(1,mines-1);
 render();
};

$("minePlus").onclick=()=>{
 mines=Math.min(24,mines+1);
 render();
};

render();
</script>
</body>
</html>
