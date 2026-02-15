<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Training Pro Tracker v7</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root { --main: #00adb5; --bg: #121212; --card: #1e1e1e; --text: #eeeeee; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: var(--text); padding: 10px; display: flex; flex-direction: column; align-items: center; }
        .container { max-width: 600px; width: 100%; }
        h1, h2 { color: var(--main); text-align: center; margin-top: 5px; }
        .last-session { font-size: 0.9rem; color: #ffde7d; text-align: center; margin-bottom: 10px; font-weight: bold; }
        .card { background: var(--card); padding: 15px; border-radius: 12px; margin-bottom: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.5); }
        
        /* Trainings-Ansicht */
        .exercise-block { margin-bottom: 20px; border-bottom: 2px solid #333; padding-bottom: 10px; }
        .ex-title { font-weight: bold; font-size: 1.2rem; cursor: pointer; color: var(--main); text-decoration: underline; display: inline-block; margin-bottom: 10px; }
        .set-row { display: flex; align-items: center; justify-content: space-between; margin-bottom: 8px; }
        .set-label { font-size: 0.8rem; color: #888; width: 45px; }
        
        /* Spinner & Inputs */
        .input-group { display: flex; align-items: center; gap: 5px; background: #333; border-radius: 8px; padding: 2px; }
        .input-group button { background: #444; width: 40px; height: 40px; padding: 0; font-size: 1.2rem; border-radius: 5px; color: white; border: none; cursor: pointer; }
        .input-group input { background: transparent; border: none; width: 50px; color: white; text-align: center; font-size: 1rem; font-weight: bold; }
        input::-webkit-outer-spin-button, input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }

        button { cursor: pointer; background: var(--main); border: none; color: white; padding: 10px 15px; border-radius: 6px; font-weight: bold; }
        
        /* Editor-Reihenfolge */
        .edit-row { display: flex; justify-content: space-between; align-items: center; background: #2a2a2a; padding: 10px; border-radius: 8px; margin-bottom: 8px; }
        .order-btns { display: flex; gap: 4px; }
        .order-btns button { padding: 8px 12px; background: #444; font-size: 1.1rem; }

        /* Navigation & Timer */
        .timer-box { position: sticky; top: 5px; z-index: 100; text-align: center; background: #222; border: 2px solid var(--main); }
        .timer-display { font-size: 2.2rem; font-weight: bold; color: #ffde7d; }
        .nav-btns { display: flex; gap: 5px; justify-content: center; margin-bottom: 15px; }
        
        #chart-modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.95); z-index: 1000; padding: 20px 10px; }
        .modal-content { background: #222; margin: auto; padding: 20px; width: 95%; max-width: 500px; border-radius: 15px; }
    </style>
</head>
<body>

<div class="container">
    <div id="last-session-info" class="last-session">Lade Daten...</div>

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
        <button id="toggle-plan-btn" onclick="togglePlan()">Zu Plan B</button>
        <button style="background:#393e46" onclick="showView('train')">Training</button>
        <button style="background:#393e46" onclick="showView('edit')">Editor</button>
    </div>

    <div id="view-train">
        <h2 id="plan-title">Training A</h2>
        <div id="exercise-list" class="card"></div>
        <button style="width:100%; height: 60px; background:#28a745; font-size: 1.1rem;" onclick="saveStats()">FORTSCHRITT SPEICHERN</button>
    </div>

    <div id="view-edit" style="display:none;">
        <h2>Reihenfolge & Übungen</h2>
        <div class="card">
            <select id="edit-plan-select" onchange="renderEditor()" style="width:100%; padding:10px; background:#333; color:white; border-radius:5px; margin-bottom:15px; border:none;">
                <option value="A">Plan A bearbeiten</option>
                <option value="B">Plan B bearbeiten</option>
            </select>
            <div id="editor-list"></div>
            <hr style="border: 1px solid #333; margin: 15px 0;">
            <div style="display:flex; gap:5px;">
                <input type="text" id="new-ex-name" placeholder="Neue Übung..." style="flex:1; padding:12px; background:#333; color:white; border-radius:5px; border:none;">
                <button onclick="addExercise()" style="background:#28a745">+</button>
            </div>
        </div>
    </div>
</div>

<div id="chart-modal">
    <div class="modal-content">
        <h2 id="modal-title" style="margin-top:0;">Übung</h2>
        <canvas id="weightChart"></canvas>
        <canvas id="repsChart" style="margin-top:20px;"></canvas>
        <button onclick="closeModal()" style="width:100%; background:#ff4b2b; margin-top:20px;">Schließen</button>
    </div>
</div>

<script>
    // --- AUDIO ---
    const shortBeep = new Audio("https://actions.google.com/sounds/v1/alarms/beep_short.ogg");
    const longBeep = new Audio("https://actions.google.com/sounds/v1/alarms/alarm_clock_short_beep.ogg");

    function playSound(type) {
        if (type === 'short') { shortBeep.currentTime = 0; shortBeep.play().catch(e => {}); }
        else { longBeep.currentTime = 0; longBeep.play().catch(e => {}); }
    }

    // --- DATEN ---
    let currentPlan = 'A';
    let plans = JSON.parse(localStorage.getItem('myPlans')) || {
        'A': ["Kniebeugen", "Klimmzüge", "Bankdrücken LH", "LH-Rudern", "Dips (Ringe)"],
        'B': ["Kreuzheben", "Schulterdrücken", "Australian Pullups", "Ausfallschritte", "Liegestütze Ringe"]
    };

    function updateLastSessionDisplay() {
        const last = JSON.parse(localStorage.getItem('lastSessionLog'));
        const info = document.getElementById('last-session-info');
        info.innerText = last ? `Letztes Training: ${last.date} (Plan ${last.plan})` : "Noch kein Training gespeichert";
    }

    function showView(view) {
        document.getElementById('view-train').style.display = view === 'train' ? 'block' : 'none';
        document.getElementById('view-edit').style.display = view === 'edit' ? 'block' : 'none';
        if(view === 'train') renderExercises(); else renderEditor();
    }

    // --- SPINNER & SYNC ---
    function changeVal(id, delta) {
        const input = document.getElementById(id);
        let val = parseFloat(input.value) || 0;
        val = Math.max(0, val + delta);
        input.value = val;
        if(id.includes('kg') && id.endsWith('-1')) {
            const exName = id.replace('kg-', '').replace('-1', '');
            document.getElementById(`kg-${exName}-2`).value = val;
            document.getElementById(`kg-${exName}-3`).value = val;
        }
    }

    function renderExercises() {
        const list = document.getElementById('exercise-list');
        list.innerHTML = '';
        document.getElementById('plan-title').innerText = `Training ${currentPlan}`;
        plans[currentPlan].forEach(ex => {
            const data = JSON.parse(localStorage.getItem('stats-'+ex)) || { s1:['0','0'], s2:['0','0'], s3:['0','0'] };
            let html = `<div class="exercise-block"><span class="ex-title" onclick="openStats('${ex}')">${ex}</span>`;
            for(let i=1; i<=3; i++) {
                const sData = data[`s${i}`] || ['0','0'];
                html += `<div class="set-row"><span class="set-label">Satz ${i}</span>
                    <div class="input-group"><button onclick="changeVal('kg-${ex}-${i}', -2.5)">-</button><input type="number" id="kg-${ex}-${i}" value="${sData[0]}" step="0.5"><button onclick="changeVal('kg-${ex}-${i}', 2.5)">+</button></div>
                    <div class="input-group"><button onclick="changeVal('reps-${ex}-${i}', -1)">-</button><input type="number" id="reps-${ex}-${i}" value="${sData[1]}"><button onclick="changeVal('reps-${ex}-${i}', 1)">+</button></div></div>`;
            }
            list.innerHTML += html + `</div>`;
        });
    }

    // --- VERSCHIEBEN & EDITOR ---
    function renderEditor() {
        const p = document.getElementById('edit-plan-select').value;
        const list = document.getElementById('editor-list'); list.innerHTML = '';
        plans[p].forEach((ex, i) => {
            list.innerHTML += `
                <div class="edit-row">
                    <span style="flex:1;">${ex}</span>
                    <div class="order-btns">
                        <button onclick="moveEx('${p}', ${i}, -1)">↑</button>
                        <button onclick="moveEx('${p}', ${i}, 1)">↓</button>
                        <button style="background:#ff4b2b" onclick="deleteEx('${p}', ${i})">X</button>
                    </div>
                </div>`;
        });
    }

    function moveEx(planKey, index, dir) {
        const newIndex = index + dir;
        if (newIndex >= 0 && newIndex < plans[planKey].length) {
            const temp = plans[planKey][index];
            plans[planKey][index] = plans[planKey][newIndex];
            plans[planKey][newIndex] = temp;
            localStorage.setItem('myPlans', JSON.stringify(plans));
            renderEditor();
        }
    }

    function addExercise() {
        const p = document.getElementById('edit-plan-select').value;
        const name = document.getElementById('new-ex-name').value;
        if(name) { plans[p].push(name); localStorage.setItem('myPlans', JSON.stringify(plans)); document.getElementById('new-ex-name').value=''; renderEditor(); }
    }

    function deleteEx(p, i) { if(confirm('Übung löschen?')) { plans[p].splice(i, 1); localStorage.setItem('myPlans', JSON.stringify(plans)); renderEditor(); } }

    // --- SPEICHERN & TIMER ---
    function saveStats() {
        const now = new Date();
        const dateStr = now.toLocaleDateString('de-DE', { weekday: 'long', day: '2-digit', month: '2-digit', year: 'numeric' });
        plans[currentPlan].forEach(ex => {
            let exData = { date: dateStr, s1:[], s2:[], s3:[] };
            for(let i=1; i<=3; i++) { exData[`s${i}`] = [document.getElementById(`kg-${ex}-${i}`).value, document.getElementById(`reps-${ex}-${i}`).value]; }
            localStorage.setItem('stats-'+ex, JSON.stringify(exData));
            let hist = JSON.parse(localStorage.getItem('hist-'+ex)) || [];
            hist.push(exData); localStorage.setItem('hist-'+ex, JSON.stringify(hist));
        });
        localStorage.setItem('lastSessionLog', JSON.stringify({ date: dateStr, plan: currentPlan }));
        updateLastSessionDisplay(); playSound('short'); alert('Training gespeichert!');
    }

    let timerInterval;
    function startTimer(seconds) {
        clearInterval(timerInterval); playSound('short');
        let timeLeft = seconds;
        const display = document.getElementById('timer-display');
        timerInterval = setInterval(() => {
            let m = Math.floor(timeLeft / 60); let s = timeLeft % 60;
            display.innerText = `${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}`;
            if (timeLeft <= 3 && timeLeft > 0) playSound('short');
            if (timeLeft === 0) { clearInterval(timerInterval); display.innerText = "GO!"; playSound('long'); }
            timeLeft--;
        }, 1000);
    }
    function stopTimer() { clearInterval(timerInterval); document.getElementById('timer-display').innerText = "00:00"; }
    function togglePlan() { currentPlan = currentPlan === 'A' ? 'B' : 'A'; document.getElementById('toggle-plan-btn').innerText = `Zu Plan ${currentPlan==='A'?'B':'A'}`; renderExercises(); }

    // --- CHARTS ---
    let wChart, rChart;
    function openStats(ex) {
        document.getElementById('chart-modal').style.display = 'block';
        document.getElementById('modal-title').innerText = ex;
        const hist = JSON.parse(localStorage.getItem('hist-'+ex)) || [];
        const labels = hist.slice(-10).map(h => h.date.split(',')[1] || h.date);
        const weights = hist.slice(-10).map(h => ((parseFloat(h.s1[0])||0)+(parseFloat(h.s2[0])||0)+(parseFloat(h.s3[0])||0))/3);
        const reps = hist.slice(-10).map(h => (parseInt(h.s1[1])||0)+(parseInt(h.s2[1])||0)+(parseInt(h.s3[1])||0));
        if(wChart) wChart.destroy(); if(rChart) rChart.destroy();
        wChart = new Chart(document.getElementById('weightChart'), { type: 'line', data: { labels, datasets: [{ label: 'Ø kg', data: weights, borderColor: '#00adb5' }] }, options: { color: 'white' } });
        rChart = new Chart(document.getElementById('repsChart'), { type: 'bar', data: { labels, datasets: [{ label: 'Wdh Total', data: reps, backgroundColor: '#ffde7d' }] }, options: { color: 'white' } });
    }
    function closeModal() { document.getElementById('chart-modal').style.display = 'none'; }

    updateLastSessionDisplay();
    renderExercises();
</script>
</body>
</html>
