<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Nasłuchuj i Generuj Obraz</title>
    <script src="https://cdn.jsdelivr.net/pyodide/v0.23.0/full/pyodide.js"></script>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 30px; }
        button { padding: 10px 20px; margin: 20px; }
        #canvas { border: 1px solid black; margin-top: 20px; }
    </style>
</head>
<body>
    <h1>Nasłuchuj dźwięk i generuj obraz</h1>
    <button id="startButton" onclick="startListening()">Rozpocznij nasłuchiwanie</button>
    <button id="stopButton" onclick="stopListening()" disabled>Zatrzymaj nasłuchiwanie</button>
    <canvas id="canvas" width="200" height="200"></canvas>

    <script>
        let audioContext;
        let analyser;
        let microphone;
        let scriptProcessor;
        let audioChunks = [];
        let listening = false;

        // Ładowanie Pyodide
        let pyodideReady = (async () => {
            console.log("Ładowanie Pyodide...");
            let pyodide = await loadPyodide();
            await pyodide.loadPackage(['numpy', 'pillow']);
            console.log("Pyodide załadowany.");
            return pyodide;
        })();

        async function startListening() {
            document.getElementById('startButton').disabled = true;
            document.getElementById('stopButton').disabled = false;

            audioContext = new (window.AudioContext || window.webkitAudioContext)();
            analyser = audioContext.createAnalyser();
            scriptProcessor = audioContext.createScriptProcessor(4096, 1, 1);

            // Uzyskanie dostępu do mikrofonu
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            microphone = audioContext.createMediaStreamSource(stream);
            microphone.connect(analyser);
            analyser.connect(scriptProcessor);
            scriptProcessor.connect(audioContext.destination);

            audioChunks = [];
            listening = true;

            scriptProcessor.onaudioprocess = (event) => {
                if (!listening) return;
                const audioData = event.inputBuffer.getChannelData(0);
                audioChunks.push(new Float32Array(audioData));
            };
        }

        function stopListening() {
            document.getElementById('startButton').disabled = false;
            document.getElementById('stopButton').disabled = true;
            listening = false;
            
            scriptProcessor.disconnect();
            analyser.disconnect();
            microphone.disconnect();
            
            processAudioChunks();
        }

        async function processAudioChunks() {
            let pyodide = await pyodideReady;

            // Konwersja audioChunks na jeden duży Float32Array
            const audioData = Float32Array.from(audioChunks.flat());

            // Python skrypt do generowania obrazu
            const pythonCode = `
from io import BytesIO
import numpy as np
from PIL import Image

def generate_image_from_sound(audio_data):
    # Przyjmujemy audio_data jako tablicę NumPy
    audio_array = np.array(audio_data)
    
    # Normalizacja danych dźwiękowych
    audio_array = (audio_array - np.min(audio_array)) / (np.max(audio_array) - np.min(audio_array))
    
    # Przekształcanie na obrazek
    size = int(len(audio_array) ** 0.5)  # Ustal rozmiar obrazka (np. 200x200)
    image_array = (audio_array[:size*size] * 255).astype(np.uint8).reshape((size, size))

    # Konwertowanie do obrazu
    img = Image.fromarray(image_array)
    img_io = BytesIO()
    img.save(img_io, format='PNG')
    return img_io.getvalue()
`;

            // Uruchomienie skryptu Pythona
            pyodide.runPython(pythonCode);
            const generateImageFromSound = pyodide.globals.get('generate_image_from_sound');
            
            // Wywołanie funkcji z Pythonem
            const imageBytes = generateImageFromSound(audioData);
            
            // Wyświetlanie obrazu
            const blob = new Blob([imageBytes], { type: 'image/png' });
            const url = URL.createObjectURL(blob);
            
            const img = new Image();
            img.onload = function() {
                const canvas = document.getElementById('canvas');
                const context = canvas.getContext('2d');
                context.clearRect(0, 0, canvas.width, canvas.height);
                context.drawImage(img, 0, 0, canvas.width, canvas.height);
            };
            img.src = url;
        }
    </script>
</body>
</html>
