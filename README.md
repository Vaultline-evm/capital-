<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>حاسبة النمو</title>
<style>
  *{box-sizing:border-box}
  body{margin:0;padding:16px;font-family:system-ui,Tahoma,sans-serif;background:#070b16;color:#e8eefc}
  .wrap{max-width:900px;margin:auto}
  .grid{display:grid;grid-template-columns:1fr 1fr;gap:10px}
  @media(max-width:700px){.grid{grid-template-columns:1fr}}
  .card{background:#0d1424;border:1px solid #1e2a44;border-radius:14px;padding:14px;margin-bottom:12px}
  label{display:block;font-size:12px;color:#9fb0cf;margin-bottom:5px}
  input{width:100%;padding:10px;border-radius:10px;background:#080e1c;border:1px solid #22304e;color:#fff;font-size:15px;direction:ltr;text-align:right;font-family:inherit}
  input:focus{outline:none;border-color:#38bdf8}
  .stats{display:grid;grid-template-columns:1fr 1fr 1fr;gap:10px}
  @media(max-width:600px){.stats{grid-template-columns:1fr}}
  .stat{background:#080e1c;border:1px solid #1c2740;border-radius:12px;padding:11px}
  .stat span{display:block;font-size:11px;color:#8ea0c0;margin-bottom:5px}
  .stat b{font-size:16px;color:#e8eefc}
  .stat small{display:block;font-size:10.5px;color:#6b7c9c;margin-top:4px;min-height:12px}
  canvas{display:block;width:100%;background:#080e1c;border:1px solid #1c2740;border-radius:12px}
</style>
</head>
<body>
<div class="wrap">
  <div class="card">
    <div class="grid">
      <div><label>رأس المال الابتدائي</label><input id="capital" type="number" min="0" step="any"></div>
      <div><label>الهدف النهائي</label><input id="target2" type="number" min="0" step="any"></div>
      <div><label>المخاطرة لكل صفقة (%)</label><input id="risk" type="number" min="0" step="any"></div>
      <div><label>نسبة العائد/الخسارة R:R</label><input id="rr" type="number" min="0" step="any"></div>
      <div><label>نسبة الصفقات الرابحة (%)</label><input id="winrate" type="number" min="0" max="100" step="any"></div>
      <div><label>عدد الصفقات في اليوم</label><input id="tradesday" type="number" min="0" step="any"></div>
    </div>
  </div>

  <div class="card">
    <div class="stats">
      <div class="stat"><span>معامل النمو/صفقة</span><b id="r-mult">—</b></div>
      <div class="stat"><span>عدد الصفقات</span><b id="r-trades">—</b></div>
      <div class="stat"><span>المدة (أيام تقويمية)</span><b id="r-time">—</b><small id="r-time-sub"></small></div>
    </div>
  </div>

  <div class="card"><canvas id="chart"></canvas></div>
</div>

<script>
(function(){
  var $=function(id){return document.getElementById(id)};
  var F=['capital','risk','rr','winrate','tradesday','target2'];
  var chartData=null;
  function num(id){var v=$(id).value.trim();if(v==='')return null;var n=parseFloat(v);return isNaN(n)?null:n}
  function fmt(n,d){if(n==null||!isFinite(n))return'—';if(d===undefined){var a=Math.abs(n);d=a>=10000?0:(a>=100?1:2)}return n.toLocaleString('en-US',{minimumFractionDigits:d,maximumFractionDigits:d})}
  function short(v){var a=Math.abs(v);if(a>=1e9)return(v/1e9).toFixed(1).replace(/\.0$/,'')+'B';if(a>=1e6)return(v/1e6).toFixed(1).replace(/\.0$/,'')+'M';if(a>=1e3)return(v/1e3).toFixed(1).replace(/\.0$/,'')+'K';return String(Math.round(v*100)/100)}
  function cal(td){return td*7/5}
  function fmtDays(d){if(d==null||!isFinite(d))return'—';if(d===0)return'0 يوم';if(d<10)return fmt(d,d<1?2:1)+' يوم';return fmt(d,0)+' يوم'}
  function set(id,t){$(id).textContent=t}
  function nice(max,c){if(!(max>0))return 1;var r=max/c,m=Math.pow(10,Math.floor(Math.log10(r))),n=r/m,s;if(n<=1)s=1;else if(n<=2)s=2;else if(n<=2.5)s=2.5;else if(n<=5)s=5;else s=10;return s*m}

  function update(){
    var C=num('capital'),risk=num('risk'),rr=num('rr'),win=num('winrate'),tpd=num('tradesday'),T=num('target2');
    set('r-mult','—');set('r-trades','—');set('r-time','—');$('r-time-sub').textContent='';chartData=null;
    var ok=C>0&&risk>0&&risk<100&&rr>0&&win>=0&&win<=100&&tpd>0;
    if(!ok){draw();return}

    var r=risk/100,w=win/100,up=1+r*rr,down=1-r;
    var m=Math.pow(up,w)*Math.pow(down,1-w);
    set('r-mult',fmt(m,4)+'×');

    if(T==null||T<=0){draw();return}
    if(T<=C){set('r-trades','0');set('r-time','0 يوم');draw();return}
    if(!(m>1)){set('r-trades','غير قابل');set('r-time','—');draw();return}

    var trades=Math.log(T/C)/Math.log(m);
    var td=trades/tpd;
    var cd=cal(td);
    set('r-trades',fmt(Math.ceil(trades),0));
    set('r-time',fmtDays(cd));
    $('r-time-sub').textContent='أيام تداول: '+fmt(td,td<10?1:0);

    var totalTrades=Math.min(Math.max(trades,5),50000);
    var totalTd=totalTrades/tpd;
    var totalCd=cal(totalTd);
    var N=240,points=[];
    for(var i=0;i<=N;i++){
      var d=totalTd*i/N;
      points.push({d:cal(d),c:C*Math.pow(m,d*tpd)});
    }
    chartData={points:points,maxDays:totalCd,maxY:Math.max(T,points[points.length-1].c)*1.08,target:T};
    draw();
  }

  function draw(){
    var cv=$('chart'),box=cv.parentElement,W=Math.max(300,box.clientWidth||600),H=340,dpr=window.devicePixelRatio||1;
    cv.width=Math.round(W*dpr);cv.height=Math.round(H*dpr);cv.style.width=W+'px';cv.style.height=H+'px';
    var ctx=cv.getContext('2d');ctx.setTransform(dpr,0,0,dpr,0,0);ctx.clearRect(0,0,W,H);
    ctx.font='12px system-ui,Tahoma,sans-serif';ctx.direction='rtl';
    if(!chartData)return;

    var p=chartData.points,maxD=chartData.maxDays,maxY=chartData.maxY,T=chartData.target;
    var padL=12,padR=78,padT=20,padB=38,plotW=Math.max(10,W-padL-padR),plotH=Math.max(10,H-padT-padB),x0=padL,y0=padT+plotH;
    function X(d){return x0+(maxD>0?(d/maxD)*plotW:0)}
    function Y(c){return y0-(maxY>0?(c/maxY)*plotH:0)}

    ctx.strokeStyle='#16213a';ctx.lineWidth=1;ctx.fillStyle='#7b8cab';ctx.textAlign='right';ctx.textBaseline='middle';
    var ys=nice(maxY,5);
    for(var v=0;v<=maxY+1e-9;v+=ys){var py=Y(v);ctx.beginPath();ctx.moveTo(x0,py);ctx.lineTo(x0+plotW,py);ctx.stroke();ctx.fillText(short(v),W-6,py)}
    var xs=nice(maxD,5);ctx.textAlign='center';ctx.textBaseline='top';
    for(var d=0;d<=maxD+1e-9;d+=xs){var px=X(d);ctx.beginPath();ctx.moveTo(px,padT);ctx.lineTo(px,y0);ctx.stroke();ctx.fillStyle='#7b8cab';ctx.fillText(short(d),px,y0+6)}

    var ty=Y(T);
    if(ty>=padT&&ty<=y0){ctx.save();ctx.setLineDash([6,4]);ctx.strokeStyle='#f59e0b';ctx.lineWidth=1.5;ctx.beginPath();ctx.moveTo(x0,ty);ctx.lineTo(x0+plotW,ty);ctx.stroke();ctx.restore()}

    ctx.save();ctx.beginPath();ctx.rect(x0,padT,plotW,plotH);ctx.clip();
    ctx.beginPath();ctx.moveTo(X(p[0].d),y0);
    for(var i=0;i<p.length;i++)ctx.lineTo(X(p[i].d),Y(p[i].c));
    ctx.lineTo(X(p[p.length-1].d),y0);ctx.closePath();
    var g=ctx.createLinearGradient(0,padT,0,y0);g.addColorStop(0,'rgba(34,197,94,.3)');g.addColorStop(1,'rgba(34,197,94,0)');ctx.fillStyle=g;ctx.fill();
    ctx.beginPath();
    for(var j=0;j<p.length;j++){var lx=X(p[j].d),ly=Y(p[j].c);j?ctx.lineTo(lx,ly):ctx.moveTo(lx,ly)}
    ctx.strokeStyle='#22c55e';ctx.lineWidth=2.5;ctx.lineJoin='round';ctx.lineCap='round';ctx.stroke();
    ctx.restore();
  }

  F.forEach(function(id){$(id).addEventListener('input',update)});
  var rt;window.addEventListener('resize',function(){clearTimeout(rt);rt=setTimeout(draw,120)});
  update();
})();
</script>
</body>
</html>
