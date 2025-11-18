<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Agent Vocal - Salon de Coiffure</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 20px;
        }

        .container {
            background: white;
            padding: 40px;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            max-width: 500px;
            width: 100%;
            text-align: center;
        }

        h1 {
            color: #333;
            margin-bottom: 10px;
            font-size: 28px;
        }

        .subtitle {
            color: #666;
            margin-bottom: 30px;
            font-size: 14px;
        }

        .status {
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 20px;
            font-weight: 500;
            transition: all 0.3s ease;
        }

        .status.ready {
            background: #e8f5e9;
            color: #2e7d32;
        }

        .status.recording {
            background: #ffebee;
            color: #c62828;
            animation: pulse 1.5s infinite;
        }

        .status.processing {
            background: #fff3e0;
            color: #e65100;
        }

        .status.speaking {
            background: #e3f2fd;
            color: #1565c0;
        }

        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.7; }
        }

        .mic-button {
            width: 120px;
            height: 120px;
            border-radius: 50%;
            border: none;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            font-size: 48px;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 10px 30px rgba(102, 126, 234, 0.4);
            margin: 20px auto;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .mic-button:hover {
            transform: scale(1.05);
            box-shadow: 0 15px 40px rgba(102, 126, 234, 0.6);
        }

        .mic-button:active {
            transform: scale(0.95);
        }

        .mic-button.recording {
            background: linear-gradient(135deg, #f44336 0%, #e91e63 100%);
            animation: pulse 1.5s infinite;
        }

        .mic-button:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }

        .instructions {
            margin-top: 30px;
            padding: 20px;
            background: #f5f5f5;
            border-radius: 10px;
            text-align: left;
            font-size: 14px;
            color: #555;
        }

        .instructions h3 {
            color: #333;
            margin-bottom: 10px;
            font-size: 16px;
        }

        .instructions ol {
            margin-left: 20px;
        }

        .instructions li {
            margin: 8px 0;
        }

        .error {
            background: #ffebee;
            color: #c62828;
            padding: 15px;
            border-radius: 10px;
            margin-top: 20px;
            display: none;
        }

        .success-badge {
            background: #e8f5e9;
            color: #2e7d32;
            padding: 10px;
            border-radius: 8px;
            margin-bottom: 20px;
            font-size: 13px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎤 Agent Vocal</h1>
        <p class="subtitle">Salon de Coiffure - Assistant IA</p>

        <div class="success-badge">
            ✅ Connecté à votre webhook N8N
        </div>

        <div class="status ready" id="status">
            Prêt à vous écouter
        </div>

        <button class="mic-button" id="micButton" onclick="toggleRecording()">
            🎤
        </button>

        <div class="instructions">
            <h3>📋 Comment utiliser :</h3>
            <ol>
                <li>Cliquez sur le microphone</li>
                <li>Autorisez l'accès au micro si demandé</li>
                <li>Parlez (max 30 secondes)</li>
                <li>Cliquez à nouveau pour arrêter</li>
                <li>Écoutez la réponse de l'agent</li>
            </ol>
        </div>

        <div class="error" id="error"></div>
    </div>

    <script>
        let mediaRecorder;
        let audioChunks = [];
        let isRecording = false;
        let recordingTimeout;

        const statusEl = document.getElementById('status');
        const micButton = document.getElementById('micButton');
        const errorEl = document.getElementById('error');

        // ✅ URL de votre webhook N8N (déjà configurée)
        const WEBHOOK_URL = 'https://maxims-autoagents.app.n8n.cloud/webhook/voice-agent';

        function updateStatus(message, className) {
            statusEl.textContent = message;
            statusEl.className = `status ${className}`;
        }

        function showError(message) {
            errorEl.textContent = message;
            errorEl.style.display = 'block';
            setTimeout(() => {
                errorEl.style.display = 'none';
            }, 5000);
        }

        async function toggleRecording() {
            if (!isRecording) {
                await startRecording();
            } else {
                stopRecording();
            }
        }

        async function startRecording() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                
                mediaRecorder = new MediaRecorder(stream);
                audioChunks = [];

                mediaRecorder.ondataavailable = (event) => {
                    audioChunks.push(event.data);
                };

                mediaRecorder.onstop = async () => {
                    const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
                    await sendAudioToAgent(audioBlob);
                    
                    // Arrêter le stream
                    stream.getTracks().forEach(track => track.stop());
                };

                mediaRecorder.start();
                isRecording = true;
                
                micButton.classList.add('recording');
                micButton.textContent = '⏹️';
                updateStatus('🎙️ Enregistrement en cours... (Cliquez pour arrêter)', 'recording');

                // Arrêt automatique après 30 secondes
                recordingTimeout = setTimeout(() => {
                    if (isRecording) {
                        stopRecording();
                    }
                }, 30000);

            } catch (error) {
                console.error('Erreur microphone:', error);
                showError('❌ Impossible d\'accéder au microphone. Vérifiez les permissions.');
            }
        }

        function stopRecording() {
            if (mediaRecorder && isRecording) {
                clearTimeout(recordingTimeout);
                mediaRecorder.stop();
                isRecording = false;
                
                micButton.classList.remove('recording');
                micButton.textContent = '🎤';
                micButton.disabled = true;
                
                updateStatus('⏳ Traitement en cours...', 'processing');
            }
        }

        async function sendAudioToAgent(audioBlob) {
            try {
                const formData = new FormData();
                formData.append('audio', audioBlob, 'recording.wav');

                updateStatus('🤖 L\'agent réfléchit...', 'processing');

                const response = await fetch(WEBHOOK_URL, {
                    method: 'POST',
                    body: formData
                });

                if (!response.ok) {
                    throw new Error(`Erreur HTTP: ${response.status}`);
                }

                const audioResponseBlob = await response.blob();
                await playAudioResponse(audioResponseBlob);

            } catch (error) {
                console.error('Erreur:', error);
                showError('❌ Erreur de connexion avec l\'agent. Vérifiez que le workflow N8N est actif.');
                micButton.disabled = false;
                updateStatus('Prêt à vous écouter', 'ready');
            }
        }

        async function playAudioResponse(audioBlob) {
            try {
                const audioUrl = URL.createObjectURL(audioBlob);
                const audio = new Audio(audioUrl);

                updateStatus('🔊 L\'agent vous répond...', 'speaking');

                audio.onended = () => {
                    URL.revokeObjectURL(audioUrl);
                    micButton.disabled = false;
                    updateStatus('Prêt à vous écouter', 'ready');
                };

                audio.onerror = () => {
                    showError('❌ Erreur lors de la lecture de la réponse');
                    micButton.disabled = false;
                    updateStatus('Prêt à vous écouter', 'ready');
                };

                await audio.play();

            } catch (error) {
                console.error('Erreur lecture audio:', error);
                showError('❌ Impossible de lire la réponse audio');
                micButton.disabled = false;
                updateStatus('Prêt à vous écouter', 'ready');
            }
        }

        // Vérifier si le navigateur supporte l'API
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
            showError('❌ Votre navigateur ne supporte pas l\'enregistrement audio');
            micButton.disabled = true;
        }
    </script>
</body>
</html>
