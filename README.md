<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Binance Futures Signal Scanner</title>
<style>
*{box-sizing:border-box}
body{margin:0;padding:16px;background:#10151f;color:#e8edf5;
font-family:Arial,sans-serif}
.container{max-width:1100px;margin:auto}
h2{color:#f0b90b;margin-bottom:5px}
.sub{color:#9aa8bb;font-size:13px}
.controls{display:flex;gap:10px;flex-wrap:wrap;margin:18px 0}
button{background:#f0b90b;border:0;padding:11px 18px;
border-radius:7px;font-weight:bold;cursor:pointer}
.status{padding:12px;background:#1c2635;border-radius:8px;
margin-bottom:14px;font-size:13px}
.summary{display:flex;gap:10px;flex-wrap:wrap;margin-bottom:14px}
.box{background:#1c2635;padding:12px 16px;border-radius:8px;flex:1;
min-width:100px}
.box strong{display:block;font-size:21px;margin-top:5px}
.tablewrap{overflow-x:auto;background:#1c2635;border-radius:9px}
table{width:100%;border-collapse:collapse;min-width:850px}
th,td{padding:12px 10px;text-align:right;border-bottom:1px solid #303b4d;
font-size:13px;white-space:nowrap}
th{color:#aab6c7;background:#222e40}
th:first-child,td:first-child{text-align:left}
.long{color:#20d98b;font-weight:bold}
.short{color:#ff626d;font-weight:bold}
.wait{color:#aab6c7}
.small{color:#9aa8bb;font-size:12px;margin-top:15px;line-height:1.6}
.error{color:#ff7777}
</style>
</head>
<body>
<div class="container">
<h2>BINANCE FUTURES SCANNER</h2>
<div class="sub">Top 20 USDT Perpetual | 5 Minute Signals</div>

<div class="controls">
<button onclick="scan()">SCAN NOW</button>
<button onclick="toggleAuto()" id="autoBtn">AUTO: ON</button>
</div>

<div class="status" id="status">Connecting to Binance Futures...</div>

<div class="summary">
<div class="box">PUMP / LONG<strong class="long" id="longCount">0</strong></div>
<div class="box">DUMP / SHORT<strong class="short" id="shortCount">0</strong></div>
<div class="box">WAIT<strong id="waitCount">0</strong></div>
<div class="box">SCANNED<strong id="totalCount">0</strong></div>
</div>

<div class="tablewrap">
<table>
<thead>
<tr>
<th>COIN</th>
<th>24H %</th>
<th>SIGNAL</th>
<th>POWER</th>
<th>RSI</th>
<th>ENTRY</th>
<th>STOP LOSS</th>
<th>TAKE PROFIT</th>
</tr>
</thead>
<tbody id="results">
<tr><td colspan="8" style="text-align:center">Waiting for scan...</td></tr>
</tbody>
</table>
</div>

<div class="small">
<b>Signal rules:</b> EMA 9/21/50, RSI 14, MACD, volume and candle direction.
A signal requires at least 4 of 6 confirmations.<br>
Stop Loss = 1.5 × ATR. Take Profit = 3 × ATR.<br>
Data source: Binance public USD-M Futures API. No API key required.<br>
Signals are indicators, not guaranteed predictions. Always verify before trading.
</div>
</div>

<script>
const API = "https://fapi.binance.com";
const EXCLUDED = ["USDC","BUSD","FDUSD","TUSD","USDE","DAI","USDP"];
let auto = true;
let busy = false;
let timer = null;

async function getJSON(url) {
  const response = await fetch(url);
  if (!response.ok) throw new Error("Binance API error: " + response.status);
  return response.json();
}

function ema(values, period) {
  const k = 2 / (period + 1);
  let result = values[0];
  const out = [result];
  for (let i = 1; i < values.length; i++) {
    result = values[i] * k + result * (1-k);
    out.push(result);
  }
  return out;
}

function rsi(values, period=14) {
  let gain=0, loss=0;
  for(let i=values.length-period-1;i<values.length-1;i++){
    const d=values[i+1]-values[i];
    if(d>0) gain+=d; else loss-=d;
  }
  if(loss===0) return 100;
  const rs=(gain/period)/(loss/period);
  return 100-(100/(1+rs));
}

function average(values) {
  return values.reduce((a,b)=>a+b,0)/values.length;
}

function analyze(rows) {
  // Binance klines: remove currently forming candle
  const c = rows.slice(0,-1);
  if(c.length<60) throw new Error("Insufficient candles");

  const close=c.map(x=>+x[4]);
  const high=c.map(x=>+x[2]);
  const low=c.map(x=>+x[3]);
  const volume=c.map(x=>+x[5]);
  const open=c.map(x=>+x[1]);

  const e9=ema(close,9);
  const e21=ema(close,21);
  const e50=ema(close,50);

  const r=rsi(close,14);

  const fast=ema(close,12);
  const slow=ema(close,26);
  const macd=fast.map((v,i)=>v-slow[i]);
  const sig=ema(macd,9);
  const hist=macd.map((v,i)=>v-sig[i]);

  const i=close.length-1;
  const prev=i-1;

  const tr=[];
  for(let j=1;j<high.length;j++){
    tr.push(Math.max(
      high[j]-low[j],
      Math.abs(high[j]-close[j-1]),
      Math.abs(low[j]-close[j-1])
    ));
  }
  const atr=average(tr.slice(-14));

  const avgVol=average(volume.slice(-21,-1));
  const volumeOK=volume[i]>avgVol*1.2;

  const longChecks=[
    e9[i]>e21[i] && e21[i]>e50[i],
    close[i]>e9[i],
    r>=52 && r<=70,
    hist[i]>0 && hist[i]>hist[prev],
    volumeOK,
    close[i]>open[i]
  ];

  const shortChecks=[
    e9[i]<e21[i] && e21[i]<e50[i],
    close[i]<e9[i],
    r>=30 && r<=48,
    hist[i]<0 && hist[i]<hist[prev],
    volumeOK,
    close[i]<open[i]
  ];

  const L=longChecks.filter(Boolean).length;
  const S=shortChecks.filter(Boolean).length;

  let signal="WAIT", sl=null, tp=null, power=Math.max(L,S);

  if(L>=4 && L>S){
    signal="PUMP / LONG";
    sl=close[i]-1.5*atr;
    tp=close[i]+3*atr;
    power=L;
  } else if(S>=4 && S>L){
    signal="DUMP / SHORT";
    sl=close[i]+1.5*atr;
    tp=close[i]-3*atr;
    power=S;
  }

  return {signal, entry:close[i], sl, tp, rsi:r, power};
}

function fmt(n) {
  if(n===null || n===undefined) return "-";
  return Number(n).toPrecision(7).replace(/\.?0+$/,"");
}

async function scan() {
  if(busy) return;
  busy=true;
  const status=document.getElementById("status");
  status.textContent="Scanning Binance Futures... Please wait.";
  const output=[];

  try {
    const [info,tickers]=await Promise.all([
      getJSON(API+"/fapi/v1/exchangeInfo"),
      getJSON(API+"/fapi/v1/ticker/24hr")
    ]);

    const tickerMap=new Map(tickers.map(t=>[t.symbol,t]));

    const eligible=info.symbols
      .filter(s=>s.status==="TRADING"
        && s.contractType==="PERPETUAL"
        && s.quoteAsset==="USDT"
        && !EXCLUDED.includes(s.baseAsset))
      .map(s=>{
        const t=tickerMap.get(s.symbol);
        return t ? {
          symbol:s.symbol,
          volume:+t.quoteVolume,
          change:+t.priceChangePercent
        } : null;
      })
      .filter(x=>x && x.volume>0)
      .sort((a,b)=>b.volume-a.volume)
      .slice(0,20);

    let next=0;
    async function worker(){
      while(next<eligible.length){
        const item=eligible[next++];
        try{
          const candles=await getJSON(
            API+"/fapi/v1/klines?symbol="+item.symbol+
            "&interval=5m&limit=100"
          );
          const a=analyze(candles);
          output.push({...item,...a});
        }catch(err){
          output.push({...item,signal:"ERROR",error:err.message});
        }
      }
    }

    await Promise.all(Array.from({length:4},()=>worker()));

    output.sort((a,b)=>{
      const rank={ "PUMP / LONG":0, "DUMP / SHORT":1, "WAIT":2, "ERROR":3 };
      return rank[a.signal]-rank[b.signal] || b.power-a.power;
    });

    const tbody=document.getElementById("results");
    tbody.innerHTML="";

    let longs=0, shorts=0, waits=0;

    for(const x of output){
      if(x.signal==="PUMP / LONG") longs++;
      else if(x.signal==="DUMP / SHORT") shorts++;
      else waits++;

      const cls=x.signal==="PUMP / LONG"?"long":
        x.signal==="DUMP / SHORT"?"short":"wait";

      const tr=document.createElement("tr");
      const cells=[
        x.symbol.replace("USDT",""),
        x.change.toFixed(2)+"%",
        x.signal,
        x.power!==undefined?x.power+"/6":"-",
        x.rsi!==undefined?x.rsi.toFixed(1):"-",
        fmt(x.entry),
        fmt(x.sl),
        fmt(x.tp)
      ];

      cells.forEach((value,index)=>{
        const td=document.createElement("td");
        td.textContent=value;
        if(index===2) td.className=cls;
        if(x.error && index===2) td.title=x.error;
        tr.appendChild(td);
      });
      tbody.appendChild(tr);
    }

    document.getElementById("longCount").textContent=longs;
    document.getElementById("shortCount").textContent=shorts;
    document.getElementById("waitCount").textContent=waits;
    document.getElementById("totalCount").textContent=output.length;

    status.textContent="Last scan: "+new Date().toLocaleString()+
      " | Top 20 ranked by 24h quote volume.";

  }catch(err){
    status.innerHTML='<span class="error">'+
      "Could not connect: "+err.message+
      ". Try again or check Binance API access."+"</span>";
  }finally{
    busy=false;
  }
}

function toggleAuto(){
  auto=!auto;
  document.getElementById("autoBtn").textContent=
    "AUTO: "+(auto?"ON":"OFF");
  if(auto) startTimer();
  else if(timer) clearInterval(timer);
}

function startTimer(){
  if(timer) clearInterval(timer);
  timer=setInterval(()=>{if(auto) scan();},300000);
}

scan();
startTimer();
</script>
</body>
</html>
