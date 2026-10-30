require("dotenv").config();

const { Telegraf, Markup } = require("telegraf");
const express = require("express");
const crypto = require("crypto");

const BOT_TOKEN = process.env.BOT_TOKEN;
const WEBAPP_URL = process.env.WEBAPP_URL;
const PORT = process.env.PORT || 3000;

if (!BOT_TOKEN || !WEBAPP_URL) {
  throw new Error("Укажи BOT_TOKEN и WEBAPP_URL в переменных окружения");
}

const bot = new Telegraf(BOT_TOKEN);
const app = express();

app.use(express.json());

const users = new Map();

function getUser(id) {
  if (!users.has(id)) {
    users.set(id, {
      balance: 1000,
      game: null
    });
  }
  return users.get(id);
}

// Проверка Telegram Web App
function verify(initData) {
  if (!initData) return null;

  const params = new URLSearchParams(initData);
  const hash = params.get("hash");
  if (!hash) return null;

  params.delete("hash");

  const checkString = [...params.entries()]
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([k, v]) => `${k}=${v}`)
    .join("\n");

  const secret = crypto
    .createHmac("sha256", "WebAppData")
    .update(BOT_TOKEN)
    .digest();

  const calculated = crypto
    .createHmac("sha256", secret)
    .update(checkString)
    .digest("hex");

  const a = Buffer.from(hash, "hex");
  const b = Buffer.from(calculated, "hex");

  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) {
    return null;
  }

  const authDate = Number(params.get("auth_date"));
  if (!authDate || Date.now() / 1000 - authDate > 86400) {
    return null;
  }

  try {
    const user = JSON.parse(params.get("user") || "{}");
    return user.id ? String(user.id) : null;
  } catch {
    return null;
  }
}

function auth(req, res, next) {
  const id = verify(req.headers["x-telegram-init-data"]);

  if (!id) {
    return res.status(401).json({ error: "Открой игру через Telegram" });
  }

  req.userId = id;
  next();
}

function multiplier(mines, opened) {
  let probability = 1;

  for (let i = 0; i < opened; i++) {
    probability *= (25 - mines - i) / (25 - i);
  }

  return Math.floor((0.97 / probability) * 100) / 100;
}

function state(user) {
  const g = user.game;

  return {
    balance: user.balance,
    game: g ? {
      bet: g.bet,
      mines: g.mines,
      openedCells: g.opened,
      opened: g.opened.length,
      multiplier: g.multiplier,
      win: g.win,
      status: g.status,
      revealedBombs: g.status === "lost" ? g.bombs : []
    } : null
  };
}

app.get("/api/state", auth, (req, res) => {
  res.json(state(getUser(req.userId)));
});

app.post("/api/start", auth, (req, res) => {
  const user = getUser(req.userId);
  const bet = Number(req.body.bet);
  const mines = Number(req.body.mines);

  if (user.game?.status === "playing") {
    return res.status(400).json({ error: "Сначала закончи игру" });
  }

  if (!Number.isSafeInteger(bet) || bet < 1 || bet > 1000000) {
    return res.status(400).json({ error: "Неверная ставка" });
  }

  if (!Number.isInteger(mines) || mines < 1 || mines > 24) {
    return res.status(400).json({ error: "Выбери от 1 до 24 бомб" });
  }

  if (bet > user.balance) {
    return res.status(400).json({ error: "Недостаточно кредитов" });
  }

  const bombs = [];

  while (bombs.length < mines) {
    const n = crypto.randomInt(25);
    if (!bombs.includes(n)) bombs.push(n);
  }

  user.balance -= bet;

  user.game = {
    bet,
    mines,
    bombs,
    opened: [],
    multiplier: 1,
    win: 0,
    status: "playing"
  };

  res.json(state(user));
});

