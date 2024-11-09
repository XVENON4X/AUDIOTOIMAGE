#12
<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Obraz na Dźwięk</title>
    <script src="https://cdn.jsdelivr.net/pyodide/v0.23.0/full/pyodide.js"></script>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
        input, button { margin: 10px; }
        audio { margin-top: 20px; }
    </style>
</head>
<body>
    <h1>Konwertuj Obraz na Dźwięk</h1>
    <input type="file" id="imageInput" accept="image/*">
    <button onclick="processImage()">Generuj Dźwięk</button>
    <br>
    <audio id="audioPlayer" controls></audio>
    
    <script>
        async function loadPyodideAndPackages() {
            console.log("Ładowanie Pyodide...");
            let pyodide = await loadPyodide();
            await pyodide.loadPackage(['numpy', 'pillow', 'soundfile']);
            console.log("Pyodide załadowany.");
            return pyodide;
        }
        
        let pyodideReady = loadPyodideAndPackages();

        async function processImage() {
            let fileInput = document.getElementById('imageInput');
            if (fileInput.files.length === 0) {
                alert("Proszę wybrać plik obrazu.");
                return;
            }
            
            const file = fileInput.files[0];
            const reader = new FileReader();
            
            reader.onload = async function(event) {
                let pyodide = await pyodideReady;
                
                // Python skrypt
                const pythonCode = `
from io import BytesIO
from PIL import Image
import numpy as np
import soundfile as sf

def generate_sound_from_image(image_bytes):
    image = Image.open(BytesIO(image_bytes))
    image = image.convert("RGB")
    image = image.resize((50, 50))  # Opcjonalne skalowanie
    
    # Konwersja obrazu na bity
    image_data = np.array(image)
    bit_stream = np.unpackbits(image_data.flatten())
    
    # Parametry dźwięku
    sample_rate = 44100
    duration = 0.005
    frequency_1 = 1000
    frequency_2 = 2000

    num_samples = int(sample_rate * duration)
    t = np.linspace(0, duration, num_samples, endpoint=False)
    tone_0 = np.sin(2 * np.pi * frequency_1 * t)
    tone_1 = np.sin(2 * np.pi * frequency_2 * t)

    audio_data = np.concatenate([tone_1 if bit else tone_0 for bit in bit_stream])
    audio_data = np.array(audio_data, dtype=np.float32)
    
    # Zapis dźwięku do pliku WAV w pamięci
    wav_io = BytesIO()
    sf.write(wav_io, audio_data, sample_rate, format='WAV')
    return wav_io.getvalue()
`;
                // Wczytanie obrazu jako bajtów
                const imageBytes = new Uint8Array(event.target.result);
                
                // Uruchomienie skryptu Pythona
                pyodide.runPython(pythonCode);
                let audioBytes = pyodide.globals.get('generate_sound_from_image')(imageBytes);
                
                // Utworzenie obiektu Blob i odtwarzacza audio
                const audioBlob = new Blob([audioBytes], { type: 'audio/wav' });
                const audioUrl = URL.createObjectURL(audioBlob);
                
                const audioPlayer = document.getElementById('audioPlayer');
                audioPlayer.src = audioUrl;
                audioPlayer.play();
                
                console.log("Dźwięk został wygenerowany i odtworzony.");
            };
            
            reader.readAsArrayBuffer(file);
        }
    </script>
</body>
</html>
