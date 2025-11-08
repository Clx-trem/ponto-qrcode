<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ponto Eletrônico - QR Code</title>
<style>
/* SEU CSS EXISTENTE (mantido) */
:root{ --blue:#0b4f78; --green:#2e9b4f; --yellow:#ffb739; --red:#ef5350; --muted:#6b7280; --card:#ffffff; --bg:#f4f7fb; }
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
@media(max-width:720px){ header{flex-direction:column;align-items:flex-start} .controls{width:100%;justify-content:space-between} table{font-size:13px} }
/* QR Code container */
#qrcodeContainer{margin-top:12px;}
</style>
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/html5-qrcode@2.3.7/minified/html5-qrcode.min.js"></script>
</head>
<body>

/* ---------- DADOS E ESTADO ---------- */
let colaboradores = [];
let pontos = [];
let colabEmEdicao = null;
let scanQRLeitura = {};

/* ---------- FUNÇÕES DE RENDER ---------- */
function renderColaboradores(filtro='') {
 const body = document.getElementById('colabBody');
 body.innerHTML='';
 colaboradores.slice().sort((a,b)=>(a.nome||'').localeCompare(b.nome||'')).filter(c=> (c.nome||'').toLowerCase().includes(filtro.toLowerCase()) || (c.cargo||'').toLowerCase().includes(filtro.toLowerCase()) ).forEach((c,i)=>{
 const tr=document.createElement('tr');
 tr.innerHTML=`
 <td>${i+1}</td>
 <td>${c.id}</td>
 <td>${c.nome||''}</td>
 <td>${c.cargo||''}</td>
 <td>${c.matricula||''} <span class="small">${c.email||''}</span></td>
 <td>${c.turno||''}</td>
 <td><div id="qrcode_${c.id}" style="width:60px;height:60px"></div></td>
 <td>
 <button class="add btnEntrada">Entrada</button>
 <button class="secondary btnSaida">Saída</button>
 <button class="secondary editBtn">Editar</button>
 <button class="danger delBtn">Excluir</button>
 </td>
 `;
 tr.querySelector('.btnEntrada').onclick=()=> registrarPonto(c.id,'Entrada');
 tr.querySelector('.btnSaida').onclick=()=> registrarPonto(c.id,'Saída');
 tr.querySelector('.editBtn').onclick=()=> abrirModalEditar(c);
 tr.querySelector('.delBtn').onclick=()=> removerColab(c.id);
 body.appendChild(tr);
 // gerar QRCode com valor id do colaborador
 QRCode.toCanvas(document.getElementById(`qrcode_${c.id}`), c.id, {width:60});
 });
}

/* ---------- REGISTRAR PONTO ---------- */
function registrarPonto(idColab,tipo){
 const c = colaboradores.find(x=>x.id===idColab);
 if(!c) return alert('Colaborador não encontrado!');
 const now=new Date();
 const p={id:Date.now().toString(),idColab,nome:c.nome,matricula:c.matricula,email:c.email,tipo,data:now.toLocaleDateString('pt-BR'),hora:now.toLocaleTimeString('pt-BR',{hour12:false}),horarioISO:now.toISOString()};
 pontos.push(p);
 renderEntradasSaidas();
}

/* ---------- ENTRADAS/SAÍDAS ---------- */
function renderEntradasSaidas(){
 const entBody=document.getElementById('entradasBody');
 const saiBody=document.getElementById('saidasBody');
 entBody.innerHTML=''; saiBody.innerHTML='';
 let eIdx=1,sIdx=1;
 pontos.forEach(p=>{
 if(p.tipo==='Entrada') entBody.innerHTML+=`<tr><td>${eIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td></td></tr>`;
 if(p.tipo==='Saída') saiBody.innerHTML+=`<tr><td>${sIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td></td></tr>`;
 });
}

/* ---------- QR CODE SCANNER ---------- */
function iniciarQRCodeScanner(){
 const html5QrCode = new Html5Qrcode("qrReader");
 html5QrCode.start({facingMode:"environment"},
 {fps:10, qrbox:250},
 (decodedText,decodedResult)=>{
 document.getElementById('qrResult').innerText=`QR Lido: ${decodedText}`;
 const colab = colaboradores.find(c=>c.id===decodedText);
 if(!colab) return;
 // alternar entrada/saida ao bater via QR
 const last = scanQRLeitura[decodedText];
 const tipo = last==='Entrada'?'Saída':'Entrada';
 scanQRLeitura[decodedText]=tipo;
 registrarPonto(decodedText,tipo);
 }
 ).catch(err=>console.warn(err));
}

/* ---------- INICIALIZAÇÃO ---------- */
function inicializar(){
 // EXEMPLO DE DADOS
 colaboradores=[
 {id:'1',nome:'Carlos',cargo:'Dev',matricula:'001',email:'carlos@ex.com',turno:'Manhã'},
 {id:'2',nome:'Ana',cargo:'RH',matricula:'002',email:'ana@ex.com',turno:'Tarde'}
 ];
 renderColaboradores();
 renderEntradasSaidas();
 iniciarQRCodeScanner();
}
window.onload=inicializar;
</script>
</body>
</html>
