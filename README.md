
MINES TELEGRAM — FULL PROJECT

==============================
1. package.json
==============================

{
  "name": "telegram-mines",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "bot": "node bot.js"
  },
  "dependencies": {
    "dotenv": "^16.4.5",
    "express": "^4.21.2",
    "node-telegram-bot-api": "^0.66.0"
  }
}

==============================
2. .env
==============================

PORT=3000
BOT_TOKEN=YOUR_BOT_TOKEN
PUBLIC_URL=https://your-domain.com

==============================
3. server.js
==============================

const express = require("express");
const crypto = require("crypto");
const fs = require("fs");
require("dotenv").config();

const app = express();
app.use(express.json());
app.use(express.static("public"));

const DB = "data.json";
const START_BALANCE = 1000;
const EDGE = 0.03;

let data = {users:{},games:{}};

if(fs.existsSync(DB)){
  data = JSON.parse(fs.readFileSync(DB,"utf8"));
}

function save(){
  fs.writeFileSync(DB,JSON.stringify(data,null,2));
}

function getUser(id){
  if(!data.users[id]){
    data.users[id]={balance:START_BALANCE};
  }
  return data.users[id];
}

function telegramUser(req){
  const initData=req.headers["x-telegram-init-data"];

  if(!initData){
    if(process.env.NODE_ENV==="production") return null;
    return {id:10001,first_name:"Demo"};
  }

  const params=new URLSearchParams(initData);
  const hash=params.get("hash");

  if(!hash || !process.env.BOT_TOKEN) return null;

  params.delete("hash");

  const check=[...params.entries()]
    .sort(([a],[b])=>a.localeCompare(b))
    .map(([k,v])=>`${k}=${v}`).join("\n");

  const secret=crypto.createHmac("sha256","WebAppData")
    .update(process.env.BOT_TOKEN).digest();

  const expected=crypto.createHmac("sha256",secret)
    .update(check).digest("hex");

  const a=Buffer.from(hash,"hex");
  const b=Buffer.from(expected,"hex");

  if(a.length!==b.length || !crypto.timingSafeEqual(a,b)){
    return null;
  }

  const authDate=Number(params.get("auth_date"));

  if(!authDate || Date.now()/1000-authDate>86400){
    return null;
  }

  try{
    return JSON.parse(params.get("user"));
  }catch{
    return null;
  }
}

function auth(req,res,next){
  const user=telegramUser(req);

  if(!user || !user.id){
    return res.status(401).json({error:"Telegram authorization failed"});
  }

  req.user=user;
  next();
}

function multiplier(mines,opened){
  let m=1;

  for(let i=0;i<opened;i++){
    m *= (25-i)/(25-mines-i);
  }

  return Math.floor(m*(1-EDGE)*100)/100;
}

function gameResponse(g){
  return {
    bet:g.bet,
    mines:g.mines,
    revealed:g.revealed,
    status:g.status,
    multiplier:multiplier(g.mines,g.revealed.length),
    payout:g.payout || 0,
    board:["lost","cashed_out"].includes(g.status)
      ? g.minePositions : null
  };
}

app.get("/api/me",auth,(req,res)=>{
  const user=getUser(req.user.id);
  const game=data.games[req.user.id];

  save();

  res.json({
    balance:user.balance,
    game:game && game.status==="active"
      ? gameResponse(game) : null
  });
});

