[# AUDIOTOIMAGE

<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Przesyłanie obrazu przez dźwięk</title>
</head>
<body>
    <h1>Przesyłanie obrazu przez dźwięk</h1>
    <button onclick="startRecording()">Rozpocznij nagrywanie</button>
    <canvas id="imageCanvas" width="300" height="300"></canvas>
    <script>
        let audioContext;
        let analyser;
        let microphone;
        let dataArray;
        let isRecording = false;
        // Funkcja rozpoczynająca nagrywanie dźwięku
        function startRecording() {
            if (isRecording) return;
            // Uzyskanie dostępu do mikrofonu
            navigator.mediaDevices.getUserMedia({ audio: true })
                .then((stream) => {
                    audioContext = new (window.AudioContext || window.webkitAudioContext)();
                    analyser = audioContext.createAnalyser();
                    analyser.fftSize = 2048;
                    microphone = audioContext.createMediaStreamSource(stream);
                    microphone.connect(analyser);
                    dataArray = new Uint8Array(analyser.frequencyBinCount);
                    isRecording = true;
                    processAudio();
                })
                .catch((error) => {
                    console.error("Błąd uzyskiwania dostępu do mikrofonu:", error);
                });
        }
        // Funkcja do analizy dźwięku i dekodowania go na bity
        function processAudio() {
            if (!isRecording) return;
            analyser.getByteFrequencyData(dataArray);
            // Analiza częstotliwości
            let threshold = 100; // Określa próg wykrywania częstotliwości
            let bitStream = '';
            dataArray.forEach((value, index) => {
                // Wykrywanie bitów (0 i 1) na podstawie częstotliwości
                if (index < 10) { // Pierwsze pasmo częstotliwości (0-1000Hz) odpowiada 0, a 1000-2000Hz to 1
                    if (value > threshold) {
                        bitStream += '1';
                    } else {
                        bitStream += '0';
                    }
                }
            });
            // Debugowanie: wyświetlanie bitów w konsoli
            console.log('Odebrany strumień bitów:', bitStream);
            // Jeżeli zebrano wystarczającą liczbę bitów, próbujemy dekodować obraz
            if (bitStream.length >= 8 * 3 * 3) {  // Załóżmy, że obraz ma 3x3 piksele
                let binaryData = bitStream.slice(0, 8 * 3 * 3); // Tylko pierwsze 9 pikseli (3x3)
                let pixels = decodeImage(binaryData);
                drawImageOnCanvas(pixels);
            }
            requestAnimationFrame(processAudio); // Przetwarzanie w pętli
        }
        // Funkcja dekodowania bitów na obraz
        function decodeImage(binaryData) {
            let pixels = [];
            for (let i = 0; i < binaryData.length; i += 24) {
                let r = parseInt(binaryData.slice(i, i + 8), 2);
                let g = parseInt(binaryData.slice(i + 8, i + 16), 2);
                let b = parseInt(binaryData.slice(i + 16, i + 24), 2);
                pixels.push({ r, g, b });
            }
            return pixels;
        }
        // Funkcja rysowania obrazu na płótnie
        function drawImageOnCanvas(pixels) {
            let canvas = document.getElementById("imageCanvas");
            let ctx = canvas.getContext("2d");
            let imageData = ctx.createImageData(3, 3); // 3x3 obrazek
            pixels.forEach((pixel, index) => {
                imageData.data[index * 4] = pixel.r;      // Red
                imageData.data[index * 4 + 1] = pixel.g;  // Green
                imageData.data[index * 4 + 2] = pixel.b;  // Blue
                imageData.data[index * 4 + 3] = 255;      // Alpha (pełna przezroczystość)
            });
            ctx.putImageData(imageData, 0, 0);
        }
    </script>
</body>
</html>
](https://github.com/XVENON4X/AUDIOTOIMAGE/settings/pages)
