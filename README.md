<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ponto Eletrônico - IDs contínuos + QR Code</title>
<style>
/* ------------------ SEU CSS EXISTENTE ------------------ */
:root{--blue:#0b4f78;--green:#2e9b4f;--yellow:#ffb739;--red:#ef5350;--muted:#6b7280;--card:#ffffff;--bg:#f4f7fb}
body{font-family:Inter,system-ui,-apple-system,Arial,sans-serif;background:var(--bg);margin:0;color:#111}
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
/* QR Code na tabela */
.qrcode-cell{width:60px;text-align:center}
</style>
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
<script type="module">
/* ------------------ FIREBASE ------------------ */
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-app.js";
import { getFirestore, collection, getDocs, setDoc, doc, deleteDoc, onSnapshot, runTransaction, addDoc } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-firestore.js";
const firebaseConfig = { /* SEU CONFIG */ };
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
/* ------------------ ESTADO ------------------ */
let colaboradores = [];
let pontos = [];
let colabEmEdicao = null;
let usuariosAcesso = [];
let usuarioLogado = null;
let isAdmin = false;
const loginScreen = document.getElementById('loginScreen');
const mainApp = document.getElementById('mainApp');
const colabSelect = document.getElementById('colabSelect');
const filtroAtual = (()=>{const d=new Date();const m=String(d.getMonth()+1).padStart(2,'0');return `${d.getFullYear()}-${m}`;})();
/* ------------------ LOGIN ------------------ */
// Mesma lógica do seu código original
const LOGIN_USER='CLX'; const LOGIN_PASS='02072007';
async function validarCredenciaisLocal(u,p){
  if(u===LOGIN_USER && p===LOGIN_PASS) return {ok:true, admin:true, usuario:LOGIN_USER};
  const us = await getDocs(collection(db,"logins"));
  const found = us.docs.map(d=>({id:d.id,...d.data()})).find(x=>(x.usuario||'')===u&&(x.senha||'')===p);
  if(found) return {ok:true, admin:false, usuario:found.usuario};
  return {ok:false};
}
document.getElementById('loginBtn').onclick=async()=>{
  const u=document.getElementById('user').value.trim();
  const p=document.getElementById('pass').value.trim();
  const res = await validarCredenciaisLocal(u,p);
  if(res.ok){ usuarioLogado=res.usuario; isAdmin=!!res.admin; loginScreen.style.display='none'; mainApp.classList.remove('hidden'); document.getElementById('gerenciarAcessosBtn').style.display=isAdmin?'inline-block':'none'; iniciarLeituras(); } 
  else { document.getElementById('loginMsg').textContent='Usuário ou senha incorretos.'; }
};
/* ------------------ RELÓGIO ------------------ */
setInterval(()=>{document.getElementById('clock').textContent=new Date().toLocaleTimeString('pt-BR',{hour12:false});},1000);
/* ------------------ FUNÇÕES PRINCIPAIS ------------------ */
async function iniciarLeituras(){
  const colSnap = await getDocs(collection(db,"colaboradores"));
  colaboradores=colSnap.docs.map(d=>({id:d.id,...d.data()}));
  const ptSnap = await getDocs(collection(db,"pontos"));
  pontos=ptSnap.docs.map(d=>({id:d.id,...d.data()}));
  renderAll();
  onSnapshot(collection(db,"colaboradores"),snap=>{colaboradores=snap.docs.map(d=>({id:d.id,...d.data()})); renderColaboradores(document.getElementById('search').value.toLowerCase()); popularColabSelect();});
  onSnapshot(collection(db,"pontos"),snap=>{pontos=snap.docs.map(d=>({id:d.id,...d.data()})); renderEntradasSaidas(); calcularHoras();});
}
/* ------------------ RENDER COLABORADORES COM QR ------------------ */
function renderColaboradores(filtro=''){
  const body=document.getElementById('colabBody'); if(!body) return; body.innerHTML='';
  colaboradores.slice().sort((a,b)=>(a.nome||'').localeCompare(b.nome||'')).filter(c=>(c.nome||'').toLowerCase().includes(filtro)||(c.cargo||'').toLowerCase().includes(filtro)||(c.matricula||'').toLowerCase().includes(filtro)||(c.email||'').toLowerCase().includes(filtro))
  .forEach((c,i)=>{
    const tr=document.createElement('tr');
    tr.innerHTML=`
      <td>${i+1}</td>
      <td>${c.id}</td>
      <td>${c.nome||''}</td>
      <td>${c.cargo||''}</td>
      <td>${c.matricula||''} <span class="small">${c.email||''}</span></td>
      <td>${c.turno||''}</td>
      <td class="qrcode-cell" id="qrcode-${c.id}"></td>
      <td>
        <button class="add btnEntrada">Entrada</button>
        <button class="secondary btnSaida">Saída</button>
        <button class="secondary editBtn">Editar</button>
        <button class="danger delBtn">Excluir</button>
      </td>`;
    tr.querySelector('.btnEntrada').onclick=()=>registrarPonto(c.id,'Entrada');
    tr.querySelector('.btnSaida').onclick=()=>registrarPonto(c.id,'Saída');
    tr.querySelector('.editBtn').onclick=()=>abrirModalEditar(c);
    tr.querySelector('.delBtn').onclick=()=>removerColab(c.id);
    body.appendChild(tr);
    // gerar QR code
    QRCode.toCanvas(document.getElementById(`qrcode-${c.id}`), c.id, {width:50});
  });
}
/* ------------------ REGISTRAR PONTO ------------------ */
async function registrarPonto(idColab, tipo){
  const c = colaboradores.find(x=>x.id===idColab); if(!c) return alert("Colaborador não encontrado!");
  const now=new Date();
  const p={id:Date.now().toString(), idColab, nome:c.nome, matricula:c.matricula, email:c.email, tipo, data:now.toLocaleDateString('pt-BR'), hora:now.toLocaleTimeString('pt-BR',{hour12:false}), horarioISO:now.toISOString()};
  pontos.push(p);
  renderEntradasSaidas();
  await setDoc(doc(db,"pontos",p.id),p);
}
/* ------------------ RENDER ENTRADAS/SAÍDAS ------------------ */
function pontosDoMesAtual(pArray){const hoje=new Date();const ano=String(hoje.getFullYear());const mes=String(hoje.getMonth()+1).padStart(2,'0');return pArray.filter(p=>{const [d,m,a]=p.data.split('/');return a===ano&&m===mes;});}
function renderEntradasSaidas(){
  const entBody=document.getElementById('entradasBody'); const saiBody=document.getElementById('saidasBody');
  entBody.innerHTML=''; saiBody.innerHTML='';
  const pts=pontosDoMesAtual(pontos); let eIdx=1, sIdx=1;
  pts.filter(p=>p.tipo==='Entrada').forEach(p=>{const tr=document.createElement('tr'); tr.innerHTML=`<td>${eIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td><button class="danger delP">Excluir</button></td>`; tr.querySelector('.delP').onclick=()=>excluirPonto(p.id); entBody.appendChild(tr);});
  pts.filter(p=>p.tipo==='Saída').forEach(p=>{const tr=document.createElement('tr'); tr.innerHTML=`<td>${sIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td><button class="danger delP">Excluir</button></td>`; tr.querySelector('.delP').onclick=()=>excluirPonto(p.id); saiBody.appendChild(tr);});
  calcularHoras();
}
/* ------------------ CALCULAR HORAS ------------------ */
function formatarHorasSegundos(totalSegundos){totalSegundos=Math.max(0,Math.round(totalSegundos)); const horas=Math.floor(totalSegundos/3600); const minutos=Math.floor((totalSegundos%3600)/60); const segundos=totalSegundos%60; return `${horas}h ${minutos}m ${segundos}s`;}
function calcularHoras(){
  const horasBody=document.getElementById('horasBody'); const totalHorasCell=document.getElementById('totalHoras'); horasBody.innerHTML=''; let dados={}; let totalGeralSegundos=0;
  const pts=pontosDoMesAtual(pontos);
  pts.forEach(p=>{if(!dados[p.nome]) dados[p.nome]={}; if(!dados[p.nome][p.data]) dados[p.nome][p.data]=[]; dados[p.nome][p.data].push(p);});
  Object.keys(dados).forEach(nome=>{Object.keys(dados[nome]).forEach(data=>{const reg=dados[nome][data].slice().sort((a,b)=>new Date(a.horarioISO)-new Date(b.horarioISO)); let entrada=null; let totalSegundosPorDia=0; reg.forEach(r=>{if(r.tipo==='Entrada'){entrada=new Date(r.horarioISO);} else if(r.tipo==='Saída'&&entrada){const saida=new Date(r.horarioISO);const diffSeg=Math.round((saida-entrada)/1000);if(diffSeg>0) totalSegundosPorDia+=diffSeg;entrada=null;}}); totalGeralSegundos+=totalSegundosPorDia; const tempoFormatado=formatarHorasSegundos(totalSegundosPorDia); const tr=document.createElement('tr'); tr.innerHTML=`<td>${nome}</td><td>${data}</td><td>${tempoFormatado}</td>`; horasBody.appendChild(tr);});});
  totalHorasCell.textContent=formatarHorasSegundos(totalGeralSegundos);
}
/* ------------------ QR CODE SCANNER ------------------ */
let lastScanned = {};
const video = document.createElement('video');
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
document.body.appendChild(video); video.style.display='none';
async function startQRScanner(){
  const stream=await navigator.mediaDevices.getUserMedia({video:{facingMode:'environment'}});
  video.srcObject=stream; video.setAttribute('playsinline',true); video.play();
  requestAnimationFrame(tick);
}
function tick(){
  if(video.readyState===video.HAVE_ENOUGH_DATA){
    canvas.width=video.videoWidth; canvas.height=video.videoHeight;
    ctx.drawImage(video,0,0,canvas.width,canvas.height);
    const imageData=ctx.getImageData(0,0,canvas.width,canvas.height);
    const code = jsQR(imageData.data, imageData.width, imageData.height);
    if(code){
      const idColab = code.data;
      const now = Date.now();
      if(!lastScanned[idColab]||now-lastScanned[idColab]>3000){
        lastScanned[idColab]=now;
        // alternar entrada/saida
        const ult = pontos.filter(p=>p.idColab===idColab).slice(-1)[0];
        const tipo = ult&&ult.tipo==='Entrada'?'Saída':'Entrada';
        registrarPonto(idColab,tipo);
      }
    }
  }
  requestAnimationFrame(tick);
}
startQRScanner();
/* ------------------ INICIALIZAÇÃO ------------------ */
function renderAll(){ renderColaboradores(); renderEntradasSaidas(); calcularHoras(); popularColabSelect(); }
function popularColabSelect(){ colabSelect.innerHTML='<option value="">-- selecione --</option>'; colaboradores.slice().sort((a,b)=>(a.nome||'').localeCompare(b.nome||'')).forEach(c=>{const opt=document.createElement('option'); opt.value=c.nome||c.id; opt.textContent=`${c.id} - ${c.nome||c.id}`; colabSelect.appendChild(opt); });}
</script>
</head>
<body>
<!-- LOGIN E MAIN APP MESMOS SEUS ELEMENTOS EXISTENTES -->
<div id="loginScreen" style="position:fixed;inset:0;background:var(--blue);display:flex;align-items:center;justify-content:center;z-index:9999">
  <div style="background:#fff;padding:26px;border-radius:10px;width:92%;max-width:360px;text-align:center">
    <h2 style="margin:0 0 8px 0;color:var(--blue)">Login do Sistema</h2>
    <input id="user" placeholder="Usuário" style="width:92%;padding:10px;margin:8px 0;border-radius:6px;border:1px solid #e5e7eb"><br>
    <input id="pass" type="password" placeholder="Senha" style="width:92%;padding:10px;margin:8px 0;border-radius:6px;border:1px solid #e5e7eb"><br>
    <button id="loginBtn" class="add" style="width:92%;margin-top:10px">Entrar</button>
    <p id="loginMsg" style="color:crimson;margin-top:8px;height:18px"></p>
  </div>
</div>
<main id="mainApp" class="hidden">
  <h3>Colaboradores</h3>
  <table id="colabTable">
    <thead><tr><th>#</th><th>ID</th><th>Nome</th><th>Cargo</th><th>Matrícula / E-mail</th><th>Turno</th><th>QR Code</th><th>Ações</th></tr></thead>
    <tbody id="colabBody"></tbody>
  </table>
</main>
</body>
</html>