app.post("/api/start",auth,(req,res)=>{
  const user=getUser(req.user.id);

  const bet=Number(req.body.bet);
  const mines=Number(req.body.mines);

  if(!Number.isFinite(bet) || bet<1 || bet>100000 ||
     Math.round(bet*100)!==bet*100){
    return res.status(400).json({error:"Invalid bet"});
  }

  if(![2,3,5,7].includes(mines)){
    return res.status(400).json({error:"Invalid mines"});
  }

  if(data.games[req.user.id]?.status==="active"){
    return res.status(400).json({error:"Finish current game"});
  }

  if(user.balance<bet){
    return res.status(400).json({error:"Not enough credits"});
  }

  user.balance-=bet;

  const tiles=Array.from({length:25},(_,i)=>i);

  for(let i=24;i>0;i--){
    const j=crypto.randomInt(i+1);
    [tiles[i],tiles[j]]=[tiles[j],tiles[i]];
  }

  const game={
    bet,
    mines,
    minePositions:tiles.slice(0,mines),
    revealed:[],
    status:"active",
    payout:0
  };

  data.games[req.user.id]=game;

  save();

  res.json({
    balance:user.balance,
    game:gameResponse(game)
  });
});

app.post("/api/reveal",auth,(req,res)=>{
  const game=data.games[req.user.id];

  if(!game || game.status!=="active"){
    return res.status(400).json({error:"No active game"});
  }

  const tile=Number(req.body.tile);

  if(!Number.isInteger(tile) || tile<0 || tile>24){
    return res.status(400).json({error:"Invalid tile"});
  }

  if(game.revealed.includes(tile)){
    return res.status(400).json({error:"Already opened"});
  }

  if(game.minePositions.includes(tile)){
    game.status="lost";
    save();

    return res.json({
      hit:true,
      balance:getUser(req.user.id).balance,
      game:gameResponse(game)
    });
  }

  game.revealed.push(tile);
  save();

  res.json({
    hit:false,
    balance:getUser(req.user.id).balance,
    game:gameResponse(game)
  });
});

app.post("/api/cashout",auth,(req,res)=>{
  const game=data.games[req.user.id];

  if(!game || game.status!=="active" ||
     game.revealed.length===0){
    return res.status(400).json({error:"Open a safe tile first"});
  }

  const user=getUser(req.user.id);

  game.payout=Math.floor(
    game.bet*multiplier(game.mines,game.revealed.length)*100
  )/100;

  game.status="cashed_out";
  user.balance+=game.payout;

  save();

  res.json({
    balance:user.balance,
    game:gameResponse(game)
  });
});

app.listen(process.env.PORT || 3000,()=>{
  console.log("MINES SERVER STARTED");
});

==============================
4. bot.js
==============================

require("dotenv").config();

const TelegramBot=require("node-telegram-bot-api");

const bot=new TelegramBot(
  process.env.BOT_TOKEN,
  {polling:true}
);

bot.onText(/\/start/,async msg=>{
  bot.sendMessage(
    msg.chat.id,
    "💣 Добро пожаловать в MINES!",
    {
      reply_markup:{
        inline_keyboard:[[
          {
            text:"🎮 ИГРАТЬ",
            web_app:{
              url:process.env.PUBLIC_URL
            }
          }
        ]]
      }
    }
  );
});

console.log("BOT STARTED");

==============================
5. public/index.html
==============================

<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>MINES</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>

<style>

*{box-sizing:border-box}

body{
margin:0;
background:#050506;
color:white;
font-family:Arial,sans-serif;
}

.app{
max-width:500px;
margin:auto;
padding:15px;
}

.logo{
background:#191a1c;
border-radius:14px;
padding:20px;
text-align:center;
font-size:27px;
font-weight:900;
margin-bottom:25px;
}

.balance{
font-size:22px;
margin:20px 5px;
}

.settings{
display:grid;
grid-template-columns:1fr 1fr;
gap:12px;
margin-bottom:20px;
}

.settings div,
.settings select{
background:#252629;
color:white;
border:2px solid #34363b;
border-radius:14px;
padding:18px;
text-align:center;
font-size:20px;
}

.board{
display:grid;
grid-template-columns:repeat(5,1fr);
gap:7px;
}

.tile{
aspect-ratio:1;
background:#191a1c;
border:2px solid #303136;
border-radius:9px;
color:#444;
font-size:30px;
cursor:pointer;
}

.tile.safe{
background:#263b19;
border-color:#9be522;
}

