<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>My Character.AI - Fully Custom & Editable</title>
  <style>
    :root { --bg: #0d0d0d; --card: #1a1a1a; --accent: #00d4ff; --text: #f0f0f0; --gray: #333; }
    body { margin:0; font-family: system-ui, sans-serif; background:var(--bg); color:var(--text); display:flex; flex-direction:column; height:100vh; }
    header { padding:1.2rem; background:#000; text-align:center; font-size:1.6rem; color:var(--accent); font-weight:bold; }
    #sidebar { width:320px; background:var(--card); padding:1rem; overflow-y:auto; border-right:1px solid #333; }
    #characters { display:flex; flex-direction:column; gap:0.8rem; }
    .char-item { 
      background:var(--gray); border-radius:12px; padding:0.8rem; display:flex; align-items:center; gap:12px; 
      position:relative; transition:0.2s;
    }
    .char-item:hover { background:#444; }
    .char-item.active { background:var(--accent); color:black; font-weight:bold; }
    .char-avatar { width:48px; height:48px; border-radius:50%; object-fit:cover; border:3px solid var(--accent); background:#555; }
    .char-name { flex:1; font-size:1.1rem; }
    .edit-btn { background:transparent; border:none; cursor:pointer; font-size:1.3rem; padding:4px 8px; border-radius:6px; }
    .edit-btn:hover { background:rgba(255,255,255,0.2); }
    #main { flex:1; display:flex; flex-direction:column; }
    #chat { flex:1; padding:1.5rem; overflow-y:auto; display:flex; flex-direction:column; gap:1rem; }
    .message { max-width:78%; padding:1rem 1.2rem; border-radius:20px; line-height:1.5; word-wrap:break-word; }
    .user { align-self:flex-end; background:var(--accent); color:black; }
    .bot { align-self:flex-start; background:#333; }
    #input-area { padding:1rem; background:var(--card); display:flex; gap:0.8rem; border-top:1px solid #333; }
    input { flex:1; padding:1rem; border:none; border-radius:50px; background:#333; color:white; font-size:1rem; }
    button { padding:0 1.8rem; background:var(--accent); border:none; border-radius:50px; cursor:pointer; font-weight:bold; }
    dialog { background:var(--card); color:var(--text); border:none; border-radius:20px; padding:2rem; width:90%; max-width:540px; box-shadow:0 20px 60px rgba(0,0,0,0.8); }
    dialog::backdrop { background:rgba(0,0,0,0.92); }
    form { display:grid; gap:1.2rem; }
    label { display:flex; flex-direction:column; font-size:0.95rem; }
    input, textarea, select { padding:0.9rem; border-radius:12px; border:none; background:#333; color:white; margin-top:0.4rem; }
    .avatar-preview { width:130px; height:130px; border-radius:50%; object-fit:cover; border:5px solid var(--accent); margin:0 auto; display:block; background:#555; }
    .avatar-upload { text-align:center; margin:1.5rem 0; }
    .delete-photo { background:#ff3333; color:white; border:none; padding:0.5rem 1rem; border-radius:8px; cursor:pointer; margin-top:0.8rem; font-size:0.9rem; }
    .modal-buttons { display:flex; gap:1rem; justify-content:flex-end; margin-top:1rem; }
    .modal-buttons button { padding:0.8rem 1.5rem; border-radius:50px; font-weight:bold; }
    .cancel-btn { background:#555; }
    .save-btn { background:var(--accent); color:black; }
  </style>
</head>
<body>
  <header>My Character.AI – Unlimited & Fully Editable</header>
  
  <div style="display:flex; flex:1;">
    <aside id="sidebar">
      <button id="new-char" style="width:100%; padding:1.1rem; background:var(--accent); color:black; border:none; border-radius:12px; margin-bottom:1rem; cursor:pointer; font-size:1.2rem; font-weight:bold;">
        + New Character
      </button>
      <div id="characters"></div>
    </aside>

    <section id="main">
      <div id="chat"></div>
      <div id="input-area">
        <input type="text" id="user-input" placeholder="Say something..." autocomplete="off"/>
        <button id="send">Send</button>
      </div>
    </section>
  </div>

  <!-- Modal -->
  <dialog id="char-modal">
    <h2 id="modal-title">Create New Character</h2>
    <form id="char-form">
      <div class="avatar-upload">
        <img id="avatar-preview" class="avatar-preview" src="" alt="Avatar" />
        <input type="file" id="avatar-input" accept="image/*" hidden />
        <label for="avatar-input" style="cursor:pointer; color:var(--accent); margin-top:1rem; display:inline-block;">Change Photo</label>
        <p style="font-size:0.85rem; color:#aaa; margin:0.5rem 0;">or drag & drop</p>
        <button type="button" class="delete-photo" id="delete-photo" style="display:none;">Remove Photo</button>
      </div>

      <label>Name *<input type="text" name="name" placeholder="e.g. Luna" required /></label>
      <label>Description (personality, backstory...)<textarea name="description" rows="4" placeholder="Optional – leave blank for truly empty character"></textarea></label>
      <label>Age <input type="number" name="age" min="0" placeholder="Leave blank" /></label>
      <label>Gender 
        <select name="gender">
          <option value="">Not specified</option>
          <option>Male</option>
          <option>Female</option>
          <option>Non-binary</option>
          <option>Other</option>
        </select>
      </label>
      <label>Species <input type="text" name="species" placeholder="e.g. Elf, Robot, Cat" /></label>

      <div class="modal-buttons">
        <button type="button" class="cancel-btn" id="cancel-modal">Cancel</button>
        <button type="submit" class="save-btn">Save Character</button>
      </div>
    </form>
  </dialog>

  <script>
    // === Data ===
    const STORAGE_KEY = 'my-character-ai-v3';
    let characters = JSON.parse(localStorage.getItem(STORAGE_KEY)) || [];
    let activeChar = null;
    let messages = [];

    const chatEl = document.getElementById('chat');
    const charactersEl = document.getElementById('characters');
    const modal = document.getElementById('char-modal');
    const form = document.getElementById('char-form');
    const avatarPreview = document.getElementById('avatar-preview');
    const avatarInput = document.getElementById('avatar-input');
    const deletePhoto = document.getElementById('delete-photo');

    let currentAvatar = null;
    let editingIndex = null;

    // === Render Characters ===
    function renderCharacters() {
      charactersEl.innerHTML = '';
      characters.forEach((char, i) => {
        const div = document.createElement('div');
        div.className = `char-item ${activeChar?.id === char.id ? 'active' : ''}`;

        const img = document.createElement('img');
        img.className = 'char-avatar';
        img.src = char.avatar || 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iNDgiIGhlaWdodD0iNDgiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PHJlY3Qgd2lkdGg9IjEwMCUiIGhlaWdodD0iMTAwJSIgZmlsbD0iIzMzMyI+PC9yZWN0Pjx0ZXh0IHg9IjUwJSIgeT0iNTUlIiBmb250LXNpemU9IjI0IiBmaWxsPSIjOTk5IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkeT0iLjNlbSI+PC90ZXh0Pjwvc3ZnPg==';

        const name = document.createElement('div');
        name.className = 'char-name';
        name.textContent = char.name || 'Unnamed';

        const editBtn = document.createElement('button');
        editBtn.className = 'edit-btn';
        editBtn.textContent = 'Edit';
        editBtn.onclick = (e) => {
          e.stopPropagation();
          openEditModal(i);
        };

        div.appendChild(img);
        div.appendChild(name);
        div.appendChild(editBtn);
        div.onclick = () => switchCharacter(char);
        charactersEl.appendChild(div);
      });
    }

    // === Switch & Edit ===
    function switchCharacter(char) {
      activeChar = char;
      messages = JSON.parse(localStorage.getItem(`msgs-${char.id}`)) || [];
      renderMessages();
      renderCharacters();
    }

    function openEditModal(index) {
      editingIndex = index;
      const char = characters[index];

      modal.querySelector('#modal-title').textContent = 'Edit Character';
      form.name.value = char.name || '';
      form.description.value = char.description || '';
      form.age.value = char.age || '';
      form.gender.value = char.gender || '';
      form.species.value = char.species || '';

      currentAvatar = char.avatar;
      avatarPreview.src = char.avatar || 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMTMwIiBoZWlnaHQ9IjEzMCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSIjMzMzIj48L3JlY3Q+PHRleHQgeD0iNTAlIiB5PSI1NSUiIGZvbnQtc2l6ZT0iNTUiIGZpbGw9IiM5OTkiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGR5PSIuM2VtIj4+PC90ZXh0Pjwvc3ZnPg==';
      deletePhoto.style.display = char.avatar ? 'inline-block' : 'none';

      modal.showModal();
    }

    // === Avatar Handling ===
    avatarInput.onchange = () => {
      const file = avatarInput.files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = e => {
        currentAvatar = e.target.result;
        avatarPreview.src = currentAvatar;
        deletePhoto.style.display = 'inline-block';
      };
      reader.readAsDataURL(file);
    };

    deletePhoto.onclick = () => {
      currentAvatar = null;
      avatarPreview.src = 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMTMwIiBoZWlnaHQ9IjEzMCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSIjMzMzIj48L3JlY3Q+PHRleHQgeD0iNTAlIiB5PSI1NSUiIGZvbnQtc2l6ZT0iNTUiIGZpbGw9IiM5OTkiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGR5PSIuM2VtIj4+PC90ZXh0Pjwvc3ZnPg==';
      deletePhoto.style.display = 'none';
      avatarInput.value = '';
    };

    // Drag & drop
    modal.addEventListener('dragover', e => { e.preventDefault(); modal.style.transform = 'scale(1.02)'; });
    modal.addEventListener('dragleave', () => modal.style.transform = '');
    modal.addEventListener('drop', e => {
      e.preventDefault();
      modal.style.transform = '';
      const file = e.dataTransfer.files[0];
      if (file && file.type.startsWith('image/')) {
        avatarInput.files = e.dataTransfer.files;
        avatarInput.dispatchEvent(new Event('change'));
      }
    });

    // === Modal Controls ===
    document.getElementById('new-char').onclick = () => {
      editingIndex = null;
      modal.querySelector('#modal-title').textContent = 'Create New Character';
      form.reset();
      currentAvatar = null;
      avatarPreview.src = 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMTMwIiBoZWlnaHQ9IjEzMCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSIjMzMzIj48L3JlY3Q+PHRleHQgeD0iNTAlIiB5PSI1NSUiIGZvbnQtc2l6ZT0iNTUiIGZpbGw9IiM5OTkiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGR5PSIuM2VtIj4+PC90ZXh0Pjwvc3ZnPg==';
      deletePhoto.style.display = 'none';
      modal.showModal();
    };

    document.getElementById('cancel-modal').onclick = () => modal.close();

    // === Save Character ===
    form.onsubmit = e => {
      e.preventDefault();
      const fd = new FormData(form);
      const char = {
        id: editingIndex !== null ? characters[editingIndex].id : Date.now().toString(),
        name: fd.get('name').trim(),
        description: fd.get('description').trim(),
        age: fd.get('age') ? parseInt(fd.get('age')) : null,
        gender: fd.get('gender'),
        species: fd.get('species').trim() || null,
        avatar: currentAvatar
      };

      if (editingIndex !== null) {
        characters[editingIndex] = char;
      } else {
        characters.push(char);
      }

      localStorage.setItem(STORAGE_KEY, JSON.stringify(characters));
      renderCharacters();
      modal.close();

      if (!activeChar || activeChar.id === char.id) {
        switchCharacter(char);
      }
    };

    // === Chat ===
    function renderMessages() {
      chatEl.innerHTML = '';
      messages.forEach(m => {
        const div = document.createElement('div');
        div.className = `message ${m.role}`;
        div.textContent = m.content;
        chatEl.appendChild(div);
      });
      chatEl.scrollTop = chatEl.scrollHeight;
    }

    document.getElementById('send').onclick = sendMessage;
    document.getElementById('user-input').addEventListener('keydown', e => {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        sendMessage();
      }
    });

    async function sendMessage() {
      if (!activeChar) return alert('Select or create a character first!');
      const input = document.getElementById('user-input');
      const msg = input.value.trim();
      if (!msg) return;
      input.value = '';

      messages.push({ role: 'user', content: msg });
      renderMessages();

      const prompt = `You are ${activeChar.name}.
${activeChar.description ? 'Personality: ' + activeChar.description : ''}
${activeChar.age ? 'Age: ' + activeChar.age : ''}
${activeChar.gender ? 'Gender: ' + activeChar.gender : ''}
${activeChar.species ? 'Species: ' + activeChar.species : ''}
Reply in first person. Stay in character.`;

      try {
        const res = await fetch('https://api.x.ai/v1/chat/completions', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer YOUR_XAI_API_KEY_HERE'
          },
          body: JSON.stringify({
            model: 'grok-beta',
            messages: [
              { role: 'system', content: prompt },
              ...messages.slice(-25)
            ],
            temperature: 0.9,
            max_tokens: 700
          })
        });
        const data = await res.json();
        const reply = data.choices[0].message.content;
        messages.push({ role: 'bot', content: reply });
        renderMessages();
        localStorage.setItem(`msgs-${activeChar.id}`, JSON.stringify(messages));
      } catch (err) {
        chatEl.innerHTML += `<div class="message bot">Error: ${err.message}</div>`;
      }
    }

    // === Init ===
    renderCharacters();
    if (characters.length > 0) switchCharacter(characters[0]);
  </script>
</body>
</html>
