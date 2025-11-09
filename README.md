<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ponto Eletrônico CLX (Firebase + QR)</title>

<!-- Bootstrap (visual igual) -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">

<style>
  body{background:#f4f6f9;font-family:Arial,Helvetica,sans-serif}
  .navbar{background:#001f3f}
  .navbar-brand{color:#fff;font-weight:700}
  .card{border-radius:12px}
  .btn-clx{background:#001f3f;color:#fff;border:none}
  .qr-box img{width:160px}
  .avatar{width:48px;height:48px;border-radius:6px;object-fit:cover}
  .small-muted{font-size:13px;color:#6b7280}
  @media (max-width:768px){ .layout{grid-template-columns:1fr} }
</style>
</head>
<body>
<nav class="navbar mb-4 px-3">
  <span class="navbar-brand">Ponto Eletrônico — CLX</span>
</nav>

<div class="container">
  <div class="row g-4">
    <!-- Left: cadastro -->
    <div class="col-md-4">
      <div class="card p-3 shadow-sm">
        <h5>Colaboradores</h5>
        <label class="small-muted">Matrícula</label>
        <input id="matriculaInput" class="form-control" placeholder="12345">
        <label class="small-muted mt-2">Nome</label>
        <input id="nomeInput" class="form-control" placeholder="Nome completo">
        <label class="small-muted mt-2">Foto (opcional)</label>
        <input id="fotoInput" type="file" class="form-control" accept="image/*">
        <div class="d-grid gap-2 mt-3">
          <button id="btnSalvarCol" class="btn btn-clx">Salvar / Atualizar</button>
          <button id="btnGerarQR" class="btn btn-outline-secondary">Gerar QR (preview)</button>
          <button id="btnLimpar" class="btn btn-light">Limpar</button>
        </div>
        <div id="qrPreview" class="qr-box text-center mt-3"></div>
        <hr>
        <h6 class="small-muted">Lista de colaboradores</h6>
        <div style="max-height:280px; overflow:auto;">
          <table class="table table-sm">
            <thead><tr><th>Mat</th><th>Nome</th><th>Foto</th><th></th></tr></thead>
            <tbody id="colTable"></tbody>
          </table>
        </div>
      </div>
    </div>
    <!-- Right: painel + histórico -->
    <div class="col-md-8">
      <div class="card p-3 shadow-sm">
        <div class="d-flex justify-content-between align-items-center">
          <h5>Painel</h5>
          <div>
            <button id="btnExport" class="btn btn-clx me-2">Exportar Excel</button>
            <button id="btnReload" class="btn btn-light">Atualizar</button>
          </div>
        </div>
        <div class="row mt-3">
          <div class="col-md-5">
            <h6 class="small-muted">Bater ponto (QR Scanner)</h6>
            <div id="reader" style="width:100%;"></div>
            <div id="scanStatus" class="small-muted mt-2"></div>
          </div>
          <div class="col-md-7">
            <h6 class="small-muted">Histórico</h6>
            <div style="max-height:300px; overflow:auto;">
              <table class="table table-sm">
                <thead><tr><th>Colab</th><th>Tipo</th><th>Data</th><th>Hora</th><th>Lat,Lon</th><th>Horas</th></tr></thead>
                <tbody id="histTable"></tbody>
              </table>
            </div>
          </div>
        </div>
        <h6 class="small-muted mt-3">Gráfico horas por colaborador</h6>
        <canvas id="chartHoras" style="max-height:200px"></canvas>
      </div>
    </div>

  </div>
</div>

<!-- Libraries -->
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<!-- Firebase setup and main -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-app.js";
  import { getFirestore, collection, doc, setDoc, getDoc, addDoc, getDocs, deleteDoc, query, orderBy } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";
  import { getStorage, ref as sref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-storage.js";

  // ---------- CONFIGURE AQUI (já preenchida com sua config) ----------
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

  // helpers
  const $ = id => document.getElementById(id);

  // references
  const colaboradoresColl = () => collection(db, 'colaboradores');
  const pontosColl = () => collection(db, 'pontos');

  // state
  let scanner = null;
  let chart = null;
  let lastScanAt = 0;

  // ---------- Colaboradores ----------
  async function salvarColaborador(){
    const matricula = $('matriculaInput').value.trim();
    const nome = $('nomeInput').value.trim();
    const file = $('fotoInput').files[0];

    if(!matricula || !nome){ alert('Preencha matrícula e nome'); return; }

    let fotoUrl = null;
    if(file){
      try{
        const ref = sref(storage, `colaborador_fotos/${matricula}_${Date.now()}`);
        const res = await uploadBytes(ref, file);
        fotoUrl = await getDownloadURL(res.ref);
      } catch(e){
        console.error(e);
        alert('Erro no upload da foto');
        return;
      }
    } else {
      // keep existing photo if any
      try{
        const snap = await getDoc(doc(db, 'colaboradores', matricula));
        if(snap.exists() && snap.data().foto) fotoUrl = snap.data().foto;
      }catch(e){}
    }

    try{
      await setDoc(doc(db, 'colaboradores', matricula), { matricula, nome, foto: fotoUrl || null });
      alert('Colaborador salvo');
      $('matriculaInput').value=''; $('nomeInput').value=''; $('fotoInput').value='';
      $('qrPreview').innerHTML = '';
      await carregarColaboradores();
    }catch(e){
      console.error(e);
      alert('Erro ao salvar colaborador');
    }
  }

  async function carregarColaboradores(){
    const tbody = $('colTable');
    tbody.innerHTML = '';
    try{
      const snap = await getDocs(query(colaboradoresColl(), orderBy('matricula')));
      snap.forEach(docSnap => {
        const d = docSnap.data();
        const tr = document.createElement('tr');
        tr.innerHTML = `
          <td>${d.matricula}</td>
          <td>${d.nome}</td>
          <td>${d.foto ? `<img src="${d.foto}" class="avatar">` : '—'}</td>
          <td></td>
        `;
        const td = tr.children[3];

        const btnEdit = document.createElement('button'); btnEdit.className='btn btn-sm btn-light me-1'; btnEdit.innerText='Editar';
        btnEdit.onclick = ()=> { $('matriculaInput').value=d.matricula; $('nomeInput').value=d.nome; };

        const btnQR = document.createElement('button'); btnQR.className='btn btn-sm btn-outline-secondary me-1'; btnQR.innerText='QR';
        btnQR.onclick = ()=> gerarQRPreview(d.matricula);

        const btnDel = document.createElement('button'); btnDel.className='btn btn-sm btn-danger'; btnDel.innerText='Excluir';
        btnDel.onclick = async ()=> { if(!confirm('Excluir?')) return; await deleteDoc(doc(db,'colaboradores',d.matricula)); await carregarColaboradores(); };

        td.appendChild(btnEdit); td.appendChild(btnQR); td.appendChild(btnDel);
        tbody.appendChild(tr);
      });
    }catch(e){
      console.error(e);
    }
  }

  function gerarQRPreview(matricula=null){
    const m = matricula || $('matriculaInput').value.trim();
    if(!m){ alert('Informe matrícula'); return; }
    QRCode.toDataURL(m, { width:220 }).then(url => {
      $('qrPreview').innerHTML = `<div style="text-align:center"><img src="${url}" /><div class="small-muted">${m}</div></div>`;
    }).catch(err => { console.error(err); alert('Erro gerar QR'); });
  }

  // ---------- Scanner / Bater ponto ----------
  function iniciarScanner(){
    if(scanner) return;
    scanner = new Html5Qrcode("reader");
    Html5Qrcode.getCameras().then(cameras => {
      if(cameras && cameras.length){
        scanner.start(cameras[0].id, { fps: 10, qrbox: 250 }, onScanSuccess).catch(err=>{
          $('scanStatus').innerText = 'Erro iniciar câmera: ' + err.message;
        });
      } else {
        $('scanStatus').innerText = 'Nenhuma câmera encontrada';
      }
    }).catch(err=>{
      $('scanStatus').innerText = 'Erro câmera: ' + err.message;
    });
  }

  async function onScanSuccess(decodedText){
    const now = Date.now();
    if(now - lastScanAt < 1500) return; // debounce
    lastScanAt = now;

    const matricula = decodedText.trim();
    $('scanStatus').innerText = `QR lido: ${matricula} — processando...`;

    // buscar colaborador
    let nome = matricula;
    try{
      const snap = await getDoc(doc(db, 'colaboradores', matricula));
      if(snap.exists()) nome = snap.data().nome || matricula;
    }catch(e){ console.error(e); }

    // busca última ação do usuário (descendente)
    let ultimoTipo = null;
    try{
      const snapP = await getDocs(query(pontosColl(), orderBy('hora','desc')));
      const arr = snapP.docs.map(d => d.data()).filter(p => p.id === matricula);
      if(arr.length) ultimoTipo = arr[0].tipo;
    }catch(e){ console.error(e); }

    const tipo = (ultimoTipo === 'entrada') ? 'saida' : 'entrada';

    // geolocalização (opcional)
    let lat=null, lon=null;
    try{
      const pos = await new Promise((res, rej)=> navigator.geolocation.getCurrentPosition(res, rej, { timeout:5000 }));
      lat = pos.coords.latitude; lon = pos.coords.longitude;
    }catch(e){ /* ignore */ }

    const agora = new Date();
    const registro = { id: matricula, nome, tipo, data: agora.toLocaleDateString('pt-BR'), hora: agora.toISOString(), lat, lon };

    try{
      await addDoc(pontosColl(), registro);
      $('scanStatus').innerText = `Ponto ${tipo} registrado: ${nome} às ${agora.toLocaleTimeString()}`;
      await carregarHistorico();
    }catch(e){
      console.error(e);
      $('scanStatus').innerText = 'Erro ao registrar ponto';
    }
  }

  // ---------- Histórico / cálculo ----------
  async function carregarHistorico(){
    const tbody = $('histTable');
    tbody.innerHTML = '';
    try{
      const snap = await getDocs(query(pontosColl(), orderBy('hora','asc')));
      const pontos = snap.docs.map(d => d.data());

      const grupos = {};
      pontos.forEach(p => {
        const key = `${p.id}|${p.data}`;
        grupos[key] = grupos[key] || [];
        grupos[key].push(p);
      });

      const horasPor = {};

      for(const key of Object.keys(grupos)){
        const regs = grupos[key].sort((a,b) => new Date(a.hora) - new Date(b.hora));
        let total = 0;
        for(let i=0;i<regs.length;i++){
          if(regs[i].tipo==='entrada' && regs[i+1] && regs[i+1].tipo==='saida'){
            total += (new Date(regs[i+1].hora) - new Date(regs[i].hora)) / 3600000;
          }
        }
        regs.forEach(r => {
          const tr = document.createElement('tr');
          tr.innerHTML = `<td>${r.nome}</td><td>${r.tipo}</td><td>${r.data}</td><td>${new Date(r.hora).toLocaleTimeString()}</td><td>${r.lat? r.lat.toFixed(4)+','+r.lon.toFixed(4): '—'}</td><td>${total.toFixed(2)}</td>`;
          tbody.appendChild(tr);
        });
        const nome = regs[0] ? regs[0].nome : '—';
        horasPor[nome] = (horasPor[nome] || 0) + total;
      }

      atualizarChart(horasPor);

    }catch(e){
      console.error(e);
      alert('Erro carregar histórico (veja console)');
    }
  }

  // ---------- Export XLSX ----------
  async function exportarExcel(){
    try{
      const snap = await getDocs(query(pontosColl(), orderBy('hora','asc')));
      const pontos = snap.docs.map(d => d.data());
      if(!pontos.length){ alert('Sem dados'); return; }
      const sheet = pontos.map(p => ({
        Colaborador: p.nome,
        Matricula: p.id,
        Tipo: p.tipo,
        Data: p.data,
        Hora: new Date(p.hora).toLocaleTimeString(),
        Lat: p.lat || '',
        Lon: p.lon || ''
      }));
      const ws = XLSX.utils.json_to_sheet(sheet);
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, 'Pontos');
      XLSX.writeFile(wb, `pontos_${new Date().toISOString().slice(0,10)}.xlsx`);
    }catch(e){ console.error(e); alert('Erro exportar (veja console)'); }
  }

  // ---------- Chart ----------
  function atualizarChart(obj){
    const ctx = $('chartHoras').getContext('2d');
    if(chart) chart.destroy();
    const labels = Object.keys(obj);
    const data = Object.values(obj);
    chart = new Chart(ctx, {
      type:'bar',
      data:{ labels, datasets:[{ label:'Horas', data, backgroundColor:'rgba(2,86,204,0.7)' }] },
      options:{ responsive:true, scales:{ y:{ beginAtZero:true } } }
    });
  }

  // ---------- Init & bindings ----------
  $('btnSalvarCol').addEventListener('click', salvarColaborador);
  $('btnGerarQR').addEventListener('click', () => gerarQRPreview(null));
  $('btnLimpar').addEventListener('click', () => { $('matriculaInput').value=''; $('nomeInput').value=''; $('fotoInput').value=''; $('qrPreview').innerHTML=''; });
  $('btnExport').addEventListener('click', exportarExcel);
  $('btnReload').addEventListener('click', carregarHistorico);

  // expose small helper to console
  window._clx = { carregarColaboradores, carregarHistorico, salvarColaborador, exportarExcel };

  // startup
  (async function start(){
    await carregarColaboradores();
    await carregarHistorico();
    iniciarScanner();
  })();

  // end of module

</script>
</body>
</html>
