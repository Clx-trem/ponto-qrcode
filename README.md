<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Ponto Eletrônico QR Code</title>

<style>
body{font-family:Arial;background:#f2f2f2;padding:20px;display:flex;justify-content:center}
.container{background:#fff;padding:20px;border-radius:10px;max-width:900px;width:100%;box-shadow:0 0 10px #0002}
input,button{padding:10px;margin:5px;width:90%}
#reader{margin-top:20px}
table{width:100%;margin-top:20px;border-collapse:collapse}
th,td{border:1px solid #ccc;padding:5px;text-align:center}
.alert.atraso{color:red;font-weight:bold}
.alert.extra{color:green;font-weight:bold}
</style>

</head>
<body>
<div class="container">

<!-- LOGIN -->
<div id="loginDiv">
    <h2>Login</h2>
    <input id="idInput" placeholder="Matrícula">
    <input id="nomeInput" placeholder="Nome">
    <label><input type="checkbox" id="adminCheckbox"> Admin</label>
    <button onclick="login()">Entrar</button>
</div>

<!-- DASHBOARD -->
<div id="dashboardDiv" style="display:none;">
    <h2>Bem-vindo, <span id="nomeUsuario"></span> (ID: <span id="idUsuario"></span>)</h2>

    <div id="adminButtons" style="display:none;">
        <button onclick="mostrarTodosColaboradores()">Ver Todos os Colaboradores</button>
    </div>

    <div id="userButtons">
        <h3>Scanner QR Code</h3>
        <div id="reader" style="width:300px;"></div>
        <div id="status"></div>
        <button onclick="exportarExcel()">Exportar Excel</button>
    </div>

    <h3>Histórico</h3>
    <table>
        <thead>
            <tr>
                <th>Nome</th><th>Tipo</th><th>Data</th><th>Hora</th><th>Total</th><th>Alerta</th>
            </tr>
        </thead>
        <tbody id="tabelaPontos"></tbody>
    </table>

    <canvas id="graficoHoras"></canvas>

    <button onclick="logout()">Sair</button>
</div>

</div>

<!-- LIBS -->
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<!-- FIREBASE + SISTEMA -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-app.js";
import { getFirestore, collection, addDoc, getDocs, query, orderBy } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-firestore.js";

const firebaseConfig={
  apiKey:"AIzaSyCiZXZ9vW-4L471ej9jWg_1MAStD44pTqo",
  authDomain:"ponto-qrcode-29f9d.firebaseapp.com",
  projectId:"ponto-qrcode-29f9d",
  storageBucket:"ponto-qrcode-29f9d.firebasestorage.app",
  messagingSenderId:"900058332385",
  appId:"1:900058332385:web:2ecdabb9b4027bc3748ba0"
};

const app=initializeApp(firebaseConfig);
const db=getFirestore(app);
window.firebaseDB={db,collection,addDoc,getDocs,query,orderBy};

// ---- LOGIN ----
function login() {
  const usuario = document.getElementById('loginUser').value.trim();
  const senha = document.getElementById('loginPass').value.trim();

  if (usuario === 'CLX' && senha === '02072007') {
    document.getElementById('loginBox').classList.add('hidden');
    document.getElementById('dashboard').classList.remove('hidden');
  } else {
    alert('Usuário ou senha incorretos');
  }
}

function logout(){
    localStorage.clear();
    loginDiv.style.display="block";
    dashboardDiv.style.display="none";
}

// ---- DASHBOARD ----
async function iniciarDashboard(){
    const id=localStorage.getItem("usuarioID");
    const nome=localStorage.getItem("usuarioNome");
    const admin=localStorage.getItem("usuarioAdmin")==="true";

    if(!id||!nome)return;

    loginDiv.style.display="none";
    dashboardDiv.style.display="block";

    nomeUsuario.textContent=nome;
    idUsuario.textContent=id;

    if(admin){
        adminButtons.style.display="block";
        userButtons.style.display="none";
    } else {
        iniciarScanner();
    }
    await carregarHistorico(admin);
}

// ---- SCANNER ----
function iniciarScanner(){
    const scanner=new Html5QrcodeScanner("reader",{fps:10,qrbox:250});
    scanner.render(onScanSuccess);
}

async function onScanSuccess(decoded){
    const matricula=decoded;
    const agora=new Date();
    const pontosRef=collection(db,"pontos");
    const dados=await getDocs(query(pontosRef,orderBy("hora","desc")));
    const historico=dados.docs.map(d=>d.data()).filter(r=>r.id===matricula);

    let tipo="entrada";
    if(historico[0] && historico[0].tipo==="entrada") tipo="saida";

    await addDoc(pontosRef,{
        id:matricula,
        nome:matricula,
        tipo,
        data:agora.toLocaleDateString("pt-BR"),
        hora:agora.toISOString()
    });

    status.innerText=`Ponto ${tipo} registrado: ${agora.toLocaleTimeString()}`;
    await carregarHistorico();
}

// ---- HISTÓRICO ----
async function carregarHistorico(admin=false){
    const id=localStorage.getItem("usuarioID");
    const tabela=tabelaPontos;
    tabela.innerHTML="";

    const pontosRef=collection(db,"pontos");
    const dados=await getDocs(query(pontosRef,orderBy("hora","asc")));
    let historico=dados.docs.map(d=>d.data());
    if(!admin) historico=historico.filter(r=>r.id===id);

    const porDia={};
    historico.forEach(r=>{
        const key=r.id+"_"+r.data;
        porDia[key]=porDia[key]||[];
        porDia[key].push(r);
    });

    const horasTotais={};

    for(const k in porDia){
        const regs=porDia[k];
        let total=0;
        for(let i=0;i<regs.length;i++){
            if(regs[i].tipo==="entrada" && regs[i+1] && regs[i+1].tipo==="saida"){
                total+=(new Date(regs[i+1].hora)-new Date(regs[i].hora))/3600000;
            }
        }

        regs.forEach(r=>{
            const tr=document.createElement("tr");
            let alerta="";
            if(total<8) alerta='<span class="alert atraso">Atraso</span>';
            if(total>8) alerta='<span class="alert extra">Extra</span>';
            tr.innerHTML=`<td>${r.nome}</td><td>${r.tipo}</td><td>${r.data}</td><td>${new Date(r.hora).toLocaleTimeString()}</td><td>${total.toFixed(2)}</td><td>${alerta}</td>`;
            tabela.appendChild(tr);
        });

        const nome=porDia[k][0].nome;
        horasTotais[nome]=(horasTotais[nome]||0)+total;
    }

    atualizarGrafico(horasTotais);
}

// ---- EXPORTAR ----
async function exportarExcel(){
    const admin=localStorage.getItem("usuarioAdmin")==="true";
    const id=localStorage.getItem("usuarioID");

    const dados=await getDocs(query(collection(db,"pontos"),orderBy("hora","asc")));
    let historico=dados.docs.map(d=>d.data());
    if(!admin) historico=historico.filter(r=>r.id===id);

    const ws=XLSX.utils.json_to_sheet(historico);
    const wb=XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb,ws,"Pontos");
    XLSX.writeFile(wb,"pontos.xlsx");
}

// ---- GERAL ----
function mostrarTodosColaboradores(){ carregarHistorico(true); }

let chart=null;
function atualizarGrafico(dados){
    const ctx=document.getElementById("graficoHoras").getContext("2d");
    if(chart) chart.destroy();
    chart=new Chart(ctx,{
        type:"bar",
        data:{labels:Object.keys(dados),datasets:[{label:"Horas",data:Object.values(dados)}]},
        options:{scales:{y:{beginAtZero:true}}}
    });
}

// ---- EXPOSE GLOBAL ----
window.login = login;
window.logout = logout;
window.exportarExcel = exportarExcel;
window.mostrarTodosColaboradores = mostrarTodosColaboradores;

if(localStorage.getItem("usuarioID")) iniciarDashboard();

</script>
</body>
</html>
