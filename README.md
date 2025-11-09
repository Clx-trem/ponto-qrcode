<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Gerenciador de Ponto — CLX</title>

<!-- Visual (tema azul) -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet"/>
<style>
  :root{--primary:#1e88e5;--muted:#6b7280}
  body{background:#e9f4ff;font-family:Arial,Helvetica,sans-serif;margin:0;padding:18px}
  .navbar{background:var(--primary);color:#fff;padding:12px;border-radius:8px}
  .card{border-radius:10px;box-shadow:0 6px 20px rgba(2,6,23,0.06);background:#fff}
  .small-muted{color:var(--muted);font-size:13px}
  .avatar{width:44px;height:44px;border-radius:8px;object-fit:cover}
  table th{background:var(--primary);color:#fff}
  .qr-preview img{width:160px}
  .btn-clx{background:var(--primary);color:#fff;border:none}
  @media (max-width:900px){ .layout{flex-direction:column} }
</style>
</head>
<body>

<div class="container">
  <div class="navbar mb-4">
    <div class="d-flex justify-content-between align-items-center">
      <div><strong>Gerenciador de Ponto — CLX</strong></div>
      <div class="small-muted">Scanner • Registro • Relatórios</div>
    </div>
  </div>

  <div class="row g-4">
    <!-- Colaboradores / Gerar QR -->
    <div class="col-md-4">
      <div class="card p-3">
        <h5>Colaboradores</h5>
        <label class="small-muted">Pesquisar matrícula</label>
        <input id="searchMat" class="form-control mb-2" placeholder="Digite matrícula e pressione Enter">
        <div class="d-grid gap-2">
          <button id="btnLoadCols" class="btn btn-clx">Carregar Colaboradores</button>
          <button id="btnRefresh" class="btn btn-outline-secondary">Atualizar Histórico</button>
        </div>
        <hr/>
        <div style="max-height:300px;overflow:auto;margin-top:8px">
          <table class="table table-sm">
            <thead><tr><th>Matr</th><th>Nome</th><th></th></tr></thead>
            <tbody id="colTable"></tbody>
          </table>
        </div>
        <div id="qrPreview" class="qr-preview text-center mt-2"></div>
      </div>
    </div>
    <!-- Painel: scanner + histórico + gráfico -->
    <div class="col-md-8">
      <div class="card p-3">
        <div class="d-flex justify-content-between align-items-center">
          <h5>Painel</h5>
          <div>
            <button id="btnExport" class="btn btn-clx btn-sm me-2">Exportar Excel</button>
          </div>
        </div>
        <div class="row mt-3 layout">
          <div class="col-md-5">
            <h6 class="small-muted">Bater ponto</h6>
            <div id="reader" style="width:100%;"></div>
            <div id="scanStatus" class="small-muted mt-2"></div>
          </div>
          <div class="col-md-7">
            <h6 class="small-muted">Últimos registros</h6>
            <div style="max-height:300px; overflow:auto;">
              <table class="table table-sm">
                <thead><tr><th>Colab</th><th>Tipo</th><th>Data</th><th>Hora</th><th>Lat,Lon</th><th>Horas</th></tr></thead>
                <tbody id="histTable"></tbody>
              </table>
            </div>
          </div>
        </div>
        <h6 class="small-muted mt-3">Gráfico horas por colaborador</h6>
        <canvas id="chartHoras" height="120"></canvas>
      </div>
    </div>
  </div>
</div>

<!-- Libs -->
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<!-- Firebase + App logic -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
  import {
    getFirestore, collection, doc, getDoc, setDoc, addDoc, getDocs, query, where, orderBy
  } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

  // --- FIREBASE CONFIG (USE A SUA) ---
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

  // helpers
  const $ = id => document.getElementById(id);
  const colColl = () => collection(db, 'colaboradores');
  const pontosColl = () => collection(db, 'pontos');

  // state
  let scanner = null;
  let lastScanAt = 0;
  let chart = null;

  // --- carregar lista de colaboradores (limitado) ---
  async function carregarColaboradores(term = '') {
    const tbody = $('colTable');
    tbody.innerHTML = '';
    try {
      const snap = await getDocs(query(colColl(), orderBy('matricula')));
      snap.forEach(s => {
        const d = s.data();
        // se termo informado, filtrar
        if (term && !d.matricula.includes(term) && !d.nome.toLowerCase().includes(term.toLowerCase())) return;
        const tr = document.createElement('tr');
        tr.innerHTML = `<td>${d.matricula}</td><td>${d.nome}</td><td></td>`;
        const td = tr.children[2];

        const btnQR = document.createElement('button'); btnQR.className='btn btn-sm btn-outline-secondary me-1'; btnQR.innerText='QR';
        btnQR.onclick = () => gerarQRPreview(d.matricula);

        const btnFill = document.createElement('button'); btnFill.className='btn btn-sm btn-light'; btnFill.innerText='Preencher';
        btnFill.onclick = () => { $('searchMat').value = d.matricula; };

        td.appendChild(btnQR);
        td.appendChild(btnFill);

        tbody.appendChild(tr);
      });
    } catch (e) {
      console.error('Erro carregar colaboradores', e);
      alert('Erro ao carregar colaboradores (veja console).');
    }
  }

  // --- gerar QR preview ---
  function gerarQRPreview(matricula) {
    if (!matricula) { alert('Informe matrícula'); return; }
    QRCode.toDataURL(matricula, { width: 200 }).then(url => {
      $('qrPreview').innerHTML = `<div style="text-align:center"><img src="${url}" alt="QR"/><div class="small-muted">${matricula}</div></div>`;
    }).catch(err => {
      console.error(err);
      alert('Erro ao gerar QR');
    });
  }

  // --- iniciar scanner ---
  function iniciarScanner() {
    if (scanner) return;
    scanner = new Html5Qrcode("reader");
    Html5Qrcode.getCameras().then(cameras => {
      if (cameras && cameras.length) {
        scanner.start(cameras[0].id, { fps: 10, qrbox: 250 }, onScanSuccess).catch(err => {
          $('scanStatus').innerText = 'Erro iniciar câmera: ' + err.message;
        });
      } else {
        $('scanStatus').innerText = 'Nenhuma câmera encontrada';
      }
    }).catch(err => {
      $('scanStatus').innerText = 'Erro câmera: ' + err.message;
    });
  }

  // --- quando QR lido ---
  async function onScanSuccess(decodedText) {
    const now = Date.now();
    if (now - lastScanAt < 1200) return; // debounce
    lastScanAt = now;

    const matricula = decodedText.trim();
    $('scanStatus').innerText = `QR lido: ${matricula} — verificando...`;

    try {
      // buscar colaborador por campo 'matricula'
      const q = query(colColl(), where('matricula', '==', matricula));
      const snap = await getDocs(q);
      if (snap.empty) {
        $('scanStatus').innerText = '❌ Matrícula não encontrada';
        return;
      }
      const colData = snap.docs[0].data();
      const nome = colData.nome || matricula;

      // buscar último ponto do usuário (desc)
      const pontosSnap = await getDocs(query(pontosColl(), orderBy('hora', 'desc')));
      const arr = pontosSnap.docs.map(d => d.data()).filter(p => p.id === matricula);

      let tipo = 'entrada';
      if (arr.length && arr[0].tipo === 'entrada') tipo = 'saida';

      // tentar pegar localização (opcional)
      let lat = null, lon = null;
      try {
        const pos = await new Promise((res, rej) => navigator.geolocation.getCurrentPosition(res, rej, { timeout: 5000 }));
        lat = pos.coords.latitude; lon = pos.coords.longitude;
      } catch (e) { /* ignorar se sem permissão */ }

      const agora = new Date();
      // salvar ponto
      await addDoc(pontosColl(), {
        id: matricula,
        nome,
        tipo,
        data: agora.toLocaleDateString('pt-BR'),
        hora: agora.toISOString(),
        lat,
        lon
      });

      $('scanStatus').innerText = `✅ Ponto ${tipo} registrado: ${nome} às ${agora.toLocaleTimeString()}`;
      carregarHistorico();
    } catch (e) {
      console.error('Erro registrar ponto', e);
      $('scanStatus').innerText = 'Erro ao registrar ponto (veja console)';
    }
  }

  // --- carregar histórico e calcular horas ---
  async function carregarHistorico() {
    const tbody = $('histTable');
    tbody.innerHTML = '';
    try {
      const snap = await getDocs(query(pontosColl(), orderBy('hora', 'asc')));
      const pontos = snap.docs.map(d => d.data());

      // agrupar por id + data
      const grupos = {};
      pontos.forEach(p => {
        const key = `${p.id}|${p.data}`;
        grupos[key] = grupos[key] || [];
        grupos[key].push(p);
      });

      const horasPor = {};

      for (const key of Object.keys(grupos)) {
        const regs = grupos[key].sort((a, b) => new Date(a.hora) - new Date(b.hora));
        let total = 0;
        for (let i = 0; i < regs.length; i++) {
          if (regs[i].tipo === 'entrada' && regs[i + 1] && regs[i + 1].tipo === 'saida') {
            total += (new Date(regs[i + 1].hora) - new Date(regs[i].hora)) / 3600000;
          }
        }

        regs.forEach(r => {
          const tr = document.createElement('tr');
          tr.innerHTML = `<td>${r.nome}</td><td>${r.tipo}</td><td>${r.data}</td><td>${new Date(r.hora).toLocaleTimeString()}</td><td>${r.lat? r.lat.toFixed(4)+','+r.lon.toFixed(4) : '—'}</td><td>${total.toFixed(2)}</td>`;
          tbody.appendChild(tr);
        });

        const nome = regs[0] ? regs[0].nome : '—';
        horasPor[nome] = (horasPor[nome] || 0) + total;
      }

      atualizarChart(horasPor);
    } catch (e) {
      console.error('Erro carregar histórico', e);
      alert('Erro ao carregar histórico. Veja console.');
    }
  }

  // --- exportar XLSX ---
  async function exportarExcel(){
    try{
      const snap = await getDocs(query(pontosColl(), orderBy('hora','asc')));
      const dados = snap.docs.map(d => d.data());
      if(!dados.length){ alert('Sem dados para exportar'); return; }
      const sheet = dados.map(r => ({
        Colaborador: r.nome,
        Matricula: r.id,
        Tipo: r.tipo,
        Data: r.data,
        Hora: new Date(r.hora).toLocaleTimeString(),
        Latitude: r.lat || '',
        Longitude: r.lon || ''
      }));
      const ws = XLSX.utils.json_to_sheet(sheet);
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, 'Pontos');
      XLSX.writeFile(wb, `pontos_${new Date().toISOString().slice(0,10)}.xlsx`);
    }catch(e){
      console.error('Erro exportar', e);
      alert('Erro ao exportar (veja console).');
    }
  }

  // --- chart ---
  function atualizarChart(obj) {
    const ctx = document.getElementById('chartHoras').getContext('2d');
    if(chart) chart.destroy();
    const labels = Object.keys(obj);
    const data = Object.values(obj);
    chart = new Chart(ctx, {
      type: 'bar',
      data: { labels, datasets: [{ label: 'Horas', data, backgroundColor: 'rgba(30,136,229,0.8)' }] },
      options: { responsive:true, scales: { y: { beginAtZero:true } } }
    });
  }

  // --- bindings ---
  $('btnLoadCols').addEventListener('click', ()=> carregarColaboradores());
  $('btnRefresh').addEventListener('click', ()=> carregarHistorico());
  $('btnExport').addEventListener('click', ()=> exportarExcel());
  $('searchMat').addEventListener('keydown', (e)=> { if(e.key==='Enter') { e.preventDefault(); carregarColaboradores($('searchMat').value.trim()); } });

  // --- init ---
  (async function init(){
    await carregarColaboradores();
    await carregarHistorico();
    iniciarScanner();
  })();

  // expose for debugging
  window._ger = { carregarColaboradores, carregarHistorico, exportarExcel, gerarQRPreview, iniciarScanner };

</script>
</body>
</html>
