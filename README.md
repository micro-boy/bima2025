# BIMA 2025

# Metode Pembacaan Sensor dan Pengumpulan Data untuk Sistem FitWork

Untuk sistem pemantauan fisik seperti FitWork dengan platform Arduino/ESP, pemilihan metode pembacaan sensor dan pengumpulan data yang tepat sangat krusial. Berikut saya jelaskan beberapa metode dengan kelebihan dan kekurangannya.

## Metode Pembacaan Sensor

### 1. Metode Polling

Metode polling adalah teknik di mana mikrokontroler secara berkala memeriksa sensor untuk data baru pada interval waktu tertentu.

```cpp
void loop() {
  // Baca sensor setiap 1 detik
  static unsigned long lastRead = 0;
  if (millis() - lastRead >= 1000) {
    readAllSensors();
    lastRead = millis();
  }
  
  // Kode lain berjalan tanpa blocking
}
```

**Kelebihan:**
- Sederhana untuk diimplementasikan
- Predictable timing
- Tidak memerlukan konfigurasi hardware khusus

**Kekurangan:**
- Kurang efisien untuk daya
- Bisa melewatkan peristiwa singkat antara interval polling
- Tidak ideal untuk sensor yang perlu respons cepat

### 2. Metode Interrupt-driven

Pembacaan sensor berbasis interrupt terjadi ketika sensor sendiri memicu mikroprosesor untuk membaca data.

```cpp
void setup() {
  // Mengatur pin interrupt dari sensor
  pinMode(INTERRUPT_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(INTERRUPT_PIN), handleSensorInterrupt, RISING);
}

void handleSensorInterrupt() {
  // Fungsi yang dijalankan saat sensor mengirim sinyal
  readSensorData();
}
```

**Kelebihan:**
- Lebih efisien untuk daya
- Responsif terhadap perubahan cepat
- Tidak melewatkan peristiwa penting

**Kekurangan:**
- Lebih kompleks untuk diimplementasikan
- Memerlukan hardware yang mendukung interrupt
- Perlu perhatian pada race conditions dan timing

### 3. Metode Hybrid (Rekomendasi untuk FitWork)

Kombinasi polling untuk sensor regular dengan interrupt untuk sensor kritis.

```cpp
void setup() {
  // Setup untuk sensor heartrate - menggunakan interrupt
  pinMode(HR_INTERRUPT_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(HR_INTERRUPT_PIN), handleHeartRateEvent, RISING);
  
  // Setup untuk sensor lainnya dengan polling
}

void loop() {
  // Polling untuk sensor-sensor non-kritis
  static unsigned long lastEnvironmentRead = 0;
  if (millis() - lastEnvironmentRead >= 60000) {  // baca lingkungan setiap menit
    readEnvironmentalSensors();
    lastEnvironmentRead = millis();
  }
  
  static unsigned long lastAccelRead = 0;
  if (millis() - lastAccelRead >= 5000) {  // baca akselerometer setiap 5 detik
    readAccelerometerData();
    lastAccelRead = millis();
  }
}
```

## Protokol Komunikasi Sensor

### 1. I²C (Inter-Integrated Circuit)

I²C adalah protokol serial yang menggunakan dua jalur: SDA (data) dan SCL (clock), yang memungkinkan beberapa sensor berkomunikasi dengan mikrokontroler.

```cpp
#include <Wire.h>

void setup() {
  Wire.begin();  // Inisialisasi I2C bus
}

void readMPU6050() {
  Wire.beginTransmission(MPU6050_ADDR);
  Wire.write(0x3B);  // Register awal
  Wire.endTransmission(false);
  Wire.requestFrom(MPU6050_ADDR, 14, true);  // Baca 14 bytes data
  
  // Proses data...
}
```

**Ideal untuk:** Senspr MPU6050, MAX30102, dan sebagian besar sensor dalam sistem FitWork

### 2. SPI (Serial Peripheral Interface)

SPI menggunakan empat jalur (MOSI, MISO, SCK, SS) dan menawarkan komunikasi lebih cepat dari I²C, tetapi membutuhkan lebih banyak pin.

**Ideal untuk:** Sensor dengan kebutuhan data throughput tinggi atau pembacaan sangat cepat

