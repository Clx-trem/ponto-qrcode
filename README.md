<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Controle de Ponto com QR Code</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css">
<script src="https://unpkg.com/html5-qrcode@2.3.8/minified/html5-qrcode.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
</head>
<body class="bg-gray-100 p-4">

<header class="flex items-center justify-between mb-4">
  <h1 class="text-2xl font-bold">Controle de Ponto</h1>
  <div class="controls flex items-center">
    <!-- Botão scanner QR será inserido via JS -->
  </div>
</header>

<main class="space-y-4">

  <!-- Tabela de colaboradores -->
  <table id="colabTable" class="w-full bg-white shadow rounded">
    <thead>
      <tr class="bg-gray-200">
        <th>#</th><th>ID</th><th>Nome</th><th>Cargo</th><th>Matrícula / Email</th><th>Turno</th><th>Ações</th>
      </tr>
    </thead>
    <tbody id="colabBody"></tbody>
  </table>

  <!-- Tabela Entradas -->
  <table id="entradasTable" class="w-full bg-white shadow rounded">
    <thead>
      <tr class="bg-gray-200"><th>#</th><th>ID</th><th>Nome</th><th>Data</th><th>Hora</th><th>Ações</th></tr>
    </thead>
    <tbody id="entradasBody"></tbody>
  </table>

  <!-- Tabela Saídas -->
  <table id="saidasTable" class="w-full bg-white shadow rounded">
    <thead>
      <tr class="bg-gray-200"><th>#</th><th>ID</th><th>Nome</th><th>Data</th><th>Hora</th><th>Ações</th></tr>
    </thead>
    <tbody id="saidasBody"></tbody>
  </table>

  <!-- Horas -->
  <table id="horasTable" class="w-full bg-white shadow rounded">
    <thead>
      <tr class="bg-gray-200"><th>Colaborador</th><th>Data</th><th>Horas</th></tr>
    </thead>
    <tbody id="horasBody"></tbody>
    <tfoot>
      <tr class="bg-gray-100"><td colspan="2">Total Geral</td><td id="totalHoras">0h 0m 0s</td></tr>
    </tfoot>
  </table>

</main>

<!-- Modal de Colaborador -->
<div id="colabModal" class="modal hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
  <div class="bg-white p-4 rounded w-96">
    <h2 id="colabModalTitle" class="text-lg font-bold mb-2">Adicionar Colaborador</h2>
    <input id="nomeInput" placeholder="Nome" class="w-full mb-2 p-2 border rounded"/>
    <input id="cargoInput" placeholder="Cargo" class="w-full mb-2 p-2 border rounded"/>
    <input id="matriculaInput" placeholder="Matrícula" class="w-full mb-2 p-2 border rounded"/>
    <input id="emailInput" placeholder="Email" class="w-full mb-2 p-2 border rounded"/>
    <input id="turnoInput" placeholder="Turno" class="w-full mb-2 p-2 border rounded"/>
    <div class="flex justify-end space-x-2 mt-2">
      <button id="cancelColab" class="secondary">Cancelar</button>
      <button id="saveColab" class="add">Salvar</button>
    </div>
  </div>
</div>

<!-- Modal Scanner QR -->
<div id="qrScannerModal" class="modal hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
  <div class="bg-white p-4 rounded w-96 text-center">
    <h3 class="text-lg font-bold mb-2">Scanner QR Code</h3>
    <div id="qrScanner" style="width:100%; height:300px;"></div>
    <button id="closeScannerBtn" class="secondary mt-2">Fechar Scanner</button>
  </div>
</div>

