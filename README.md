        .eliminated { opacity: 0.3; text-decoration: line-through; border-color: gray; pointer-events: none; }
        .ready-text { color: yellow; font-size: 0.8em; }

        /* Tabla de Resultados */
        .feedback-table {
            margin-top: 10px;
            background: #111;
            border: 1px dashed var(--hacker-green);
            padding: 5px;
            height: 80px;
            overflow-y: auto;
            font-size: 0.9em;
            text-align: center;
        }
        .feedback-row {
            letter-spacing: 2px;
            margin-bottom: 3px;
        }

        /* Chat */
        #chat-box {
            height: 150px;
            overflow-y: scroll;
            border: 1px solid var(--hacker-green);
            text-align: left;
            padding: 10px;
            margin-top: 20px;
        }

        /* Tutorial Modal */
        #tutorial-modal {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 1000;
            max-width: 400px;
            width: 90%;
            border-color: cyan;
            box-shadow: 0 0 20px cyan;
        }
        #tutorial-overlay {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.8); z-index: 999;
        }
    </style>
</head>
<body>

<audio id="bg-music" loop></audio>

<div id="profile-badge">🏆 Trofeos: <span id="trophy-count">0</span></div>

<div class="container">
    <h1>> HACK THE CODE _</h1>

    <button onclick="openTutorial()" style="margin-bottom: 15px; border-color: cyan; color: cyan;">📖 ¿Cómo jugar?</button>

    <div id="menu" class="card">
        <input type="text" id="username" placeholder="Tu nombre de Hacker">
        <br>
        <button onclick="hostGame()">Crear Partida</button>
        <br><br>
        <input type="text" id="join-id" placeholder="ID de la sala">
        <button onclick="joinGame()">Unirse</button>
    </div>

    <div id="lobby" class="card hidden">
        <h3>SALA: <span id="display-id" style="color: white; user-select: all;"></span></h3>
        
        <div id="qr-container" style="margin: 15px 0;"></div>

        <div id="host-controls" class="hidden" style="margin: 15px 0; border: 1px dashed gray; padding: 10px;">
            <p>🎵 Host: Elegir Música (Opcional)</p>
            <input type="file" id="music-file" accept="audio/*" class="hidden">
            <button onclick="document.getElementById('music-file').click()">Subir MP3/WAV</button>
            <br><span id="music-name" style="font-size: 0.8em; color: gray;">Ninguna</span>
        </div>
        <p id="client-music-status" style="color: cyan; font-size: 0.8em;"></p>

        <p>Tu Código (6 números): 
            <input type="password" id="my-secret-code" maxlength="6" pattern="\d{6}" placeholder="123456" oninput="this.value = this.value.replace(/[^0-9]/g, '')">
        </p>
        <button id="ready-btn" onclick="toggleReady()">ESTOY LISTO</button>
        
        <div id="player-list" style="margin-top: 20px; text-align: left;"></div>
    </div>

    <div id="game-screen" class="hidden">
        <div class="card">
            <h4>TU CÓDIGO: <span id="show-my-code" style="color: white;"></span></h4>
        </div>
        <div id="rivals-list"></div>
    </div>

    <div id="victory-screen" class="card hidden" style="border-color: gold; box-shadow: 0 0 15px gold;">
        <h2 id="victory-text" style="color: gold;"></h2>
        <button id="btn-play-again" class="hidden" onclick="playAgain()">Seguir jugando</button>
        <button onclick="location.reload()">Salir</button>
    </div>

    <div id="chat-container" class="hidden" style="width: 100%;">
        <div id="chat-box"></div>
        <input type="text" id="chat-input" placeholder="Mensaje..." style="width: 75%;">
        <button onclick="sendChatMessage()">Enviar</button>
    </div>
</div>

<div id="tutorial-overlay" class="hidden"></div>
<div id="tutorial-modal" class="card hidden">
    <h3 style="color: cyan;">TUTORIAL (<span id="tut-step">1</span>/8)</h3>
    <p id="tut-text" style="min-height: 60px;">Bienvenido a Hack the Code, el juego donde tu mente es tu mejor arma.</p>
    <button onclick="nextTutorial()">Siguiente</button>
    <button onclick="closeTutorial()" style="border-color: red; color: red;">Cerrar</button>
</div>

