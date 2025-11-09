<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ponto Eletrônico QR Code — Tudo</title>

<!-- Styles -->
<style>
  :root{--card-bg:#fff;--muted:#666}
  body{font-family:Inter,Arial,Helvetica,sans-serif;background:#f3f4f6;margin:0;padding:18px}
  .wrap{max-width:1200px;margin:0 auto}
  header{display:flex;align-items:center;justify-content:space-between;gap:12px;margin-bottom:14px}
  h1{margin:0;font-size:20px}
  .layout{display:grid;grid-template-columns:360px 1fr;gap:14px}
  .card{background:var(--card-bg);padding:14px;border-radius:8px;box-shadow:0 6px 18px rgba(0,0,0,0.06)}
  input,select,textarea,button{font-size:14px}
  input,select,textarea{width:100%;padding:8px;margin:6px 0;border:1px solid #ddd;border-radius:6px;box-sizing:border-box}
  button{padding:10px 12px;border-radius:8px;border:0;background:#0f172a;color:#fff;cursor:pointer}
  .small{font-size:13px;color:var(--muted)}
  .row{display:flex;gap:8px}
  .col{display:flex;flex-direction:column;gap:8px}
  table{width:100%;border-collapse:collapse;margin-top:8px;font-size:13px}
  th,td{padding:8px;border:1px solid #eee;text-align:left}
  .qr-canvas{width:120px;height:120px;display:block;margin:6px 0}
  .actions{display:flex;gap:8px;flex-wrap:wrap}
  .hidden{display:none}
  .avatar{width:64px;height:64px;border-radius:8px;object-fit:cover;border:1px solid #eee}
  .badge{display:inline-block;padding:4px 8px;border-radius:999px;background:#eef2ff;color:#1e3a8a;font-weight:600;font-size:12px}
  @media (max-width:980px){ .layout{grid-template-columns:1fr} }
</style>

<!-- Firebase (modular) and libs -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-app.js";
  import { getFirestore, collection, doc, setDoc, getDoc, addDoc, getDocs, deleteDoc, query, orderBy, where } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";
  import { getStorage, ref as sref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-storage.js";

  // ======= CONFIGURE AQUI seu Firebase (já preenchei com a sua config) =======
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

  // expose some firebase utilities globally
  window._FB = { db, storage, collection, doc, setDoc, getDoc, addDoc, getDocs, deleteDoc, query, orderBy, where, sref, uploadBytes, getDownloadURL };
</script>

<!-- QR scanner + generator + xlsx + chart -->
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
<div class="wrap">
  <header>
    <h1>Ponto Eletrônico com QR — Painel Completo</h1>
    <div class="small">Local e nuvem (Firestore + Storage)</div>
  </header>

  <div class="layout">
    <!-- left column: admin tools -->
    <div class="card">
      <h3>Admin — Colaboradores</h3>
      <label class="small">Matrícula</label>
      <input id="colMatricula" placeholder="ex: 12345">
      <label class="small">Nome</label>
      <input id="colNome" placeholder="Nome completo">
      <label class="small">Foto (opcional)</label>
      <input id="colFoto" type="file" accept="image/*">
      <div class="actions">
        <button id="btnSalvarCol">Salvar / Atualizar</button>
        <button id="btnGerarQR">Gerar QR (preview)</button>
        <button id="btnLimpar">Limpar</button>
      </div>
      <div id="qrPreview" class="small" style="margin-top:10px"></div>
      <h4 style="margin-top:14px">Lista de colaboradores</h4>
      <div style="max-height:320px;overflow:auto">
        <table id="colTable">
          <thead><tr><th>Mat</th><th>Nome</th><th>Foto</th><th>Ações</th></tr></thead>
          <tbody></tbody>
        </table>
      </div>
    </div>
    <!-- right column: scanner, histórico, dashboard -->
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
          <h4 class="small">Histórico / Relatórios</h4>
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

<!-- Main script -->
<script type="module">
import { collection, doc, setDoc, getDoc, addDoc, getDocs, deleteDoc, query, orderBy, where } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";
import { ref as sref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-storage.js";

const { db, storage } = window._FB;

// helpers
const $ = (id) => document.getElementById(id);
const colColl = () => collection(db, 'colaboradores');
const pontosColl = () => collection(db, 'pontos');

// state
let chart = null;
let scanner = null;

// ---------- Colaboradores ----------
async function salvarColaborador(){
  const matricula = $('colMatricula').value.trim();
  const nome = $('colNome').value.trim();
  const file = $('colFoto').files[0];

  if(!matricula || !nome){ return alert('Preencha matrícula e nome'); }

  let fotoUrl = null;
  if(file){
    const ref = sref(storage, `colaborador_fotos/${matricula}_${Date.now()}`);
    const b = await uploadBytes(ref, file);
    fotoUrl = await getDownloadURL(b.ref);
  } else {
    // se já existe doc com foto, preserve (fetch below)
    const existing = await getDoc(doc(db, 'colaboradores', matricula));
    if(existing.exists() && existing.data().foto) fotoUrl = existing.data().foto;
  }

  await setDoc(doc(db, 'colaboradores', matricula), { matricula, nome, foto: fotoUrl || null });
  alert('Colaborador salvo.');
  $('colMatricula').value=''; $('colNome').value=''; $('colFoto').value='';
  await carregarColaboradores();
}

async function carregarColaboradores(){
  const tbody = document.querySelector('#colTable tbody');
  tbody.innerHTML = '';
  const snap = await getDocs(query(colColl(), orderBy('matricula')));
  snap.forEach(d => {
    const data = d.data();
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td>${data.matricula}</td>
      <td>${data.nome}</td>
      <td>${ data.foto ? `<img src="${data.foto}" class="avatar" />` : '—' }</td>
      <td></td>
    `;
    const tdActions = tr.children[3];
    const btnFill = document.createElement('button'); btnFill.innerText='Editar'; btnFill.onclick = ()=> {
      $('colMatricula').value = data.matricula;
      $('colNome').value = data.nome;
    };
    const btnQR = document.createElement('button'); btnQR.innerText='QR'; btnQR.onclick = ()=> gerarQRPreview(data.matricula);
    const btnDel = document.createElement('button'); btnDel.innerText='Excluir'; btnDel.onclick = async ()=>{
      if(!confirm('Excluir colaborador?')) return;
      await deleteDoc(doc(db,'colaboradores',data.matricula));
      await carregarColaboradores();
    };
    tdActions.appendChild(btnFill); tdActions.appendChild(btnQR); tdActions.appendChild(btnDel);
    tbody.appendChild(tr);
  });
}

// ---------- QR generation ----------
function gerarQRPreview(matricula=null){
  const m = matricula || $('colMatricula').value.trim();
  if(!m){ alert('Informe matrícula'); return; }
  const target = document.createElement('div');
  QRCode.toDataURL(m, { width:240 })
    .then(url => {
      $('qrPreview').innerHTML = `<div style="text-align:center"><img src="${url}" style="width:220px"/><div class="small">${m}</div></div>`;
    }).catch(err => { console.error(err); alert('Erro ao gerar QR'); });
}

// ---------- Scanner & registro de ponto ----------
function iniciarScanner(){
  if(scanner) return;
  scanner = new Html5Qrcode("reader");
  Html5Qrcode.getCameras().then(cameras => {
    if(cameras && cameras.length){
      scanner.start(cameras[0].id, { fps: 10, qrbox: 250 }, onScanSuccess).catch(err=>{
        $('scanStatus').innerText = 'Erro ao iniciar câmera: ' + err.message;
      });
    } else {
      $('scanStatus').innerText = 'Nenhuma câmera encontrada';
    }
  }).catch(err => {
    $('scanStatus').innerText = 'Erro (câmera): ' + err.message;
  });
}

let lastScanAt = 0;
async function onScanSuccess(decodedText, decodedResult){
  const now = Date.now();
  if(now - lastScanAt < 2000) return; // debounce 2s
  lastScanAt = now;

  const matricula = decodedText.trim();
  $('scanStatus').innerText = `QR lido: ${matricula} — buscando colaborador...`;

  // buscar colaborador
  const docRef = doc(db, 'colaboradores', matricula);
  const colSnap = await getDoc(docRef);
  let nome = matricula;
  let foto = null;
  if(colSnap.exists()){ nome = colSnap.data().nome; foto = colSnap.data().foto || null; }

  // calcular tipo (entrada/saida) consultando últimos pontos do dia
  const pontosSnap = await getDocs(query(pontosColl(), orderBy('hora','desc')));
  const allPontos = pontosSnap.docs.map(d=>d.data()).filter(p=>p.id === matricula);

  // define tipo: se última ação foi 'entrada' sem saída após -> saída; else entrada
  let tipo = 'entrada';
  if(allPontos.length){
    // find last tipo chronologically
    const last = allPontos[0];
    if(last.tipo === 'entrada') tipo = 'saida';
    else tipo = 'entrada';
  }

  // pegar localização
  let lat=null, lon=null;
  try{
    const pos = await new Promise((res,rej)=>{
      navigator.geolocation.getCurrentPosition(res, rej, { timeout:5000 });
    });
    lat = pos.coords.latitude; lon = pos.coords.longitude;
  }catch(e){
    // sem permissão ou erro: fica null
  }

  const agora = new Date();
  const registro = {
    id: matricula,
    nome,
    tipo,
    data: agora.toLocaleDateString('pt-BR'),
    hora: agora.toISOString(),
    lat: lat,
    lon: lon
  };

  await addDoc(pontosColl(), registro);
  $('scanStatus').innerText = `Ponto ${registro.tipo} registrado: ${nome} às ${agora.toLocaleTimeString()}`;
  await carregarHistorico();
}

// ---------- Histórico / cálculo horas ----------
async function carregarHistorico(){
  const tbody = document.querySelector('#pontosTable tbody');
  tbody.innerHTML = '';
  const snap = await getDocs(query(pontosColl(), orderBy('hora','asc')));
  const docs = snap.docs.map(d => d.data());

  // agrupar por id+data
  const grupos = {};
  docs.forEach(r => {
    const key = r.id + '|' + r.data;
    grupos[key] = grupos[key] || [];
    grupos[key].push(r);
  });

  const horasPor = {};
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
      tr.innerHTML = `<td>${r.nome}</td><td>${r.tipo}</td><td>${r.data}</td><td>${new Date(r.hora).toLocaleTimeString()}</td><td>${r.lat? r.lat.toFixed(4)+','+r.lon.toFixed(4): '—'}</td><td>${total.toFixed(2)}</td>`;
      tbody.appendChild(tr);
    });
    const nome = regs[0] ? regs[0].nome : '—';
    horasPor[nome] = (horasPor[nome] || 0) + total;
  }

  atualizarChart(horasPor);
}

// ---------- Export Excel ----------
async function exportarExcel(){
  const snap = await getDocs(query(pontosColl(), orderBy('hora','asc')));
  const dados = snap.docs.map(d => d.data());
  if(!dados.length){ alert('Sem dados'); return; }
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
}

// ---------- Chart ----------
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

// ---------- util ----------
function pontosColl(){ return collection(db,'pontos'); }

// ---------- bind events ----------
document.getElementById('btnSalvarCol').addEventListener('click', salvarColaborador);
document.getElementById('btnGerarQR').addEventListener('click', ()=> gerarQRPreview(null));
document.getElementById('btnLimpar').addEventListener('click', ()=> { $('colMatricula').value=''; $('colNome').value=''; $('colFoto').value=''; $('qrPreview').innerHTML=''; });
document.getElementById('btnExportExcel').addEventListener('click', exportarExcel);
document.getElementById('btnRefresh').addEventListener('click', carregarHistorico);

// ---------- init ----------
(async function init(){
  await carregarColaboradores();
  await carregarHistorico();
  iniciarScanner();
})();

// expose for console if needed
window._app = { carregarColaboradores, carregarHistorico, salvarColaborador, gerarQRPreview, exportarExcel };

</script>
</body>
</html>
