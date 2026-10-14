# ds_1014_245204
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<title>キーホルダー</title>
<style>
  html, body {
    margin: 0;
    height: 100%;
    overflow: hidden;
    background: #f4efe8;
    touch-action: none;
    -webkit-user-select: none;
    user-select: none;
  }
  canvas {
    display: block;
    width: 100vw;
    height: 100vh;
    cursor: grab;
  }
  canvas.dragging { cursor: grabbing; }
  .hint {
    position: fixed;
    left: 0; right: 0; bottom: 24px;
    text-align: center;
    font: 14px/1.4 -apple-system, "Hiragino Sans", "Yu Gothic", sans-serif;
    color: #9a8f82;
    pointer-events: none;
    transition: opacity .6s;
  }
</style>
</head>
<body>
<canvas id="c"></canvas>
<div class="hint" id="hint">触ってみて</div>

<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  const hint = document.getElementById('hint');

  let W, H, DPR;
  function resize() {
    DPR = window.devicePixelRatio || 1;
    W = window.innerWidth;
    H = window.innerHeight;
    canvas.width = W * DPR;
    canvas.height = H * DPR;
    ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
    if (pts) pts[0].x = pts[0].px = W / 2;
  }

  // ---- 物理モデル(ベルレ積分) ----
  const CHAIN = 5;          // チェーンの点の数(アンカー除く)
  const LINK = 22;          // チェーンの1節の長さ
  const CHARM_LEN = 70;     // チャームの吊り穴〜下端の距離
  const GRAVITY = 0.55;
  const DAMP = 0.992;

  let pts = null, cons = [];
  function init() {
    pts = []; cons = [];
    const ax = window.innerWidth / 2, ay = 70;
    pts.push({ x: ax, y: ay, px: ax, py: ay, pin: true });
    for (let i = 1; i <= CHAIN; i++) {
      // 少し斜めに置いて、最初から軽く揺れるように
      const x = ax + i * 4, y = ay + i * LINK;
      pts.push({ x, px: x, y, py: y });
      cons.push([i - 1, i, LINK]);
    }
    const top = pts[CHAIN];
    const bx = top.x + 20, by = top.y + CHARM_LEN;
    pts.push({ x: bx, y: by, px: bx, py: by });
    cons.push([CHAIN, CHAIN + 1, CHARM_LEN]);
  }

  resize();
  init();
  window.addEventListener('resize', resize);

  const IA = CHAIN;      // チャームの吊り穴側の点
  const IB = CHAIN + 1;  // チャームの下端側の点

  function step() {
    for (const p of pts) {
      if (p.pin) continue;
      const vx = (p.x - p.px) * DAMP;
      const vy = (p.y - p.py) * DAMP;
      p.px = p.x; p.py = p.y;
      p.x += vx;
      p.y += vy + GRAVITY;
    }
    for (let it = 0; it < 14; it++) {
      for (const [i, j, len] of cons) {
        const a = pts[i], b = pts[j];
        const dx = b.x - a.x, dy = b.y - a.y;
        const d = Math.hypot(dx, dy) || 0.0001;
        const diff = (d - len) / d;
        const wa = a.pin ? 0 : 1, wb = b.pin ? 0 : 1;
        const w = wa + wb;
        if (!w) continue;
        a.x += dx * diff * (wa / w);
        a.y += dy * diff * (wa / w);
        b.x -= dx * diff * (wb / w);
        b.y -= dy * diff * (wb / w);
      }
      // 画面の端で止める
      for (const p of pts) {
        if (p.pin) continue;
        if (p.x < 10) p.x = 10;
        if (p.x > W - 10) p.x = W - 10;
        if (p.y > H - 10) p.y = H - 10;
      }
    }
  }

  // ---- 入力(マウス/タッチ共通) ----
  let grab = null;       // 掴んでいる点のindex
  let last = null;       // 直前のポインタ位置
  function pos(e) { return { x: e.clientX, y: e.clientY }; }

  function distToSeg(p, a, b) {
    const abx = b.x - a.x, aby = b.y - a.y;
    let t = ((p.x - a.x) * abx + (p.y - a.y) * aby) / (abx * abx + aby * aby);
    t = Math.max(0, Math.min(1, t));
    return { d: Math.hypot(p.x - (a.x + abx * t), p.y - (a.y + aby * t)), t };
  }

  function pick(p) {
    const s = distToSeg(p, pts[IA], pts[IB]);
    if (s.d < 60) return s.t > 0.4 ? IB : IA;
    let best = null, bd = 28;
    for (let i = 1; i < CHAIN; i++) {
      const d = Math.hypot(p.x - pts[i].x, p.y - pts[i].y);
      if (d < bd) { bd = d; best = i; }
    }
    return best;
  }

  function poke(p, vx, vy) {
    // 触れた/なぞった勢いを速度として与える
    const s = distToSeg(p, pts[IA], pts[IB]);
    if (s.d < 55) {
      for (const i of [IA, IB]) {
        const k = i === IB ? s.t : 1 - s.t;
        pts[i].px -= vx * 0.35 * k;
        pts[i].py -= vy * 0.35 * k;
      }
      return true;
    }
    return false;
  }

  function hideHint() { hint.style.opacity = 0; }

  canvas.addEventListener('pointerdown', e => {
    canvas.setPointerCapture(e.pointerId);
    const p = pos(e);
    last = p;
    grab = pick(p);
    if (grab !== null) {
      canvas.classList.add('dragging');
      hideHint();
      // タップしただけでもプルッと揺れるように
      pts[IB].px -= (Math.random() - 0.5) * 6;
    }
  });

  canvas.addEventListener('pointermove', e => {
    const p = pos(e);
    if (grab !== null) {
      const g = pts[grab];
      g.x = p.x; g.y = p.y;
      g.px = g.x; g.py = g.y;
    } else if (last) {
      // マウスを乗せてなぞっただけでも揺れる
      if (poke(p, p.x - last.x, p.y - last.y)) hideHint();
    }
    last = p;
  });

  function release() {
    if (grab !== null) {
      // 離した時、ポインタの勢いが残るようにする
      canvas.classList.remove('dragging');
      grab = null;
    }
  }
  canvas.addEventListener('pointerup', release);
  canvas.addEventListener('pointercancel', release);

  // ---- 描画 ----
  function drawChain() {
    ctx.lineWidth = 3.2;
    ctx.strokeStyle = '#b9b3aa';
    ctx.lineCap = 'round';
    for (let i = 0; i < IA; i++) {
      const a = pts[i], b = pts[i + 1];
      const mx = (a.x + b.x) / 2, my = (a.y + b.y) / 2;
      const ang = Math.atan2(b.y - a.y, b.x - a.x);
      ctx.save();
      ctx.translate(mx, my);
      ctx.rotate(ang);
      ctx.beginPath();
      ctx.ellipse(0, 0, LINK * 0.62, 4.6, 0, 0, Math.PI * 2);
      ctx.stroke();
      ctx.restore();
    }
  }

  function roundRect(x, y, w, h, r) {
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.arcTo(x + w, y, x + w, y + h, r);
    ctx.arcTo(x + w, y + h, x, y + h, r);
    ctx.arcTo(x, y + h, x, y, r);
    ctx.arcTo(x, y, x + w, y, r);
    ctx.closePath();
  }

  function drawCharm() {
    const a = pts[IA], b = pts[IB];
    const ang = Math.atan2(b.y - a.y, b.x - a.x) - Math.PI / 2;
    ctx.save();
    ctx.translate(a.x, a.y);
    ctx.rotate(ang);

    // 影
    ctx.shadowColor = 'rgba(80,60,40,.22)';
    ctx.shadowBlur = 18;
    ctx.shadowOffsetY = 8;

    // 風船の本体(結び目側を吊り下げた形)
    const g = ctx.createRadialGradient(-14, 92, 6, 0, 96, 56);
    g.addColorStop(0, '#ff6b6b');
    g.addColorStop(0.65, '#e5262e');
    g.addColorStop(1, '#b3151d');
    ctx.fillStyle = g;
    ctx.beginPath();
    ctx.moveTo(0, 40);
    ctx.bezierCurveTo(-14, 44, -46, 62, -44, 92);
    ctx.bezierCurveTo(-42, 122, -20, 138, 0, 138);
    ctx.bezierCurveTo(20, 138, 42, 122, 44, 92);
    ctx.bezierCurveTo(46, 62, 14, 44, 0, 40);
    ctx.closePath();
    ctx.fill();

    ctx.shadowColor = 'transparent';

    // 結び目
    ctx.fillStyle = '#a8121a';
    ctx.beginPath();
    ctx.moveTo(-7, 31);
    ctx.lineTo(7, 31);
    ctx.lineTo(3, 41);
    ctx.lineTo(-3, 41);
    ctx.closePath();
    ctx.fill();

    // つや
    ctx.fillStyle = 'rgba(255,255,255,.6)';
    ctx.beginPath(); ctx.ellipse(-22, 88, 7, 15, 0.35, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = 'rgba(255,255,255,.35)';
    ctx.beginPath(); ctx.arc(-16, 114, 3.5, 0, Math.PI * 2); ctx.fill();

    // 金具のリング
    ctx.strokeStyle = '#b9b3aa';
    ctx.lineWidth = 3.2;
    ctx.beginPath(); ctx.arc(0, 22, 9, 0, Math.PI * 2); ctx.stroke();

    ctx.restore();
  }

  function drawAnchor() {
    const a = pts[0];
    ctx.fillStyle = '#d8d0c4';
    ctx.beginPath(); ctx.arc(a.x, a.y, 9, 0, Math.PI * 2); ctx.fill();
    ctx.strokeStyle = '#b9b3aa';
    ctx.lineWidth = 3.2;
    ctx.beginPath(); ctx.arc(a.x, a.y, 15, 0, Math.PI * 2); ctx.stroke();
  }

  function frame() {
    step();
    ctx.clearRect(0, 0, W, H);
    drawAnchor();
    drawChain();
    drawCharm();
    requestAnimationFrame(frame);
  }
  frame();
})();
</script>
</body>
</html>
