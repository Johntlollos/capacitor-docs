
<!DOCTYPE html>
<html lang="it">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no" />
<title>Dominio Supremo - Gioco Competitivo</title>
<style>
  body { font-family: Arial, sans-serif; background: #121212; color: #eee; margin: 0; }
  #container { max-width: 960px; margin: 10px auto; background: #222; padding: 15px; border-radius: 8px; }
  select, button, input, textarea { font-size: 1em; margin: 5px 0; width: 100%; padding: 5px; border-radius: 4px; border: none; }
  #gameCanvas { background: #333; display: block; margin: 10px auto; border-radius: 8px; width: 100%; height: auto; max-width: 640px; }
  #chatBox { height: 150px; overflow-y: auto; background: #111; padding: 8px; border-radius: 5px; margin-bottom: 5px; }
  #messageInput { width: calc(100% - 80px); display: inline-block; }
  #sendMsgBtn { width: 70px; display: inline-block; }
  #friendList { background: #222; padding: 10px; margin: 10px 0; border-radius: 6px; max-height: 100px; overflow-y: auto; }
  .friendItem { margin: 3px 0; display: flex; justify-content: space-between; align-items: center; }
  .friendItem button { font-size: 0.8em; padding: 2px 6px; }
  #shop, #skinShop { background: #222; padding: 10px; margin: 10px 0; border-radius: 6px; max-height: 180px; overflow-y: auto; }
  .item { background: #444; margin: 5px 0; padding: 5px; border-radius: 5px; display: flex; justify-content: space-between; align-items: center; }
  .item button { padding: 3px 8px; }
  #status { margin-top: 10px; font-weight: bold; }
  .hidden { display: none; }
  #modeSelect { width: 200px; }

  /* Controlli touch */
  #touchControls {
    max-width: 640px;
    margin: 10px auto;
    display: flex;
    justify-content: space-between;
    user-select: none;
  }
  #touchMovement {
    width: 150px;
    height: 150px;
    position: relative;
    background: #222;
    border-radius: 10px;
    display: grid;
    grid-template-columns: 50px 50px 50px;
    grid-template-rows: 50px 50px 50px;
    gap: 5px;
  }
  .touch-btn {
    background: #555;
    border-radius: 8px;
    text-align: center;
    line-height: 50px;
    font-weight: bold;
    font-size: 20px;
    color: #eee;
    user-select: none;
  }
  .touch-btn:active {
    background: #0af;
    color: white;
  }
  #attackBtn {
    width: 100px;
    height: 100px;
    background: #a33;
    border-radius: 50%;
    font-size: 24px;
    line-height: 100px;
    color: white;
    text-align: center;
    font-weight: bold;
    user-select: none;
  }
  #attackBtn:active {
    background: #d55;
  }

  @media (max-width: 500px) {
    #container { padding: 10px; }
    #touchControls { flex-direction: column; align-items: center; gap: 15px; }
  }
</style>
</head>
<body>
<div id="container">
  <h1>Dominio Supremo</h1>
  <label for="modeSelect">Scegli modalità:</label>
  <select id="modeSelect">
    <option value="combat">Combattimento</option>
    <option value="stargame">Sala Giochi - Raccolta Stelle</option>
  </select>
  <button id="startGameBtn">Avvia Partita</button>
  <canvas id="gameCanvas" width="640" height="360"></canvas>

  <div id="touchControls" class="hidden">
    <div id="touchMovement">
      <div></div><div class="touch-btn" id="upBtn">↑</div><div></div>
      <div class="touch-btn" id="leftBtn">←</div><div></div><div class="touch-btn" id="rightBtn">→</div>
      <div></div><div class="touch-btn" id="downBtn">↓</div><div></div>
    </div>
    <div id="attackBtn">ATTACCA</div>
  </div>

  <div id="status"></div>

  <h3>Negozio Casse (1€ a cassa - simulato)</h3>
  <div id="shop"></div>

  <h3>Negozio Skin (10€ a skin - simulato)</h3>
  <div id="skinShop"></div>

  <h3>Amici</h3>
  <input type="text" id="friendNameInput" placeholder="Nome amico" />
  <button id="addFriendBtn">Aggiungi amico</button>
  <div id="friendList"></div>

  <h3>Chat Lobby</h3>
  <div id="chatBox"></div>
  <input type="text" id="messageInput" placeholder="Scrivi messaggio..." />
  <button id="sendMsgBtn">Invia</button>

  <button id="resetBtn" style="margin-top:20px; background:#a33; color:#fff;">Cancella Tutti i Dati</button>
</div>

