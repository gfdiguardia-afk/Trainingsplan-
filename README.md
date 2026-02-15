<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ultimate Training Tracker</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root { --main: #00adb5; --bg: #121212; --card: #1e1e1e; --text: #eeeeee; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: var(--text); padding: 10px; display: flex; flex-direction: column; align-items: center; }
        .container { max-width: 600px; width: 100%; }
        h1, h2 { color: var(--main); text-align: center; }
        .card { background: var(--card); padding: 15px; border-radius: 12px; margin-bottom: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.5); }
        .exercise-block { margin-bottom: 20px; border-bottom: 2px solid #333; padding-bottom: 10px; }
        .ex-title { font-weight: bold; font-size: 1.2rem; cursor: pointer; color: var(--main); text-decoration: underline; display: inline-block; margin-bottom: 8px; }
        .set-row { display: flex; align-items: center; gap: 10px; margin-bottom: 5px; }
        .set-label { width: 60px; font-size: 0.9rem; color: #888; }
        input { background: #333; border: 1px solid #444; color: white; padding: 8px; border-radius: 5px; width: 60px; text-align: center; font-size: 1rem; }
        button { cursor: pointer; background: var(--main); border: none; color: white; padding: 10px 15px; border-radius: 6px; font-weight: bold; }
        .timer-box { position: sticky; top: 5px; z-index: 100; text-align: center; background: #222; border: 2px solid var(--main); }
        .timer-display { font-size: 2rem; font-weight: bold; color: #ffde7d; }
        .nav-btns { display: flex; gap: 5px; justify-content: center; margin-bottom: 15px; }
        
        /* Modal für Charts */
        #chart-modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.9); z-index: 1000; overflow-y: auto; padding-top: 20px; }
        .modal-content { background: #222; margin: auto; padding: 20px; width: 90%; max-width: 500px; border-radius: 15px; }
        canvas { margin-bottom: 30px; }
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

    <div id="view-edit" style="display:none;">
        <h2>Editor</h2>
        <div class="card">
            <select id="edit-plan-select" onchange="renderEditor()" style="width:100%; padding:10px; background:#333; color:white; border-radius:5px;">
                <option value="A">Plan A</option>
                <option value="B">Plan B</option>
            </select>
            <div id="editor-list" style="margin-top:10px;"></div>
            <input type="text" id="new-ex-name" placeholder="Name..." style="width:60%; margin-top:10px;">
            <button onclick="addExercise()">+</button>
        </div>
    </div>
</div>

<div id="chart-modal">
    <div class="modal-content">
        <h2 id="modal-title">Übung</h2>
        <canvas id="weightChart"></canvas>
        <canvas id="repsChart"></canvas>
        <button onclick="closeModal()" style="width:100%; background:#ff4b2b">Schließen</button>
    </div>
</div>

<script>
    let audioCtx = null;
    function initAudio() { if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)(); }
    function beep(freq, duration, vol = 0.1) {
        initAudio(); const osc = audioCtx.createOscillator(); const gain = audioCtx.createGain();
        osc.connect(gain); gain.connect(audioCtx.destination);
        osc.frequency.value = freq; gain.gain.value = vol;
        osc.start(); setTimeout(() => osc.stop(), duration);
    }

    let currentPlan = 'A';
    let plans = JSON.parse(localStorage.getItem('myPlans')) || {
        'A': ["Kniebeugen", "Klimmzüge", "Bankdrücken LH", "LH-Rudern", "Dips (Ringe)"],
        'B': ["Kreuzheben", "Schulterdrücken", "Australian Pullups", "Ausfallschritte", "Liegestütze Ringe"]
    };

    function showView(view) {
        document.getElementById('view-train').style.display = view === 'train' ? 'block' : 'none';
        document.getElementById('view-edit').style.display = view === 'edit' ? 'block' : 'none';
        if(view === 'train') renderExercises(); else renderEditor();
    }

    function renderExercises() {
        const list = document.getElementById('exercise-list');
        list.innerHTML = '';
        document.getElementById('plan-title').innerText = `Training ${currentPlan}`;
        
        plans[currentPlan].forEach(ex => {
            const data = JSON.parse(localStorage.getItem('stats-'+ex)) || { s1:['',''], s2:['',''], s3:['',''] };
            let html = `<div class="exercise-block">
                <span class="ex-title" onclick="openStats('${ex}')">${ex}</span>`;
            for(let i=1; i<=3; i++) {
                html += `
                    <div class="set-row">
                        <span class="set-label">Satz ${i}</span>
                        <input type="number" id="kg-${ex}-${i}" value="${data[`s${i}`][0]}" oninput="syncWeight('${ex}', ${i})" placeholder="kg">
                        <input type="number" id="reps-${ex}-${i}" value="${data[`s${i}`][1]}" placeholder="reps">
                    </div>`;
            }
            list.innerHTML += html + `</div>`;
        });
    }

    // Smart Fill Funktion
    function syncWeight(ex, setNum) {
        if (setNum === 1) {
            const val = document.getElementById(`kg-${ex}-1`).value;
            document.getElementById(`kg-${ex}-2`).value = val;
            document.getElementById(`kg-${ex}-3`).value = val;
        }
    }

    function saveStats() {
        const date = new Date().toISOString().split('T')[0];
        plans[currentPlan].forEach(ex => {
            let exerciseData = { date: date };
            for(let i=1; i<=3; i++) {
                const kg = document.getElementById(`kg-${ex}-${i}`).value;
                const reps = document.getElementById(`reps-${ex}-${i}`).value;
                exerciseData[`s${i}`] = [kg, reps];
            }
            localStorage.setItem('stats-'+ex, JSON.stringify(exerciseData));
            
            // Historie für Charts speichern
            let history = JSON.parse(localStorage.getItem('hist-'+ex)) || [];
            // Nur speichern, wenn heute noch nicht gespeichert oder Update
            history = history.filter(h => h.date !== date);
            history.push(exerciseData);
            localStorage.setItem('hist-'+ex, JSON.stringify(history));
        });
        beep(800, 200); alert('Gespeichert!');
    }

    // CHART LOGIK
    let weightChart, repsChart;
    function openStats(ex) {
        document.getElementById('chart-modal').style.display = 'block';
        document.getElementById('modal-title').innerText = ex;
        const history = JSON.parse(localStorage.getItem('hist-'+ex)) || [];
        
        const labels = history.map(h => h.date);
        const avgWeights = history.map(h => {
            const sum = (parseFloat(h.s1[0])||0) + (parseFloat(h.s2[0])||0) + (parseFloat(h.s3[0])||0);
            return (sum / 3).toFixed(1);
        });
        const totalReps = history.map(h => (parseInt(h.s1[1])||0) + (parseInt(h.s2[1])||0) + (parseInt(h.s3[1])||0));

        if(weightChart) weightChart.destroy();
        if(repsChart) repsChart.destroy();

        const ctxW = document.getElementById('weightChart').getContext('2d');
        weightChart = new Chart(ctxW, {
            type: 'line',
            data: { labels, datasets: [{ label: 'Durchschnitt kg', data: avgWeights, borderColor: '#00adb5', tension: 0.1 }] },
            options: { plugins: { legend: { labels: { color: 'white' } } } }
        });

        const ctxR = document.getElementById('repsChart').getContext('2d');
        repsChart = new Chart(ctxR, {
            type: 'bar',
            data: { labels, datasets: [{ label: 'Total Wiederholungen', data: totalReps, backgroundColor: '#ffde7d' }] },
            options: { plugins: { legend: { labels: { color: 'white' } } } }
        });
    }

    function closeModal() { document.getElementById('chart-modal').style.display = 'none'; }

    // Timer & Editor (wie vorher)
    let timerInterval;
    function startTimer(seconds) {
        initAudio(); clearInterval(timerInterval); beep(600, 100);
        let timeLeft = seconds;
        const display = document.getElementById('timer-display');
        timerInterval = setInterval(() => {
            display.innerText = `00:${timeLeft.toString().padStart(2, '0')}`;
            if (timeLeft <= 3 && timeLeft > 0) beep(600, 150);
            if (timeLeft === 0) { clearInterval(timerInterval); display.innerText = "GO!"; beep(900, 800, 0.3); }
            timeLeft--;
        }, 1000);
    }
    function stopTimer() { clearInterval(timerInterval); document.getElementById('timer-display').innerText = "00:00"; }
    function togglePlan() { currentPlan = currentPlan === 'A' ? 'B' : 'A'; renderExercises(); }
    function renderEditor() {
        const p = document.getElementById('edit-plan-select').value;
        const list = document.getElementById('editor-list'); list.innerHTML = '';
        plans[p].forEach((ex, i) => { list.innerHTML += `<div style="display:flex; justify-content:space-between; margin-bottom:5px;"><span>${ex}</span><button onclick="deleteEx('${p}', ${i})">X</button></div>`; });
    }
    function addExercise() {
        const p = document.getElementById('edit-plan-select').value; const name = document.getElementById('new-ex-name').value;
        if(name) { plans[p].push(name); localStorage.setItem('myPlans', JSON.stringify(plans)); renderEditor(); }
    }
    function deleteEx(p, i) { plans[p].splice(i, 1); localStorage.setItem('myPlans', JSON.stringify(plans)); renderEditor(); }

    renderExercises();
</script>
</body>
</html>
