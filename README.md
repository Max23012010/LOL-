<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Trippy Infinite Zoom — Debug Friendly</title>
<style>
  html,body{margin:0;padding:0;height:100%;overflow:hidden;background:#000;}
  canvas{position:absolute;top:0;left:0;width:100vw;height:100vh;display:block;filter:contrast(1.3) brightness(1.2) saturate(1.6);}
  .overlay{position:fixed;bottom:20px;width:100%;text-align:center;color:white;font-family:monospace;pointer-events:none;}
  .controls{position:fixed;top:12px;left:12px;z-index:999;pointer-events:auto;color:#eee;font-family:system-ui,monospace;}
  .controls input[type="text"]{width:360px;padding:6px;border-radius:6px;border:1px solid rgba(255,255,255,0.12);background:rgba(0,0,0,0.4);color:#fff}
  .controls button{margin-left:8px;padding:6px 10px;border-radius:6px;background:#222;color:#fff;border:1px solid rgba(255,255,255,0.06)}
  .hint{opacity:0.9;font-size:13px;margin-top:6px;color:#ccc}
</style>
</head>
<body>
<canvas id="zoomCanvas"></canvas>
<div class="overlay">🌀 Move your mouse or touch — fall deeper 🌀</div>

<div class="controls" aria-hidden="false">
  <div>
    <input id="imgUrl" type="text" placeholder="Paste image URL (CORS-friendly) or leave blank for default" />
    <button id="loadBtn">Load</button>
    <button id="toggleBtn">Pause</button>
  </div>
  <div class="hint">Use a CORS-enabled image (Access-Control-Allow-Origin:*). For local tests, host via a local server.</div>
</div>

<script>
(function(){
  // Basic guards
  const canvas = document.getElementById('zoomCanvas');
  if (!canvas) { console.error('Canvas not found.'); return; }
  const ctx = canvas.getContext && canvas.getContext('2d');
  if (!ctx) { console.error('2D context unavailable.'); return; }

  // HiDPI support
  function resizeCanvas() {
    const dpr = Math.max(1, window.devicePixelRatio || 1);
    canvas.style.width = window.innerWidth + 'px';
    canvas.style.height = window.innerHeight + 'px';
    canvas.width = Math.round(window.innerWidth * dpr);
    canvas.height = Math.round(window.innerHeight * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // keep drawing coordinates in CSS pixels
  }
  window.addEventListener('resize', resizeCanvas);
  resizeCanvas();

  // Input / parallax
  let mouseX = 0, mouseY = 0;
  function setPointer(px, py){
    mouseX = (px / window.innerWidth - 0.5) * 2;
    mouseY = (py / window.innerHeight - 0.5) * 2;
  }
  window.addEventListener('mousemove', (e) => setPointer(e.clientX, e.clientY));
  window.addEventListener('touchmove', (e) => {
    if (e.touches && e.touches[0]) setPointer(e.touches[0].clientX, e.touches[0].clientY);
  }, {passive:true});

  // Image loader with CORS-aware handling
  let img = new Image();
  let animRunning = true;
  const defaultSrc = 'https://www.dropbox.com/scl/fi/xmja9rqztz8gwk6jjux4i/Studio_20231231_193239-removebg-preview.png?raw=1';

  function createImage(src){
    const i = new Image();
    i.crossOrigin = 'anonymous';
    i.decoding = 'async';
    i.src = src;
    return i;
  }

  function loadAndStart(src){
    if (!src) src = defaultSrc;
    img = createImage(src);
    console.info('Attempting to load image:', src);
    img.onload = () => {
      console.info('Image loaded:', src, 'size', img.width + 'x' + img.height);
      startAnimation();
    };
    img.onerror = (ev) => {
      console.error('Image load failed:', ev);
      // fallback to a small data URI to avoid taint and still show something
      const fallback = 'data:image/svg+xml;charset=utf-8,' + encodeURIComponent(
        '<svg xmlns="http://www.w3.org/2000/svg" width="512" height="512"><rect width="100%" height="100%" fill="#222"/><text x="50%" y="50%" font-size="28" fill="#fff" text-anchor="middle" dominant-baseline="central">Image failed to load</text></svg>'
      );
      img = createImage(fallback);
      img.onload = () => { console.warn('Using fallback placeholder image.'); startAnimation(); };
      img.onerror = () => console.error('Fallback also failed; check console/network.');
    };
  }

  // Controls
  document.getElementById('loadBtn').addEventListener('click', () => {
    const url = document.getElementById('imgUrl').value.trim();
    loadAndStart(url || defaultSrc);
  });
  const toggleBtn = document.getElementById('toggleBtn');
  toggleBtn.addEventListener('click', () => {
    animRunning = !animRunning;
    toggleBtn.textContent = animRunning ? 'Pause' : 'Resume';
    if (animRunning) requestAnimationFrame(drawLoop);
  });

  // Animation
  let stopFlag = false;
  let lastT = 0;
  function startAnimation(){
    stopFlag = false;
    lastT = 0;
    if (animRunning) requestAnimationFrame(drawLoop);
  }

  function drawLoop(t){
    try {
      if (!animRunning || stopFlag) return;
      const dt = t - lastT;
      lastT = t;
      // clear and draw
      ctx.clearRect(0,0,canvas.width,canvas.height);
      ctx.save();

      // CSS-pixel center + parallax in CSS pixels
      const cx = window.innerWidth / 2 + mouseX * Math.min(window.innerWidth, 180);
      const cy = window.innerHeight / 2 + mouseY * Math.min(window.innerHeight, 180);
      ctx.translate(cx, cy);

      // compute a base scale so the image covers the screen (cover behavior)
      const baseScale = Math.max(window.innerWidth / img.width, window.innerHeight / img.height) * 1.05;

      // adapt layers to device performance
      const maxLayers = Math.max(12, Math.round( Math.min(60, 36 * (window.devicePixelRatio || 1)) ));
      const layers = Math.min(maxLayers, 60);

      for (let i = 0; i < layers; i++) {
        const oscill = 1 + 0.05 * Math.sin(t/300 + i/5);
        const layerScale = baseScale * Math.pow(0.9, i) * oscill;
        ctx.save();
        ctx.scale(layerScale, layerScale);
        ctx.rotate(0.03 * Math.sin(t/400 + i/5));
        ctx.globalAlpha = Math.max(0, 1 - i / layers);
        ctx.drawImage(img, -img.width / 2, -img.height / 2);
        ctx.restore();
      }

      ctx.restore();
      requestAnimationFrame(drawLoop);
    } catch (err) {
      console.error('Animation error:', err);
      // Specific advice for CORS/taint issues
      if (err && (err.name === 'SecurityError' || /taint/i.test(err.message || ''))) {
        console.error('Canvas was tainted due to cross-origin image. Solutions:');
        console.error(' • Host the image on a server that sends Access-Control-Allow-Origin: *');
        console.error(' • Use image.crossOrigin = "anonymous" and ensure the host allows cross-origin requests');
        console.error(' • Host image on your domain or use GitHub Pages / a CORS-friendly host');
        // stop further requests to avoid spamming errors
        stopFlag = true;
      }
    }
  }

  // Start with default image
  loadAndStart(defaultSrc);
})();
</script>

<!-- Hidden autoplay SoundCloud audio -->
<iframe 
  style="visibility:hidden; position:absolute; width:0; height:0; border:none;" 
  allow="autoplay" 
  src="https://w.soundcloud.com/player/?url=https%3A//soundcloud.com/juggsi-music/no-signal-slowed&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false">
</iframe>

</body>
</html>