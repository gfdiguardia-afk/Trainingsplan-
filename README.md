<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Profi Training Tracker - 3 Sätze</title>
    <style>
        :root { --main: #00adb5; --bg: #121212; --card: #1e1e1e; --text: #eeeeee; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: var(--text); padding: 10px; display: flex; flex-direction: column; align-items: center; }
        .container { max-width: 600px; width: 100%; }
        h1, h2 { color: var(--main); text-align: center; margin-bottom: 10px; }
        .card { background: var(--card); padding: 15px; border-radius: 12px; margin-bottom: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.5); }
        .exercise-block { margin-bottom: 20px; border-bottom: 2px solid #333; padding-bottom: 10px; }
        .ex-title { font-weight: bold; font-size: 1.1rem; margin-bottom: 8px; display: block; color: var(--main); }
        .set-row { display: flex; align-items: center; gap: 10px; margin-bottom: 5px; }
        .set-label { width: 60px; font-size: 0.9rem; color: #888; }
        input { background: #333; border: 1px solid #444; color: white; padding: 6px; border-radius: 5px; width: 55px; text-align: center; }
        button { cursor: pointer; background: var(--main); border: none; color: white; padding: 10px 15px; border-radius: 6px; font-weight: bold; }
        .timer-box { position: sticky; top: 5px; z-index: 100; text-align: center; background: #222; border: 2px solid var(--main); }
        .timer-display { font-size: 2rem; font-weight: bold; color: #ffde7d; }
        .nav-btns { display: flex; gap: 5px; justify-content: center; margin-bottom: 15px; }
        .edit-mode { display: none; }
        .delete-btn { background: #ff4b2b; padding: 5px; font-size: 0.7rem; }
    </style>
</head>
<body>

<div class="container">
    <div class="card timer-box">
        <div id="timer-display" class="timer-display">00:00</div>
        <div style="display:flex; gap:5px; justify-content: center;">
            <button onclick="startTimer(30)">30s</button>
            <button onclick="startTimer(60)">60s</button>
            <button onclick="startTimer(90)">90s</button>
            <button style="background:#555" onclick="stopTimer()">Rst</button>
        </div>
    </div>

    <div class="nav-btns">
        <button id="toggle-plan-btn" onclick="togglePlan()">Wechsel zu B</button>
        <button style="background:#393e46" onclick="showView('train')">Training</button>
        <button style="background:#393e46" onclick="showView('edit')">Editor</button>
    </div>

    <div id="view-train">
        <h2 id="plan-title">Training A</h2>
        <div id="exercise-list" class="card"></div>
        <button style="width:100%; height: 50px; background:#28a745" onclick="saveStats()">FORTSCHRITT SPEICHERN</button>
    </div>

    <div id="view-edit" class="edit-mode">
        <h2>Übungen anpassen</h2>
        <div class="card">
            <select id="edit-plan-select" onchange="renderEditor()" style="width:100%; padding:10px; margin-bottom:10px; background:#333; color:white;">
                <option value="A">Plan A bearbeiten</option>
                <option value="B">Plan B bearbeiten</option>
            </select>
            <div id="editor-list"></div>
            <hr>
            <input type="text" id="new-ex-name" placeholder="Neue Übung..." style="width: 60%;">
            <button onclick="addExercise()">Hinzufügen</button>
        </div>
    </div>
</div>

<script>
    // AUDIO ENGINE (Web Audio API)
    let audioCtx = null;

    function initAudio() {
        if (!audioCtx) {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        }
    }

    function beep(freq, duration, vol = 0.1) {
        initAudio();
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.frequency.value = freq;
        gain.gain.value = vol;
        osc.start();
        setTimeout(() => osc.stop(), duration);
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
            const data = JSON.parse(localStorage.getItem('stats-'+ex)) || { s1:['',''], s2:['',''], s3:['',''] };
            
            let html = `<div class="exercise-block">
                <span class="ex-title">${ex}</span>`;
            
            for(let i=1; i<=3; i++) {
                let valKg = data[`s${i}`][0];
                let valReps = data[`s${i}`][1];
                html += `
                    <div class="set-row">
                        <span class="set-label">Satz ${i}</span>
                        <input type="number" id="kg-${ex}-${i}" value="${valKg}" placeholder="kg">
                        <input type="number" id="reps-${ex}-${i}" value="${valReps}" placeholder="reps">
                    </div>`;
            }
            html += `</div>`;
            list.innerHTML += html;
        });
    }

    function saveStats() {
        plans[currentPlan].forEach(ex => {
            let exerciseData = {};
            for(let i=1; i<=3; i++) {
                const kg = document.getElementById(`kg-${ex}-${i}`).value;
                const reps = document.getElementById(`reps-${ex}-${i}`).value;
                exerciseData[`s${i}`] = [kg, reps];
            }
            localStorage.setItem('stats-'+ex, JSON.stringify(exerciseData));
        });
        beep(800, 200);
        alert('Alle Sätze gespeichert!');
    }

    function renderEditor() {
        const p = document.getElementById('edit-plan-select').value;
        const list = document.getElementById('editor-list');
        list.innerHTML = '';
        plans[p].forEach((ex, index) => {
            list.innerHTML += `<div style="display:flex; justify-content:space-between; margin-bottom:5px;">
                <span>${ex}</span>
                <button class="delete-btn" onclick="deleteEx('${p}', ${index})">Entfernen</button>
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

    // TIMER MIT NEUEN SOUNDS
    let timerInterval;
    function startTimer(seconds) {
        initAudio(); // Aktiviert Audio beim Klick
        clearInterval(timerInterval);
        beep(600, 100); 
        let timeLeft = seconds;
        const display = document.getElementById('timer-display');
        
        timerInterval = setInterval(() => {
            const mins = Math.floor(timeLeft / 60);
            const secs = timeLeft % 60;
            display.innerText = `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
            
            if (timeLeft <= 3 && timeLeft > 0) {
                beep(600, 150); // Pip Pip Pip
            } else if (timeLeft === 0) {
                clearInterval(timerInterval);
                display.innerText = "GO!";
                beep(900, 800, 0.3); // PIIIIIIEP
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