.tile.mine{
background:#492126;
border-color:#ff5555;
}

.multipliers{
display:grid;
grid-template-columns:repeat(5,1fr);
gap:6px;
margin:18px 0 25px;
}

.mult{
background:#252629;
padding:15px 2px;
border-radius:12px;
text-align:center;
color:#aeb2bd;
font-weight:bold;
}

.mult.active{
background:#526e19;
color:white;
}

.bet-panel{
background:#191a1c;
border:2px solid #292a2d;
border-radius:18px;
padding:12px;
display:grid;
grid-template-columns:1.2fr .8fr;
gap:10px;
}

.bet-input{
background:#34363a;
border-radius:12px;
height:75px;
display:flex;
align-items:center;
padding:5px;
}

.bet-input input{
width:100%;
min-width:0;
background:transparent;
border:none;
color:white;
font-size:20px;
outline:none;
}

.bet-input button{
background:transparent;
border:none;
color:white;
font-size:25px;
}

.bet-button{
background:#b5ed0b;
color:black;
border:none;
border-radius:14px;
font-size:25px;
font-weight:bold;
}

.quick{
display:flex;
gap:8px;
}

.quick button{
flex:1;
background:#303136;
border:none;
color:white;
padding:12px;
border-radius:10px;
font-weight:bold;
}

.cashout,.newround{
width:100%;
padding:18px;
margin-top:15px;
border:none;
border-radius:13px;
background:#303136;
color:white;
font-size:19px;
font-weight:bold;
}

.cashout:not(:disabled){
background:#b5ed0b;
color:black;
}

.cashout:disabled{
opacity:.4;
}

#message{
text-align:center;
color:#b5ed0b;
margin-top:20px;
min-height:20px;
}

</style>
</head>

<body>

<div class="app">

<div class="logo">100HP ❤️ × MINES</div>

<div class="balance">
💳 <span id="balance">1000</span> ₽
</div>

<div class="settings">
<div>⭐ MINES</div>

<select id="mines">
<option value="2">💣 2</option>
<option value="3" selected>💣 3</option>
<option value="5">💣 5</option>
<option value="7">💣 7</option>
</select>
</div>

<div id="board" class="board"></div>

<div id="multipliers" class="multipliers"></div>

<div class="bet-panel">

<div class="bet-input">
<input id="bet" type="number" value="25" min="1">
<button id="minus">−</button>
<button id="plus">+</button>
</div>

<button id="start" class="bet-button">Bet ★</button>

<div class="quick">
<button id="half">/2</button>
<button id="double">×2</button>
</div>

</div>

<button id="cashout" class="cashout" disabled>
CASH OUT · <span id="potential">0.00</span>
</button>

<button id="newround" class="newround" hidden>
НОВЫЙ РАУНД
</button>

<p id="message">Выбери ставку и нажми Bet</p>

</div>

<script src="app.js"></script>

</body>
</html>

==============================
6. public/app.js
==============================

const tg=window.Telegram?.WebApp;

if(tg){
  tg.ready();
  tg.expand();
}

const $=id=>document.getElementById(id);

const initData=tg?.initData || "";

let game=null;
let busy=false;

async function api(path,body){
  const r=await fetch("/api/"+path,{
    method:body?"POST":"GET",
    headers:{
      "Content-Type":"application/json",
      "X-Telegram-Init-Data":initData
    },
    body:body?JSON.stringify(body):undefined
  });

  const d=await r.json();

  if(!r.ok) throw new Error(d.error || "Error");

  return d;
}

function message(text){
  $("message").textContent=text;
}

function balance(value){
  $("balance").textContent=Number(value).toLocaleString("ru-RU");
}

