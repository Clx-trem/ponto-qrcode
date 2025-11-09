<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Ponto QRCode</title>
<style>
body { font-family: Arial, sans-serif; background:#f2f2f2; margin:0; padding:0; }
.container { max-width:500px; margin:40px auto; background:white; padding:30px; border-radius:12px; box-shadow:0 0 10px rgba(0,0,0,0.1); }
h1 { text-align:center; }
input { width:100%; padding:12px; margin-top:10px; border:1px solid #ccc; border-radius:6px; }
button { width:100%; padding:14px; margin-top:20px; background:black; color:white; border:none; border-radius:6px; cursor:pointer; }
button:hover { opacity:0.8; }
.hidden { display:none; }
</style>
</head>
<body>

<div id="loginBox" class="container">
    <h1>Login</h1>
    <input id="loginUser" placeholder="Usuário">
    <input id="loginPass" type="password" placeholder="Senha">
    <button onclick="login()">Entrar</button>
</div>

<div id="dashboard" class="container hidden">
    <h1>Painel</h1>
    <p>Login realizado com sucesso!</p>
    <button onclick="logout()">Sair</button>
</div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.6.0/firebase-app.js";

const firebaseConfig = {
  apiKey: "AIzaSyCiZXZ9vW-4L471ej9jWg_1MAStD44pTqo",
  authDomain: "ponto-qrcode-29f9d.firebaseapp.com",
  projectId: "ponto-qrcode-29f9d",
  storageBucket: "ponto-qrcode-29f9d.firebasestorage.app",
  messagingSenderId: "900058332385",
  appId: "1:900058332385:web:2ecdabb9b4027bc3748ba0"
};

initializeApp(firebaseConfig);

function login() {
    const usuario = document.getElementById('loginUser').value.trim();
    const senha = document.getElementById('loginPass').value.trim();

    if(usuario === 'CLX' && senha === '02072007'){
        document.getElementById('loginBox').classList.add('hidden');
        document.getElementById('dashboard').classList.remove('hidden');
    } else {
        alert('Usuário ou senha incorretos');
    }
}

function logout(){
    document.getElementById('dashboard').classList.add('hidden');
    document.getElementById('loginBox').classList.remove('hidden');
}

// Tornar funções visíveis para o HTML
window.login = login;
window.logout = logout;

</script>
</body>
</html>
