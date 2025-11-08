<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Ponto Eletrônico QR Code</title>

<!-- CSS Unificado -->
<style>
body {
    font-family: Arial, sans-serif;
    background-color: #f5f5f5;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
}
.login-container, .dashboard-container {
    background-color: #fff;
    padding: 20px 30px;
    border-radius: 10px;
    box-shadow: 0 4px 8px rgba(0,0,0,0.2);
    text-align: center;
}
input {
    padding: 10px;
    margin: 10px 0;
    width: 80%;
}
button {
    padding: 10px 20px;
    margin: 5px;
    cursor: pointer;
}
table {
    width: 100%;
    margin-top: 10px;
    border-collapse: collapse;
}
th, td {
    border: 1px solid #ddd;
    padding: 8px;
}
</style>

<!-- Firebase SDK -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-app.js";
import { getFirestore, collection, addDoc, getDocs, query, orderBy } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";

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
window.firebaseApp = { db, collection, addDoc, getDocs, query, orderBy };
</script>

<!-- Bibliotecas Externas -->
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
</head>
<body>

<!-- LOGIN -->
<div class="login-container" id="loginDiv">
    <h1>Login</h1>
    <input type="text" id="idInput" placeholder="Digite sua matrícula">
    <input type="text" id="nomeInput" placeholder="Digite seu nome">
    <label><input type="checkbox" id="adminCheckbox"> Login como admin</label>
    <button onclick="login()">Entrar</button>
</div>

<!-- DASHBOARD -->
<div class="dashboard-container" id="dashboardDiv" style="display:none;">
    <h1>Bem-vindo, <span id="nomeUsuario"></span> (ID: <span id="idUsuario"></span>)</h1>

    <div id="adminButtons" style="display:none;">
        <button onclick="mostrarTodosColaboradores()">Ver Todos Colaboradores</button>
    </div>

    <div id="userButtons">
        <h3>Bater ponto via QR Code</h3>
        <div id="reader" style="width:300px;"></div>
        <div id="status"></div>
        <button onclick="exportarExcel()">Exportar para Excel</button>
    </div>

    <h2>Histórico de Pontos</h2>
    <table>
        <thead>
            <tr>
                <th>Colaborador</th>
                <th>Tipo</th>
                <th>Data</th>
                <th>Hora</th>
                <th>Horas Trabalhadas</th>
            </tr>
        </thead>
        <tbody id="tabelaPontos"></tbody>
    </table>

    <button onclick="logout()">Sair</button>
</div>

<!-- JS Unificado -->
<script type="module">
const loginDiv = document.getElementById('loginDiv');
const dashboardDiv = document.getElementById('dashboardDiv');

// --- LOGIN ---
function login() {
    const id = document.getElementById('idInput').value.trim();
    const nome = document.getElementById('nomeInput').value.trim();
    const admin = document.getElementById('adminCheckbox').checked;
    if(!id || !nome){ alert('Preencha os campos!'); return; }
    localStorage.setItem('usuarioID', id);
    localStorage.setItem('usuarioNome', nome);
    localStorage.setItem('usuarioAdmin', admin?'true':'false');
    iniciarDashboard();
}

// --- DASHBOARD ---
async function iniciarDashboard(){
    const id = localStorage.getItem('usuarioID');
    const nome = localStorage.getItem('usuarioNome');
    const admin = localStorage.getItem('usuarioAdmin')==='true';
    if(!id || !nome){ loginDiv.style.display='block'; dashboardDiv.style.display='none'; return; }

    loginDiv.style.display='none';
    dashboardDiv.style.display='block';

    document.getElementById('nomeUsuario').textContent=nome;
    document.getElementById('idUsuario').textContent=id;

    if(admin){
        document.getElementById('adminButtons').style.display='block';
        document.getElementById('userButtons').style.display='none';
    } else {
        iniciarScanner();
    }
    await carregarHistorico(admin);
}

// --- LOGOUT ---
function logout(){
    localStorage.clear();
    loginDiv.style.display='block';
    dashboardDiv.style.display='none';
}

// --- SCANNER QR ---
function iniciarScanner(){
    const scanner = new Html5QrcodeScanner("reader",{fps:10,qrbox:250});
    scanner.render(onScanSuccess);
}

