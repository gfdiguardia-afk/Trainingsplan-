<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Profi Training Tracker</title>
    <style>
        :root { --main: #00adb5; --bg: #121212; --card: #1e1e1e; --text: #eeeeee; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: var(--text); padding: 20px; display: flex; flex-direction: column; align-items: center; }
        .container { max-width: 500px; width: 100%; }
        h1, h2 { color: var(--main); text-align: center; margin-bottom: 10px; }
        .card { background: var(--card); padding: 20px; border-radius: 12px; margin-bottom: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.5); }
        .exercise-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; padding-bottom: 8px; border-bottom: 1px solid #333; }
        input { background: #333; border: 1px solid #444; color: white; padding: 8px; border-radius: 5px; width: 60px; text-align: center; }
        input[type="text"] { width: 150px; text-align: left; }
        button { cursor: pointer; background: var(--main); border: none; color: white; padding: 10px 15px; border-radius: 6px; font-weight: bold; transition: 0.2s; }
        button:active { transform: scale(0.95); }
        .timer-box { position: sticky; top: 10px; z-index: 100; text-align: center; background: #222; border: 2px solid var(--main); }
        .timer-display { font-size: 2.5rem; font-weight: bold; color: #ffde7d; margin-bottom: 10px; }
        .nav-btns { display: flex; gap: 10px; justify-content: center; margin-bottom: 20px; }
        .edit-mode { display: none; }
        .delete-btn { background: #ff4b2b; padding: 5px 10px; font-size: 0.8rem; }
        .add-btn { background: #28a745; width: 100%; margin-top: 10px; }
    </style>
</head>
<body>

<div class="container">
    <div class="card timer-box">
        <div id="timer-display" class="timer-display">00:00</div>
        <div style="display:flex; gap:8px; justify-content: center;">
            <button onclick="startTimer(30)">30s</button>
            <button onclick="startTimer(60)">60s</button>
            <button onclick="startTimer(90)">90s</button>
            <button style="background:#555" onclick="stopTimer()">Reset</button>
        </div>
    </div>

    <div class="nav-btns">
        <button id="toggle-plan-btn" onclick="togglePlan()">Wechsel zu B</button>
        <button style="background:#393e46" onclick="showView('train')">Training</button>
        <button style="background:#393e46" onclick="showView('edit')">Einstellungen</button>
    </div>

    <div id="view-train">
        <h2 id="plan-title">Training A</h2>
        <div id="exercise-list" class="card"></div>
        <button style="width:100%; height: 50px;" onclick="saveStats()">FORTSCHRITT SPEICHERN</button>
    </div>

    <div id="view-edit" class="edit-mode">
        <h2>Plan Bearbeiten</h2>
        <div class="card">
            <select id="edit-plan-select" onchange="renderEditor()" style="width:100%; padding:10px; margin-bottom:10px; background:#333; color:white;">
                <option value="A">Plan A bearbeiten</option>
                <option value="B">Plan B bearbeiten</option>
            </select>
            <div id="editor-list"></div>
            <input type="text" id="new-ex-name" placeholder="Neue Übung Name" style="width: 70%;">
            <button class="add-btn" onclick="addExercise()">Übung hinzufügen</button>
        </div>
    </div>
</div>

<script>
    // Audio-Kontext für die Pieptöne
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    function beep(freq, duration, vol = 0.1) {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.connect(gain); gain.connect(audioCtx.destination);
        osc.frequency.value = freq; gain.gain.value = vol;
        osc.start(); setTimeout(() => osc.stop(), duration);
    }

    let currentPlan = 'A';
    let plans = JSON.parse(localStorage.getItem('myPlans')) || {
        'A': ["Kniebeugen", "Klimmzüge", "Bankdrücken LH", "LH-Rudern", "Dips (Ringe)", "Toes to Bar"],
        'B': ["Kreuzheben", "Schulterdrücken", "Australian Pullups", "Ausfallschritte", "Liegestütze Ringe", "Bizeps Curls", "Scheibenwischer"]
    };

    function showView(view) {
        document.getElementById('view-train').style.display = view === 'train' ? 'block' : 'none';
        document.getElementById('view-edit').style.display = view === 'edit' ? 'block' : 'none';
        if(view === 'train') renderExercises();
        else renderEditor();
    }

    function renderExercises() {
        const list = document.getElementById('exercise-list');
        list.innerHTML = '';
        document.getElementById('plan-title').innerText = `Training ${currentPlan}`;
        plans[currentPlan].forEach(ex => {
            const data = JSON.parse(localStorage.getItem('stats-'+ex)) || { kg: '', reps: '' };
            list.innerHTML += `
                <div class="exercise-row">
                    <span>${ex}</span>
                    <div>
                        <input type="number" id="kg-${ex}" value="${data.kg}" placeholder="kg">
                        <input type="number" id="reps-${ex}" value="${data.reps}" placeholder="x">
                    </div>
                </div>`;
        });
    }

    function renderEditor() {
        const p = document.getElementById('edit-plan-select').value;
        const list = document.getElementById('editor-list');
        list.innerHTML = '';
        plans[p].forEach((ex, index) => {
            list.innerHTML += `
                <div class="exercise-row">
                    <span>${ex}</span>
                    <button class="delete-btn" onclick="deleteEx('${p}', ${index})">Löschen</button>
                </div>`;
        });
    }

    function addExercise() {
        const p = document.getElementById('edit-plan-select').value;
        const name = document.getElementById('new-ex-name').value;
        if(name) {
            plans[p].push(name);
            localStorage.setItem('myPlans', JSON.stringify(plans));
            document.getElementById('new-ex-name').value = '';
            renderEditor();
        }
    }

    function deleteEx(p, index) {
        plans[p].splice(index, 1);
        localStorage.setItem('myPlans', JSON.stringify(plans));
        renderEditor();
    }

    function togglePlan() {
        currentPlan = currentPlan === 'A' ? 'B' : 'A';
        document.getElementById('toggle-plan-btn').innerText = `Wechsel zu ${currentPlan === 'A' ? 'B' : 'A'}`;
        renderExercises();
    }

    function saveStats() {
        plans[currentPlan].forEach(ex => {
            const kg = document.getElementById(`kg-${ex}`).value;
            const reps = document.getElementById(`reps-${ex}`).value;
            localStorage.setItem('stats-'+ex, JSON.stringify({ kg, reps }));
        });
        beep(800, 200);
        alert('Gespeichert!');
    }

    // TIMER LOGIK MIT SOUNDS
    let timerInterval;
    function startTimer(seconds) {
        clearInterval(timerInterval);
        beep(600, 100); // Klick-Piep
        let timeLeft = seconds;
        const display = document.getElementById('timer-display');
        
        timerInterval = setInterval(() => {
            const mins = Math.floor(timeLeft / 60);
            const secs = timeLeft % 60;
            display.innerText = `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
            
            if (timeLeft <= 3 && timeLeft > 0) {
                beep(600, 150); // Kurz-Piep für 3, 2, 1
            } else if (timeLeft === 0) {
                clearInterval(timerInterval);
                display.innerText = "GO!";
                beep(900, 800, 0.2); // Langer PIIIIEP bei Null
            }
            timeLeft--;
        }, 1000);
    }

    function stopTimer() {
        clearInterval(timerInterval);
        document.getElementById('timer-display').innerText = "00:00";
    }

    renderExercises();
</script>
</body>
</html>
