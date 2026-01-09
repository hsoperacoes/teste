<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Teste Scanner EAN-13</title>

<!-- Dynamsoft -->
<script src="https://cdn.jsdelivr.net/npm/dynamsoft-javascript-barcode@9.6.21/dist/dbr.js"></script>

<!-- Html5Qrcode -->
<script src="https://unpkg.com/html5-qrcode"></script>

<style>
body{
  margin:0;
  background:#000;
  color:#fff;
  font-family:Arial, sans-serif;
  display:flex;
  flex-direction:column;
  align-items:center;
}

h1{font-size:18px;margin:12px}

.controls{
  display:flex;
  gap:8px;
  margin-bottom:10px;
}

button{
  padding:10px 14px;
  font-size:14px;
  font-weight:bold;
  border:none;
  border-radius:6px;
  cursor:pointer;
}

.btn-dyn{background:#7c3aed;color:#fff}
.btn-bd{background:#2563eb;color:#fff}
.btn-zx{background:#059669;color:#fff}
.btn-stop{background:#991b1b;color:#fff}

#camera{
  width:100%;
  max-width:520px;
  aspect-ratio:3/4;
  background:#000;
}

#result{
  margin-top:10px;
  font-size:16px;
  font-weight:bold;
  color:#22c55e;
}

small{color:#aaa;margin-top:6px;text-align:center}
</style>
</head>
<body>

<h1>Teste de Scanner EAN-13</h1>

<div class="controls">
  <button class="btn-dyn" onclick="startDynamsoft()">Dynamsoft</button>
  <button class="btn-bd" onclick="startBarcodeDetector()">BarcodeDetector</button>
  <button class="btn-zx" onclick="startZXing()">Html5Qrcode</button>
  <button class="btn-stop" onclick="stopAll()">Parar</button>
</div>

<div id="camera"></div>
<div id="result">—</div>
<small>Somente EAN-13 (13 dígitos)</small>

<script>
let scannerDyn = null;
let zxScanner = null;
let streamBD = null;
let bdRunning = false;

// ========= UTIL =========
function normalizeEAN13(v){
  const n = String(v||"").replace(/\D/g,"");
  return n.length === 13 ? n : "";
}
function showResult(code){
  document.getElementById("result").textContent = "EAN LIDO: " + code;
}

// ========= STOP =========
async function stopAll(){
  if(scannerDyn){
    try{
      await scannerDyn.hide();
      await scannerDyn.stop();
      await scannerDyn.destroyContext();
    }catch(e){}
    scannerDyn = null;
  }
  if(zxScanner){
    try{ await zxScanner.stop(); }catch(e){}
    zxScanner = null;
  }
  if(streamBD){
    streamBD.getTracks().forEach(t=>t.stop());
    streamBD = null;
    bdRunning = false;
  }
  document.getElementById("camera").innerHTML = "";
  document.getElementById("result").textContent = "—";
}

// ========= 1. DYNAMSOFT (TELA CLARA) =========
async function startDynamsoft(){
  await stopAll();

  const cam = document.getElementById("camera");
  cam.innerHTML = "";

  scannerDyn = await Dynamsoft.DBR.BarcodeScanner.createInstance();

  // ⚠️ remove overlay escuro
  scannerDyn.showOverlay = false;
  scannerDyn.showScanRegion = false;

  // foco + resolução
  await scannerDyn.setVideoSettings({
    video:{
      facingMode:{ideal:"environment"},
      width:{ideal:1920},
      height:{ideal:1080}
    }
  });

  let settings = await scannerDyn.getRuntimeSettings();
  settings.expectedBarcodesCount = 1;
  settings.deblurLevel = 9;
  settings.barcodeFormatIds = Dynamsoft.DBR.EnumBarcodeFormat.BF_EAN_13;
  settings.barcodeFormatIds_2 = 0;
  await scannerDyn.updateRuntimeSettings(settings);

  scannerDyn.onFrameRead = results=>{
    for(const r of results){
      const ean = normalizeEAN13(r.barcodeText);
      if(ean){
        showResult(ean);
        return;
      }
    }
  };

  await scannerDyn.show(cam);
}

// ========= 2. BARCODE DETECTOR (NATIVO CHROME) =========
async function startBarcodeDetector(){
  await stopAll();
  if(!("BarcodeDetector" in window)){
    alert("BarcodeDetector não suportado neste navegador.");
    return;
  }

  const cam = document.getElementById("camera");
  cam.innerHTML = "<video autoplay playsinline></video>";
  const video = cam.querySelector("video");

  streamBD = await navigator.mediaDevices.getUserMedia({
    video:{facingMode:"environment"}
  });
  video.srcObject = streamBD;

  const detector = new BarcodeDetector({formats:["ean_13"]});
  bdRunning = true;

  async function loop(){
    if(!bdRunning) return;
    try{
      const codes = await detector.detect(video);
      if(codes.length){
        const ean = normalizeEAN13(codes[0].rawValue);
        if(ean) showResult(ean);
      }
    }catch(e){}
    requestAnimationFrame(loop);
  }
  loop();
}

// ========= 3. HTML5QRCODE / ZXING =========
async function startZXing(){
  await stopAll();

  const cam = document.getElementById("camera");
  cam.innerHTML = "";

  zxScanner = new Html5Qrcode("camera");
  await zxScanner.start(
    { facingMode:"environment" },
    { fps:10, qrbox:{width:250,height:250} },
    text=>{
      const ean = normalizeEAN13(text);
      if(ean) showResult(ean);
    }
  );
}
</script>
</body>
</html>