app.post("/api/click", auth, (req, res) => {
  const user = getUser(req.userId);
  const game = user.game;
  const cell = Number(req.body.cell);

  if (!game || game.status !== "playing") {
    return res.status(400).json({ error: "Нет активной игры" });
  }

  if (!Number.isInteger(cell) || cell < 0 || cell > 24) {
    return res.status(400).json({ error: "Неверная клетка" });
  }

  if (game.opened.includes(cell)) {
    return res.status(400).json({ error: "Клетка уже открыта" });
  }

  if (game.bombs.includes(cell)) {
    game.status = "lost";
    game.win = 0;
    return res.json({ ...state(user), result: "bomb" });
  }

  game.opened.push(cell);
  game.multiplier = multiplier(game.mines, game.opened.length);
  game.win = Math.floor(game.bet * game.multiplier);

  res.json({ ...state(user), result: "star" });
});

app.post("/api/cashout", auth, (req, res) => {
  const user = getUser(req.userId);
  const game = user.game;

  if (!game || game.status !== "playing" || !game.opened.length) {
    return res.status(400).json({ error: "Сначала найди звезду" });
  }

  const payout = game.win;
  game.status = "cashed";
  user.balance += payout;

  res.json({ ...state(user), result: "cashed", payout });
});

// Игровая страница создаётся прямо здесь
app.get("/", (req, res) => {
  res.type("html").send(`<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<meta name="theme-color" content="#10131c">
<title>MINES</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<style>
*{box-sizing:border-box}
body{margin:0;background:#101522;color:white;font-family:Arial,sans-serif}
.app{max-width:460px;margin:auto;padding:15px}
header{display:flex;justify-content:space-between;align-items:center;margin-bottom:18px}
.logo{font-size:25px;font-weight:900}
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
.cell{aspect-ratio:1;border:0;border-radius:9px;background:linear-gradient(145deg,#405a7e,#293d5a);box-shadow:0 4px #17263b;color:white;font-size:23px}
.cell.star{background:#17694c;box-shadow:0 4px #10442f}
.cell.bomb{background:#a22d43;box-shadow:0 4px #601b2a}
label{display:block;color:#aab7cc;font-size:12px;margin:15px 0 7px}
.control{display:flex;align-items:center;justify-content:space-between;background:#253249;padding:7px;border-radius:10px}
.control button{width:42px;height:38px;border:0;border-radius:8px;background:#405a7e;color:white;font-size:22px}
input{width:60%;background:none;color:white;border:0;text-align:center;font-size:19px;outline:0}
.actions{display:grid;gap:9px;margin-top:17px}
.action{height:49px;border:0;border-radius:10px;font-weight:bold;font-size:15px}
.start{background:#20c77e;color:#062c1d}
.cash{background:#f1b943;color:#382500}
.action:disabled{opacity:.4}
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
<div class="heading"><div><h1>MINES</h1><p>Открывай звёзды и забирай выигрыш</p></div><b>💣 <span id="bombLabel">3</span></b></div>
<div class="stats">
<div class="stat"><span>СТАВКА</span><strong id="betLabel">50</strong></div>
<div class="stat"><span>КОЭФФИЦИЕНТ</span><strong id="mult">x1.00</strong></div>
<div class="stat"><span>ВЫИГРЫШ</span><strong id="win">0</strong></div>
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
const tg=window.Telegram.WebApp;
tg.ready();tg.expand();

const $=id=>document.getElementById(id);
let gameState=null,bet=50,mines=3,busy=false;

function msg(t){$("message").textContent=t}
function num(n){return Math.floor(n||0).toLocaleString("ru-RU")}

async function api(path,body){
 const response=await fetch("/api/"+path,{
  method:body?"POST":"GET",
  headers:{
   "Content-Type":"application/json",
   "x-telegram-init-data":tg.initData
  },
  body:body?JSON.stringify(body):undefined
 });
 const data=await response.json();
 if(!response.ok)throw new Error(data.error||"Ошибка");
 return data;
}

function render(){
 if(!gameState)return;
 const g=gameState.game;
 const playing=g?.status==="playing";
 $("balance").textContent=num(gameState.balance);
 $("betLabel").textContent=num(g?.bet??bet)+" ⭐";
 $("bombLabel").textContent=g?.mines??mines;
 $("mines").textContent=mines;
 $("mult").textContent="x"+(g?.multiplier??1).toFixed(2);
 $("win").textContent=num(g?.win??0)+" ⭐";
 $("cash").textContent="ЗАБРАТЬ "+num(g?.win??0)+" ⭐";
 $("cash").disabled=busy||!playing||!g.opened;
 $("start").disabled=busy||playing;
 $("start").textContent=playing?"ИГРА ИДЁТ":"ИГРАТЬ";
 $("bet").disabled=busy||playing;
 $("betMinus").disabled=busy||playing;
 $("betPlus").disabled=busy||playing;
 $("mineMinus").disabled=busy||playing||mines<=1;
 $("minePlus").disabled=busy||playing||mines>=24;
 $("board").innerHTML="";
 for(let i=0;i<25;i++){
  const b=document.createElement("button");
  b.className="cell";
  const opened=g?.openedCells?.includes(i);
  const bomb=g?.revealedBombs?.includes(i);
  if(opened){b.textContent="⭐";b.classList.add("star");b.disabled=true}
  else if(bomb){b.textContent="💣";b.classList.add("bomb");b.disabled=true}
  else{
   b.disabled=!playing||busy;
   if(g?.status==="lost"){b.textContent="·";b.disabled=true}
   else b.onclick=()=>clickCell(i);
  }
  $("board").appendChild(b);
 }
}

async function run(fn){
 if(busy)return;
 busy=true;render();
 try{await fn()}
 catch(e){msg(e.message)}
 finally{busy=false;render()}
}

async function clickCell(cell){
 await run(async()=>{
  gameState=await api("click",{cell});
  if(gameState.result==="bomb")msg("💣 БОМБА! Ставка и выигрыш сгорели.");
  else msg("⭐ Звезда! Можешь продолжить или забрать выигрыш.");
 });
}

$("start").onclick=()=>run(async()=>{
 bet=Math.floor(Number($("bet").value));
 if(!Number.isSafeInteger(bet)||bet<1)throw Error("Неверная ставка");
 gameState=await api("start",{bet,mines});
 msg("Игра началась! Найди звезду ⭐");
});

$("cash").onclick=()=>run(async()=>{
 const result=await api("cashout");
 gameState=result;
 msg("🎉 Забрал "+num(result.payout)+" виртуальных кредитов!");
});

$("betMinus").onclick=()=>{$("bet").value=bet=Math.max(1,bet-10)};
$("betPlus").onclick=()=>{$("bet").value=bet=Math.min(1000000,bet+10)};
$("bet").onchange=()=>{bet=Math.max(1,Math.floor(Number($("bet").value)||1));$("bet").value=bet};
$("mineMinus").onclick=()=>{mines=Math.max(1,mines-1);render()};
$("minePlus").onclick=()=>{mines=Math.min(24,mines+1);render()};

api("state").then(s=>{gameState=s;render()}).catch(e=>msg(e.message));
</script>
</body>
</html>`);
});

bot.start(ctx => ctx.reply(
  "💣 Добро пожаловать в MINES!\n\n" +
  "⭐ Открывай звёзды.\n" +
  "💣 Избегай бомб.\n" +
  "💰 Забирай виртуальные выигрыши.",
  Markup.inlineKeyboard([
    [Markup.button.webApp("🎮 ИГРАТЬ", WEBAPP_URL)]
  ])
));

bot.command("game", ctx => ctx.reply(
  "Запускай свою игру MINES:",
  Markup.inlineKeyboard([
    [Markup.button.webApp("🎮 ОТКРЫТЬ MINES", WEBAPP_URL)]
  ])
));

app.listen(PORT, () => {
  console.log("Игра запущена на порту " + PORT);
});

bot.launch();
console.log("Telegram бот запущен");

process.once("SIGINT", () => bot.stop("SIGINT"));
process.once("SIGTERM", () => bot.stop("SIGTERM"));