// --- REGISTRAR PONTO ---
async function onScanSuccess(decodedText){
    const matricula = decodedText;
    const db = window.firebaseApp.db;
    const pontosRef = window.firebaseApp.collection(db,'pontos');
    const agora = new Date();

    const snapshot = await window.firebaseApp.getDocs(window.firebaseApp.query(pontosRef, window.firebaseApp.orderBy('hora','desc')));
    const historico = snapshot.docs.map(doc=>doc.data()).filter(r=>r.id===matricula);

    let tipo = 'entrada';
    if(historico.length>0){
        const ultimaEntrada = [...historico].reverse().find(r=>r.tipo==='entrada');
        const ultimaSaida = [...historico].reverse().find(r=>r.tipo==='saida');
        if(ultimaEntrada && (!ultimaSaida || new Date(ultimaSaida.hora)<new Date(ultimaEntrada.hora))){
            tipo='saida';
        }
    }

    await window.firebaseApp.addDoc(pontosRef,{
        id: matricula,
        nome: matricula,
        tipo,
        data: agora.toLocaleDateString('pt-BR'),
        hora: agora.toISOString()
    });

    document.getElementById('status').innerText=`Ponto ${tipo} registrado para ${matricula} às ${agora.toLocaleTimeString()}`;
    await carregarHistorico();
}

// --- CARREGAR HISTÓRICO ---
async function carregarHistorico(admin=false){
    const id = localStorage.getItem('usuarioID');
    const tabela = document.getElementById('tabelaPontos');
    if(!tabela) return;
    const db = window.firebaseApp.db;
    const pontosRef = window.firebaseApp.collection(db,'pontos');
    const snapshot = await window.firebaseApp.getDocs(window.firebaseApp.query(pontosRef, window.firebaseApp.orderBy('hora','asc')));
    let historico = snapshot.docs.map(doc=>doc.data());
    if(!admin) historico = historico.filter(r=>r.id===id);

    tabela.innerHTML='';

    const registrosPorDia = {};
    historico.forEach(reg=>{
        const key = reg.id+'_'+reg.data;
        if(!registrosPorDia[key]) registrosPorDia[key]=[];
        registrosPorDia[key].push(reg);
    });

    for(let key in registrosPorDia){
        const regs = registrosPorDia[key];
        let totalHoras = 0;
        for(let i=0;i<regs.length;i++){
            if(regs[i].tipo==='entrada' && regs[i+1] && regs[i+1].tipo==='saida'){
                totalHoras += (new Date(regs[i+1].hora)-new Date(regs[i].hora))/3600000;
            }
        }
        regs.forEach(reg=>{
            const tr=document.createElement('tr');
            tr.innerHTML=`<td>${reg.nome}</td><td>${reg.tipo}</td><td>${reg.data}</td><td>${new Date(reg.hora).toLocaleTimeString()}</td><td>${totalHoras.toFixed(2)}</td>`;
            tabela.appendChild(tr);
        });
    }
}

// --- EXPORTAR EXCEL ---
async function exportarExcel(){
    const admin = localStorage.getItem('usuarioAdmin')==='true';
    const id = localStorage.getItem('usuarioID');
    const db = window.firebaseApp.db;
    const pontosRef = window.firebaseApp.collection(db,'pontos');
    const snapshot = await window.firebaseApp.getDocs(window.firebaseApp.query(pontosRef, window.firebaseApp.orderBy('hora','asc')));
    let historico = snapshot.docs.map(doc=>doc.data());
    if(!admin) historico = historico.filter(r=>r.id===id);

    if(historico.length===0){alert('Nenhum ponto registrado!'); return;}

    const ws = XLSX.utils.json_to_sheet(historico.map(r=>({
        Colaborador:r.nome,
        Tipo:r.tipo,
        Data:r.data,
        Hora:new Date(r.hora).toLocaleTimeString()
    })));
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, "Pontos");
    XLSX.writeFile(wb, `Pontos_${admin?'Todos':localStorage.getItem('usuarioNome')}.xlsx`);
}

// --- MOSTRAR TODOS COLABORADORES ---
async function mostrarTodosColaboradores(){
    await carregarHistorico(true);
}

// Inicializa dashboard se já estiver logado
if(localStorage.getItem('usuarioID')) iniciarDashboard();
</script>

</body>
</html>
