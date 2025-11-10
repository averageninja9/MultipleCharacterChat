<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Character.AI-style Group Chat</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
  :root { --bg: #1e1e1e; --msg: #2d2d2d; --user: #007bff; --bot: #333; --accent: #00ff9d; }
  body { font-family: system-ui, sans-serif; background: var(--bg); color: #eee; margin:0; padding:0; height:100vh; display:flex; flex-direction:column; }
  #header { padding: 10px; background: #111; border-bottom: 1px solid #444; text-align:center; }
  #characters { padding:10px; background:#111; overflow-x:auto; white-space:nowrap; border-bottom:1px solid #444; }
  .char { display:inline-block; text-align:center; margin:0 10px; width:80px; cursor:pointer; }
  .char img { width:60px; height:60px; border-radius:50%; border:3px solid #444; object-fit:cover; }
  .char small { display:block; font-size:10px; opacity:0.8; }
  .char.active { border-color:var(--accent); }
  #chat { flex:1; overflow-y:auto; padding:15px; display:flex; flex-direction:column; gap:12px; }
  .msg { max-width:80%; padding:10px 14px; border-radius:18px; line-height:1.4; }
  .msg.user { align-self:flex-end; background:var(--user); color:#fff; }
  .msg.bot { align-self:flex-start; background:var(--msg); border-bottom-left-radius:4px; }
  .msg .name { font-weight:bold; font-size:0.85em; opacity:0.8; display:block; margin-bottom:4px; }
  #inputbar { display:flex; padding:10px; background:#111; gap:8px; }
  #input { flex:1; padding:12px; border:none; border-radius:24px; background:#333; color:#fff; }
  #send { background:var(--accent); color:#000; border:none; width:44px; height:44px; border-radius:50%; font-size:20px; cursor:pointer; }
  button { background:var(--accent); color:#000; border:none; padding:8px 12px; border-radius:8px; cursor:pointer; }
  dialog { background:#222; color:#fff; border:1px solid #444; border-radius:12px; padding:20px; max-width:400px; }
  dialog input, dialog textarea, dialog select { width:100%; padding:8px; margin:8px 0; background:#333; border:1px solid #555; border-radius:6px; color:#fff; }
  .preview { width:100px; height:100px; border-radius:50%; object-fit:cover; margin:10px 0; border:3px solid var(--accent); }
</style>
</head>
<body>

<div id="header">
  <h1>Character Group Chat</h1>
  <button onclick="saveChat()">💾 Download Chat</button>
  <button onclick="document.getElementById('loadFile').click()">📂 Load Chat</button>
  <input type="file" id="loadFile" accept=".json" style="display:none">
  <button onclick="showEditDialog()">+ Add Character</button>
</div>

<div id="characters"></div>

<div id="chat"></div>

<div id="inputbar">
  <input id="input" placeholder="Type a message..." autocomplete="off">
  <button id="send">➤</button>
</div>

<dialog id="editDialog">
  <h2 id="dialogTitle">Add Character</h2>
  <input type="hidden" id="editIndex">
  <label>Name <input id="charName"></label>
  <label>Age <input id="charAge" placeholder="optional"></label>
  <label>Gender <input id="charGender" placeholder="optional"></label>
  <label>Species <input id="charSpecies" placeholder="optional"></label>
  <label>Description<br><textarea id="charDesc" rows="3" placeholder="optional"></textarea></label>
  <label>Photo (click or drop)
    <div id="dropArea" style="border:2px dashed #666; padding:20px; text-align:center; margin:10px 0;">
      <img id="photoPreview" class="preview" src="" style="display:none">
      <p>Drop image here or click</p>
    </div>
    <input type="file" id="photoInput" accept="image/*" style="display:none">
  </label>
  <div style="text-align:right; margin-top:20px;">
    <button onclick="saveCharacter()">Save</button>
    <button onclick="closeDialog()">Cancel</button>
    <button id="deleteBtn" style="background:#f44; display:none;" onclick="deleteCharacter()">Delete</button>
  </div>
</dialog>

<script>
let characters = []; // {name, desc, age, gender, species, photoUrl}
let messages = [];   // {sender:"user" or index, text}
let currentChar = -1; // -1 = all characters visible

loadFromStorage();

renderCharacters();
renderChat();

document.getElementById('send').onclick = sendMessage;
document.getElementById('input').onkeydown = e => { if(e.key==='Enter') sendMessage(); };
document.getElementById('loadFile').onchange = loadChat;
document.getElementById('dropArea').onclick = () => document.getElementById('photoInput').click();
document.getElementById('photoInput').onchange = e => handlePhoto(e.target.files[0]);
document.getElementById('dropArea').ondragover = e => {e.preventDefault(); e.target.style.borderColor='white';};
document.getElementById('dropArea').ondragleave = e => {e.target.style.borderColor='#666';};
document.getElementById('dropArea').ondrop = e => {e.preventDefault(); e.target.style.borderColor='#666'; if(e.dataTransfer.files[0]) handlePhoto(e.dataTransfer.files[0]);};

function handlePhoto(file) {
  if(!file || !file.type.startsWith('image/')) return;
  const reader = new FileReader();
  reader.onload = ev => {
    document.getElementById('photoPreview').src = ev.target.result;
    document.getElementById('photoPreview').style.display = 'block';
    document.querySelector('#dropArea p').style.display = 'none';
  };
  reader.readAsDataURL(file);
}

function showEditDialog(index = -1) {
  const dialog = document.getElementById('editDialog');
  document.getElementById('editIndex').value = index;
  document.getElementById('dialogTitle').textContent = index===-1 ? 'Add Character' : 'Edit Character';
  document.getElementById('deleteBtn').style.display = index===-1 ? 'none' : 'inline-block';

  if(index === -1) {
    dialog.querySelectorAll('input, textarea').forEach(el=>el.value='');
    document.getElementById('photoPreview').src = '';
    document.getElementById('photoPreview').style.display = 'none';
    document.querySelector('#dropArea p').style.display = 'block';
  } else {
    const c = characters[index];
    document.getElementById('charName').value = c.name || '';
    document.getElementById('charAge').value = c.age || '';
    document.getElementById('charGender').value = c.gender || '';
    document.getElementById('charSpecies').value = c.species || '';
    document.getElementById('charDesc').value = c.desc || '';
    if(c.photoUrl) {
      document.getElementById('photoPreview').src = c.photoUrl;
      document.getElementById('photoPreview').style.display = 'block';
      document.querySelector('#dropArea p').style.display = 'none';
    }
  }
  dialog.showModal();
}

function closeDialog() {
  document.getElementById('editDialog').close();
}

function saveCharacter() {
  const index = parseInt(document.getElementById('editIndex').value);
  const char = {
    name: document.getElementById('charName').value.trim() || 'Unnamed',
    age: document.getElementById('charAge').value.trim() || '',
    gender: document.getElementById('charGender').value.trim() || '',
    species: document.getElementById('charSpecies').value.trim() || '',
    desc: document.getElementById('charDesc').value.trim() || '',
    photoUrl: document.getElementById('photoPreview').src || ''
  };

  if(index === -1) characters.push(char);
  else characters[index] = char;

  saveToStorage();
  renderCharacters();
  closeDialog();
}

function deleteCharacter() {
  const index = parseInt(document.getElementById('editIndex').value);
  characters.splice(index,1);
  saveToStorage();
  renderCharacters();
  closeDialog();
}

function renderCharacters() {
  const cont = document.getElementById('characters');
  cont.innerHTML = `<div class="char active" onclick="selectChar(-1)">
    <img src="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><circle cx='50' cy='50' r='40' fill='%23444'/></svg>">
    <small>All</small>
  </div>`;

  characters.forEach((c,i) => {
    const div = document.createElement('div');
    div.className = 'char';
    div.onclick = () => selectChar(i);
    div.ondblclick = () => showEditDialog(i); // double-click to edit
    div.innerHTML = `
      <img src="${c.photoUrl || 'data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><circle cx=%2250%22 cy=%2250%22 r=%2240%22 fill=%23${i%2?'6a6':'468'}/></svg>'}" title="${c.name}\n${c.desc}">
      <div>${c.name}</div>
      <small>${c.age||''} ${c.species||''}</small>
    `;
    cont.appendChild(div);
  });
}

function selectChar(i) {
  currentChar = i;
  document.querySelectorAll('.char').forEach((el,idx) => el.classList.toggle('active', idx===i+1));
  renderChat();
}

function renderChat() {
  const chat = document.getElementById('chat');
  chat.innerHTML = '';
  messages.forEach(m => {
    if(currentChar !== -1 && m.sender !== 'user' && m.sender !== currentChar) return;
    const div = document.createElement('div');
    div.className = 'msg ' + (m.sender==='user'?'user':'bot');
    if(m.sender !== 'user') {
      const c = characters[m.sender];
      div.innerHTML = `<span class="name">${c.name}</span>${m.text}`;
    } else {
      div.textContent = m.text;
    }
    chat.appendChild(div);
  });
  chat.scrollTop = chat.scrollHeight;
}

function sendMessage() {
  const input = document.getElementById('input');
  const text = input.value.trim();
  if(!text) return;
  messages.push({sender:'user', text});
  input.value = '';

  // Simple reply from all characters (or just the selected one)
  setTimeout(() => {
    const responders = currentChar === -1 ? characters.map((_,i)=>i) : [currentChar];
    responders.forEach(i => {
      const c = characters[i];
      const reply = generateReply(c, text);
      messages.push({sender:i, text: reply});
    });
    saveToStorage();
    renderChat();
  }, 500);
}

function generateReply(char, userMsg) {
  // Very simple personality-based reply (you can replace with real AI later)
  const greetings = ['Hey!', 'Yo', 'Hi there', 'Hello~', 'Sup'];
  const endings = [' :3', ' ^^', '!', '...', ' 💕', ' 🔥'];
  const name = char.name || 'Someone';
  return `${name}: ${greetings[Math.floor(Math.random()*greetings.length)]} You said "${userMsg}". That's cool${endings[Math.floor(Math.random()*endings.length)]}`;
}

function saveToStorage() {
  localStorage.setItem('characters', JSON.stringify(characters));
  localStorage.setItem('messages', JSON.stringify(messages));
}

function loadFromStorage() {
  const c = localStorage.getItem('characters');
  const m = localStorage.getItem('messages');
  if(c) characters = JSON.parse(c);
  if(m) messages = JSON.parse(m);
}

function saveChat() {
  const data = {characters, messages, savedAt: new Date().toISOString()};
  const blob = new Blob([JSON.stringify(data, null, 2)], {type:'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `groupchat_${new Date().toISOString().slice(0,10)}.json`;
  a.click();
}

function loadChat(e) {
  const file = e.target.files[0];
  if(!file) return;
  const reader = new FileReader();
  reader.onload = ev => {
    try {
      const data = JSON.parse(ev.target.result);
      characters = data.characters || [];
      messages = data.messages || [];
      saveToStorage();
      renderCharacters();
      renderChat();
      alert('Chat loaded!');
    } catch(err) { alert('Invalid file'); }
  };
  reader.readAsText(file);
}
</script>
</body>
</html>
