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
    display:flex; gap:8px; flex-wrap:wrap;
    width:100%; max-width:560px;
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
  /* área do vídeo */
  #camera{
    width:100%;
    min-height:420px;
    background:#000;
  }

  /* Overlay “toque para próximo” (clica em qualquer lugar) */
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

<h1>Teste EAN-13 (Android) — Dynamsoft “claro” + Html5Qrcode com trava por toque</h1>

<div class="controls">
  <button class="btn-dyn" onclick="startDynamsoft()">Dynamsoft</button>
  <button class="btn-zx" onclick="startHtml5()">Html5Qrcode</button>
  <button class="btn-stop" onclick="stopAll()">Parar</button>
</div>

<div class="panel">
  <div><strong>Como funciona:</strong></div>
  <div>• Lê apenas <strong>EAN-13</strong> (13 dígitos).</div>
  <div>• Html5Qrcode: leu 1 vez → <strong>trava</strong> → você <strong>toca em qualquer lugar</strong> para liberar o próximo.</div>
  <div class="warn">Se o Dynamsoft não abrir, vai aparecer o erro aqui embaixo.</div>
</div>

<div id="cameraWrap">
  <div id="camera"></div>

  <!-- qualquer clique no overlay libera o próximo scan -->
  <div id="tapOverlay" onclick="unlockScanAnywhere()">
    <div class="box">
      ✅ LIDO E TRAVADO<br/>
      Toque em qualquer lugar para ler o próximo
      <small>Isso evita disparo duplo/múltiplo.</small>
    </div>
  </div>
</div>

<div id="result">—</div>
<div id="status">Pronto.</div>

<script>
  let dynScanner = null;
  let html5 = null;

  // trava de leitura (para html5 e também pode usar no dynamsoft se quiser)
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

    // Dynamsoft
    if(dynScanner){
      try{
        await dynScanner.hide();
        await dynScanner.stop();
        await dynScanner.destroyContext();
      }catch(e){}
      dynScanner = null;
    }

    // Html5Qrcode
    if(html5){
      try{ await html5.stop(); }catch(e){}
      html5 = null;
    }
  }

  // =========================
  // 1) DYNAMSOFT (AJUSTADO)
  // =========================
  async function startDynamsoft(){
    await stopAll();

    setStatus("Iniciando Dynamsoft...");

    try{
      // ✅ COLE SUA LICENÇA AQUI (sem isso, em alguns casos no Android ele nem abre direito)
      // Exemplo: Dynamsoft.DBR.BarcodeScanner.license = "SUA_LICENÇA";
      Dynamsoft.DBR.BarcodeScanner.license = "COLE_SUA_LICENCA_AQUI";

      // ✅ garante que ele ache o engine (WASM/worker)
      Dynamsoft.DBR.BarcodeScanner.engineResourcePath =
        "https://cdn.jsdelivr.net/npm/dynamsoft-javascript-barcode@9.6.21/dist/";

      // cria scanner
      dynScanner = await Dynamsoft.DBR.BarcodeScanner.createInstance();

      // ✅ “tela clara”: remove overlay/scan region
      // (se a sua versão não aceitar, não quebra — só ignora)
      try{ dynScanner.showOverlay = false; }catch(e){}
      try{ dynScanner.showScanRegion = false; }catch(e){}

      // vídeo com câmera traseira
      try{
        await dynScanner.setVideoSettings({
          video:{
            facingMode:{ideal:"environment"},
            width:{ideal:1920},
            height:{ideal:1080}
          }
        });
      }catch(e){}

      // runtime settings: só EAN-13
      let settings = await dynScanner.getRuntimeSettings();
      settings.expectedBarcodesCount = 1;
      settings.deblurLevel = 9;
      settings.barcodeFormatIds = Dynamsoft.DBR.EnumBarcodeFormat.BF_EAN_13;
      settings.barcodeFormatIds_2 = 0;
      await dynScanner.updateRuntimeSettings(settings);

      dynScanner.onFrameRead = results => {
        if(!results || !results.length) return;
        if(locked) return;

        for(const r of results){
          const ean = normalizeEAN13(r.barcodeText);
          if(!ean) continue;

          const now = Date.now();
          // anti-disparo: se repetir muito rápido, ignora
          if(ean === lastRead && (now - lastReadAt) < 700) return;

          lastRead = ean; lastReadAt = now;
          showResult(ean);
          beep();
          lockScan();
          setStatus("Dynamsoft: lido e travado. Toque para liberar o próximo.");
          return;
        }
      };

      await dynScanner.show(document.getElementById("camera"));

      setStatus("Dynamsoft rodando. Aponte um EAN-13.");
    }catch(err){
      setStatus(
        "ERRO Dynamsoft:\n" + (err && (err.message || err.toString()) ? (err.message || err.toString()) : String(err)) +
        "\n\nDICAS:\n- Cole a licença real em DLS2eyJoYW5kc2hha2VDb2RlIjoiMTA1MDEzMzM1LU1UQTFNREV6TXpNMUxYZGxZaTFVY21saGJGQnliMm8iLCJtYWluU2VydmVyVVJMIjoiaHR0cHM6Ly9tZGxzLmR5bmFtc29mdG9ubGluZS5jb20vIiwib3JnYW5pemF0aW9uSUQiOiIxMDUwMTMzMzUiLCJzdGFuZGJ5U2VydmVyVVJMIjoiaHR0cHM6Ly9zZGxzLmR5bmFtc29mdG9ubGluZS5jb20vIiwiY2hlY2tDb2RlIjoxNzIwODEyNjE2fQ==\n- Teste em HTTPS (GitHub Pages)\n- Permita câmera no Chrome",
        "err"
      );
    }
  }

  // =========================
  // 2) HTML5QRCODE (TRAVA POR TOQUE)
  // =========================
  async function startHtml5(){
    await stopAll();
    setStatus("Iniciando Html5Qrcode...");

    try{
      // monta container do html5 dentro de #camera
      document.getElementById("camera").innerHTML = `<div id="html5box" style="width:100%"></div>`;

      html5 = new Html5Qrcode("html5box");

      // config: o “quadrado” que você curtiu
      const config = {
        fps: 12,
        qrbox: { width: 250, height: 250 },
        // importante: tenta focar mais rápido em barras
        experimentalFeatures: { useBarCodeDetectorIfSupported: true }
      };

      locked = false;

      await html5.start(
        { facingMode: "environment" },
        config,
        (decodedText) => {
          if(locked) return;

          const ean = normalizeEAN13(decodedText);
          if(!ean) return;

          const now = Date.now();
          // anti-disparo extra (mesmo destravado)
          if(ean === lastRead && (now - lastReadAt) < 700) return;

          lastRead = ean; lastReadAt = now;

          showResult(ean);
          beep();
          lockScan();
          setStatus("Html5Qrcode: lido e travado. Toque para liberar o próximo.");
        },
        (err) => {
          // silencioso (pra não lotar)
        }
      );

      setStatus("Html5Qrcode rodando. Aponte um EAN-13.");
    }catch(err){
      setStatus("ERRO Html5Qrcode:\n" + (err.message || err.toString()), "err");
    }
  }
</script>

</body>
</html>
