<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Ponto Eletrônico</title>

<!-- XLSX -->
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
<!-- QRCode.js -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>

<style>
body { font-family: Arial, sans-serif; margin:20px; }
table { width: 100%; border-collapse: collapse; margin-bottom:20px; }
th, td { border: 1px solid #ccc; padding: 5px; text-align:left; }
th { background:#f3f4f6; }
button { margin:2px; }
.hidden { display:none; }
.small { font-size:0.8em; color:#555; }
</style>
</head>
<body>

<h1>Ponto Eletrônico</h1>

<button id="addColabBtn">Adicionar Colaborador</button>
<button id="limparTodosColabsBtn">Apagar todos colaboradores</button>
<button id="limparTodosPontosBtn">Apagar todos pontos</button>
<button id="baixarBtn">Exportar Excel</button>

<table>
  <thead>
    <tr>
      <th>#</th><th>ID</th><th>Nome</th><th>Cargo</th><th>Matrícula / Email</th><th>Turno</th><th>QR Code</th><th>Ações</th>
    </tr>
  </thead>
  <tbody id="colabBody"></tbody>
</table>

<h2>Entradas</h2>
<table>
  <thead><tr><th>#</th><th>ID Colab</th><th>Nome</th><th>Data</th><th>Hora</th><th>Ações</th></tr></thead>
  <tbody id="entradasBody"></tbody>
</table>

<h2>Saídas</h2>
<table>
  <thead><tr><th>#</th><th>ID Colab</th><th>Nome</th><th>Data</th><th>Hora</th><th>Ações</th></tr></thead>
  <tbody id="saidasBody"></tbody>
</table>

<h2>Horas Trabalhadas (mês atual)</h2>
<table>
  <thead><tr><th>Colaborador</th><th>Data</th><th>Horas</th></tr></thead>
  <tbody id="horasBody"></tbody>
  <tfoot><tr><td><b>Total</b></td><td></td><td id="totalHoras"></td></tr></tfoot>
</table>

<!-- Modal Adicionar/Editar -->
<div id="colabModal" class="hidden" style="position:fixed;top:10%;left:50%;transform:translateX(-50%);background:#fff;padding:20px;border:1px solid #ccc;">
  <h3 id="colabModalTitle">Adicionar Colaborador</h3>
  <label>Nome: <input id="nomeInput"></label><br>
  <label>Cargo: <input id="cargoInput"></label><br>
  <label>Matrícula: <input id="matriculaInput"></label><br>
  <label>Email: <input id="emailInput"></label><br>
  <label>Turno: <input id="turnoInput"></label><br><br>
  <button id="saveColab">Salvar</button>
  <button id="cancelColab">Cancelar</button>
</div>

<script>
// ----------- VARIÁVEIS E ESTADO -----------
let colaboradores = [];
let pontos = [];
let colabEmEdicao = null;
let filtroAtual = 'Atual'; // pode ser usado no Excel

// Mock temPermissao (substitua com sua lógica real)
function temPermissao(acao) { return true; }

// ----------- UTILIDADES -----------
function formatarHorasSegundos(totalSegundos) {
  totalSegundos = Math.max(0, Math.round(totalSegundos));
  const horas = Math.floor(totalSegundos / 3600);
  const minutos = Math.floor((totalSegundos % 3600) / 60);
  const segundos = totalSegundos % 60;
  return `${horas}h ${minutos}m ${segundos}s`;
}

function pontosDoMesAtual(pArray) {
  const hoje = new Date();
  const ano = String(hoje.getFullYear());
  const mes = String(hoje.getMonth()+1).padStart(2,'0');
  return pArray.filter(p => {
    const [d,m,a] = p.data.split('/');
    return a === ano && m === mes;
  });
}

// ----------- RENDER COLABORADORES -----------
function renderColaboradores(filtro='') {
  const body = document.getElementById('colabBody');
  body.innerHTML = '';
  colaboradores
    .slice()
    .sort((a,b) => (a.nome||'').localeCompare(b.nome||''))
    .filter(c => (c.nome||'').toLowerCase().includes(filtro.toLowerCase()) || (c.cargo||'').toLowerCase().includes(filtro.toLowerCase()))
    .forEach((c,i)=>{
      const tr = document.createElement('tr');
      tr.innerHTML = `
        <td>${i+1}</td>
        <td>${c.id}</td>
        <td>${c.nome||''}</td>
        <td>${c.cargo||''}</td>
        <td>${c.matricula||''} <span class="small">${c.email||''}</span></td>
        <td>${c.turno||''}</td>
        <td id="qrcode-${c.id}"></td>
        <td>
          <button class="add btnEntrada">Entrada</button>
          <button class="secondary btnSaida">Saída</button>
          <button class="secondary editBtn">Editar</button>
          <button class="danger delBtn">Excluir</button>
        </td>
      `;

      // QR Code fixo = 2
      new QRCode(tr.querySelector(`#qrcode-${c.id}`), {text:"2", width:60, height:60});

      tr.querySelector('.btnEntrada').onclick = ()=>registrarPonto(c.id,'Entrada');
      tr.querySelector('.btnSaida').onclick = ()=>registrarPonto(c.id,'Saída');
      tr.querySelector('.editBtn').onclick = ()=>abrirModalEditar(c);
      tr.querySelector('.delBtn').onclick = ()=>removerColab(c.id);

      body.appendChild(tr);
    });
}

// ----------- CRUD COLABORADORES -----------
function abrirModalAdicionar(){
  colabEmEdicao=null;
  document.getElementById('colabModalTitle').textContent='Adicionar Colaborador';
  ['nomeInput','cargoInput','matriculaInput','emailInput','turnoInput'].forEach(id=>document.getElementById(id).value='');
  document.getElementById('colabModal').classList.remove('hidden');
}

function abrirModalEditar(c){
  colabEmEdicao=c;
  document.getElementById('colabModalTitle').textContent='Editar Colaborador';
  document.getElementById('nomeInput').value=c.nome||'';
  document.getElementById('cargoInput').value=c.cargo||'';
  document.getElementById('matriculaInput').value=c.matricula||'';
  document.getElementById('emailInput').value=c.email||'';
  document.getElementById('turnoInput').value=c.turno||'';
  document.getElementById('colabModal').classList.remove('hidden');
}

document.getElementById('cancelColab').onclick = ()=>document.getElementById('colabModal').classList.add('hidden');

document.getElementById('saveColab').onclick = ()=>{
  const nome = document.getElementById('nomeInput').value.trim();
  if(!nome) return alert('Informe o nome do colaborador');
  const obj = {
    nome,
    cargo:document.getElementById('cargoInput').value.trim(),
    matricula:document.getElementById('matriculaInput').value.trim(),
    email:document.getElementById('emailInput').value.trim(),
    turno:document.getElementById('turnoInput').value.trim()
  };
  if(colabEmEdicao){ // editar
    Object.assign(colabEmEdicao,obj);
  } else { // novo
    obj.id = Date.now().toString();
    colaboradores.push(obj);
  }
  document.getElementById('colabModal').classList.add('hidden');
  renderColaboradores();
  renderEntradasSaidas();
};

// ----------- PONTO -----------
function registrarPonto(idColab,tipo){
  const c = colaboradores.find(x=>x.id===idColab);
  if(!c) return alert("Colaborador não encontrado!");
  const now = new Date();
  const p = {id:Date.now().toString(),idColab,nome:c.nome,matricula:c.matricula,email:c.email,tipo,data:now.toLocaleDateString('pt-BR'),hora:now.toLocaleTimeString('pt-BR',{hour12:false}),horarioISO:now.toISOString()};
  pontos.push(p);
  renderEntradasSaidas();
}

// ----------- RENDER ENTRADAS / SAÍDAS -----------
function renderEntradasSaidas(){
  const entBody = document.getElementById('entradasBody');
  const saiBody = document.getElementById('saidasBody');
  entBody.innerHTML=''; saiBody.innerHTML='';
  const pts = pontosDoMesAtual(pontos);
  let eIdx=1, sIdx=1;
  pts.filter(p=>p.tipo==='Entrada').forEach(p=>{
    const tr=document.createElement('tr');
    tr.innerHTML=`<td>${eIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td></td>`;
    entBody.appendChild(tr);
  });
  pts.filter(p=>p.tipo==='Saída').forEach(p=>{
    const tr=document.createElement('tr');
    tr.innerHTML=`<td>${sIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td></td>`;
    saiBody.appendChild(tr);
  });
  calcularHoras();
}

// ----------- CALCULAR HORAS -----------
function calcularHoras(){
  const horasBody=document.getElementById('horasBody');
  const totalHorasCell=document.getElementById('totalHoras');
  horasBody.innerHTML='';
  let dados={}; let totalGeralSegundos=0;
  const pts = pontosDoMesAtual(pontos);
  pts.forEach(p=>{
    if(!dados[p.nome]) dados[p.nome]={};
    if(!dados[p.nome][p.data]) dados[p.nome][p.data]=[];
    dados[p.nome][p.data].push(p);
  });
  Object.keys(dados).forEach(nome=>{
    Object.keys(dados[nome]).forEach(data=>{
      const reg=dados[nome][data].slice().sort((a,b)=>new Date(a.horarioISO)-new Date(b.horarioISO));
      let entrada=null, totalSegundosPorDia=0;
      reg.forEach(r=>{
        if(r.tipo==='Entrada') entrada=new Date(r.horarioISO);
        else if(r.tipo==='Saída' && entrada){
          const diffSeg=Math.round((new Date(r.horarioISO)-entrada)/1000);
          if(diffSeg>0) totalSegundosPorDia+=diffSeg;
          entrada=null;
        }
      });
      totalGeralSegundos+=totalSegundosPorDia;
      const tr=document.createElement('tr');
      tr.innerHTML=`<td>${nome}</td><td>${data}</td><td>${formatarHorasSegundos(totalSegundosPorDia)}</td>`;
      horasBody.appendChild(tr);
    });
  });
  totalHorasCell.textContent=formatarHorasSegundos(totalGeralSegundos);
}

// ----------- REMOVER COLABORADOR -----------
function removerColab(id){
  if(confirm("Excluir colaborador permanentemente?")){
    colaboradores = colaboradores.filter(c=>c.id!==id);
    pontos = pontos.filter(p=>p.idColab!==id);
    renderColaboradores();
    renderEntradasSaidas();
  }
}

// ----------- BOTÕES GLOBAIS -----------
document.getElementById('addColabBtn').onclick = abrirModalAdicionar;
document.getElementById('limparTodosColabsBtn').onclick = ()=>{
  if(confirm("Apagar TODOS os colaboradores e pontos?")){
    colaboradores=[]; pontos=[];
    renderColaboradores();
    renderEntradasSaidas();
  }
};
document.getElementById('limparTodosPontosBtn').onclick = ()=>{
  if(confirm("Apagar todos os pontos?")){
    pontos=[];
    renderEntradasSaidas();
  }
};

// ----------- EXPORT EXCEL -----------
document.getElementById('baixarBtn').onclick=()=>{
  const ptsMes = pontosDoMesAtual(pontos);
  const entradas = [['#','ID Colab','Nome','Data','Hora']];
  const saidas = [['#','ID Colab','Nome','Data','Hora']];
  ptsMes.forEach((p,i)=>{
    if(p.tipo==='Entrada') entradas.push([i+1,p.idColab,p.nome,p.data,p.hora]);
    else saidas.push([i+1,p.idColab,p.nome,p.data,p.hora]);
  });
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb,XLSX.utils.aoa_to_sheet(entradas),'Entradas');
  XLSX.utils.book_append_sheet(wb,XLSX.utils.aoa_to_sheet(saidas),'Saídas');
  XLSX.writeFile(wb, `Pontos_${filtroAtual}.xlsx`);
};

// ----------- INICIALIZAÇÃO -----------
renderColaboradores();
renderEntradasSaidas();
</script>
</body>
</html>
