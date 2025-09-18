<script>
(function(){
  const $ = s => document.querySelector(s);
  const todayIso = () => { const d=new Date(); d.setHours(0,0,0,0); return d.toISOString().slice(0,10); };
  const enc = obj => JSON.stringify(obj); // שינוי: אין צורך ב-Base64
  const dec = str => JSON.parse(str); // שינוי: אין צורך ב-Base64

  // State in localStorage
  function getState(){
    const h = localStorage.getItem('coin_bear_state'); // שינוי: קורא מ-localStorage
    if(!h) return {balance:0, logs:[]};
    try { return dec(h); } catch(e){ return {balance:0, logs:[]}; }
  }
  function setState(s, toastMsg){
    const minimal = {...s};
    if(minimal.logs.length>120) minimal.logs = minimal.logs.slice(minimal.logs.length-120);
    localStorage.setItem('coin_bear_state', enc(minimal)); // שינוי: כותב ל-localStorage
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
  document.getElementById('btnBad').addEventListener('click',  ()=>setToday(-1));
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