### 3. OneWire

Protokol komunikasi yang hanya membutuhkan satu kabel, cocok untuk sensor simpel seperti DS18B20.

**Ideal untuk:** Sensor sederhana dengan kebutuhan data rendah

## Metode Pengumpulan dan Transmisi Data

### 1. Batch Processing

Data dikumpulkan dalam batch dan ditransmisikan periodik untuk menghemat daya.

```cpp
void loop() {
  // Baca dan simpan data setiap 10 detik
  if (millis() - lastReadTime >= 10000) {
    SensorData newData = readAllSensors();
    dataBuffer.add(newData);
    lastReadTime = millis();
  }
  
  // Kirim batch data setiap 5 menit
  if (millis() - lastSendTime >= 300000 || dataBuffer.isFull()) {
    sendDataToServer(dataBuffer);
    dataBuffer.clear();
    lastSendTime = millis();
  }
}
```

**Kelebihan:**
- Sangat efisien untuk baterai
- Mengurangi overhead koneksi
- Menangani masalah konektivitas intermiten

**Kekurangan:**
- Delay dalam ketersediaan data
- Risiko kehilangan data jika terjadi crash sebelum transmisi

### 2. Real-time Transmission

Data ditransmisikan segera setelah diperoleh, ideal untuk monitoring medis.

```cpp
void loop() {
  if (millis() - lastReadTime >= 1000) {
    SensorData data = readAllSensors();
    sendDataImmediately(data);
    lastReadTime = millis();
  }
}
```

**Kelebihan:**
- Data tersedia segera
- Monitoring real-time memungkinkan respons cepat
- Simplifikasi manajemen memori lokal

**Kekurangan:**
- Konsumsi daya tinggi
- Sangat bergantung pada konektivitas
- Overhead transmisi yang signifikan

### 3. Adaptive Transmission (Rekomendasi untuk FitWork)

Transmisi disesuaikan berdasarkan konteks dan kepentingan data.

```cpp
void processAndTransmitData(SensorData data) {
  // Analisis data untuk menentukan signifikansi
  bool isSignificant = analyzeDataSignificance(data);
  
  if (isSignificant || heartRateAbnormal) {
    // Transmisi segera untuk data penting
    sendDataImmediately(data);
  } else {
    // Tambahkan ke buffer untuk transmisi batch
    dataBuffer.add(data);
  }
  
  // Kirim batch data berdasarkan jadwal atau ketika buffer hampir penuh
  if (millis() - lastBatchSend >= batchInterval || dataBuffer.size() > threshold) {
    sendBatchData(dataBuffer);
    dataBuffer.clear();
    lastBatchSend = millis();
  }
}
```

Pendekatan ini mengirim data vital signs secara real-time saat mendeteksi anomali, sementara data lingkungan dan aktivitas normal dikirim dalam batch.

## Teknologi Penyimpanan dan Transmisi Data

### 1. Penyimpanan Lokal

**SD Card** - Untuk buffer data sementara jika konektivitas terputus.

```cpp
#include <SD.h>

void saveToSD(SensorData data) {
  File dataFile = SD.open("fitwork.csv", FILE_WRITE);
  if (dataFile) {
    dataFile.println(data.toCSV());
    dataFile.close();
  }
}
```

**SPIFFS/LittleFS** - Sistem file pada flash memory ESP yang lebih efisien.

### 2. Protokol Transmisi

**MQTT** - Protokol lightweight yang ideal untuk IoT, dengan model publish-subscribe.

```cpp
#include <PubSubClient.h>

void sendDataMQTT(SensorData data) {
  char payload[256];
  sprintf(payload, "{\"heartRate\":%d,\"steps\":%d,\"temp\":%.1f}", 
          data.heartRate, data.steps, data.temperature);
  
  mqttClient.publish("fitwork/user123/health", payload);
}
```

**HTTP/REST** - Pengiriman data dengan API calls ke server backend.

```cpp
#include <HTTPClient.h>

void sendDataHTTP(SensorData data) {
  HTTPClient http;
  http.begin("https://fitwork-api.example.com/data");
  http.addHeader("Content-Type", "application/json");
  
  String payload = data.toJSON();
  int httpCode = http.POST(payload);
  
  // Handle response...
  http.end();
}
```