<script>
    // --- MOTOR SFX (WEB AUDIO API) ---
    let audioCtx;
    function initAudio() {
        if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        if (audioCtx.state === 'suspended') audioCtx.resume();
    }
    function playTone(freq, type, duration, vol=0.1) {
        try {
            if (!audioCtx) return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.type = type;
            osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
            gain.gain.setValueAtTime(vol, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + duration);
        } catch(e) {}
    }
    function sfxClick() { initAudio(); playTone(600, 'sine', 0.1, 0.05); }
    function sfxHack() { initAudio(); playTone(400, 'sawtooth', 0.15, 0.05); }
    function sfxHit() { initAudio(); playTone(1000, 'sine', 0.1, 0.1); setTimeout(()=>playTone(1500, 'sine', 0.2, 0.1), 100); }
    function sfxMiss() { initAudio(); playTone(200, 'sawtooth', 0.3, 0.1); }
    function sfxEliminated() { initAudio(); playTone(300, 'square', 0.5, 0.1); setTimeout(()=>playTone(200, 'square', 0.5, 0.1), 250); }
    function sfxWin() { initAudio(); [523, 659, 783, 1046].forEach((f, i) => setTimeout(() => playTone(f, 'sine', 0.3, 0.1), i*150)); }

    // Interceptar clics para sonido de botones
    document.addEventListener('click', (e) => { if(e.target.tagName === 'BUTTON') sfxClick(); });

    // --- LÓGICA DEL TUTORIAL ---
    const tutSteps = [
        "Bienvenido a Hack the Code, el juego donde tu mente es tu mejor arma.",
        "El objetivo es descubrir y hackear los códigos secretos de tus rivales antes de que ellos descubran el tuyo.",
        "Para empezar, crea una partida como 'Host' o únete con un 'ID de Sala'.",
        "Una vez en la sala, elige tu código secreto de 6 dígitos. ¡No se lo digas a nadie!",
        "Cuando todos estén listos, comenzará el juego. Verás a tus rivales en pantalla.",
        "Escribe un código de 6 dígitos en el panel de un rival y presiona 'HACKEAR'.",
        "Recibirás feedback: '✅' significa número correcto en la posición correcta. '❌' significa incorrecto.",
        "Usa la lógica para descifrar sus códigos. ¡El último hacker en pie gana!"
    ];
    let currentTutStep = 0;
    function openTutorial() {
        currentTutStep = 0;
        document.getElementById('tutorial-modal').classList.remove('hidden');
        document.getElementById('tutorial-overlay').classList.remove('hidden');
        updateTutorial();
    }
    function nextTutorial() {
        currentTutStep++;
        if(currentTutStep >= tutSteps.length) closeTutorial();
        else updateTutorial();
    }
    function updateTutorial() {
        document.getElementById('tut-step').innerText = currentTutStep + 1;
        document.getElementById('tut-text').innerText = tutSteps[currentTutStep];
    }
    function closeTutorial() {
        document.getElementById('tutorial-modal').classList.add('hidden');
        document.getElementById('tutorial-overlay').classList.add('hidden');
    }

    // --- VARIABLES DE JUEGO ---
    let peer;
    let conn; 
    let connections = []; 
    let myId;
    let isHost = false;
    let myData = { name: '', code: '', ready: false, eliminated: false };
    let players = [];
    let trophies = parseInt(localStorage.getItem('hackTheCode_trophies')) || 0;
    
    // Música compartida
    let sharedMusicData = null;
    let sharedMusicName = "";

    document.getElementById('trophy-count').innerText = trophies;

    // Lector de música (Solo Host)
    document.getElementById('music-file').addEventListener('change', function(e) {
        const file = e.target.files[0];
        if(!file) return;
        document.getElementById('music-name').innerText = file.name;
        sharedMusicName = file.name;
        const reader = new FileReader();
        reader.onload = function(evt) {
            sharedMusicData = evt.target.result;
            broadcast({ type: 'MUSIC', data: sharedMusicData, name: sharedMusicName });
        };
        reader.readAsDataURL(file);
    });

    // --- LÓGICA DE CONEXIÓN ---

    function hostGame() {
        const nameInput = document.getElementById('username').value.trim();
        if (!nameInput) return alert("Ingresa tu nombre primero.");
        
        myData.name = nameInput;
        isHost = true;
        initPeer();
        
        peer.on('open', (id) => {
            myId = id;
            document.getElementById('display-id').innerText = id;
            // Generar QR
            document.getElementById('qr-container').innerHTML = `<img src="https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=${id}&color=00ff41&bgcolor=000" alt="QR Code Sala">`;
            document.getElementById('host-controls').classList.remove('hidden');
            showLobby();
            addOrUpdatePlayer({ id: myId, name: myData.name, ready: false, eliminated: false });
        });

        peer.on('connection', (c) => {
            if (players.length >= 10) {
                c.send({ type: 'ERROR', msg: 'La sala está llena (Máximo 10 jugadores)' });
                setTimeout(() => c.close(), 500);
                return;
            }
            
            c.on('data', (data) => handleData(data, c));
            c.on('open', () => {
                connections.push(c);
                // Si el Host ya subió música, enviarla al nuevo jugador
                if (sharedMusicData) {
                    c.send({ type: 'MUSIC', data: sharedMusicData, name: sharedMusicName });
                }
            });
            c.on('close', () => {
                players = players.filter(p => p.id !== c.peer);
                connections = connections.filter(conn => conn.peer !== c.peer);
                broadcast({ type: 'PLAYER_LIST', list: players });
                updatePlayerListUI();
                checkAllReady();
            });
        });
    }

    function joinGame() {
        const nameInput = document.getElementById('username').value.trim();
        const hostId = document.getElementById('join-id').value.trim();
        
        if (!nameInput) return alert("Ingresa tu nombre primero.");
        if (!hostId) return alert("Introduce el ID de la sala.");
        
        myData.name = nameInput;
        initPeer();
        
        peer.on('open', (id) => {
            myId = id;
            conn = peer.connect(hostId);
            
            conn.on('open', () => {
                conn.send({ type: 'JOIN', id: myId, name: myData.name });
                document.getElementById('display-id').innerText = hostId;
                showLobby();
            });
            
            conn.on('data', (data) => handleData(data, conn));
            conn.on('close', () => alert("Conexión con el host perdida."));
            conn.on('error', () => alert("Error al conectar a la sala."));
        });
    }

    function initPeer() {
        peer = new Peer();
        peer.on('error', (err) => alert("Error de conexión: " + err.type));
    }

    function showLobby() {
        document.getElementById('menu').classList.add('hidden');
        document.getElementById('lobby').classList.remove('hidden');
    }

    // --- MANEJO DE DATOS ---

    function handleData(data, connection) {
        switch(data.type) {
            case 'ERROR':
                alert(data.msg);
                location.reload();
                break;
            case 'JOIN': 
                addOrUpdatePlayer({ id: data.id, name: data.name, ready: false, eliminated: false });
                broadcast({ type: 'PLAYER_LIST', list: players });
                break;
            case 'PLAYER_LIST': 
                players = data.list;
                updatePlayerListUI();
                break;
            case 'MUSIC':
                document.getElementById('bg-music').src = data.data;
                document.getElementById('client-music-status').innerText = "🎵 Canción del Host lista: " + data.name;
                break;
            case 'READY': 
                let p = players.find(player => player.id === data.id);
                if (p) {
                    p.ready = data.ready;
                    broadcast({ type: 'PLAYER_LIST', list: players });
                    updatePlayerListUI();
                    checkAllReady();
                }
                break;
            case 'START_GAME':
                players = data.list;
                runGameStart();
                break;
            case 'GUESS':
                // Aquí llega el ataque y verificamos el código
                if (data.to === myId) {
                    let resultArr = [];
                    let isCorrect = true;
                    for (let i = 0; i < 6; i++) {
                        if (data.val[i] === myData.code[i]) { resultArr.push('✅'); } 
                        else { resultArr.push('❌'); isCorrect = false; }
                    }

                    // Enviar respuesta al atacante
                    let feedbackData = { type: 'GUESS_RESULT', from: myId, to: data.from, guess: data.val, result: resultArr };
                    
                    if (isHost) {
                        let attackerConn = connections.find(c => c.peer === data.from);
                        if (attackerConn) attackerConn.send(feedbackData);
                    } else {
                        conn.send(feedbackData);
                    }

                    if (isCorrect && !myData.eliminated) {
                        sfxEliminated();
                        alert("¡CRÍTICO! Han descifrado tu código. Has sido eliminado.");
                        myData.eliminated = true;
                        if (isHost) {
                            markEliminated(myId);
                            broadcast({ type: 'ELIMINATED', playerId: myId });
                            checkWinCondition();
                        } else {
                            conn.send({ type: 'ELIMINATED', playerId: myId });
                        }
                    }
                } else if (isHost) { 
                    // El Host simplemente rutea la petición al jugador atacado
                    let targetConn = connections.find(c => c.peer === data.to);
                    if (targetConn) targetConn.send(data);
                }
                break;
            case 'GUESS_RESULT':
                // Recibir los resultados de mi propio ataque
                if (data.to === myId) {
                    handleFeedback(data);
                } else if (isHost) {
                    // El Host rutea el resultado de vuelta al atacante
                    let attackerConn = connections.find(c => c.peer === data.to);
                    if (attackerConn) attackerConn.send(data);
                }
                break;
            case 'ELIMINATED':
                markEliminated(data.playerId);
                checkWinCondition();
                break;
            case 'CHAT':
                displayChat(data.name, data.msg);
                break;
            case 'GAME_OVER':
                showVictory(data.winnerId, data.winnerName);
                break;
            case 'RESET':
                resetToLobby();
                break;
        }
    }

    // --- LÓGICA DE SALA Y LISTO ---

    function toggleReady() {
        const codeInput = document.getElementById('my-secret-code').value;
        if (codeInput.length !== 6) return alert("El código debe tener exactamente 6 números.");
        
        myData.code = codeInput;
        myData.ready = true;
        
        document.getElementById('ready-btn').innerText = "ESPERANDO A LOS DEMÁS...";
        document.getElementById('ready-btn').disabled = true;
        document.getElementById('my-secret-code').disabled = true;

        if (isHost) {
            let self = players.find(p => p.id === myId);
            self.ready = true;
            broadcast({ type: 'PLAYER_LIST', list: players });
            updatePlayerListUI();
            checkAllReady();
        } else {
            conn.send({ type: 'READY', id: myId, ready: true });
        }
    }

    function checkAllReady() {
        if (!isHost) return;
        if (players.length < 2) return; 
        
        const allReady = players.every(p => p.ready);
        if (allReady) {
            broadcast({ type: 'START_GAME', list: players });
            runGameStart();
        }
    }

    // --- LÓGICA DEL JUEGO ---

    function runGameStart() {
        initAudio(); // Inicializar contexto de audio al interactuar
        document.getElementById('lobby').classList.add('hidden');
        document.getElementById('game-screen').classList.remove('hidden');
        document.getElementById('chat-container').classList.remove('hidden');
        
        document.getElementById('show-my-code').innerText = myData.code;
        myData.eliminated = false;
        
        // Iniciar música si hay
        const bgAudio = document.getElementById('bg-music');
        if(bgAudio.src && bgAudio.src !== window.location.href) {
            bgAudio.play().catch(e => console.log("Audio prevent by browser", e));
        }

        renderRivals();
    }

    function renderRivals() {
        const list = document.getElementById('rivals-list');
        list.innerHTML = '';
        players.forEach(p => {
            if (p.id === myId) return;
            const div = document.createElement('div');
            div.id = `player-${p.id}`;
            div.className = 'rival-card card';
            div.innerHTML = `
                <p><strong>${p.name}</strong></p>
                <input type="text" id="guess-${p.id}" maxlength="6" pattern="\\d{6}" placeholder="CÓDIGO" oninput="this.value = this.value.replace(/[^0-9]/g, '')">
                <button onclick="sendGuess('${p.id}')">HACKEAR</button>
                <div id="feedback-${p.id}" class="feedback-table"></div> `;
            list.appendChild(div);
        });
    }

    function sendGuess(targetId) {
        sfxHack();
        if (myData.eliminated) return alert("Estás eliminado, no puedes hackear.");
        
        const guessInput = document.getElementById(`guess-${targetId}`);
        const guess = guessInput.value;
        
        if (guess.length !== 6) return alert("Debes intentar con 6 números.");

        if (isHost) {
            // Host envía su ataque al cliente objetivo usando network para que fluya correctamente
            let targetConn = connections.find(c => c.peer === targetId);
            if (targetConn) targetConn.send({ type: 'GUESS', from: myId, to: targetId, val: guess });
        } else {
            conn.send({ type: 'GUESS', from: myId, to: targetId, val: guess });
        }
        guessInput.value = '';
    }

    function handleFeedback(data) {
        // Reproducir efecto sonoro según el acierto
        if (data.result.includes('✅')) sfxHit();
        else sfxMiss();

        const feedbackDiv = document.getElementById(`feedback-${data.from}`);
        if (feedbackDiv) {
            let row = document.createElement('div');
            row.className = 'feedback-row';
            row.innerText = `${data.guess} ➔ ${data.result.join('')}`;
            feedbackDiv.appendChild(row);
            feedbackDiv.scrollTop = feedbackDiv.scrollHeight; 
        }
    }

    function markEliminated(id) {
        let p = players.find(player => player.id === id);
        if (p) p.eliminated = true;

        const el = document.getElementById(`player-${id}`);
        if (el) {
            el.classList.add('eliminated');
            el.querySelector('button').disabled = true;
            el.querySelector('input').disabled = true;
        }
        
        if (id === myId) {
            document.getElementById('show-my-code').innerText += " (ELIMINADO)";
            document.getElementById('show-my-code').style.color = "red";
        }
    }

    function checkWinCondition() {
        if (!isHost) return;

        const activePlayers = players.filter(p => !p.eliminated);
        if (activePlayers.length === 1) {
            const winner = activePlayers[0];
            showVictory(winner.id, winner.name);
            broadcast({ type: 'GAME_OVER', winnerId: winner.id, winnerName: winner.name });
        } else if (activePlayers.length === 0) {
            showVictory(null, "Nadie");
            broadcast({ type: 'GAME_OVER', winnerId: null, winnerName: "Nadie" });
        }
    }

    function showVictory(winnerId, winnerName) {
        document.getElementById('bg-music').pause(); // Detener música
        sfxWin(); // Sonido de victoria

        document.getElementById('game-screen').classList.add('hidden');
        document.getElementById('victory-screen').classList.remove('hidden');
        
        const vicText = document.getElementById('victory-text');
        
        if (winnerId === myId) {
            vicText.innerText = "¡ERES EL MEJOR HACKER! ¡HAS GANADO!";
            addTrophy();
        } else {
            vicText.innerText = `EL GANADOR ES: ${winnerName}`;
        }

        if (isHost) {
            document.getElementById('btn-play-again').classList.remove('hidden');
        }
    }

    function addTrophy() {
        trophies++;
        localStorage.setItem('hackTheCode_trophies', trophies);
        document.getElementById('trophy-count').innerText = trophies;
    }

    // --- REINICIAR PARTIDA (Solo Host) ---

    function playAgain() {
        if (!isHost) return;
        broadcast({ type: 'RESET' });
        resetToLobby();
    }

    function resetToLobby() {
        document.getElementById('victory-screen').classList.add('hidden');
        document.getElementById('game-screen').classList.add('hidden');
        document.getElementById('lobby').classList.remove('hidden');
        
        document.getElementById('my-secret-code').value = '';
        document.getElementById('my-secret-code').disabled = false;
        document.getElementById('ready-btn').innerText = "ESTOY LISTO";
        document.getElementById('ready-btn').disabled = false;
        document.getElementById('show-my-code').style.color = "white";
        document.getElementById('chat-box').innerHTML = '';

        myData.ready = false;
        myData.eliminated = false;
        players.forEach(p => {
            p.ready = false;
            p.eliminated = false;
        });

        updatePlayerListUI();
    }

    // --- FUNCIONES AUXILIARES ---

    function broadcast(data) {
        connections.forEach(c => c.send(data));
    }

    function updatePlayerListUI() {
        const list = document.getElementById('player-list');
        list.innerHTML = players.map(p => {
            let status = p.ready ? "<span class='ready-text'>[LISTO]</span>" : "[Esperando...]";
            return `<div>> ${p.name} ${status}</div>`;
        }).join('');
    }

    function addOrUpdatePlayer(newPlayer) {
        let existing = players.find(p => p.id === newPlayer.id);
        if (existing) {
            existing.name = newPlayer.name;
            existing.ready = newPlayer.ready;
        } else {
            players.push(newPlayer);
        }
        updatePlayerListUI();
    }

    function sendChatMessage() {
        const msg = document.getElementById('chat-input').value.trim();
        if (!msg) return;
        
        const data = { type: 'CHAT', name: myData.name, msg: msg };
        displayChat("Tú", msg);
        
        if (isHost) broadcast(data);
        else conn.send(data);
        
        document.getElementById('chat-input').value = '';
    }

    function displayChat(name, msg) {
        const box = document.getElementById('chat-box');
        box.innerHTML += `<div><strong>${name}:</strong> ${msg}</div>`;
        box.scrollTop = box.scrollHeight;
    }
</script>

</body>
</html>