<script>
(() => {
  // --- Setup personaggi e skin ---
  const personaggi = [];
  const NUM_PERSONAGGI = 64;
  for (let i = 1; i <= NUM_PERSONAGGI; i++) {
    personaggi.push({
      id: i,
      nome: `Personaggio ${i}`,
      rarita: ['Comune', 'Raro', 'Epico', 'Leggendario'][Math.floor(Math.random()*4)],
      skin: [
        {id: 1, nome: 'Skin Base', prezzo: 0},
        {id: 2, nome: 'Skin Alternativa 1', prezzo: 10},
        {id: 3, nome: 'Skin Alternativa 2', prezzo: 10},
      ],
      sbloccato: i === 1,
    });
  }

  // --- Stato globale giocatore ---
  let stato = {
    coppe: 0,
    livello: 1,
    casse: 0,
    personaggiSbloccati: [personaggi[0].id],
    skinAcquistate: [],
    amici: [],
    chatLog: [],
    sceltaPersonaggio: personaggi[0].id,
    sceltaSkin: "1-1",
  };

  // --- Salvataggio locale ---
  function salvaStato() {
    localStorage.setItem('DominioSupremoSalvataggio', JSON.stringify(stato));
  }
  function caricaStato() {
    const dati = localStorage.getItem('DominioSupremoSalvataggio');
    if (dati) {
      stato = JSON.parse(dati);
    }
  }
  function resettaDati() {
    if (confirm("Sei sicuro di voler cancellare tutti i dati?")) {
      localStorage.removeItem('DominioSupremoSalvataggio');
      stato = {
        coppe: 0,
        livello: 1,
        casse: 0,
        personaggiSbloccati: [personaggi[0].id],
        skinAcquistate: [],
        amici: [],
        chatLog: [],
        sceltaPersonaggio: personaggi[0].id,
        sceltaSkin: "1-1",
      };
      aggiornaUI();
      alert("Dati cancellati!");
    }
  }

  // --- Aggiorna livello in base a coppe ---
  function aggiornaLivello() {
    stato.livello = Math.min(500, Math.floor(stato.coppe / 1000) + 1);
  }

  // --- UI Shop casse ---
  const shopDiv = document.getElementById('shop');
  function aggiornaShopCasse() {
    shopDiv.innerHTML = '';
    const btn = document.createElement('button');
    btn.textContent = 'Acquista Cassa - 1€';
    btn.onclick = () => {
      stato.casse++;
      aggiornaUI();
      alert('Hai acquistato una cassa!');
      salvaStato();
    };
    shopDiv.appendChild(btn);
    shopDiv.appendChild(document.createElement('br'));
    shopDiv.appendChild(document.createTextNode(`Casse disponibili: ${stato.casse}`));
  }

  // --- UI Shop Skin ---
  const skinShopDiv = document.getElementById('skinShop');
  function aggiornaShopSkin() {
    skinShopDiv.innerHTML = '';
    personaggi.forEach(p => {
      p.skin.forEach(s => {
        if (s.prezzo > 0 && !stato.skinAcquistate.includes(`${p.id}-${s.id}`)) {
          const div = document.createElement('div');
          div.className = 'item';
          div.textContent = `${p.nome} - ${s.nome} - Prezzo: ${s.prezzo}€`;
          const btn = document.createElement('button');
          btn.textContent = 'Acquista';
          btn.onclick = () => {
            stato.skinAcquistate.push(`${p.id}-${s.id}`);
            aggiornaUI();
            alert(`Hai acquistato la skin "${s.nome}" per ${p.nome}!`);
            salvaStato();
          };
          div.appendChild(btn);
          skinShopDiv.appendChild(div);
        }
      });
    });
  }

  // --- Amici ---
  const friendListDiv = document.getElementById('friendList');
  function aggiornaListaAmici() {
    friendListDiv.innerHTML = '';
    stato.amici.forEach((amico, idx) => {
      const div = document.createElement('div');
      div.className = 'friendItem';
      div.textContent = amico;
      const btn = document.createElement('button');
      btn.textContent = 'Rimuovi';
      btn.onclick = () => {
        stato.amici.splice(idx,1);
        aggiornaUI();
        salvaStato();
      };
      div.appendChild(btn);
      friendListDiv.appendChild(div);
    });
  }

  // --- Chat ---
  const chatBox = document.getElementById('chatBox');
  const messageInput = document.getElementById('messageInput');
  function aggiornaChat() {
    chatBox.innerHTML = '';
    stato.chatLog.forEach(msg => {
      const p = document.createElement('p');
      p.textContent = msg;
      chatBox.appendChild(p);
    });
    chatBox.scrollTop = chatBox.scrollHeight;
  }
  function inviaMessaggio() {
    const msg = messageInput.value.trim();
    if(msg.length === 0) return;
    stato.chatLog.push(msg);
    if(stato.chatLog.length > 100) stato.chatLog.shift(); // max 100 messaggi
    aggiornaChat();
    messageInput.value = '';
    salvaStato();
  }

  // --- Logica di gioco semplificata ---
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  let gameRunning = false;
  let mode = 'combat';

  function aggiornaUI() {
    aggiornaLivello();
    aggiorna
