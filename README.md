<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mein Trainingsplan A/B</title>
    <style>
        body { font-family: sans-serif; background: #121212; color: white; padding: 20px; display: flex; flex-direction: column; align-items: center; }
        .container { max-width: 500px; width: 100%; }
        h1, h2 { color: #00adb5; text-align: center; }
        .plan-card { background: #1e1e1e; padding: 15px; border-radius: 10px; margin-bottom: 20px; border-left: 5px solid #00adb5; }
        .exercise { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; padding-bottom: 5px; border-bottom: 1px solid #333; }
        input { width: 50px; background: #333; border: 1px solid #444; color: white; padding: 5px; border-radius: 4px; text-align: center; }
        button { cursor: pointer; background: #00adb5; border: none; color: white; padding: 10px; border-radius: 5px; font-weight: bold; transition: 0.3s; }
        button:hover { background: #007a82; }
        .timer-box { position: sticky; top: 10px; background: #222; padding: 15px; border-radius: 10px; display: flex; flex-direction: column; align-items: center; box-shadow: 0 4px 10px rgba(0,0,0,0.5); z-index: 100; margin-bottom: 20px; }
        .timer-display { font-size: 2rem; font-weight: bold; margin-bottom: 10px; color: #ffde7d; }
        .timer-btns { display: flex; gap: 10px; }
    </style>
</head>
<body>

<div class="container">
    <h1>Fitness Tracker</h1>

    <div class="timer-box">
        <div id="timer-display" class="timer-display">00:00</div>
        <div class="timer-btns">
            <button onclick="startTimer(30)">30s</button>
            <button onclick="startTimer(60)">60s</button>
            <button onclick="startTimer(90)">90s</button>
            <button style="background:#ff4b2b" onclick="stopTimer()">Stop</button>
        </div>
    </div>

    <div style="text-align: center; margin-bottom: 20px;">
        <button id="toggle-btn" onclick="togglePlan()">Zu Plan B wechseln</button>
    </div>

    <h2 id="current-plan-title">Training A (Kraft)</h2>
    <div id="exercise-list"></div>
    
    <button style="width: 100%; margin-top: 20px; background: #393e46;" onclick="saveProgress()">Fortschritt Speichern</button>
</div>

<script>
    let currentPlan = 'A';
    const exercises = {
        'A': ["Kniebeugen", "Klimmzüge", "Bankdrücken LH", "LH-Rudern", "Dips (Ringe)", "Toes to Bar"],
        'B': ["Kreuzheben", "Schulterdrücken", "Australian Pullups", "Ausfallschritte", "Liegestütze Ringe", "Bizeps Curls", "Scheibenwischer"]
    };

    function renderExercises() {
        const list = document.getElementById('exercise-list');
        list.innerHTML = '';
        document.getElementById('current-plan-title').innerText = `Training ${currentPlan}`;
        
        exercises[currentPlan].forEach(ex => {
            const savedData = JSON.parse(localStorage.getItem(ex)) || { kg: '', reps: '' };
            const div = document.createElement('div');
            div.className = 'exercise';
            div.innerHTML = `
                <span>${ex}</span>
                <div>
                    <input type="number" placeholder="kg" id="kg-${ex}" value="${savedData.kg}"> kg
                    <input type="number" placeholder="x" id="reps-${ex}" value="${savedData.reps}"> reps
                </div>
            `;
            list.appendChild(div);
        });
    }

    function togglePlan() {
        currentPlan = currentPlan === 'A' ? 'B' : 'A';
        document.getElementById('toggle-btn').innerText = `Zu Plan ${currentPlan === 'A' ? 'B' : 'A'} wechseln`;
        renderExercises();
    }

    function saveProgress() {
        exercises[currentPlan].forEach(ex => {
            const kg = document.getElementById(`kg-${ex}`).value;
            const reps = document.getElementById(`reps-${ex}`).value;
            localStorage.setItem(ex, JSON.stringify({ kg, reps }));
        });
        alert('Fortschritt für Plan ' + currentPlan + ' gespeichert!');
    }

    // TIMER LOGIK
    let timerInterval;
    function startTimer(seconds) {
        clearInterval(timerInterval);
        let timeLeft = seconds;
        const display = document.getElementById('timer-display');
        
        timerInterval = setInterval(() => {
            const mins = Math.floor(timeLeft / 60);
            const secs = timeLeft % 60;
            display.innerText = `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
            if (timeLeft <= 0) {
                clearInterval(timerInterval);
                display.innerText = "FERTIG!";
                const audio = new Audio('https://actions.google.com/sounds/v1/alarms/beep_short.ogg');
                audio.play();
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

