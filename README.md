<!DOCTYPE html>
<html lang="he" dir="rtl">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>דוב המטבעות — גרסת מובייל (ללא אחסון מקומי)</title>
<style>
  :root{
    --bg:#0f172a; --card:#111827; --ink:#e5e7eb; --muted:#94a3b8;
    --good:#22c55e; --bad:#ef4444; --btn:#1f2937; --accent:#fbbf24;
  }
  html,body{height:100%;margin:0;background:linear-gradient(180deg,#0b1223,#0f172a);
    font-family:system-ui,-apple-system,Segoe UI,Roboto,Heebo,Arial,sans-serif;color:var(--ink);}
  .wrap{max-width:720px;margin:0 auto;padding:18px}
  header{display:flex;align-items:center;gap:12px}
  .bear{font-size:50px;filter:drop-shadow(0 6px 16px rgba(0,0,0,.35))}
  .title{font-weight:800;font-size:clamp(18px,4vw,26px)}
  .sub{color:var(--muted);font-size:13px}
  .card{background:rgba(17,24,39,.75);backdrop-filter: blur(8px); border:1px solid rgba(255,255,255,.06);
    border-radius:16px;padding:14px;margin-top:14px}
  .balance{display:flex;align-items:center;justify-content:space-between;gap:12px}
  .num{font-weight:800;font-size:32px}
  .pill{background:rgba(255,255,255,.06);border:1px solid rgba(255,255,255,.08);
    padding:6px 10px;border-radius:999px;color:var(--muted);font-size:12px}
  .controls{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-top:12px}
  button{cursor:pointer;border:1px solid rgba(255,255,255,.08); background:var(--btn); color:var(--ink);
    padding:14px 12px;border-radius:12px;font-weight:800;font-size:16px}
  button.primary{background:linear-gradient(180deg,#16a34a,#15803d); border-color:#064e3b}
  button.danger{background:linear-gradient(180deg,#b91c1c,#7f1d1d); border-color:#450a0a}
  button.ghost{background:rgba(255,255,255,.05);grid-column: span 2}
  .calendar{display:grid;grid-template-columns: repeat(7, 1fr); gap:6px;margin-top:10px}
  .dot{aspect-ratio:1;border-radius:10px;display:grid;place-items:center;font-size:14px;
    border:1px solid rgba(255,255,255,.08);}
  .dot.good{background:rgba(34,197,94,.15);color:#86efac}
  .dot.bad{background:rgba(239,68,68,.15);color:#fecaca}
  .dot.none{background:rgba(255,255,255,.04);color:#94a3b8}
  .hint{color:var(--muted);font-size:13px;margin-top:6px}
  .footer{margin-top:10px;color:var(--muted);font-size:12px}
  .toast{position:fixed;inset:auto 0 18px 0;display:grid;place-items:center;pointer-events:none}
  .toast > div{background:#111827cc;color:#e5e7eb;border:1px solid rgba(255,255,255,.06);padding:10px 14px;border-radius:999px;opacity:0;transform:translateY(8px);transition:.25s}
  .toast.show > div{opacity:1;transform:none}
</style>
</head>
<body>
<div class="wrap">
  <header>
    <div class="bear">🐻‍❄️🪙</div>
    <div>
      <div class="title">דוב המטבעות — מובייל</div>
      <div class="sub">הכל נשמר בכתובת (URL). הוסיפי למסך הבית לשמירה קלה.</div>
    </div>
  </header>

  <section class="card">
    <div class="balance">
      <div><span class="num" id="balance">0</span> מטבעות</div>
      <div class="pill" id="streak">רצף: 0</div>
    </div>
    <div class="controls">
      <button class="primary" id="btnGood">לא אכלתי קינוח (+1)</button>
      <button class="danger" id="btnBad">אכלתי קינוח (−1)</button>
      <button class="ghost" id="btnUndo">בטלי פעולה אחרונה ↩︎</button>
      <button class="ghost" id="btnShare">שמרי קיצור דרך/שיתוף</button>
    </div>
    <div class="hint" id="todayHint"></div>
    <div class="calendar" id="calendar"></div>
    <div class="footer" id="footer"></div>
  </section>
</div>

<div class="toast" id="toast"><div id="toastMsg">עודכן</div></div>

<script>
(function(){
  const $ = s => document.querySelector(s);
  const todayIso = () => { const d=new Date(); d.setHours(0,0,0,0); return d.toISOString().slice(0,10); };

  // State in localStorage
  function getState(){
    const h = localStorage.getItem('coin_bear_state');
    if(!h) return {balance:0, logs:[]};
    try { return JSON.parse(h); } catch(e){ return {balance:0, logs:[]}; }
  }

  function setState(s, toastMsg){
    const minimal = {...s};
    if(minimal.logs.length>120) minimal.logs = minimal.logs.slice(minimal.logs.length-120);
    localStorage.setItem('coin_bear_state', JSON.stringify(minimal));
    render();
    if(toastMsg) toast(toastMsg);
  }

  function toast(msg){
    const t = document.getElementById('toast');
    document.getElementById('toastMsg').textContent = msg;
    t.classList.add('show'); setTimeout(()=>t.classList.remove('show'), 1400);
  }

  function computeStreak(logs){
    const byDate = logs.slice().sort((a,b)=>a.date.localeCompare(b.date));
    let streak = 0;
    for(let i=byDate.length-1;i>=0;i--){
      if(byDate[i].val <= 0) break;
      const lastDate = new Date(byDate[i+1]?.date || '2050-01-01');
      const currentDate = new Date(byDate[i].date);
      if(i === byDate.length - 1 || (lastDate - currentDate === 86400000)) {
        streak++;
      } else if (lastDate - currentDate > 86400000) {
        break;
      }
    }
    return streak;
  }

  function render(){
    const st = getState();
    document.getElementById('balance').textContent = st.balance;
    document.getElementById('streak').textContent = 'רצף: ' + computeStreak(st.logs);

    const today = todayIso();
    const tLog = st.logs.find(l => l.date === today);
    document.getElementById('todayHint').textContent = tLog
      ? `היום מסומן: ${tLog.val>0?'בלי קינוח (+1)':'עם קינוח (−1)'}`
      : 'עוד לא סימנת את היום';

    const cal = document.getElementById('calendar'); cal.innerHTML='';
    for(let i=13;i>=0;i--){
      const d = new Date(); d.setDate(d.getDate()-i);
      const iso = d.toISOString().slice(0,10);
      const log = st.logs.find(x=>x.date===iso);
      const el = document.createElement('div');
      el.className = 'dot ' + (log ? (log.val>0?'good':'bad') : 'none');
      el.textContent = log ? (log.val>0?'✓':'×') : '·';
      cal.appendChild(el);
    }

    document.getElementById('footer').textContent =
      `ימים מתועדים: ${st.logs.length} • בלי קינוח: ${st.logs.filter(l=>l.val>0).length} • עם קינוח: ${st.logs.filter(l=>l.val<0).length}`;
  }

  function setToday(val){
    const st = getState();
    const iso = todayIso();
    const idx = st.logs.findIndex(l => l.date===iso);
    if(idx>=0){ st.balance -= st.logs[idx].val; st.logs.splice(idx,1); }
    st.logs.push({date: iso, val});
    st.balance += val;
    setState(st, val>0 ? "כל הכבוד! +1 מטבע" : "סומן קינוח — −1 מטבע");
  }

  document.getElementById('btnGood').addEventListener('click', ()=>setToday(+1));
  document.getElementById('btnBad').addEventListener('click',  ()=>setToday(-1));
  document.getElementById('btnUndo').addEventListener('click', ()=>{
    const st = getState();
    if(!st.logs.length) return;
    const last = st.logs.pop();
    st.balance -= last.val;
    setState(st, "בוטל");
  });
  document.getElementById('btnShare').addEventListener('click', async ()=>{
    const url = location.href;
    try{
      if(navigator.share){
        await navigator.share({title:"דוב המטבעות", url});
      } else {
        await navigator.clipboard.writeText(url);
        toast("הלינק הועתק — הוסיפי למסך הבית");
      }
    }catch(e){ console.log(e); }
  });

  render();
})();
</script>
</body>
</html>
