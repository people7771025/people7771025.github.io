# people7771025.github.io
<!doctype html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>我的投資組合每日追蹤</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;600;800&display=swap" rel="stylesheet">
  <style>
    :root{--bg:#0b1020;--card:#121a36;--text:#e7ecff;--muted:#9fb0ff;--grid:#2a355f}
    *{box-sizing:border-box}
    body{margin:0;background:var(--bg);color:var(--text);font-family:"Noto Sans TC",system-ui,-apple-system,Segoe UI,Roboto}
    header{padding:20px 16px;border-bottom:1px solid var(--grid);position:sticky;top:0;background:linear-gradient(180deg,#0b1020 70%,#0b102000)}
    .wrap{max-width:1100px;margin:0 auto;padding:16px}
    .grid{display:grid;gap:16px}
    @media(min-width:900px){.grid{grid-template-columns:2fr 1fr}}
    .card{background:var(--card);border:1px solid var(--grid);border-radius:16px;padding:16px;box-shadow:0 10px 30px #0004}
    h1{font-size:24px;margin:0 0 6px;font-weight:800}
    h2{font-size:16px;margin:0 0 12px;color:var(--muted);font-weight:600}
    .row{display:flex;gap:10px;align-items:center;flex-wrap:wrap}
    label{font-size:12px;color:var(--muted)}
    input,select{background:#0f1630;color:var(--text);border:1px solid var(--grid);border-radius:10px;padding:8px 10px}
    .kpi{display:grid;grid-template-columns:repeat(3,1fr);gap:12px}
    .kpi .item{background:#0f1630;border:1px solid var(--grid);border-radius:12px;padding:12px}
    .kpi .v{font-size:20px;font-weight:800}
    .legend{display:flex;gap:10px;flex-wrap:wrap}
    .pill{padding:6px 10px;border-radius:999px;background:#0f1630;border:1px solid var(--grid);font-size:12px}
    a{color:#b9c6ff}
    footer{opacity:.7;padding:24px;text-align:center}
    canvas{width:100%;height:340px}
  </style>
</head>
<body>
  <header class="wrap">
    <h1>投資組合每日追蹤（免費版）</h1>
    <div class="row">
      <label>資料來源：<a id="sheetLink" target="_blank">Google 試算表（公開 CSV）</a></label>
    </div>
  </header>

  <main class="wrap grid">
    <section class="card">
      <h2>組合淨值走勢（含台股／上櫃／興櫃／美股／現金）</h2>
      <canvas id="equityChart"></canvas>
      <div class="legend" id="legend"></div>
    </section>

    <aside class="grid">
      <section class="card">
        <h2>摘要（最近一日）</h2>
        <div class="kpi">
          <div class="item"><div class="l">總資產</div><div class="v" id="kpi-total">—</div><div class="l" id="kpi-date">—</div></div>
          <div class="item"><div class="l">日變動</div><div class="v" id="kpi-day">—</div><div class="l" id="kpi-daypct">—</div></div>
          <div class="item"><div class="l">今年以來</div><div class="v" id="kpi-ytd">—</div><div class="l" id="kpi-ytdpct">—</div></div>
        </div>
      </section>
      <section class="card">
        <h2>設定</h2>
        <div class="row">
          <label for="csvUrl">CSV 來源（History 分頁）</label>
          <input id="csvUrl" placeholder="貼上試算表 CSV 連結" />
          <button id="save">儲存</button>
        </div>
        <p style="font-size:12px;opacity:.85">說明：於 Google 試算表「發佈到網路」→ 選取 <b>History</b> 分頁 → 格式 <b>CSV</b>，將連結貼到上方即可。</p>
      </section>
    </aside>
  </main>

  <footer>
    <small>由 Chart.js + Google Sheets CSV 驅動（僅使用免費資源）。資料非即時，預設每日更新一次。</small>
  </footer>

<script>
const state = {csv: localStorage.getItem('pf_csv') || ''}
const csvInput = document.getElementById('csvUrl')
const saveBtn = document.getElementById('save')
const sheetLink = document.getElementById('sheetLink')
const legendBox = document.getElementById('legend')
let chart

function fmt(n){
  if(n===undefined||n===null||isNaN(n)) return '—'
  return n.toLocaleString('zh-TW',{style:'currency',currency:'TWD',maximumFractionDigits:0})
}
function pct(a){
  return (a>=0?'+':'') + (a*100).toFixed(2)+'%'
}

async function loadCSV(){
  if(!state.csv){return}
  sheetLink.href = state.csv
  sheetLink.textContent = 'Google 試算表（CSV）'
  const res = await fetch(state.csv)
  const txt = await res.text()
  const parsed = Papa.parse(txt,{header:true,skipEmptyLines:true})
  const rows = parsed.data
  // 預期欄位：date, total_twd, twse_twd, tpex_twd, emerging_twd, us_twd, cash_twd
  const labels = []
  const series = {total:[], twse:[], tpex:[], emerging:[], us:[], cash:[]}
  rows.forEach(r=>{
    labels.push(r.date)
    series.total.push(+r.total_twd)
    series.twse.push(+r.twse_twd)
    series.tpex.push(+r.tpex_twd)
    series.emerging.push(+r.emerging_twd)
    series.us.push(+r.us_twd)
    series.cash.push(+r.cash_twd)
  })

  // KPI
  if(rows.length){
    const last = rows[rows.length-1]
    const prev = rows[rows.length-2]||last
    const ytd0 = rows.find(r=>r.date.slice(0,4)===last.date.slice(0,4))||rows[0]
    const lastV = +last.total_twd, prevV = +prev.total_twd, ytdV = +ytd0.total_twd
    document.getElementById('kpi-total').textContent = fmt(lastV)
    document.getElementById('kpi-date').textContent = last.date
    document.getElementById('kpi-day').textContent = fmt(lastV-prevV)
    document.getElementById('kpi-daypct').textContent = pct((lastV-prevV)/prevV)
    document.getElementById('kpi-ytd').textContent = fmt(lastV-ytdV)
    document.getElementById('kpi-ytdpct').textContent = pct((lastV-ytdV)/ytdV)
  }

  const ds = (label,data)=>({
    label, data, borderWidth:2, pointRadius:0, tension:.25
  })

  const datasets = [
    ds('總資產', series.total),
    ds('台股上市', series.twse),
    ds('上櫃', series.tpex),
    ds('興櫃', series.emerging),
    ds('美股(折台幣)', series.us),
    ds('現金', series.cash)
  ]

  const ctx = document.getElementById('equityChart')
  chart?.destroy()
  chart = new Chart(ctx,{
    type:'line',
    data:{labels,datasets},
    options:{
      responsive:true,
      maintainAspectRatio:false,
      interaction:{mode:'index',intersect:false},
      scales:{
        x:{grid:{color:'rgba(255,255,255,.06)'}},
        y:{grid:{color:'rgba(255,255,255,.06)'},ticks:{callback:(v)=>v.toLocaleString('zh-TW')}}
      },
      plugins:{
        legend:{display:false},
        tooltip:{callbacks:{label:(ctx)=>`${ctx.dataset.label}: ${fmt(ctx.raw)}`}}
      }
    }
  })

  // 自訂圖例（可點擊顯示/隱藏）
  legendBox.innerHTML=''
  datasets.forEach((d,idx)=>{
    const el=document.createElement('button')
    el.className='pill'
    el.textContent=d.label
    el.onclick=()=>{
      const vis = chart.isDatasetVisible(idx)
      chart.setDatasetVisibility(idx,!vis)
      chart.update()
    }
    legendBox.appendChild(el)
  })
}

saveBtn.onclick=()=>{
  state.csv = csvInput.value.trim()
  localStorage.setItem('pf_csv', state.csv)
  loadCSV()
}

// 初始化
if(state.csv){csvInput.value=state.csv; loadCSV()}
</script>

<!--
======================== 實作指南 ========================
一、Google 試算表結構（免費 + 每日更新）
建立名為「Portfolio」的試算表，分頁：
1) Config（持股清單 & 權重）
   - columns: market,ticker,name,shares,cost,currency,bucket
   - market: TWSE / TPEX / EMERGING / US / CASH
   - bucket: twse / tpex / emerging / us / cash  （用來彙總到圖表）
2) Prices（Apps Script 每日抓價寫入）
   - columns: date, market, ticker, close, currency
3) History（彙總後輸出到網頁用 CSV）
   - columns: date,total_twd,twse_twd,tpex_twd,emerging_twd,us_twd,cash_twd

二、Apps Script（免金鑰、全免費來源；部署於試算表 > 擴充功能 > Apps Script）
- 特色：
  * 台股上市：TWSE 官方 API（逐月日資料，取最後一筆）
    ex: https://www.twse.com.tw/exchangeReport/STOCK_DAY?response=json&date=20250101&stockNo=2330
  * 上櫃/興櫃：TPEx 開放資料（Swagger 列有 daily endpoints）
    開放平台條目與 Swagger： https://www.tpex.org.tw/openapi/swagger.json
  * 美股/ETF：Stooq 歷史日資料 CSV（免費、無金鑰），例如：
    https://stooq.com/q/d/l/?s=spy.us&i=d
  * 匯率：exchangerate.host（免費、無金鑰），例如： https://api.exchangerate.host/latest?base=USD&symbols=TWD

將下列程式碼貼到 Apps Script，新建「每日觸發器（時間驅動）」排程：
-->
<script>
/* eslint-disable */
async function httpGet(url){
  const res = UrlFetchApp.fetch(url,{muteHttpExceptions:true,headers:{'User-Agent':'AppsScript'}})
  const code = res.getResponseCode()
  if(code>=200 && code<300){return res.getContentText('utf-8')}
  throw new Error('HTTP '+code+' '+url)
}

// 1) TWSE 上市：逐月 API，回傳 json，取最後交易日收盤
function fetchTWSEClose(stockNo){
  const today = new Date()
  const y = today.getFullYear(), m = (today.getMonth()+1).toString().padStart(2,'0')
  const url = `https://www.twse.com.tw/exchangeReport/STOCK_DAY?response=json&date=${y}${m}01&stockNo=${stockNo}`
  const json = JSON.parse(httpGet(url))
  if(json.stat!=='OK') throw new Error('TWSE bad stat for '+stockNo)
  const rows = json.data
  const last = rows[rows.length-1]
  const close = parseFloat(last[6].replace(/[,]/g,''))
  const date = last[0].replace(/\//g,'-') // yyyy-MM-dd
  return {date,close,currency:'TWD'}
}

// 2) TPEx 上櫃個股：開放資料（示例 endpoint；實際以 Swagger 為準）
function fetchTPEXClose(stockNo){
  // 方案A（每日整批收盤表，過濾代碼）：
  const d = Utilities.formatDate(new Date(),'Asia/Taipei','yyyy/MM/dd')
  const url = `https://www.tpex.org.tw/web/stock/aftertrading/otc_quotes_no1430/stk_wn1430_result.php?l=zh-tw&o=data&d=${d}`
  const csv = httpGet(url)
  const lines = csv.split(/\r?\n/)
  for(const ln of lines){
    const cols = ln.split(',')
    if(cols[0]===stockNo){
      const close = parseFloat(cols[4]) // 開/高/低/收 依欄位調整
      return {date:d.replace(/\//g,'-'), close, currency:'TWD'}
    }
  }
  throw new Error('TPEX not found '+stockNo)
}

// 3) 興櫃：TPEx 興櫃個股歷史（以 Swagger 類似端點為準；此為備援抓整表再篩選）
function fetchEMGClose(stockNo){
  const d = Utilities.formatDate(new Date(),'Asia/Taipei','yyyy/MM/dd')
  const url = `https://www.tpex.org.tw/web/emergingstock/historical/price/price_result.php?l=zh-tw&o=data&d=${d}`
  const csv = httpGet(url)
  const lines = csv.split(/\r?\n/)
  for(const ln of lines){
    const cols = ln.split(',')
    if(cols[0]===stockNo){
      const close = parseFloat(cols[4])
      return {date:d.replace(/\//g,'-'), close, currency:'TWD'}
    }
  }
  throw new Error('EMG not found '+stockNo)
}

// 4) 美股/ETF（Stooq）
function fetchUSClose(symbol){
  const url = `https://stooq.com/q/d/l/?s=${symbol.toLowerCase()}.us&i=d`
  const csv = httpGet(url).trim().split(/\r?\n/)
  const hdr = csv.shift().split(',')
  const last = csv.pop().split(',')
  return {date:last[0], close:parseFloat(last[4]), currency:'USD'}
}

function getUSDTWD(){
  const url = 'https://api.exchangerate.host/latest?base=USD&symbols=TWD'
  const json = JSON.parse(httpGet(url))
  return json.rates.TWD
}

function upsertPrices(){
  const ss = SpreadsheetApp.getActive()
  const cfg = ss.getSheetByName('Config')
  const prices = ss.getSheetByName('Prices')
  const rows = cfg.getDataRange().getValues(); rows.shift() // 去表頭
  const usdTwd = getUSDTWD()
  const out = []
  for(const r of rows){
    const [market,ticker,name,shares,cost,currency,bucket] = r
    if(!ticker||!market) continue
    try{
      let rec
      if(market==='TWSE') rec = fetchTWSEClose(ticker)
      else if(market==='TPEX') rec = fetchTPEXClose(ticker)
      else if(market==='EMERGING') rec = fetchEMGClose(ticker)
      else if(market==='US') rec = fetchUSClose(ticker)
      else if(market==='CASH') { // 現金部位以 1:1 視為收盤
        const d = Utilities.formatDate(new Date(),'Asia/Taipei','yyyy-MM-dd')
        rec = {date:d, close:1, currency:currency||'TWD'}
      }
      out.push([rec.date, market, ticker, rec.close, rec.currency])
    }catch(e){
      out.push([Utilities.formatDate(new Date(),'Asia/Taipei','yyyy-MM-dd'), market, ticker, '', 'ERR:'+e.message])
    }
  }
  if(out.length){
    prices.clearContents(); prices.getRange(1,1,1,5).setValues([['date','market','ticker','close','currency']])
    prices.getRange(2,1,out.length,5).setValues(out)
  }
}

function rebuildHistory(){
  const ss = SpreadsheetApp.getActive()
  const cfg = ss.getSheetByName('Config')
  const prices = ss.getSheetByName('Prices')
  const hist = ss.getSheetByName('History')
  const usdTwd = getUSDTWD()

  const cfgRows = cfg.getDataRange().getValues(); const hdr = cfgRows.shift()
  const priceRows = prices.getDataRange().getValues(); priceRows.shift()
  const date = priceRows[0]? priceRows[0][0] : Utilities.formatDate(new Date(),'Asia/Taipei','yyyy-MM-dd')

  const buckets = {twse:0,tpex:0,emerging:0,us:0,cash:0}
  for(const pr of priceRows){
    const [d,market,ticker,close,curr] = pr
    if(!close) continue
    const conf = cfgRows.find(r=>r[0]===market && r[1]===ticker) // match
    if(!conf) continue
    const shares = conf[3]
    const bucket = conf[6]
    let value = close * shares
    // 美元轉台幣
    if(curr==='USD'){ value *= usdTwd }
    buckets[bucket] = (buckets[bucket]||0) + value
  }

  const total = buckets.twse + buckets.tpex + buckets.emerging + buckets.us + buckets.cash
  // 追加到 History（若已有該日期，可覆蓋）
  const hv = hist.getDataRange().getValues(); const hhdr = hv.shift()
  const row = [date, total, buckets.twse, buckets.tpex, buckets.emerging, buckets.us, buckets.cash]
  let wrote=false
  for(let i=0;i<hv.length;i++){
    if(hv[i][0]===date){
      hist.getRange(i+2,1,1,7).setValues([row]); wrote=true; break
    }
  }
  if(!wrote){ hist.appendRow(row) }
}

function daily(){
  upsertPrices()
  rebuildHistory()
}
</script>

<!--
三、排程與發佈
1) 在 Apps Script 建立時間驅動觸發器：每日台北時間 18:00 執行 daily()
2) 在試算表「History」分頁：檔案 > 發佈到網路 > 選擇此分頁 & 格式 CSV，複製連結
3) 把連結貼到本頁右上角的輸入框後「儲存」，網頁即會自動抓取並顯示

四、Config 填寫範例
market,ticker,name,shares,cost,currency,bucket
TWSE,2330,台積電,25,580,TWD,twse
TPEX,5289,宜鼎,30,320,TWD,tpex
EMERGING,7556,喬山-興櫃,5,120,TWD,emerging
US,NVDA,NVIDIA,200,90,USD,us
US,QQQ,Invesco QQQ,120,400,USD,us
CASH,TWD_Cash,台幣現金,1,0,TWD,cash
CASH,USD_Cash,美元現金(以1為面額),30000,0,USD,cash

五、注意事項（免費來源的限制與替代）
- TWSE：官方 API 屬於「逐月查詢」，程式已自動取最後一筆；如遇交易日休市，請於下一交易日更新
- TPEX/興櫃：官方站點提供整表 CSV 與 OpenAPI（Swagger 上可見），本範例採「抓整表再篩選」策略，避免逐檔流量；若日後欄位異動，請依實際欄位名調整 split 索引
- 美股：Stooq 為學術社群常用的免費日資料來源，符號慣例如：
  * SPY→ spy.us，QQQ→ qqq.us，BRK.B→ brk-b.us，VTI→ vti.us
  * 若個別代碼在 Stooq 缺料，可改用 Tiingo/Alpha Vantage（都有免費額度、需申請金鑰），並在 Apps Script 以 UrlFetchApp 帶入金鑰
- 匯率：exchangerate.host 無金鑰，足以做每日折算；若偏好央行資料，可改抓央行每日牌告匯率 CSV
- 隱私：將 History 分頁設為「有連結者可檢視」即可，避免整本試算表公開

六、版面（可模仿你提供的 FB 文章風格）
- 單一主圖展示總資產淨值，右側以 KPI 呈現日變動與 YTD
- 下方圖例可點擊開關各資產桶（上市 / 上櫃 / 興櫃 / 美股 / 現金）
- 全部為前端靜態檔，建議部署到 GitHub Pages 或 Netlify（免費）

七、延伸：
- 新增「回測成本」欄，計算未實現損益與 MDD
- 新增「配息現金流」分頁，History 另加 income_twd 欄位並疊加成柱狀＋淨值折線
- 新增「再平衡建議」：根據目標權重自動算出買賣金額

-->
</body>
</html>

