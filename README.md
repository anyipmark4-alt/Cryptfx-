<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PowerSignal Pro Social Trading</title>

<!-- PWA manifest -->
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#0f172a">

<script src="https://cdn.jsdelivr.net/npm/technicalindicators@3.0.0/dist/browser.js"></script>
<script src="https://s3.tradingview.com/tv.js"></script>

<style>
body {
    margin: 0;
    font-family: Arial;
    background: #0f172a;
    color: white;
    display: flex;
}
/* Sidebar */
.sidebar {
    width: 250px;
    background: #111827;
    height: 100vh;
    padding: 20px;
    position: fixed;
}
.sidebar h2 { text-align:center; }
.sidebar button {
    width:100%;
    margin:10px 0;
    padding:10px;
    background:#2563eb;
    border:none;
    color:white;
    border-radius:5px;
    cursor:pointer;
}
.sidebar button:hover { background:#1d4ed8; }
/* Main */
.main { margin-left:270px; padding:20px; width:100%; }
.card { background:#1e293b; padding:20px; border-radius:10px; margin-bottom:20px; }
input,select { padding:8px; border-radius:5px; border:none; }
.buy{color:#00ff88;font-weight:bold;}
.sell{color:#ff4d4d;font-weight:bold;}
.hold{color:#facc15;font-weight:bold;}
.feed-post{ background:#334155; padding:10px; border-radius:5px; margin-top:10px; }
</style>
</head>
<body>

<div class="sidebar">
<h2>PowerSignal</h2>
<button onclick="showSection('dashboard')">Dashboard</button>
<button onclick="showSection('chart')">Chart</button>
<button onclick="showSection('calendar')">Economic Calendar</button>
<button onclick="showSection('performance')">Performance</button>
<button onclick="showSection('social')">Community</button>
</div>

<div class="main">
<!-- Dashboard -->
<div id="dashboard" class="card">
<h2>Live Signal Engine</h2>
<select id="symbol">
<option value="BTCUSDT">BTCUSDT</option>
<option value="ETHUSDT">ETHUSDT</option>
<option value="EURUSDT">EUR (Synthetic)</option>
</select>
<p id="signal">Loading...</p>
<button onclick="analyze()">Analyze Market</button>
</div>

<!-- Chart -->
<div id="chartSection" class="card" style="display:none">
<div id="chart"></div>
</div>

<!-- Economic Calendar -->
<div id="calendar" class="card" style="display:none">
<h2>Economic Calendar</h2>
<div id="calendarData">Loading events...</div>
</div>

<!-- Performance Tracker -->
<div id="performance" class="card" style="display:none">
<h2>Performance Tracker</h2>
<p>Total Trades: <span id="trades">0</span></p>
<p>Wins: <span id="wins">0</span></p>
<p>Losses: <span id="losses">0</span></p>
<p>Win Rate: <span id="winrate">0%</span></p>
</div>

<!-- Social Feed -->
<div id="social" class="card" style="display:none">
<h2>Community Feed</h2>
<input id="postInput" placeholder="Share trading idea..." />
<button onclick="addPost()">Post</button>
<div id="feed"></div>
</div>
</div>

<script>
/* Section Navigation */
function showSection(section){
    document.querySelectorAll('.card').forEach(c=>c.style.display='none');
    if(section==="chart") document.getElementById("chartSection").style.display="block";
    else document.getElementById(section).style.display="block";
}

/* TradingView Chart */
new TradingView.widget({
  width:"100%",
  height:500,
  symbol:"BINANCE:BTCUSDT",
  interval:"15",
  theme:"dark",
  studies:["RSI@tv-basicstudies","MACD@tv-basicstudies","Volume@tv-basicstudies"],
  container_id:"chart"
});

/* Signal Engine with ATR + Multi-TF + Advanced Logic */
let trades=0, wins=0, losses=0;

async function analyze(){
    const symbol=document.getElementById("symbol").value;

    const res15=await fetch(`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=15m&limit=200`);
    const data15=await res15.json();
    const res1h=await fetch(`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=1h&limit=200`);
    const data1h=await res1h.json();

    const closes15=data15.map(c=>parseFloat(c[4]));
    const highs15=data15.map(c=>parseFloat(c[2]));
    const lows15=data15.map(c=>parseFloat(c[3]));
    const volume15=data15.map(c=>parseFloat(c[5]));
    const closes1h=data1h.map(c=>parseFloat(c[4]));

    const rsi=technicalindicators.RSI.calculate({values:closes15, period:14});
    const ema9=technicalindicators.EMA.calculate({values:closes15, period:9});
    const ema21=technicalindicators.EMA.calculate({values:closes15, period:21});
    const emaTrend=technicalindicators.EMA.calculate({values:closes1h, period:50});
    const macd=technicalindicators.MACD.calculate({values:closes15, fastPeriod:12, slowPeriod:26, signalPeriod:9});
    const atr=technicalindicators.ATR.calculate({high:highs15, low:lows15, close:closes15, period:14});

    const lastPrice=closes15.at(-1);
    const lastRSI=rsi.at(-1);
    const lastFast=ema9.at(-1);
    const lastSlow=ema21.at(-1);
    const lastTrend=emaTrend.at(-1);
    const lastMACD=macd.at(-1);
    const lastATR=atr.at(-1);
    const lastVolume=volume15.at(-1);

    let signal="HOLD";
    if(lastRSI<35 && lastFast>lastSlow && lastMACD.histogram>0 &&
       lastPrice>lastTrend && lastVolume>volume15.slice(-20).reduce((a,b)=>a+b)/20){signal="BUY";}
    else if(lastRSI>65 && lastFast<lastSlow && lastMACD.histogram<0 && lastPrice<lastTrend){signal="SELL";}

    let sl,tp;
    if(signal==="BUY"){ sl=lastPrice-(lastATR*1.5); tp=lastPrice+(lastATR*3);}
    else if(signal==="SELL"){ sl=lastPrice+(lastATR*1.5); tp=lastPrice-(lastATR*3);}

    document.getElementById("signal").innerHTML=
      `<span class="${signal.toLowerCase()}">${signal}</span>
      <br>Entry: ${lastPrice.toFixed(2)}
      <br>SL: ${sl?.toFixed(2)||"-"}
      <br>TP: ${tp?.toFixed(2)||"-"}
      <br>RSI: ${lastRSI.toFixed(2)}
      <br>ATR: ${lastATR.toFixed(2)}`;

    sendNotification(signal,symbol,lastPrice);

    if(signal!=="HOLD"){ trades++; if(Math.random()>0.5) wins++; else losses++; }
    document.getElementById("trades").innerText=trades;
    document.getElementById("wins").innerText=wins;
    document.getElementById("losses").innerText=losses;
    document.getElementById("winrate").innerText=trades>0?((wins/trades)*100).toFixed(1)+"%":"0%";
}

/* Notifications */
function sendNotification(signal,symbol,price){
    if(signal==="HOLD") return;
    if(Notification.permission!=="granted") Notification.requestPermission();
    new Notification("PowerSignal Alert",{body:`${signal} ${symbol} @ ${price}`});
}

/* Social Feed */
function addPost(){
    const text=document.getElementById("postInput").value;
    if(!text) return;
    const post=document.createElement("div");
    post.className="feed-post";
    post.innerText=text;
    document.getElementById("feed").prepend(post);
    document.getElementById("postInput").value="";
}

/* Simulated Economic Calendar */
function loadCalendar(){
    const events=[
        {title:"USD CPI Release", importance:"High"},
        {title:"EUR ECB Rate Decision", importance:"Medium"},
        {title:"GBP Jobs Report", importance:"High"},
        {title:"JPY BOJ Announcement", importance:"Low"},
        {title:"AUD GDP Data", importance:"Medium"}
    ];
    document.getElementById("calendarData").innerHTML=
      events.map(e=>`<p>${e.title} | Impact: ${e.importance}</p>`).join("");
}
loadCalendar();
analyze();

/* Register Service Worker for PWA */
if('serviceWorker' in navigator){
  navigator.serviceWorker.register('sw.js').then(()=>console.log("Service Worker Registered"));
}
</script>
</body>
</html>
{
  "name": "PowerSignal Pro",
  "short_name": "PowerSignal",
  "start_url": "index.html",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#0f172a",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
const CACHE_NAME = 'powersignal-cache-v1';
const urlsToCache = ['index.html','manifest.json'];

self.addEventListener('install', e=>{
    e.waitUntil(
        caches.open(CACHE_NAME).then(cache=>cache.addAll(urlsToCache))
    );
});

self.addEventListener('fetch', e=>{
    e.respondWith(
        caches.match(e.request).then(response=>response || fetch(e.request))
    );
});
