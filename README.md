<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Teste EAN-13 — Dynamsoft vs Html5Qrcode</title>

<script src="https://cdn.jsdelivr.net/npm/dynamsoft-javascript-barcode@9.6.21/dist/dbr.js"></script>
<script src="https://unpkg.com/html5-qrcode"></script>

<style>
  *{box-sizing:border-box}
  body{
    margin:0;
    background:#000;
    color:#fff;
    font-family:Arial, sans-serif;
    display:flex;
    flex-direction:column;
    align-items:center;
    padding:12px;
  }
  h1{font-size:16px;margin:8px 0 12px;text-align:center}

  .controls{
    display:flex;
    gap:8px;
    flex-wrap:wrap;
    width:100%;
    max-width:560px;
    justify-content:center;
    margin-bottom:10px;
  }
  button{
    padding:10px 14px;
    font-size:14px;
    font-weight:800;
    border:none;
    border-radius:8px;
    cursor:pointer;
  }
  .btn-dyn{background:#7c3aed;color:#fff}
  .btn-zx{background:#059669;color:#fff}
  .btn-stop{background:#991b1b;color:#fff}

  .panel{
    width:100%;
    max-width:560px;
    background:#141414;
    border:1px solid #2a2a2a;
    border-radius:10px;
    padding:10px;
    margin-bottom:10px;
  }

  #cameraWrap{
    width:100%;
    max-width:560px;
    border-radius:12px;
    overflow:hidden;
    border:1px solid #2a2a2a;
    background:#000;
    position:relative;
  }

  #camera{
    width:100%;
    min-height:420px;
    background:#000;
  }

  /* Overlay de trava — clique em qualquer lugar */
  #tapOverlay{
    position:absolute;
    inset:0;
    display:none;
    align-items:center;
    justify-content:center;
    background:rgba(0,0,0,0.35);
    z-index:50;
    padding:18px;
    text-align:center;
    font-weight:900;
  }
  #tapOverlay .box{
    width:min(520px, 92%);
    background:rgba(10,10,10,0.85);
    border:1px solid #334155;
    border-radius:14px;
    padding:14px 16px;
  }
  #tapOverlay small{
    display:block;
    margin-top:6px;
    font-weight:700;
    color:#cbd5e1;
  }

  #result{
    width:100%;
    max-width:560px;
    font-size:16px;
    font-weight:900;
    color:#22c55e;
    margin-top:8px;
    word-break:break-word;
  }
  #status{
    width:100%;
    max-width:560px;
    margin-top:6px;
    font-size:13px;
    color:#cbd5e1;
    white-space:pre-wrap;
  }
  .warn{color:#fbbf24}
  .err{color:#f97373}
</style>
</head>
<body>

<h1>Teste EAN-13 (Android) — Dynamsoft + Html5Qrcode</h1>

<div class="controls">
  <button class="btn-dyn" onclick="startDynamsoft()">Dynamsoft</button>
  <button class="btn-zx" onclick="startHtml5()">Html5Qrcode</button>
  <button class="btn-stop" onclick="stopAll()">Parar</button>
</div>

<div class="panel">
  <strong>Regras:</strong><br>
  • Lê apenas <b>EAN-13</b><br>
  • Leu → <b>trava</b><br>
  • Toque em qualquer lugar da tela para liberar o próximo
</div>

<div id="cameraWrap">
  <div id="camera"></div>

  <div id="tapOverlay" onclick="unlockScanAnywhere()">
    <div class="box">
      ✅ LIDO E TRAVADO<br/>
      Toque em qualquer lugar para ler o próximo
      <small>Evita disparos múltiplos</small>
    </div>
  </div>
</div>

<div id="result">—</div>
<div id="status">Pronto.</div>

<script>
  let dynScanner = null;
  let html5 = null;

  let locked = false;
  let lastRead = "";
  let lastReadAt = 0;

  function normalizeEAN13(v){
    const n = String(v||"").replace(/\D/g,"");
    return n.length === 13 ? n : "";
  }

  function setStatus(msg, cls=""){
    const el = document.getElementById("status");
    el.className = cls;
    el.textContent = msg;
  }

  function showResult(code){
    document.getElementById("result").textContent = "EAN LIDO: " + code;
  }

  function beep(){
    try{
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      const o = ctx.createOscillator();
      const g = ctx.createGain();
      o.type = "square";
      o.frequency.value = 880;
      g.gain.value = 0.08;
      o.connect(g);
      g.connect(ctx.destination);
      o.start();
      setTimeout(()=>{ o.stop(); ctx.close(); }, 120);
    }catch(e){}
    try{ if(navigator.vibrate) navigator.vibrate(60); }catch(e){}
  }

  function lockScan(){
    locked = true;
    document.getElementById("tapOverlay").style.display = "flex";
  }

  function unlockScanAnywhere(){
    locked = false;
    document.getElementById("tapOverlay").style.display = "none";
    setStatus("Liberado. Aponte o próximo EAN-13.");
  }

  async function stopAll(){
    locked = false;
    document.getElementById("tapOverlay").style.display = "none";
    document.getElementById("camera").innerHTML = "";
    document.getElementById("result").textContent = "—";
    setStatus("Parado.");

    if(dynScanner){
      try{
        await dynScanner.hide();
        await dynScanner.stop();
        await dynScanner.destroyContext();
      }catch(e){}
      dynScanner = null;
    }

    if(html5){
      try{ await html5.stop(); }catch(e){}
      html5 = null;
    }
  }

  /* =======================
     DYNAMSOFT
  ======================= */
  async function startDynamsoft(){
    await stopAll();
    setStatus("Iniciando Dynamsoft...");

    try{
      // ✅ LICENÇA APLICADA
      Dynamsoft.DBR.BarcodeScanner.license =
        "DLS2eyJoYW5kc2hha2VDb2RlIjoiMTA1MDEzMzM1LU1UQTFNREV6TXpNMUxYZGxZaTFVY21saGJGQnliMm8iLCJtYWluU2VydmVyVVJMIjoiaHR0cHM6Ly9tZGxzLmR5bmFtc29mdG9ubGluZS5jb20vIiwib3JnYW5pemF0aW9uSUQiOiIxMDUwMTMzMzUiLCJzdGFuZGJ5U2VydmVyVVJMIjoiaHR0cHM6Ly9zZGxzLmR5bmFtc29mdG9ubGluZS5jb20vIiwiY2hlY2tDb2RlIjoxNzIwODEyNjE2fQ==";

      // engine
      Dynamsoft.DBR.BarcodeScanner.engineResourcePath =
        "https://cdn.jsdelivr.net/npm/dynamsoft-javascript-barcode@9.6.21/dist/";

      dynScanner = await Dynamsoft.DBR.BarcodeScanner.createInstance();

      // tela clara
      try{ dynScanner.showOverlay = false; }catch(e){}
      try{ dynScanner.showScanRegion = false; }catch(e){}

      try{
        await dynScanner.setVideoSettings({
          video:{
            facingMode:{ideal:"environment"},
            width:{ideal:1920},
            height:{ideal:1080}
          }
        });
      }catch(e){}

      let settings = await dynScanner.getRuntimeSettings();
      settings.expectedBarcodesCount = 1;
      settings.deblurLevel = 9;
      settings.barcodeFormatIds = Dynamsoft.DBR.EnumBarcodeFormat.BF_EAN_13;
      settings.barcodeFormatIds_2 = 0;
      await dynScanner.updateRuntimeSettings(settings);

      dynScanner.onFrameRead = results=>{
        if(!results || !results.length || locked) return;

        for(const r of results){
          const ean = normalizeEAN13(r.barcodeText);
          if(!ean) continue;

          const now = Date.now();
          if(ean === lastRead && (now - lastReadAt) < 700) return;

          lastRead = ean;
          lastReadAt = now;

          showResult(ean);
          beep();
          lockScan();
          setStatus("Dynamsoft: lido e travado. Toque para liberar.");
          return;
        }
      };

      await dynScanner.show(document.getElementById("camera"));
      setStatus("Dynamsoft rodando. Aponte um EAN-13.");
    }catch(err){
      setStatus("ERRO Dynamsoft:\n" + (err.message || err.toString()), "err");
    }
  }

  /* =======================
     HTML5QRCODE
  ======================= */
  async function startHtml5(){
    await stopAll();
    setStatus("Iniciando Html5Qrcode...");

    try{
      document.getElementById("camera").innerHTML =
        `<div id="html5box" style="width:100%"></div>`;

      html5 = new Html5Qrcode("html5box");

      const config = {
        fps: 12,
        qrbox: { width: 250, height: 250 },
        experimentalFeatures: { useBarCodeDetectorIfSupported: true }
      };

      await html5.start(
        { facingMode:"environment" },
        config,
        decodedText=>{
          if(locked) return;

          const ean = normalizeEAN13(decodedText);
          if(!ean) return;

          const now = Date.now();
          if(ean === lastRead && (now - lastReadAt) < 700) return;

          lastRead = ean;
          lastReadAt = now;

          showResult(ean);
          beep();
          lockScan();
          setStatus("Html5Qrcode: lido e travado. Toque para liberar.");
        },
        ()=>{}
      );

      setStatus("Html5Qrcode rodando. Aponte um EAN-13.");
    }catch(err){
      setStatus("ERRO Html5Qrcode:\n" + (err.message || err.toString()), "err");
    }
  }
</script>

</body>
</html>
