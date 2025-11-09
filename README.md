<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ponto Eletrônico QR - Final (corrigido)</title>

<style>
  :root{--card:#fff;--muted:#666}
  body{font-family:Arial,Helvetica,sans-serif;background:#f3f4f6;margin:0;padding:18px}
  .wrap{max-width:1150px;margin:0 auto}
  header{display:flex;justify-content:space-between;align-items:center;margin-bottom:14px}
  h1{margin:0}
  .layout{display:grid;grid-template-columns:360px 1fr;gap:14px}
  .card{background:var(--card);padding:14px;border-radius:8px;box-shadow:0 6px 18px rgba(0,0,0,0.06)}
  input,select,textarea,button{font-size:14px}
  input,select,textarea{width:100%;padding:8px;margin:6px 0;border:1px solid #ddd;border-radius:6px;box-sizing:border-box}
  button{padding:9px 12px;border-radius:8px;border:0;background:#0f172a;color:#fff;cursor:pointer}
  .small{font-size:13px;color:var(--muted)}
  .actions{display:flex;gap:8px;flex-wrap:wrap}
  table{width:100%;border-collapse:collapse;margin-top:8px;font-size:13px}
  th,td{padding:8px;border:1px solid #eee;text-align:left}
  .avatar{width:56px;height:56px;border-radius:8px;object-fit:cover;border:1px solid #eee}
  .qr-preview img{width:180px}
  .hidden{display:none}
  @media (max-width:980px){ .layout{grid-template-columns:1fr} }
</style>

<!-- Firebase modular + expose -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-app.js";
  import { getFirestore } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";
  import { getStorage } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-storage.js";

  // --- COLOQUE SUA CONFIG AQUI (já preenchida com a sua) ---
  const firebaseConfig = {
    apiKey: "AIzaSyCiZXZ9vW-4L471ej9jWg_1MAStD44pTqo",
    authDomain: "ponto-qrcode-29f9d.firebaseapp.com",
    projectId: "ponto-qrcode-29f9d",
    storageBucket: "ponto-qrcode-29f9d.firebasestorage.app",
    messagingSenderId: "900058332385",
    appId: "1:900058332385:web:2ecdabb9b4027bc3748ba0"
  };

  const app = initializeApp(firebaseConfig);
  const db = getFirestore(app);
  const storage = getStorage(app);

  // expose to window for scripts below
  window._FB = { app, db, storage };
</script>

<!-- External libs -->
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
<div class="wrap">
  <header>
    <h1>Ponto Eletrônico — Sistema Completo (corrigido)</h1>
    <div class="small">Firestore + Storage • QR scanner • Export XLSX</div>
  </header>

  <div class="layout">
    <!-- Admin / Colaboradores -->
    <div class="card">
      <h3>Admin — colaboradores</h3>
      <label class="small">Matrícula</label>
      <input id="colMatricula" placeholder="ex: 12345" />
      <label class="small">Nome</label>
      <input id="colNome" placeholder="Nome completo" />
      <label class="small">Foto (opcional)</label>
      <input id="colFoto" type="file" accept="image/*" />
      <div class="actions" style="margin-top:8px">
        <button id="btnSalvarCol">Salvar / Atualizar</button>
        <button id="btnGerarQR">Gerar QR (preview)</button>
        <button id="btnLimpar">Limpar</button>
      </div>
      <div id="qrPreview" class="qr-preview small" style="margin-top:10px"></div>
      <h4 style="margin-top:14px">Lista de colaboradores</h4>
      <div style="max-height:320px;overflow:auto">
        <table id="colTable">
          <thead><tr><th>Mat</th><th>Nome</th><th>Foto</th><th>Ações</th></tr></thead>
          <tbody></tbody>
        </table>
      </div>
    </div>
    <!-- Painel, scanner, histórico -->
    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <h3>Painel</h3>
        <div>
          <button id="btnExportExcel">Exportar Excel</button>
          <button id="btnRefresh">Atualizar</button>
        </div>
      </div>
      <div style="display:flex;gap:12px;align-items:flex-start;flex-wrap:wrap;margin-top:8px">
        <div style="min-width:300px">
          <h4 class="small">Bater ponto (QR Scanner)</h4>
          <div id="reader" style="width:300px"></div>
          <div id="scanStatus" class="small" style="margin-top:6px"></div>
        </div>
        <div style="flex:1">
          <h4 class="small">Histórico / Relatório</h4>
          <table id="pontosTable">
            <thead><tr><th>Colab</th><th>Tipo</th><th>Data</th><th>Hora</th><th>Lat,Lon</th><th>Horas</th></tr></thead>
            <tbody></tbody>
          </table>
        </div>
      </div>
      <h4 style="margin-top:12px" class="small">Gráfico horas por colaborador</h4>
      <canvas id="chartHoras" height="120"></canvas>
    </div>
  </div>
</div>

<!-- Main logic (module) -->
<script type="module">
  // firebase functions we need
  import {
    collection, doc, setDoc, getDoc, addDoc, getDocs, deleteDoc, query, orderBy
  } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";

  import { ref as storageRef, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-storage.js";

  const { db, storage } = window._FB;

  // helpers
  const $ = (id) => document.getElementById(id);

  const colRef = () => collection(db, 'colaboradores');
  const pontosRef = () => collection(db, 'pontos');

  // state
  let chart = null;
  let scanner = null;
  let lastScanAt = 0;

  // ---- salvar/atualizar colaborador ----
  async function salvarColaborador(){
    const matricula = $('colMatricula').value.trim();
    const nome = $('colNome').value.trim();
    const file = $('colFoto').files[0];

    if(!matricula || !nome){
      alert('Preencha matrícula e nome.');
      return;
    }

    let fotoUrl = null;
    // se enviar foto, faz upload
    if(file){
      try{
        const ref = storageRef(storage, `colaborador_fotos/${matricula}_${Date.now()}`);
        const res = await uploadBytes(ref, file);
        fotoUrl = await getDownloadURL(res.ref);
      }catch(e){
        console.error('Erro upload foto', e);
        alert('Erro no upload da foto.');
        return;
      }
    } else {
      // manter foto já salva se existir
      const existing = await getDoc(doc(db, 'colaboradores', matricula));
      if(existing.exists() && existing.data().foto) fotoUrl = existing.data().foto;
    }

    await setDoc(doc(db, 'colaboradores', matricula), { matricula, nome, foto: fotoUrl || null });
    alert('Colaborador salvo/atualizado.');
    $('colMatricula').value=''; $('colNome').value=''; $('colFoto').value='';
    $('qrPreview').innerHTML='';
    await carregarColaboradores();
  }

  // ---- carregar colaboradores ----
  async function carregarColaboradores(){
    const tbody = document.querySelector('#colTable tbody');
    tbody.innerHTML = '';
    try{
      const snap = await getDocs(query(colRef(), orderBy('matricula')));
      snap.forEach(docSnap => {
        const data = docSnap.data();
        const tr = document.createElement('tr');
        tr.innerHTML = `<td>${data.matricula}</td><td>${data.nome}</td><td>${ data.foto ? `<img src="${data.foto}" class="avatar"/>` : '—' }</td><td></td>`;
        const tdActions = tr.children[3];

        const btnEditar = document.createElement('button'); btnEditar.innerText='Editar';
        btnEditar.onclick = ()=> {
          $('colMatricula').value = data.matricula;
          $('colNome').value = data.nome;
        };

        const btnQR = document.createElement('button'); btnQR.innerText='QR';
        btnQR.onclick = ()=> gerarQRPreview(data.matricula);

        const btnExcluir = document.createElement('button'); btnExcluir.innerText='Excluir';
        btnExcluir.onclick = async ()=> {
          if(!confirm('Excluir colaborador?')) return;
          await deleteDoc(doc(db,'colaboradores',data.matricula));
          await carregarColaboradores();
        };

        tdActions.appendChild(btnEditar);
        tdActions.appendChild(btnQR);
        tdActions.appendChild(btnExcluir);
        tbody.appendChild(tr);
      });
    }catch(e){
      console.error('Erro carregar colaboradores', e);
      alert('Erro ao carregar colaboradores. Veja console.');
    }
  }

  // ---- gerar QR preview ----
  function gerarQRPreview(matricula = null){
    const m = matricula || $('colMatricula').value.trim();
    if(!m){ alert('Informe matrícula'); return; }
    QRCode.toDataURL(m, { width:240 }).then(url => {
      $('qrPreview').innerHTML = `<div style="text-align:center"><img src="${url}" /><div class="small">${m}</div></div>`;
    }).catch(err => {
      console.error(err);
      alert('Erro ao gerar QR');
    });
  }

  // ---- iniciar scanner ----
  function iniciarScanner(){
    if(scanner) return;
    scanner = new Html5Qrcode("reader");
    Html5Qrcode.getCameras().then(cameras => {
      if(cameras && cameras.length){
        scanner.start(cameras[0].id, { fps: 10, qrbox: 250 }, onScanSuccess).catch(err => {
          $('scanStatus').innerText = 'Erro ao iniciar câmera: ' + err.message;
        });
      } else {
        $('scanStatus').innerText = 'Nenhuma câmera disponível';
      }
    }).catch(err => {
      $('scanStatus').innerText = 'Erro câmera: ' + err.message;
    });
  }

  // ---- quando QR lido ----
  async function onScanSuccess(decodedText, decodedResult){
    const now = Date.now();
    if(now - lastScanAt < 1500) return; // debounce 1.5s
    lastScanAt = now;

    const matricula = decodedText.trim();
    $('scanStatus').innerText = `QR lido: ${matricula} — processando...`;

    // buscar colaborador (nome/foto)
    let nome = matricula;
    try{
      const docRef = doc(db, 'colaboradores', matricula);
      const snap = await getDoc(docRef);
      if(snap.exists()) nome = snap.data().nome || matricula;
    }catch(e){
      console.error('Erro buscar colaborador', e);
    }

    // buscar últimos pontos do ID (decidir entrada/saida)
    let ultimoTipo = null;
    try{
      const snapP = await getDocs(query(pontosRef(), orderBy('hora', 'desc')));
      const arr = snapP.docs.map(d => d.data()).filter(p => p.id === matricula);
      if(arr.length) ultimoTipo = arr[0].tipo;
    }catch(e){
      console.error('Erro buscar pontos', e);
    }

    // determinar tipo: se ultima foi 'entrada' -> 'saida', senão 'entrada'
    const tipo = (ultimoTipo === 'entrada') ? 'saida' : 'entrada';

    // tentativa obter localização
    let lat = null, lon = null;
    try{
      const pos = await new Promise((res, rej) => navigator.geolocation.getCurrentPosition(res, rej, { timeout: 5000 }));
      lat = pos.coords.latitude; lon = pos.coords.longitude;
    }catch(e){
      // sem permissão => gravar sem coords
    }

    const agora = new Date();
    const registro = {
      id: matricula,
      nome,
      tipo,
      data: agora.toLocaleDateString('pt-BR'),
      hora: agora.toISOString(),
      lat,
      lon
    };

    try{
      await addDoc(pontosRef(), registro);
      $('scanStatus').innerText = `Ponto ${tipo} registrado: ${nome} — ${agora.toLocaleTimeString()}`;
      await carregarHistorico();
    }catch(e){
      console.error('Erro gravar ponto', e);
      $('scanStatus').innerText = 'Erro ao registrar ponto.';
    }
  }

  // ---- carregar histórico e calcular horas ----
  async function carregarHistorico(){
    const tbody = document.querySelector('#pontosTable tbody');
    tbody.innerHTML = '';
    try{
      const snap = await getDocs(query(pontosRef(), orderBy('hora', 'asc')));
      const pontos = snap.docs.map(d => d.data());

      // agrupar por id+data
      const grupos = {};
      pontos.forEach(p => {
        const key = `${p.id}|${p.data}`;
        grupos[key] = grupos[key] || [];
        grupos[key].push(p);
      });

      const horasPorColab = {};

      for(const key of Object.keys(grupos)){
        const regs = grupos[key].sort((a,b)=> new Date(a.hora) - new Date(b.hora));
        let total = 0;
        for(let i=0;i<regs.length;i++){
          if(regs[i].tipo === 'entrada' && regs[i+1] && regs[i+1].tipo === 'saida'){
            total += (new Date(regs[i+1].hora) - new Date(regs[i].hora)) / 3600000;
          }
        }

        regs.forEach(r => {
          const tr = document.createElement('tr');
          tr.innerHTML = `<td>${r.nome}</td><td>${r.tipo}</td><td>${r.data}</td><td>${new Date(r.hora).toLocaleTimeString()}</td><td>${r.lat? r.lat.toFixed(4)+','+r.lon.toFixed(4) : '—'}</td><td>${total.toFixed(2)}</td>`;
          tbody.appendChild(tr);
        });

        const nome = regs[0] ? regs[0].nome : '—';
        horasPorColab[nome] = (horasPorColab[nome] || 0) + total;
      }

      atualizarChart(horasPorColab);
    }catch(e){
      console.error('Erro carregar histórico', e);
      alert('Erro ao carregar histórico. Veja console.');
    }
  }

  // ---- exportar excel ----
  async function exportarExcel(){
    try{
      const snap = await getDocs(query(pontosRef(), orderBy('hora','asc')));
      const pontos = snap.docs.map(d=>d.data());
      if(!pontos.length){ alert('Sem dados'); return; }

      const sheet = pontos.map(p => ({
        Colaborador: p.nome,
        Matricula: p.id,
        Tipo: p.tipo,
        Data: p.data,
        Hora: new Date(p.hora).toLocaleTimeString(),
        Latitude: p.lat || '',
        Longitude: p.lon || ''
      }));
      const ws = XLSX.utils.json_to_sheet(sheet);
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, 'Pontos');
      XLSX.writeFile(wb, `pontos_${new Date().toISOString().slice(0,10)}.xlsx`);
    }catch(e){
      console.error('Erro exportar', e);
      alert('Erro ao exportar. Veja console.');
    }
  }

  // ---- chart ----
  function atualizarChart(obj){
    const ctx = document.getElementById('chartHoras').getContext('2d');
    if(chart) chart.destroy();
    const labels = Object.keys(obj);
    const data = Object.values(obj);
    chart = new Chart(ctx, {
      type: 'bar',
      data: { labels, datasets: [{ label: 'Horas', data, backgroundColor: 'rgba(34,139,230,0.7)' }] },
      options: { responsive:true, scales:{ y:{ beginAtZero:true } } }
    });
  }

  // ---- binding UI ----
  $('btnSalvarCol').addEventListener('click', salvarColaborador);
  $('btnGerarQR').addEventListener('click', ()=> gerarQRPreview(null));
  $('btnLimpar').addEventListener('click', ()=> { $('colMatricula').value=''; $('colNome').value=''; $('colFoto').value=''; $('qrPreview').innerHTML=''; });
  $('btnExportExcel').addEventListener('click', exportarExcel);
  $('btnRefresh').addEventListener('click', carregarHistorico);

  // ---- init ----
  (async function init(){
    await carregarColaboradores();
    await carregarHistorico();
    iniciarScanner();
  })();

  // expose for console/debug
  window._app = { salvarColaborador, carregarColaboradores, carregarHistorico, gerarQRPreview, exportarExcel };

  // no duplicate identifiers - pontosRef is a function above
  function pontosRef(){ return collection(db, 'pontos'); }

</script>
</body>
</html>
