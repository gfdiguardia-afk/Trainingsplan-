// --- VERBESSERTE AUDIO LOGIK ---
let audioCtx = null;

function initAudio() {
    // Erstellt den Audio-Kontext erst bei einem echten Klick
    if (!audioCtx) {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    // Falls der Browser den Ton "schlafen" gelegt hat (Suspended), wecken wir ihn auf
    if (audioCtx.state === 'suspended') {
        audioCtx.resume();
    }
}

function beep(freq, duration, vol = 0.3) {
    initAudio(); // Stellt sicher, dass Audio aktiv ist
    
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    osc.frequency.value = freq;
    // Sanftes Ein- und Ausblenden verhindert Knackgeräusche
    gain.gain.setValueAtTime(0, audioCtx.currentTime);
    gain.gain.linearRampToValueAtTime(vol, audioCtx.currentTime + 0.01);
    gain.gain.linearRampToValueAtTime(0, audioCtx.currentTime + (duration / 1000));
    
    osc.start();
    osc.stop(audioCtx.currentTime + (duration / 1000));
}

// In der startTimer Funktion muss initAudio ganz oben stehen:
function startTimer(seconds) {
    initAudio(); // WICHTIG: Aktiviert den Sound-Kanal sofort beim Drücken der 30s/60s/90s Taste
    clearInterval(timerInterval);
    beep(500, 100); // Bestätigungs-Piep
    // ... restliche Timer Logik
}