<script>
  // --------------------------
  // Dados iniciais
  // --------------------------
  let colaboradores = [];
  let pontos = [];
  let colabEmEdicao = null;

  // --------------------------
  // Funções utilitárias
  // --------------------------
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
      return a===ano && m===mes;
    });
  }

  // --------------------------
  // Renderizações
  // --------------------------
  function renderColaboradores(filtro='') {
    const body = document.getElementById('colabBody');
    body.innerHTML='';
    colaboradores.slice()
      .sort((a,b)=> (a.nome||'').localeCompare(b.nome||''))
      .filter(c=> (c.nome||'').toLowerCase().includes(filtro.toLowerCase()))
      .forEach((c,i)=>{
        const tr = document.createElement('tr');
        tr.innerHTML = `
          <td>${i+1}</td>
          <td>${c.id}</td>
          <td>${c.nome||''}</td>
          <td>${c.cargo||''}</td>
          <td>${c.matricula||''} <span class="small">${c.email||''}</span></td>
          <td>${c.turno||''}</td>
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
      });
  }

  function renderEntradasSaidas() {
    const entBody = document.getElementById('entradasBody');
    const saiBody = document.getElementById('saidasBody');
    entBody.innerHTML=''; saiBody.innerHTML='';
    const pts = pontosDoMesAtual(pontos);
    let eIdx=1,sIdx=1;
    pts.filter(p=>p.tipo==='Entrada').forEach(p=>{
      const tr=document.createElement('tr');
      tr.innerHTML=`<td>${eIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td><button class="danger delP">Excluir</button></td>`;
      tr.querySelector('.delP').onclick=()=>excluirPonto(p.id);
      entBody.appendChild(tr);
    });
    pts.filter(p=>p.tipo==='Saída').forEach(p=>{
      const tr=document.createElement('tr');
      tr.innerHTML=`<td>${sIdx++}</td><td>${p.idColab}</td><td>${p.nome}</td><td>${p.data}</td><td>${p.hora}</td><td><button class="danger delP">Excluir</button></td>`;
      tr.querySelector('.delP').onclick=()=>excluirPonto(p.id);
      saiBody.appendChild(tr);
    });
    calcularHoras();
  }

  function calcularHoras() {
    const horasBody=document.getElementById('horasBody');
    const totalHorasCell=document.getElementById('totalHoras');
    horasBody.innerHTML='';
    let dados={}; let totalGeralSegundos=0;
    const pts=pontosDoMesAtual(pontos);
    pts.forEach(p=>{if(!dados[p.nome]) dados[p.nome]={}; if(!dados[p.nome][p.data]) dados[p.nome][p.data]=[]; dados[p.nome][p.data].push(p)});
    Object.keys(dados).forEach(nome=>{
      Object.keys(dados[nome]).forEach(data=>{
        const reg=dados[nome][data].slice().sort((a,b)=> new Date(a.horarioISO)-new Date(b.horarioISO));
        let entrada=null,totalSegundosPorDia=0;
        reg.forEach(r=>{
          if(r.tipo==='Entrada') entrada=new Date(r.horarioISO);
          else if(r.tipo==='Saída' && entrada){
            const saida=new Date(r.horarioISO);
            const diffSeg=Math.round((saida-entrada)/1000);
            if(diffSeg>0) totalSegundosPorDia+=diffSeg;
            entrada=null;
          }
        });
        totalGeralSegundos+=totalSegundosPorDia;
        const tempoFormatado=formatarHorasSegundos(totalSegundosPorDia);
        const tr=document.createElement('tr');
        tr.innerHTML=`<td>${nome}</td><td>${data}</td><td>${tempoFormatado}</td>`;
        horasBody.appendChild(tr);
      });
    });
    totalHorasCell.textContent=formatarHorasSegundos(totalGeralSegundos);
  }

  // --------------------------
  // Cadastro / Modal
  // --------------------------
  const colabModal=document.getElementById('colabModal');
  const colabModalTitle=document.getElementById('colabModalTitle');
  const nomeInput=document.getElementById('nomeInput');
  const cargoInput=document.getElementById('cargoInput');
  const matriculaInput=document.getElementById('matriculaInput');
  const emailInput=document.getElementById('emailInput');
  const turnoInput=document.getElementById('turnoInput');

  function abrirModalAdicionar(){
    colabEmEdicao=null;
    colabModalTitle.textContent='Adicionar Colaborador';
    nomeInput.value=cargoInput.value=matriculaInput.value=emailInput.value=turnoInput.value='';
    colabModal.classList.remove('hidden');
  }

  function abrirModalEditar(c){
    colabEmEdicao=c;
    colabModalTitle.textContent='Editar Colaborador';
    nomeInput.value=c.nome||'';
    cargoInput.value=c.cargo||'';
    matriculaInput.value=c.matricula||'2'; // QR Code fixo
    emailInput.value=c.email||'';
    turnoInput.value=c.turno||'';
    colabModal.classList.remove('hidden');
  }

  document.getElementById('cancelColab').onclick=()=>colabModal.classList.add('hidden');

  document.getElementById('saveColab').onclick=()=>{
    const nome=nomeInput.value.trim();
    if(!nome) return alert('Informe o nome do colaborador');
    const obj={nome, cargo:cargoInput.value.trim(), matricula:matriculaInput.value.trim(), email:emailInput.value.trim(), turno:turnoInput.value.trim()};
    if(colabEmEdicao && colabEmEdicao.id){
      colabEmEdicao={...colabEmEdicao, ...obj};
    }else{
      obj.id=(colaboradores.length+1).toString();
      colaboradores.push(obj);
    }
    colabModal.classList.add('hidden');
    renderColaboradores();
  }

  // --------------------------
  // Registrar ponto
  // --------------------------
  function registrarPonto(idColab,tipo){
    const c=colaboradores.find(x=>x.id===idColab);
    if(!c) return alert('Colaborador não encontrado!');
    const now=new Date();
    const p={id:Date.now().toString(), idColab, nome:c.nome, matricula:c.matricula, tipo, data:now.toLocaleDateString('pt-BR'), hora:now.toLocaleTimeString('pt-BR',{hour12:false}), horarioISO:now.toISOString()};
    pontos.push(p);
    renderEntradasSaidas();
  }

  function excluirPonto(id){pontos=pontos.filter(p=>p.id!==id); renderEntradasSaidas();}
  function removerColab(id){colaboradores=colaboradores.filter(c=>c.id!==id); pontos=pontos.filter(p=>p.idColab!==id); renderColaboradores(); renderEntradasSaidas();}

  // --------------------------
  // QR Code Scanner
  // --------------------------
  const qrScannerModal=document.getElementById('qrScannerModal');
  const qrScannerDiv=document.getElementById('qrScanner');
  const closeScannerBtn=document.getElementById('closeScannerBtn');
  let html5QrCode=null, scannerAberto=false;

  function abrirScanner(){
    if(scannerAberto) return;
    qrScannerModal.classList.remove('hidden');
    scannerAberto=true;
    html5QrCode=new Html5Qrcode("qrScanner");
    Html5Qrcode.getCameras().then(cameras=>{
      if(cameras && cameras.length){
        html5QrCode.start(cameras[0].id,{fps:10, qrbox:250},
          qrCodeMessage=>{processarQRCode(qrCodeMessage);},
          errorMessage=>{}
        ).catch(err=>alert('Erro câmera:'+err));
      }
    }).catch(err=>alert('Erro câmeras:'+err));
  }

  closeScannerBtn.onclick=()=>{
    if(html5QrCode) html5QrCode.stop().then(()=>{html5QrCode.clear(); scannerAberto=false; qrScannerModal.classList.add('hidden');});
    else {scannerAberto=false; qrScannerModal.classList.add('hidden');}
  }

  function processarQRCode(qrValue){
    const c=colaboradores.find(col=>col.matricula===qrValue);
    if(!c){alert('Colaborador não encontrado!'); return;}
    const pontosColab=pontos.filter(p=>p.idColab===c.id).sort((a,b)=> new Date(a.horarioISO)-new Date(b.horarioISO));
    let tipo="Entrada";
    if(pontosColab.length>0){ const ultimo=pontosColab[pontosColab.length-1]; if(ultimo.tipo==='Entrada') tipo='Saída'; else tipo='Entrada'; }
    registrarPonto(c.id,tipo);
    alert(`${tipo} registrada para ${c.nome}`);
    closeScannerBtn.click();
  }

  // --------------------------
  // Botão abrir scanner
  // --------------------------
  const btnAbrirScanner=document.createElement('button');
  btnAbrirScanner.textContent='Abrir Scanner QR';
  btnAbrirScanner.className='add';
  btnAbrirScanner.style.marginLeft='10px';
  btnAbrirScanner.onclick=abrirScanner;
  document.querySelector('header .controls').appendChild(btnAbrirScanner);

  // --------------------------
  // Inicializar
  // --------------------------
  renderColaboradores();
  renderEntradasSaidas();

</script>

</body>
</html>
