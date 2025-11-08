<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ponto Eletrônico com QR Code</title>
<style>
:root{--blue:#0b4f78;--green:#2e9b4f;--yellow:#ffb739;--red:#ef5350;--muted:#6b7280;--card:#ffffff;--bg:#f4f7fb;}
body{font-family:Inter, system-ui, -apple-system, Arial, sans-serif;background:var(--bg);margin:0;color:#111}
header{background:linear-gradient(90deg,var(--blue),#0f6b96);color:#fff;padding:12px 18px;display:flex;align-items:center;justify-content:space-between;gap:12px;flex-wrap:wrap}
.logo{font-weight:700;font-size:18px}
#clock{font-weight:700}
.controls{display:flex;gap:8px;align-items:center}
button{padding:8px 12px;border:none;border-radius:8px;cursor:pointer;font-weight:600}
.add{background:var(--green);color:#fff}
.secondary{background:#e5e7eb;color:#111}
.download{background:var(--yellow);color:#111}
.danger{background:var(--red);color:#fff}
main{padding:20px;max-width:1100px;margin:20px auto}
.search{width:100%;padding:10px;border-radius:8px;border:1px solid #d1d5db;margin-bottom:14px}
table{width:100%;border-collapse:collapse;background:var(--card);border-radius:10px;overflow:hidden;box-shadow:0 6px 24px rgba(15,23,42,0.06);margin-bottom:18px}
th,td{padding:10px;border-bottom:1px solid #eef2f6;text-align:left;font-size:14px}
th{background:#fbfdfe;font-weight:700}
tr:hover td{background:#fcfdff}
.small{font-size:13px;color:var(--muted);margin-left:6px}
.muted{color:var(--muted);font-size:13px}
.modal{position:fixed;inset:0;background:rgba(0,0,0,.45);display:flex;align-items:center;justify-content:center;z-index:999}
.modal-content{background:#fff;padding:18px;border-radius:10px;width:95%;max-width:720px;box-shadow:0 10px 40px rgba(2,6,23,0.12)}
.hidden{display:none}
.flex-row{display:flex;gap:8px;align-items:center}
@media(max-width:720px){header{flex-direction:column;align-items:flex-start}.controls{width:100%;justify-content:space-between}table{font-size:13px}}
#usuariosList{max-height:220px;overflow:auto;margin-bottom:10px;border:1px solid #eef2f6;border-radius:6px;padding:8px;background:#fafafa}
.usr-row{display:flex;justify-content:space-between;align-items:center;padding:6px 8px;border-bottom:1px solid #f1f5f9}
.gear{width:36px;height:36px;border-radius:8px;display:flex;align-items:center;justify-content:center;cursor:pointer;background:transparent;border:none;color:#fff;font-size:18px}
#acessosList{max-height:300px;overflow:auto;border:1px solid #eef2f6;border-radius:6px;padding:8px;background:#fff}
.acc-row{padding:8px;border-bottom:1px solid #f1f5f9;font-size:13px}
.qr-cell{display:flex;align-items:center;gap:6px}
.qr-canvas{width:50px;height:50px;}
</style>
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode@1.5.1/build/qrcode.min.js"></script>
<script src="https://unpkg.com/html5-qrcode"></script>
</head>
<body>
<!-- Seu login, header e main permanecem aqui -->
<header>
<div style="display:flex;gap:12px;align-items:center">
<div class="logo">Ponto Eletrônico QR</div>
<div id="status" class="muted">Offline • Local Storage</div>
</div>
<div style="display:flex;gap:12px;align-items:center">
<div id="clock">--:--:--</div>
<div class="controls">
<button class="download" id="baixarBtn">Baixar Planilhas (mês atual)</button>
<button class="download" id="gerarRelatorioBtn">Relatório Horas (mês atual)</button>
<button class="secondary" id="limparTodosPontosBtn">Limpar Pontos</button>
<button class="secondary" id="limparTodosColabsBtn">Apagar Todos Colaboradores</button>
<button class="gear secondary" id="gerenciarAcessosBtn" title="Gerenciar Logins" style="display:none">⚙️</button>
<button class="secondary" id="logoutBtn">Sair</button>
</div>
</div>
</header>
<main id="mainApp" class="hidden">
<input id="search" class="search" placeholder="🔍 Pesquisar colaborador por nome, cargo, matrícula ou e-mail">
<button class="add" id="addColabBtn">Adicionar Colaborador</button>
<table id="colabTable">
<thead><tr><th>#</th><th>ID</th><th>Nome</th><th>Cargo</th><th>Matrícula / E-mail</th><th>Turno</th><th>QR</th><th>Ações</th></tr></thead>
<tbody id="colabBody"></tbody>
</table>
<h3>Scanner de QR Code</h3>
<div id="qr-reader" style="width:300px;"></div>
</main>
<!-- Modal Colaborador -->
<div id="colabModal" class="modal hidden">
<div class="modal-content">
<h3 id="colabModalTitle">Adicionar Colaborador</h3>
<input id="nomeInput" placeholder="Nome"><br>
<input id="cargoInput" placeholder="Cargo"><br>
<input id="matriculaInput" placeholder="Matrícula"><br>
<input id="emailInput" placeholder="E-mail"><br>
<input id="turnoInput" placeholder="Turno"><br>
<div style="display:flex;gap:8px;justify-content:flex-end;margin-top:10px">
<button class="secondary" id="cancelColab">Cancelar</button>
<button class="add" id="saveColab">Salvar</button>
</div>
<canvas id="qrModalCanvas"></canvas>
</div>
</div>
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-app.js";
import { getFirestore, collection, getDocs, setDoc, doc, deleteDoc, onSnapshot, runTransaction, addDoc } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-firestore.js";
const firebaseConfig = {apiKey:"SUA_API", authDomain:"SUA_AUTH", projectId:"SEU_PROJECT"};
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
let colaboradores=[];
let pontos=[];
let colabEmEdicao=null;
// --- Funções utilitárias ---
function formatarHorasSegundos(totalSegundos){const h=Math.floor(totalSegundos/3600);const m=Math.floor((totalSegundos%3600)/60);const s=totalSegundos%60;return `${h}h ${m}m ${s}s`;}
// --- Render Colaboradores com QR ---
function renderColaboradores(filtro=''){const body=document.getElementById('colabBody');body.innerHTML='';colaboradores.slice().sort((a,b)=> (a.nome||'').localeCompare(b.nome||'')).filter(c=> (c.nome||'').toLowerCase().includes(filtro) || (c.cargo||'').toLowerCase().includes(filtro)).forEach((c,i)=>{
const tr=document.createElement('tr');
tr.innerHTML=`<td>${i+1}</td><td>${c.id}</td><td>${c.nome||''}</td><td>${c.cargo||''}</td><td>${c.matricula||''} <span class='small'>${c.email||''}</span></td><td>${c.turno||''}</td><td class='qr-cell'><canvas class='qr-canvas' id='qr-${c.id}'></canvas></td><td><button class='add btnEntrada'>Entrada</button> <button class='secondary btnSaida'>Saída</button> <button class='secondary editBtn'>Editar</button> <button class='danger delBtn'>Excluir</button></td>`;
tr.querySelector('.btnEntrada').onclick=()=>registrarPonto(c.id,'Entrada');
tr.querySelector('.btnSaida').onclick=()=>registrarPonto(c.id,'Saída');
tr.querySelector('.editBtn').onclick=()=>abrirModalEditar(c);
tr.querySelector('.delBtn').onclick=()=>removerColab(c.id);
body.appendChild(tr);
// gerar QR Code para cada colaborador
QRCode.toCanvas(document.getElementById(`qr-${c.id}`), c.id, {width:50}, function(error){if(error)console.error(error);});
});}
// --- Modal QR ---
function gerarQrModal(id){QRCode.toCanvas(document.getElementById('qrModalCanvas'), id, {width:100}, e=>e&&console.error(e));}
// --- Registrar ponto ---
async function registrarPonto(idColab,tipo){const c=colaboradores.find(x=>x.id===idColab);if(!c)return;const now=new Date();const p={id:Date.now().toString(),idColab,nome:c.nome,tipo,data:now.toLocaleDateString('pt-BR'),hora:now.toLocaleTimeString('pt-BR',{hour12:false}),horarioISO:now.toISOString()};pontos.push(p);await setDoc(doc(db,'pontos',p.id),p);renderEntradasSaidas();}
function renderEntradasSaidas(){/* Seu código de entradas/saídas permanece */}
// --- Abrir/Fechar Modal ---
function abrirModalAdicionar(){colabEmEdicao=null;document.getElementById('colabModalTitle').textContent='Adicionar Colaborador';nomeInput.value=cargoInput.value=matriculaInput.value=emailInput.value=turnoInput.value='';colabModal.classList.remove('hidden');}
function abrirModalEditar(c){colabEmEdicao=c;document.getElementById('colabModalTitle').textContent='Editar Colaborador';nomeInput.value=c.nome||'';cargoInput.value=c.cargo||'';matriculaInput.value=c.matricula||'';emailInput.value=c.email||'';turnoInput.value=c.turno||'';colabModal.classList.remove('hidden');gerarQrModal(c.id);}
function fecharModalColab(){colabModal.classList.add('hidden');}
// --- Salvar Colaborador ---
document.getElementById('saveColab').onclick=async()=>{const nome=nomeInput.value.trim();if(!nome)return;const obj={nome,cargo:cargoInput.value.trim(),matricula:matriculaInput.value.trim(),email:emailInput.value.trim(),turno:turnoInput.value.trim()};
let newId;
if(colabEmEdicao&&colabEmEdicao.id){await setDoc(doc(db,'colaboradores',colabEmEdicao.id),{...colabEmEdicao,...obj});newId=colabEmEdicao.id;}
else{const counterRef=doc(db,'meta','counters');const next=await runTransaction(db,async tx=>{const snap=await tx.get(counterRef);let last=0;if(snap.exists())last=snap.data().lastId||0;const novo=last+1;tx.set(counterRef,{lastId:novo},{merge:true});return novo;});newId=String(next);await setDoc(doc(db,'colaboradores',newId),{id:newId,...obj});}
fecharModalColab();iniciarLeituras();}
// --- Leitura QR Code ---
let ultimoTipo={};
function iniciarScanner(){
const html5QrCode=new Html5Qrcode('qr-reader');
html5QrCode.start({facingMode:'environment'}, {fps:10,qrbox:250}, qrCodeMessage=>{
const idColab=qrCodeMessage.trim();const tipo=ultimoTipo[idColab]==='Entrada'?'Saída':'Entrada';ultimoTipo[idColab]=tipo;registrarPonto(idColab,tipo);alert(`Colaborador ${idColab} bateu ${tipo}`);
},errorMessage=>{});
}
// --- Inicialização ---
async function iniciarLeituras(){
const colSnap=await getDocs(collection(db,'colaboradores'));colaboradores=colSnap.docs.map(d=>({id:d.id,...d.data()}));
renderColaboradores();
iniciarScanner();
}
window.onload=iniciarLeituras;
</script>
</body>
</html>
