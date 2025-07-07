# Oxigen Vision

Proyek ini mengintegrasikan p5.js (untuk manipulasi video dan canvas), ml5.js (untuk image classification model), dan MQTT.js (untuk komunikasi real-time ke broker MQTT).
Hasil klasifikasi label dikirim secara real-time ke topic MQTT saat label berubah.

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8" />
  <title>OXIGEN Vision with MQTT Advanced</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/0.9.0/p5.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/0.9.0/addons/p5.dom.min.js"></script>
  <script src="https://unpkg.com/ml5@latest/dist/ml5.min.js"></script>
  <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
  <style>
    body {
      display: flex;
      flex-direction: column;
      align-items: center;
      background-color: #222;
      color: white;
      font-family: Arial, sans-serif;
    }

    canvas {
      border: 2px solid white;
      margin-top: 20px;
    }

    #status {
      margin-top: 10px;
      font-weight: bold;
    }

    #log {
      width: 80%;
      height: 200px;
      background-color: #000;
      color: #0f0;
      overflow-y: scroll;
      padding: 10px;
      margin-top: 20px;
      border: 1px solid #0f0;
    }
  </style>
</head>

<body>
  <h1>OXIGEN Vision</h1>
  <div id="status">Connecting to MQTT...</div>
  <div id="log"></div>

  <script>
    // =========================
    // Global Variables
    // =========================
    let classifier, video, flippedVideo;
    let label = "";
    let lastLabel = "";
    let lastConfidence = 0;

    const mqttTopicLabel = "oxigen/label";
    const mqttTopicConfidence = "oxigen/confidence";
    let mqttClient;

    // =========================
    // MQTT Setup with Auto Reconnect
    // =========================
    function setupMQTT() {
      mqttClient = mqtt.connect("wss://public:public@public.cloud.shiftr.io", {
        clientId: "OXIGEN_Vision_Client_" + Math.random().toString(16).substr(2, 8),
        reconnectPeriod: 2000 // Auto reconnect every 2 seconds
      });

      mqttClient.on("connect", () => {
        console.log("✅ MQTT Connected");
        updateStatus("MQTT Connected");
        logMessage("MQTT Connected");
        mqttClient.subscribe("oxigen/feedback");
      });

      mqttClient.on("reconnect", () => {
        console.warn("🔄 Reconnecting to MQTT...");
        updateStatus("Reconnecting to MQTT...");
        logMessage("Reconnecting to MQTT...");
      });

      mqttClient.on("message", (topic, message) => {
        logMessage(`📥 MQTT Received [${topic}]: ${message.toString()}`);
      });

      mqttClient.on("error", (err) => {
        console.error("❌ MQTT Error:", err);
        logMessage(`❌ MQTT Error: ${err.message}`);
      });
    }

    function publishLabel(labelValue, confidenceValue) {
      if (mqttClient && mqttClient.connected) {
        mqttClient.publish(mqttTopicLabel, labelValue);
        mqttClient.publish(mqttTopicConfidence, confidenceValue.toString());
        logMessage(`🚀 Published: ${labelValue} (${confidenceValue.toFixed(2)})`);
      }
    }

    function updateStatus(text) {
      document.getElementById("status").innerText = text;
    }

    function logMessage(message) {
      const logElement = document.getElementById("log");
      const time = new Date().toLocaleTimeString();
      logElement.innerHTML += `[${time}] ${message}<br>`;
      logElement.scrollTop = logElement.scrollHeight; // Auto scroll
    }

    // =========================
    // ML5 Setup
    // =========================
    function preload() {
      classifier = ml5.imageClassifier('my_model/model.json');
    }

    function setup() {
      createCanvas(320, 260);
      setupMQTT();
      initializeVideo();
    }

    function initializeVideo() {
      video = createCapture(VIDEO);
      video.size(320, 240);
      video.hide();
      flippedVideo = ml5.flipImage(video);
      classifyVideo();
    }

    function draw() {
      background(0);
      image(flippedVideo, 0, 0);

      fill(0, 255, 0);
      textSize(20);
      textAlign(CENTER);
      text(label, width / 2, height - 10);
    }

    function classifyVideo() {
      flippedVideo = ml5.flipImage(video);
      classifier.classify(flippedVideo, handleResult);
      flippedVideo.remove();
    }

    function handleResult(error, results) {
      if (error) {
        console.error("❌ Classification Error:", error);
        logMessage(`❌ Classification Error: ${error.message}`);
        classifyVideo();
        return;
      }

      if (results && results.length > 0) {
        const currentLabel = results[0].label;
        const currentConfidence = results[0].confidence;

        label = currentLabel;

        // Publish ke MQTT hanya jika label atau confidence berubah signifikan
        if (currentLabel !== lastLabel || Math.abs(currentConfidence - lastConfidence) > 0.05) {
          publishLabel(currentLabel, currentConfidence);
          lastLabel = currentLabel;
          lastConfidence = currentConfidence;
        }
      }

      classifyVideo();
    }
  </script>
</body>

</html>
```

| Bagian                   | Penjelasan                                                                                                 |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| **MQTT Setup**           | Menghubungkan client JavaScript ke broker MQTT public dari shiftr.io menggunakan protokol WebSocket.       |
| **ML5 Setup**            | Meload model image classifier (`my_model/model.json`) yang dibuat sebelumnya.                              |
| **Video Capture**        | Mengambil video dari webcam menggunakan `p5.js`.                                                           |
| **Image Classification** | Memproses frame video secara terus-menerus dan mengklasifikasikannya menggunakan model yang telah dilatih. |
| **Publish ke MQTT**      | Hasil label dikirim ke topic `oxigen/label` hanya jika label terbaru berbeda dari label sebelumnya.        |
| **UI Feedback**          | Menampilkan status koneksi MQTT dan hasil klasifikasi secara real-time di canvas.                          |
