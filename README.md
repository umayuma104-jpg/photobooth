<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>プリクラ</title>
<style>
:root{--pink:#ff6fa5;--bg:#fff5f9;--ink:#3a2a33}
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent;-webkit-user-select:none;user-select:none}
html,body{margin:0;height:100%;background:var(--bg);color:var(--ink);font-family:-apple-system,"Hiragino Sans","Hiragino Kaku Gothic ProN",sans-serif;overflow:hidden;touch-action:manipulation}
.screen{display:none;position:absolute;inset:0;padding:20px;flex-direction:column;overflow-y:auto}
.screen.active{display:flex}
.inner{margin:auto;display:flex;flex-direction:column;align-items:center;gap:14px;width:100%;max-width:900px}
h1{font-size:56px;margin:0;color:var(--pink)}
h2{font-size:26px;margin:6px 0 0}
p{margin:0;font-size:20px;text-align:center}
.btn{border:none;border-radius:999px;background:var(--pink);color:#fff;font-size:28px;font-weight:700;padding:20px 56px;box-shadow:0 6px 0 #d6457f}
.btn:active{transform:translateY(4px);box-shadow:0 2px 0 #d6457f}
.btn:disabled{opacity:.4}
.btn.sub{background:#fff;color:var(--pink);box-shadow:0 6px 0 #ffc2d9;font-size:20px;padding:12px 28px}
.group{width:100%;text-align:center}
.chips{display:flex;flex-wrap:wrap;justify-content:center}
.chip{padding:12px 22px;border-radius:999px;border:3px solid #ffc2d9;background:#fff;color:var(--ink);font-size:20px;margin:5px}
.chip small{display:block;font-size:13px;opacity:.7}
.chip.sel{background:var(--pink);border-color:var(--pink);color:#fff}
.fthumb{border:4px solid transparent;border-radius:10px;background:#fff;padding:3px;margin:5px}
.fthumb.sel{border-color:var(--pink)}
.fthumb canvas{width:80px;display:block}
.fthumb div{font-size:13px;text-align:center}
.vwrap{position:relative;width:min(92vw,calc((100vh - 180px) * 4 / 3),900px);aspect-ratio:4/3;background:#000;border-radius:16px;overflow:hidden}
video{width:100%;height:100%;object-fit:cover;transform:scaleX(-1)}
#cd{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;font-size:200px;font-weight:800;color:#fff;text-shadow:0 0 20px rgba(0,0,0,.5);pointer-events:none}
#counter{position:absolute;top:10px;left:14px;font-size:26px;font-weight:700;color:#fff;text-shadow:0 0 8px rgba(0,0,0,.6)}
#flash{position:absolute;inset:0;background:#fff;opacity:0;pointer-events:none;transition:opacity .35s}
.row{display:flex;gap:30px;align-items:center;justify-content:center;flex-wrap:wrap}
#resultImg{max-height:78vh;max-width:40vw;border-radius:8px;box-shadow:0 4px 16px rgba(0,0,0,.2);-webkit-user-select:auto;user-select:auto;-webkit-touch-callout:default}
#qr{background:#fff;padding:14px;border-radius:12px;min-width:190px;min-height:190px;display:flex;align-items:center;justify-content:center}
#qr img,#qr canvas{display:block}
.err{color:#d12b2b}
</style>
</head>
<body>

<div class="screen active" id="s-start">
  <div class="inner">
    <h1>プリクラ</h1>
    <p id="startMsg">画面をタップしてスタート！</p>
    <button class="btn" id="btnStart">スタート</button>
  </div>
</div>

<div class="screen" id="s-settings">
  <div class="inner" id="settingsBox"></div>
</div>

<div class="screen" id="s-shoot">
  <div class="inner">
    <div class="vwrap">
      <video id="video" autoplay playsinline muted></video>
      <div id="counter"></div>
      <div id="cd"></div>
      <div id="flash"></div>
    </div>
    <p>撮影開始ボタンを押すと4枚撮影されます</p>
    <div class="row">
      <button class="btn sub" id="btnBack">もどる</button>
      <button class="btn" id="btnShoot">撮影開始</button>
    </div>
  </div>
</div>

<div class="screen" id="s-working">
  <div class="inner"><h2>加工中…</h2><p>そのままちょっと待ってね</p></div>
</div>

<div class="screen" id="s-result">
  <div class="inner">
    <div class="row">
      <img id="resultImg" alt="完成">
      <div style="display:flex;flex-direction:column;align-items:center;gap:12px">
        <h2>できたよ！</h2>
        <div id="qr"></div>
        <p id="resultMsg">アップロード中…</p>
        <p style="font-size:15px">読み取ったあと、画像を長押し →<br>「“写真”に追加」で保存できます</p>
        <button class="btn sub" id="btnRetry" style="display:none">もう一度送信する</button>
        <button class="btn" id="btnEnd">おわり</button>
      </div>
    </div>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
<script>
/* ===== 設定 ===== */
const CLOUD_NAME = 'g8matvfw';
const UPLOAD_PRESET = 'purikura';
const EVENT_TEXT = '学校祭 2026';
const SHOT_COUNT = 4;
const COUNTDOWN = 5;
const RESET_SEC = 90;

/* ===== データ ===== */
const FILTERS = {
  natural:{name:'ナチュラル',p:{br:1.03,ct:1,sat:1.05,r:1,g:1,b:1},css:'brightness(1.03) saturate(1.05)'},
  pink:{name:'ピンク系',p:{br:1.08,ct:.95,sat:1.1,r:1.06,g:.97,b:1.04},css:'brightness(1.08) contrast(.95) saturate(1.1) hue-rotate(-10deg)'},
  blue:{name:'ブルー',p:{br:1.05,ct:1,sat:.95,r:.94,g:1,b:1.1},css:'brightness(1.05) saturate(.95) hue-rotate(10deg)'},
  mono:{name:'モノクロ',p:{br:1.05,ct:1.1,sat:0,r:1,g:1,b:1},css:'grayscale(1) contrast(1.1)'}
};
const BEAUTY = {
  weak:{name:'弱め',alpha:.2,bright:.02},
  normal:{name:'普通',alpha:.38,bright:.04},
  strong:{name:'強め',alpha:.55,bright:.07}
};
const PRESETS = {
  natural:{name:'ナチュラル',sub:'自然に',filter:'natural',beauty:'weak'},
  balance:{name:'おすすめ',sub:'バランス',filter:'natural',beauty:'normal'},
  heavy:{name:'しっかり盛り',sub:'華やかに',filter:'pink',beauty:'strong'}
};
const FRAMES = {
  pop:{name:'ポップピンク',bg:'#ffd6e7',accent:'#ff5c9d',dots:true,deco:'heart'},
  sky:{name:'スカイブルー',bg:'#d3eaff',accent:'#3b8cff',dots:true,deco:'star'},
  mono:{name:'シンプル白',bg:'#ffffff',accent:'#333333',dots:false,deco:null}
};
const defaultState = () => ({preset:'balance',filter:'natural',beauty:'normal',frame:'pop'});
let state = defaultState();

/* ===== ヘルパー ===== */
const $ = id => document.getElementById(id);
const sleep = ms => new Promise(r => setTimeout(r, ms));
let resetTimer = null;
function show(id){
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  $(id).classList.add('active');
}

/* ===== カメラ ===== */
const video = $('video');
let stream = null;
async function startCamera(){
  if (stream) return;
  stream = await navigator.mediaDevices.getUserMedia({
    video:{facingMode:'user',width:{ideal:1280},height:{ideal:960}},audio:false
  });
  video.srcObject = stream;
  await video.play().catch(()=>{});
}

/* ===== 設定画面 ===== */
function chipGroup(title, items, selKey, onPick){
  const g = document.createElement('div'); g.className = 'group';
  g.innerHTML = '<h2>' + title + '</h2>';
  const box = document.createElement('div'); box.className = 'chips';
  items.forEach(([key,label,sub]) => {
    const b = document.createElement('button');
    b.className = 'chip' + (state[selKey] === key ? ' sel' : '');
    b.innerHTML = label + (sub ? '<small>' + sub + '</small>' : '');
    b.onclick = () => onPick(key);
    box.appendChild(b);
  });
  g.appendChild(box);
  return g;
}
function renderSettings(){
  const box = $('settingsBox'); box.innerHTML = '';
  box.appendChild(chipGroup('① おすすめ加工',
    Object.entries(PRESETS).map(([k,v]) => [k,v.name,v.sub]), 'preset', k => {
      state.preset = k; state.filter = PRESETS[k].filter; state.beauty = PRESETS[k].beauty; renderSettings();
    }));
  box.appendChild(chipGroup('② フィルター',
    Object.entries(FILTERS).map(([k,v]) => [k,v.name]), 'filter', k => {
      state.filter = k; state.preset = null; renderSettings();
    }));
  box.appendChild(chipGroup('③ 美肌の強さ',
    Object.entries(BEAUTY).map(([k,v]) => [k,v.name]), 'beauty', k => {
      state.beauty = k; state.preset = null; renderSettings();
    }));
  const g = document.createElement('div'); g.className = 'group';
  g.innerHTML = '<h2>④ フレーム</h2>';
  const row = document.createElement('div'); row.className = 'chips';
  Object.entries(FRAMES).forEach(([k,f]) => {
    const w = document.createElement('div');
    w.className = 'fthumb' + (state.frame === k ? ' sel' : '');
    w.appendChild(composeStrip(null, f, 0.25, ''));
    const l = document.createElement('div'); l.textContent = f.name; w.appendChild(l);
    w.onclick = () => { state.frame = k; renderSettings(); };
    row.appendChild(w);
  });
  g.appendChild(row); box.appendChild(g);
  const ok = document.createElement('button');
  ok.className = 'btn'; ok.textContent = '決定';
  ok.onclick = () => { video.style.filter = FILTERS[state.filter].css; show('s-shoot'); };
  box.appendChild(ok);
}

/* ===== 画像処理 ===== */
function applyColor(ctx, w, h, p){
  const im = ctx.getImageData(0,0,w,h), d = im.data;
  for (let i = 0; i < d.length; i += 4){
    let R = d[i]*p.r*p.br, G = d[i+1]*p.g*p.br, B = d[i+2]*p.b*p.br;
    R = (R-128)*p.ct+128; G = (G-128)*p.ct+128; B = (B-128)*p.ct+128;
    const L = 0.299*R + 0.587*G + 0.114*B;
    d[i] = L+(R-L)*p.sat; d[i+1] = L+(G-L)*p.sat; d[i+2] = L+(B-L)*p.sat;
  }
  ctx.putImageData(im,0,0);
}
function soften(c, alpha){
  const w = c.width, h = c.height, s = document.createElement('canvas');
  s.width = Math.round(w/4); s.height = Math.round(h/4);
  const sx = s.getContext('2d');
  sx.imageSmoothingEnabled = true; sx.imageSmoothingQuality = 'high';
  sx.drawImage(c,0,0,s.width,s.height);
  const x = c.getContext('2d');
  x.imageSmoothingEnabled = true; x.imageSmoothingQuality = 'high';
  x.globalAlpha = alpha; x.drawImage(s,0,0,w,h); x.globalAlpha = 1;
}
function processPhoto(raw){
  const c = document.createElement('canvas'); c.width = raw.width; c.height = raw.height;
  c.getContext('2d').drawImage(raw,0,0);
  const bt = BEAUTY[state.beauty];
  soften(c, bt.alpha);
  const p = Object.assign({}, FILTERS[state.filter].p);
  p.br *= 1 + bt.bright;
  applyColor(c.getContext('2d'), c.width, c.height, p);
  return c;
}

/* ===== フレーム合成（縦長ストリップ 600x1800 を S 倍） ===== */
function heart(x,cx,cy,s){
  x.beginPath(); x.moveTo(cx,cy+s*0.9);
  x.bezierCurveTo(cx-s*1.6,cy-s*0.2,cx-s*0.7,cy-s*1.3,cx,cy-s*0.45);
  x.bezierCurveTo(cx+s*0.7,cy-s*1.3,cx+s*1.6,cy-s*0.2,cx,cy+s*0.9);
  x.fill();
}
function star(x,cx,cy,r){
  x.beginPath();
  for (let i = 0; i < 10; i++){
    const rad = i % 2 === 0 ? r : r*0.45, a = -Math.PI/2 + i*Math.PI/5;
    const px = cx + Math.cos(a)*rad, py = cy + Math.sin(a)*rad;
    i === 0 ? x.moveTo(px,py) : x.lineTo(px,py);
  }
  x.closePath(); x.fill();
}
function composeStrip(photos, f, S, dateText){
  const W = 600*S, H = 1800*S;
  const c = document.createElement('canvas'); c.width = W; c.height = H;
  const x = c.getContext('2d');
  x.fillStyle = f.bg; x.fillRect(0,0,W,H);
  if (f.dots){
    x.fillStyle = f.accent + '33';
    for (let row = 0, yy = 0; yy < H; yy += 60*S, row++){
      for (let xx = (row % 2)*30*S; xx < W; xx += 60*S){
        x.beginPath(); x.arc(xx,yy,8*S,0,7); x.fill();
      }
    }
  }
  for (let i = 0; i < SHOT_COUNT; i++){
    const px = 40*S, py = (90 + i*420)*S, pw = 520*S, ph = 390*S;
    if (photos) x.drawImage(photos[i],px,py,pw,ph);
    else { x.fillStyle = '#d9d9d9'; x.fillRect(px,py,pw,ph); }
    x.lineWidth = 6*S; x.strokeStyle = '#fff'; x.strokeRect(px,py,pw,ph);
  }
  x.lineWidth = 10*S; x.strokeStyle = f.accent; x.strokeRect(5*S,5*S,W-10*S,H-10*S);
  x.fillStyle = f.accent; x.textAlign = 'center'; x.textBaseline = 'middle';
  const font = '"Hiragino Sans",-apple-system,sans-serif';
  x.font = 'bold ' + (40*S) + 'px ' + font;
  x.fillText(EVENT_TEXT, W/2, 48*S);
  if (dateText){ x.font = (26*S) + 'px ' + font; x.fillText(dateText, W/2, 1772*S); }
  if (f.deco){
    const fn = f.deco === 'heart' ? heart : star, sz = f.deco === 'heart' ? 14*S : 18*S;
    [[60,48],[540,48],[60,1772],[540,1772]].forEach(([a,b]) => fn(x,a*S,b*S,sz));
  }
  return c;
}

/* ===== 撮影 ===== */
function capture(){
  const vw = video.videoWidth, vh = video.videoHeight;
  let sw = vw, sh = vw*3/4;
  if (sh > vh){ sh = vh; sw = vh*4/3; }
  const sx = (vw-sw)/2, sy = (vh-sh)/2;
  const c = document.createElement('canvas'); c.width = 1040; c.height = 780;
  const x = c.getContext('2d');
  x.translate(c.width,0); x.scale(-1,1);
  x.drawImage(video,sx,sy,sw,sh,0,0,c.width,c.height);
  return c;
}
function flash(){
  const f = $('flash'); f.style.transition = 'none'; f.style.opacity = 1;
  requestAnimationFrame(() => { f.style.transition = 'opacity .35s'; f.style.opacity = 0; });
}
async function runShooting(){
  $('btnShoot').disabled = true; $('btnBack').disabled = true;
  const raws = [];
  for (let i = 0; i < SHOT_COUNT; i++){
    $('counter').textContent = (i+1) + ' / ' + SHOT_COUNT;
    for (let t = COUNTDOWN; t >= 1; t--){ $('cd').textContent = t; await sleep(1000); }
    $('cd').textContent = '';
    flash(); raws.push(capture());
    await sleep(900);
  }
  $('counter').textContent = '';
  $('btnShoot').disabled = false; $('btnBack').disabled = false;
  show('s-working'); await sleep(150);
  const done = raws.map(processPhoto);
  const d = new Date();
  const dateText = d.getFullYear() + '.' + (d.getMonth()+1) + '.' + d.getDate();
  const strip = composeStrip(done, FRAMES[state.frame], 2, dateText);
  showResult(strip);
}

/* ===== 結果・アップロード・QR ===== */
let lastBlob = null;
function showResult(canvas){
  canvas.toBlob(blob => {
    lastBlob = blob;
    $('resultImg').src = URL.createObjectURL(blob);
    $('qr').innerHTML = '';
    $('btnRetry').style.display = 'none';
    $('resultMsg').className = ''; $('resultMsg').textContent = 'アップロード中…';
    show('s-result');
    clearTimeout(resetTimer); resetTimer = setTimeout(resetAll, RESET_SEC*1000);
    doUpload();
  }, 'image/jpeg', 0.92);
}
async function doUpload(){
  $('btnRetry').style.display = 'none';
  $('resultMsg').className = ''; $('resultMsg').textContent = 'アップロード中…';
  try {
    const fd = new FormData();
    fd.append('file', lastBlob, 'purikura.jpg');
    fd.append('upload_preset', UPLOAD_PRESET);
    const ctl = new AbortController();
    const to = setTimeout(() => ctl.abort(), 30000);
    const r = await fetch('https://api.cloudinary.com/v1_1/' + CLOUD_NAME + '/image/upload', {method:'POST',body:fd,signal:ctl.signal});
    clearTimeout(to);
    if (!r.ok) throw new Error(await r.text());
    const url = (await r.json()).secure_url;
    drawQR(url);
    $('resultMsg').textContent = 'スマホでQRコードを読み取ってね';
  } catch (e){
    console.error(e);
    $('resultMsg').className = 'err';
    $('resultMsg').textContent = '送信に失敗しました。Wi-Fiを確認して、もう一度送信してください';
    $('btnRetry').style.display = 'block';
  }
}
function drawQR(url){
  const q = $('qr'); q.innerHTML = '';
  if (typeof QRCode !== 'undefined'){
    new QRCode(q, {text:url,width:240,height:240,correctLevel:QRCode.CorrectLevel.M});
  } else {
    const img = new Image(); img.width = 240; img.height = 240;
    img.src = 'https://api.qrserver.com/v1/create-qr-code/?size=240x240&data=' + encodeURIComponent(url);
    q.appendChild(img);
  }
}

/* ===== 画面遷移 ===== */
function resetAll(){
  clearTimeout(resetTimer);
  state = defaultState();
  $('startMsg').textContent = '画面をタップしてスタート！';
  show('s-start');
}
$('btnStart').onclick = async () => {
  try {
    $('startMsg').textContent = 'カメラを準備しています…';
    await startCamera();
    renderSettings();
    show('s-settings');
  } catch (e){
    console.error(e);
    $('startMsg').textContent = 'カメラが使えません。Safariの設定でカメラを許可して、もう一度押してください';
  }
};
$('btnBack').onclick = () => show('s-settings');
$('btnShoot').onclick = runShooting;
$('btnRetry').onclick = doUpload;
$('btnEnd').onclick = resetAll;
</script>
</body>
</html>
