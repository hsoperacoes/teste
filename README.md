<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Teste — Leitor Chave NF (44)</title>

  <script src="https://unpkg.com/html5-qrcode"></script>

  <style>
    *{box-sizing:border-box}
    body{
      margin:0; font-family:Arial, sans-serif;
      background:#0b0b0c; color:#fff;
      min-height:100vh; display:flex; align-items:center; justify-content:center;
      padding:20px;
    }
    .wrap{
      width:100%; max-width:900px;
      background:#151517; border:1px solid #2a2a2a;
      border-radius:14px; padding:18px;
      box-shadow:0 18px 60px rgba(0,0,0,.45);
    }
    h1{margin:0 0 10px; font-size:18px}
    .row{display:flex; gap:10px; flex-wrap:wrap; align-items:center}
    input{
      flex:1; min-width:260px;
      padding:12px 12px; border-radius:10px;
      border:1px solid #333; background:#0f0f10; color:#fff;
      font-size:14px;
    }
    button{
      padding:12px 14px; border-radius:10px; border:0;
      background:#673ab7; color:#fff; font-weight:800; cursor:pointer;
    }
    button:hover{filter:brightness(1.08)}
    .muted{color:#cbd5e1; font-size:13px; margin-top:10px; line-height:1.4}
    .ok{color:#bbf7d0; font-weight:800}
    .err{color:#fca5a5; font-weight:800}

    /* Overlay do scanner */
    #scannerOverlay{
      position:fixed; inset:0;
      display:none;
      background:rgba(0,0,0,.85);
      backdrop-filter: blur(6px);
      z-index:9999;
      align-items:center; justify-content:center;
      padding:18px;
    }
    .scannerBox{
      width:min(820px, 96vw);
      background:#0b0b0c;
      border:2px solid #673ab7;
      border-radius:16px;
      padding:14px;
      position:relative;
      box-shadow:0 18px 70px rgba(103,58,183,.25);
    }
    #reader{
      width:100%;
      border-radius:12px;
      overflow:hidden;
    }
    .scannerTop{
      display:flex; justify-content:space-between; align-items:center;
      gap:10px; margin-bottom:10px;
    }
    .scannerTop .title{
      font-weight:900; letter-spacing:.3px;
      display:flex; gap:10px; align-items:center;
    }
    .badge{
      font-size:12px; font-weight:900;
      padding:6px 10px; border-radius:999px;
      background:#1f2937; border:1px solid #374151; color:#e5e7eb;
    }
    /* ✅ botão fechar SEM sobrepor nada interno do leitor */
    .btnClose{
      background:#dc2626;
    }
    .btnClose:hover{filter:brightness(1.05)}
    .hint{
      margin-top:10px; font-size:13px; color:#cbd5e1;
      background:rgba(31,41,55,.55);
      border:1px solid #374151;
      padding:10px 12px; border-radius:12px;
    }

    .log{
      margin-top:12px;
      background:#0f0f10; border:1px solid #2a2a2a;
      border-radius:12px; padding:10px; font-family:ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono";
      font-size:12px; color:#cbd5e1; max-height:140px; overflow:auto;
    }

    @media (max-width:560px){
      .scannerTop{flex-direction:column; align-items:stretch}
      button{width:100%}
    }
  </style>
</head>
<body>
  <div class="wrap">
    <h1>Teste — Leitor da Chave da NF (44 dígitos)</h1>

    <div class="row">
      <input id="chave" maxlength="44" placeholder="A chave (44) vai aparecer aqui automaticamente..." />
      <button id="btnAbrir">📷 Ler chave</button>
      <button id="btnLimpar" style="background:#374151">🧹 Limpar</button>
    </div>

    <div class="muted">
      - Se o código vier com letras/símbolos, eu extraio só os números e pego os <span class="ok">44 dígitos</span>.<br>
      - Se a câmera abrir mas não ler, tente aproximar/afastar e deixar bem focado (código de barras precisa de contraste).
    </div>

    <div id="status" class="muted" style="margin-top:8px;"></div>

    <div class="log" id="log"></div>
  </div>

  <!-- Overlay do scanner -->
  <div id="scannerOverlay">
    <div class="scannerBox">
      <div class="scannerTop">
        <div class="title">
          <span>📷 Scanner NF</span>
          <span class="badge">captura automática</span>
        </div>
        <button class="btnClose" id="btnFechar">✖ Fechar</button>
      </div>

      <div id="reader"></div>

      <div class="hint">
        Aponte para o <b>código de barras</b> da NF. Ao ler, o sistema preenche a chave e fecha automaticamente.
      </div>
    </div>
  </div>

  <script>
    // =========================
    // Utils
    // =========================
    const $ = (id) => document.getElementById(id);

    function log(msg){
      const el = $("log");
      const now = new Date();
      const ts = now.toLocaleTimeString();
      el.innerHTML = `[${ts}] ${msg}<br>` + el.innerHTML;
    }

    function setStatus(msg, kind=""){
      const el = $("status");
      el.className = "muted " + (kind === "ok" ? "ok" : kind === "err" ? "err" : "");
      el.textContent = msg || "";
    }

    // Extrai 44 dígitos de qualquer texto lido
    function extrairChave44(texto){
      const only = String(texto || "").replace(/\D/g, "");
      if (only.length < 44) return "";
      // pega a primeira sequência de 44 dentro do texto
      for (let i=0; i<=only.length-44; i++){
        const cand = only.slice(i, i+44);
        if (/^\d{44}$/.test(cand)) return cand;
      }
      return "";
    }

    // =========================
    // Scanner (Html5Qrcode)
    // =========================
    let html5 = null;
    let aberto = false;
    let travado = false; // evita múltiplos eventos

    async function abrirScanner(){
      if (aberto) return;

      $("scannerOverlay").style.display = "flex";
      aberto = true;
      travado = false;
      setStatus("Abrindo câmera...", "");

      // Cria instância sempre nova (isso evita “ficou preso”)
      html5 = new Html5Qrcode("reader");

      // Config: tenta barcode detector nativo se existir
      const config = {
        fps: 14,
        qrbox: { width: 320, height: 220 },
        experimentalFeatures: { useBarCodeDetectorIfSupported: true },
        rememberLastUsedCamera: true
      };

      // Tenta usar câmera traseira se disponível
      const cameraConfig = { facingMode: "environment" };

      try{
        await html5.start(
          cameraConfig,
          config,
          onDecoded,
          (err) => { /* pode ignorar “NotFoundException” repetitivo */ }
        );

        setStatus("Câmera aberta. Aponte para o código de barras…", "");
        log("Scanner iniciado.");
      }catch(e){
        setStatus("Falha ao abrir câmera: " + (e.message || e), "err");
        log("ERRO abrir câmera: " + (e.message || e));
        await fecharScanner();
      }
    }

    async function fecharScanner(){
      if (!aberto) return;

      try{
        if (html5){
          await html5.stop();
          await html5.clear();
        }
      }catch(e){
        // ignore
      }

      html5 = null;
      aberto = false;
      $("scannerOverlay").style.display = "none";
      log("Scanner fechado.");
    }

    async function onDecoded(decodedText, decodedResult){
      if (travado) return;

      // Log do que está lendo (ajuda a diagnosticar)
      log("LIDO: " + String(decodedText).slice(0, 120));

      const chave = extrairChave44(decodedText);
      if (!chave) return;

      travado = true;

      $("chave").value = chave;
      setStatus("✅ Chave capturada e preenchida!", "ok");
      log("CHAVE 44 OK: " + chave);

      // Fecha a câmera logo após capturar
      await new Promise(r => setTimeout(r, 120));
      await fecharScanner();
    }

    // =========================
    // UI
    // =========================
    $("btnAbrir").addEventListener("click", abrirScanner);
    $("btnFechar").addEventListener("click", fecharScanner);

    $("btnLimpar").addEventListener("click", ()=>{
      $("chave").value = "";
      setStatus("");
      log("Campo limpo.");
    });

    // ESC fecha
    window.addEventListener("keydown", (e)=>{
      if (e.key === "Escape") fecharScanner();
    });

    // Clica fora NÃO fecha (evita fechar sem querer), mas se quiser eu habilito.
  </script>
</body>
</html>