function render(){

  const board=$("board");
  board.innerHTML="";

  for(let i=0;i<25;i++){

    const button=document.createElement("button");

    button.className="tile";

    const safe=game?.revealed.includes(i);
    const mine=game?.board?.includes(i);

    if(safe){
      button.classList.add("safe");
      button.textContent="💎";
    }
    else if(mine){
      button.classList.add("mine");
      button.textContent="💣";
    }
    else{
      button.textContent="♡";
    }

    button.disabled=
      busy ||
      !game ||
      game.status!=="active" ||
      safe;

    button.onclick=()=>reveal(i);

    board.appendChild(button);
  }

  const mines=Number($("mines").value);

  $("multipliers").innerHTML=
    Array.from({length:5},(_,i)=>{

      let m=1;

      for(let j=0;j<=i;j++){
        m*=(25-j)/(25-mines-j);
      }

      m=Math.floor(m*.97*100)/100;

      return `
        <div class="mult ${
          game?.revealed.length===i+1 &&
          game.status==="active" ? "active" : ""
        }">
          ×${m.toFixed(2)}
        </div>
      `;

    }).join("");

  const active=game?.status==="active";

  $("start").disabled=busy || active;

  $("cashout").disabled=
    busy || !active || !game.revealed.length;

  $("potential").textContent=
    game ? (game.bet*game.multiplier).toFixed(2) : "0.00";

  $("newround").hidden=!(game && !active);

  $("bet").disabled=!!active;
  $("mines").disabled=!!active;
}

async function refresh(){

  try{
    const d=await api("me");

    balance(d.balance);
    game=d.game;

    render();

  }catch(e){
    message(e.message);
  }
}

async function start(){

  busy=true;
  render();

  try{

    const d=await api("start",{
      bet:Number($("bet").value),
      mines:Number($("mines").value)
    });

    balance(d.balance);
    game=d.game;

    message("Раунд начался! Открывай клетки.");

    tg?.HapticFeedback?.impactOccurred("light");

  }catch(e){
    message(e.message);
  }

  busy=false;
  render();
}

async function reveal(tile){

  if(busy || !game) return;

  busy=true;
  render();

  try{

    const d=await api("reveal",{tile});

    balance(d.balance);
    game=d.game;

    if(d.hit){

      message("💥 МИНА! Раунд проигран.");

      tg?.HapticFeedback?.impactOccurred("heavy");

    }else{

      message(
        "💎 Безопасно! Коэффициент ×"+
        game.multiplier.toFixed(2)
      );

      tg?.HapticFeedback?.impactOccurred("light");
    }

  }catch(e){
    message(e.message);
  }

  busy=false;
  render();
}

async function cashout(){

  busy=true;
  render();

  try{

    const d=await api("cashout");

    balance(d.balance);
    game=d.game;

    message(
      "🎉 Вы забрали "+
      game.payout.toFixed(2)+
      " кредитов!"
    );

  }catch(e){
    message(e.message);
  }

  busy=false;
  render();
}

$("start").onclick=start;

$("cashout").onclick=cashout;

$("newround").onclick=()=>{
  game=null;
  message("Начни новый раунд!");
  render();
};

$("minus").onclick=()=>{
  $("bet").value=Math.max(
    1,Number($("bet").value)-1
  );
};

$("plus").onclick=()=>{
  $("bet").value=Math.min(
    100000,Number($("bet").value)+1
  );
};

$("half").onclick=()=>{
  $("bet").value=Math.max(
    1,Math.floor(Number($("bet").value)/2)
  );
};

$("double").onclick=()=>{
  $("bet").value=Math.min(
    100000,Number($("bet").value)*2
  );
};

$("mines").onchange=render;

refresh();

==============================
7. ЗАПУСК
==============================

В терминале:

npm install

npm start

Во втором терминале:

npm run bot

Открой своего Telegram-бота и напиши:

/start

Важно:
- Для Telegram нужен публичный HTTPS-домен.
- В .env нужно вставить настоящий токен бота.
- Для продакшена используй постоянную базу данных и защиту от повторных запросов.
- Игра использует виртуальные кредиты, без реальных денежных ставок и выплат.