**WebSocket** - Untuk koneksi dua arah real-time jika diperlukan feedback cepat.

## Strategi Optimasi untuk Sistem FitWork

### 1. Adaptive Sampling

Frekuensi sampling yang disesuaikan berdasarkan aktivitas pengguna:

```cpp
void determineAndSetSamplingRate() {
  // Detect activity level
  int activityLevel = detectActivityLevel();
  
  switch(activityLevel) {
    case ACTIVITY_HIGH:
      // Set high sampling rate for active periods
      accelSamplingInterval = 1000;  // 1 second
      heartRateSamplingInterval = 5000;  // 5 seconds
      break;
    case ACTIVITY_MEDIUM:
      accelSamplingInterval = 3000;  // 3 seconds
      heartRateSamplingInterval = 10000;  // 10 seconds
      break;
    case ACTIVITY_LOW:
      // Lower sampling rate during rest periods
      accelSamplingInterval = 10000;  // 10 seconds
      heartRateSamplingInterval = 30000;  // 30 seconds
      break;
  }
}
```

### 2. Local Preprocessing

Melakukan preprocessing data pada device untuk mengurangi volume data yang dikirim:

```cpp
void preprocessAccelData() {
  // Collect raw samples
  for (int i = 0; i < SAMPLE_COUNT; i++) {
    samples[i] = readAccelerometer();
    delay(20);  // 50Hz sampling
  }
  
  // Process locally to extract features
  ActivityData processedData;
  processedData.averageMagnitude = calculateAverageMagnitude(samples);
  processedData.stepCount = countSteps(samples);
  processedData.activityType = classifyActivity(samples);
  
  // Only send the processed features rather than raw data
  sendProcessedData(processedData);
}
```

### 3. Event-Based Monitoring

Monitoring berdasarkan peristiwa spesifik untuk mengurangi sampling berkelanjutan:

```cpp
void detectAndMonitorEvents() {
  // Monitor for start of activity
  if (detectActivityStart()) {
    // Increase sampling rate during activity
    currentSamplingRate = HIGH_SAMPLING_RATE;
    startDetailedMonitoring();
  }
  
  // Monitor for end of activity
  if (monitoringActive && detectActivityEnd()) {
    // Return to lower sampling rate
    currentSamplingRate = LOW_SAMPLING_RATE;
    stopDetailedMonitoring();
    
    // Generate activity summary
    ActivitySummary summary = generateActivitySummary();
    sendActivitySummary(summary);
  }
}
```

## Rekomendasi Terintegrasi untuk FitWork

Berdasarkan kebutuhan sistem FitWork, saya merekomendasikan pendekatan berikut:

1. **Pembacaan Sensor**: Metode hybrid dengan polling terjadwal untuk sensor umum (akselerometer, lingkungan) dan interrupt-driven untuk data kritis (detak jantung).

2. **Protokol Komunikasi**: Gunakan I²C untuk sebagian besar sensor karena efisiensi pin dan dukungan alamat perangkat multiple.

3. **Pengumpulan Data**: Implementasikan adaptive sampling yang menyesuaikan interval pembacaan berdasarkan aktivitas pengguna dan status kesehatan.

4. **Transmisi Data**: Gunakan transmisi adaptif dengan:
   - Pengiriman real-time untuk anomali kesehatan dan data kritis
   - Batch processing untuk data aktivitas dan lingkungan reguler
   - Local preprocessing untuk mengekstrak fitur penting dan mengurangi bandwidth

5. **Protokol Transmisi**: MQTT sebagai protokol utama dengan WebSocket untuk notifikasi real-time ke aplikasi pengguna.

6. **Penyimpanan Lokal**: Implementasikan buffer data circular pada SPIFFS/LittleFS dengan failover ke SD card untuk periode offline yang lebih lama.

7. **Efisiensi Daya**: Jadwalkan deep sleep antara pembacaan sensor dan implementasikan wake-on-motion untuk aktivitas yang signifikan.

Arsitektur semacam ini akan memberikan keseimbangan optimal antara akurasi monitoring, responsivitas terhadap perubahan kesehatan signifikan, dan efisiensi daya yang diperlukan untuk sistem wearable seperti FitWork.
