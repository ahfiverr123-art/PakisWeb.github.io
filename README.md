<html lang="en">
<head>
<meta charset="UTF-8">
<title>Pakis Web Chat with Owner</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;500;700&display=swap" rel="stylesheet">

<style>
*{margin:0;padding:0;box-sizing:border-box;font-family:'Poppins',sans-serif;}
body{
  min-height:100vh;
  background: linear-gradient(270deg,#ff004c,#00ffcc,#7f00ff,#ff9900);
  background-size:800% 800%;
  animation: rgb 15s ease infinite;
  color:white;
  display:flex;
}
@keyframes rgb{
  0%{background-position:0% 50%}
  50%{background-position:100% 50%}
  100%{background-position:0% 50%}
}

/* SIDEBAR */
.sidebar{
  width:230px;
  background:rgba(0,0,0,0.65);
  padding:20px;
}
.sidebar h2{
  text-align:center;
  margin-bottom:25px;
  color:#00ffcc;
}
.sidebar button{
  width:100%;
  padding:12px;
  margin:8px 0;
  border:none;
  border-radius:12px;
  background:rgba(255,255,255,0.1);
  color:white;
  cursor:pointer;
  font-size:16px;
}
.sidebar button:hover{
  background:#00ffcc;
  color:black;
}

/* MAIN */
.main{
  flex:1;
  padding:30px;
  overflow:auto;
}
.section{display:none;animation:fade .4s}
.section.active{display:block}
@keyframes fade{from{opacity:0}to{opacity:1}}

.box{
  max-width:520px;
  background:rgba(0,0,0,0.65);
  padding:30px;
  border-radius:20px;
  box-shadow:0 0 25px rgba(0,0,0,0.4);
}

h1 span{color:#00ffcc}

input, textarea, select{
  width:100%;
  padding:12px;
  margin:10px 0;
  border:none;
  border-radius:10px;
}
button.action{
  width:100%;
  padding:12px;
  margin-top:10px;
  border:none;
  border-radius:10px;
  background:#00ffcc;
  color:black;
  font-weight:bold;
  cursor:pointer;
}
button.action:disabled{opacity:.5}

/* POINTS BOX */
#pointsBox{
  position:fixed;
  top:15px;
  right:15px;
  background:rgba(0,0,0,0.75);
  padding:10px 16px;
  border-radius:12px;
  font-weight:bold;
  color:#00ffcc;
  box-shadow:0 0 15px rgba(0,255,204,0.5);
  z-index:999;
}

/* TIC TAC TOE */
.board{
  display:grid;
  grid-template-columns:repeat(3,100px);
  gap:12px;
  justify-content:center;
  margin:20px auto;
}
.cell{
  width:100px;
  height:100px;
  background:rgba(255,255,255,0.15);
  border-radius:15px;
  font-size:50px;
  font-weight:bold;
  display:flex;
  align-items:center;
  justify-content:center;
  cursor:pointer;
  transition:.2s;
}
.cell.x{color:#00ffcc;}
.cell.o{color:#ff4dd2;}
.cell:hover{background:rgba(255,255,255,0.3);}
#gameStatus{text-align:center;margin-top:15px;font-weight:bold}

/* CHAT */
#chatBox{
  height:300px;
  overflow-y:auto;
  background:rgba(0,0,0,0.85);
  padding:10px;
  border-radius:12px;
  margin-bottom:10px;
}
.userMsg{color:#00ffcc;margin:6px 0;}
.ownerMsg{color:#ffdd00;margin:6px 0;}
</style>

<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

<script>
const firebaseConfig = {
  apiKey: "AIzaSyBKTQwhzTYm74ZRMYGo3q8EWQm_7W_vBiw",
  authDomain: "pakisweb-9032c.firebaseapp.com",
  databaseURL: "https://pakisweb-9032c-default-rtdb.firebaseio.com",
  projectId: "pakisweb-9032c",
  storageBucket: "pakisweb-9032c.appspot.com",
  messagingSenderId: "316212410418",
  appId: "1:316212410418:web:dedc95eae2a22c442ea42b"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

let user="", points=0, cooldown=false;

/* AUTH */
function createAcc(){
  const u=uname.value.trim(), p=pass.value.trim();
  if(!u||!p) return alert("Fill all fields");
  db.ref("users/"+u).get().then(s=>{
    if(s.exists()) return alert("User exists");
    db.ref("users/"+u).set({pass:p,points:0});
    alert("Account Created");
  });
}
function signIn(){
  const u=uname.value.trim(), p=pass.value.trim();
  db.ref("users/"+u).get().then(s=>{
    if(!s.exists()||s.val().pass!==p) return alert("Wrong login");
    user=u; points=s.val().points;
    login.style.display="none";
    app.style.display="flex";
    updatePoints();
    show("home");
    loadMessages();
  });
}

/* NAV */
function show(id){
  document.querySelectorAll(".section").forEach(s=>s.classList.remove("active"));
  document.getElementById(id).classList.add("active");
}

/* POINTS */
function updatePoints(){
  pointsVal.innerText=points;
  if(user) db.ref("users/"+user).update({points});
}
function earn(){
  if(cooldown) return;
  cooldown=true;
  earnBtn.disabled=true;
  points+=5;
  updatePoints();
  setTimeout(()=>{cooldown=false;earnBtn.disabled=false},2000);
}

/* SHOP PURCHASES */
function buyItem(name, cost){
  if(points>=cost){
    points-=cost;
    updatePoints();
    db.ref("purchases/"+user).push({item:name,time:Date.now()});
    alert(name + " purchased successfully!");
  } else alert("Not enough points ("+cost+")");
}

/* TIC TAC TOE */
let board=["","","","","","","","",""];
let gameOver=false;
const winSets=[[0,1,2],[3,4,5],[6,7,8],[0,3,6],[1,4,7],[2,5,8],[0,4,8],[2,4,6]];

function render(){
  board.forEach((v,i)=>{
    cells[i].innerText=v;
    cells[i].className="cell "+(v==="X"?"x":v==="O"?"o":"");
  });
}

function playerMove(i){
  if(board[i]||gameOver) return;
  board[i]="X";
  checkWin();
  aiMove();
}

function aiMove(){
  if(gameOver) return;
  let empty=board.map((v,i)=>v==""?i:null).filter(v=>v!==null);
  if(!empty.length) return;
  let move=empty[Math.floor(Math.random()*empty.length)];
  board[move]="O";
  checkWin();
}

function checkWin(){
  for(let w of winSets){
    if(board[w[0]] && board[w[0]]===board[w[1]] && board[w[1]]===board[w[2]]){
      gameStatus.innerText=board[w[0]]+" wins!";
      gameOver=true;
      render();
      return;
    }
  }
  if(!board.includes("")){
    gameStatus.innerText="Draw!";
    gameOver=true;
  }
  render();
}

function resetGame(){
  board=["","","","","","","","",""];
  gameOver=false;
  gameStatus.innerText="";
  render();
}

/* ------------------- USER CHAT ------------------- */
function sendMessage(){
  const msg = chatInput.value.trim();
  if(!msg || !user) return;

  db.ref("chats/"+user).push({
    sender: "user",
    text: msg,
    time: Date.now()
  });

  chatInput.value="";
}

function loadMessages(){
  const chatBox = document.getElementById("chatBox");
  chatBox.innerHTML="";
  
  db.ref("chats/"+user).on("child_added", snap=>{
    const data = snap.val();
    const div = document.createElement("div");
    div.classList.add(data.sender === "user" ? "userMsg" : "ownerMsg");
    div.innerText = data.sender + ": " + data.text;
    chatBox.appendChild(div);
    chatBox.scrollTop = chatBox.scrollHeight;
  });
}

// Enter key for chat (user side)
document.addEventListener("DOMContentLoaded", ()=>{
  chatInput.addEventListener("keypress", function(e){
    if(e.key==="Enter"){ e.preventDefault(); sendMessage(); }
  });
});
</script>
</head>
<body>

<div id="pointsBox">⭐ Points: <span id="pointsVal">0</span></div>

<!-- LOGIN -->
<div class="main" id="login">
  <div class="box">
    <h1>Pakis <span>Web</span></h1>
    <input id="uname" placeholder="Username">
    <input id="pass" type="password" placeholder="Password">
    <button class="action" onclick="signIn()">Sign In</button>
    <button class="action" onclick="createAcc()">Create Account</button>
  </div>
</div>

<!-- APP -->
<div id="app" style="display:none;width:100%">
  <div class="sidebar">
    <h2>Pakis Web</h2>
    <button onclick="show('home')">🏠 Home</button>
    <button onclick="show('earn')">⚡ Earn</button>
    <button onclick="show('games')">🎮 Games</button>
    <button onclick="show('shop')">🛒 Shop</button>
    <button onclick="show('chat')">💬 Chat</button>
  </div>

  <div class="main">
    <div id="home" class="section">
      <div class="box">
        <h1>Welcome to <span>Pakisboy</span></h1>
        <p>Pakisboy is a YouTube channel featuring fun, casual, and lifestyle content including shorts, challenges, and livestreams.</p>
        <p>🔗 My Channel Link: <a href="https://www.youtube.com/@Pakisboy" target="_blank" style="color:#00ffcc;text-decoration:underline;">www.youtube.com/@Pakisboy</a></p>
      </div>
    </div>

    <div id="earn" class="section">
      <div class="box">
        <h1>Earn Points</h1>
        <button id="earnBtn" class="action" onclick="earn()">1 Click (+5)</button>
        <p style="opacity:.8;margin-top:10px">2 second cooldown</p>
      </div>
    </div>

    <div id="games" class="section">
      <div class="box">
        <h1>Tic Tac Toe</h1>
        <div class="board">
          <div class="cell" onclick="playerMove(0)"></div>
          <div class="cell" onclick="playerMove(1)"></div>
          <div class="cell" onclick="playerMove(2)"></div>
          <div class="cell" onclick="playerMove(3)"></div>
          <div class="cell" onclick="playerMove(4)"></div>
          <div class="cell" onclick="playerMove(5)"></div>
          <div class="cell" onclick="playerMove(6)"></div>
          <div class="cell" onclick="playerMove(7)"></div>
          <div class="cell" onclick="playerMove(8)"></div>
        </div>
        <p id="gameStatus"></p>
        <button class="action" onclick="resetGame()">Restart</button>
      </div>
    </div>

    <div id="shop" class="section">
      <div class="box">
        <h1>Shop</h1>
        <p>165 Points = Minecraft Premium Account (Temporary)</p>
        <button class="action" onclick="buyItem('Minecraft Premium Account (Temp)',165)">Buy</button>

        <p>200 Points = Netflix Account</p>
        <button class="action" onclick="buyItem('Netflix Account',200)">Buy</button>

        <p>1000 Points = Minecraft Permanent Account</p>
        <button class="action" onclick="buyItem('Minecraft Permanent Account',1000)">Buy</button>
      </div>
    </div>

    <div id="chat" class="section">
      <div class="box">
        <h1>Chat with Owner</h1>
        <div id="chatBox"></div>
        <textarea id="chatInput" placeholder="Type a message..."></textarea>
        <button class="action" onclick="sendMessage()">Send</button>
      </div>
    </div>

  </div>
</div>

<script>
const cells=document.querySelectorAll(".cell");
render();
</script>

</body>
</html>
