# Dispositivo YuraGuard

```
/* =========================================================
   YURAGUARD
   Envia el paquete con el siguiente contenido:

   contador, sonido, presencia, caida, peligro, bateria,
   lat, lng, alt, hdop, satelites

   IA Edge Impulse:
   - bosque = normal
   - voz_humana + actividad_humana = alerta
   - actividad_humana representa motosierra + maquinaria
   ========================================================= */

#define EIDSP_QUANTIZE_FILTERBANK 0
#include <MarthCage-project-1_inferencing.h>

#include <Arduino.h>
#include <Wire.h>
#include <TinyGPSPlus.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <SPI.h>
#include <RF24.h>
#include <math.h>
#include <string.h>

// ======================
// PINES
// ======================
#define MIC_PIN 4

#define PIN_GPS_RX_ESP 10
#define PIN_GPS_TX_ESP 9

#define PIN_RADAR_RX_ESP 16
#define PIN_RADAR_TX_ESP 18

#define PIN_MPU_SDA 21
#define PIN_MPU_SCL 20

#define NRF_CE   35
#define NRF_CSN  36
#define NRF_SCK  37
#define NRF_MOSI 38
#define NRF_MISO 39

// Bateria FZ0430: S/OUT -> GPIO1
#define PIN_BATERIA 1

// Calibrado con pack real 3.54 V y lectura ADC aprox. 0.629 V
#define FACTOR_FZ0430 5.63f

// Escala de porcentaje para prototipo
#define VOLTAJE_BATERIA_MIN 3.30f
#define VOLTAJE_BATERIA_MAX 3.70f

// Cambia a 0 para probar sin FZ0430/bateria
#define USAR_SENSOR_BATERIA 1

// ======================
// AJUSTES
// ======================
#define AUDIO_GAIN 2
#define RMS_ALERTA_MIN 120
// Deteccion mejorada de caida con MPU6050
// Fase 1: caida libre si la magnitud baja de 0.55G
// Fase 2: impacto si luego sube sobre 2.0G en menos de 1 segundo
#define CAIDA_LIBRE_G_MAX 0.55f
#define CAIDA_IMPACTO_G_MIN 2.00f
#define CAIDA_TIEMPO_MAX_MS 1000UL

// La caida detectada queda pendiente hasta que se envie el siguiente paquete NRF

// Si quieres que sonido de alerta necesite presencia radar, pon 1
#define EXIGIR_PRESENCIA_PARA_TALA 0

// ======================
// CODIGOS DEL PAQUETE
// Deben coincidir con el ESP32 negro
// ======================
#define SONIDO_BOSQUE       0
#define SONIDO_ACTIVIDAD    1   // motosierra + maquinaria agrupadas
#define SONIDO_RESERVADO    2
#define SONIDO_VOZ          3

#define PELIGRO_NORMAL      0
#define PELIGRO_TALA        1
#define PELIGRO_BATERIA     2

// ======================
// OBJETOS
// ======================
HardwareSerial GPSSerial(2);
HardwareSerial RadarSerial(1);
TinyGPSPlus gps;
Adafruit_MPU6050 mpu;

RF24 radio(NRF_CE, NRF_CSN);
const byte address[6] = "YURA1";

// ======================
// PAQUETE NRF
// IMPORTANTE: igual al struct del ESP32 negro
// Tamaño esperado: 26 bytes
// ======================
struct __attribute__((packed)) YuraPacket {
  uint32_t contador;

  uint8_t sonido;
  uint8_t presencia;
  uint8_t caida;
  uint8_t peligro;
  uint8_t bateria;

  float lat;
  float lng;
  float alt;
  float hdop;

  uint8_t satelites;
};

// ======================
// EDGE IMPULSE AUDIO
// ======================
typedef struct {
  signed short *buffers[2];
  unsigned char buf_select;
  volatile unsigned char buf_ready;
  unsigned int buf_count;
  unsigned int n_samples;
} inference_t;

static inference_t inference;
static bool debug_nn = false;
static int print_results = -(EI_CLASSIFIER_SLICES_PER_MODEL_WINDOW);
static bool record_status = true;
static int adc_center = 1430;

volatile int32_t audio_peak = 0;
volatile float audio_rms = 0.0f;
volatile bool pausarAudioADC = false;

// Estado compartido de caida mejorada
volatile bool caidaPendiente = false;
volatile bool posibleCaidaLibre = false;
volatile unsigned long tiempoCaidaLibre = 0;
volatile float ultimaVibracionMPU = 0.0f;
volatile float ultimaGMPU = 1.0f;

// ======================
// ESTADO
// ======================
bool mpuOk = false;
bool nrfOk = false;
bool iaOk = false;
uint32_t contador = 0;

// ======================
// PROTOTIPOS
// ======================
void feedGPS();
void calibrar_microfono();
bool microphone_inference_start(uint32_t n_samples);
bool microphone_inference_record();
int microphone_audio_signal_get_data(size_t offset, size_t length, float *out_ptr);
void microphone_inference_end();
void capture_samples(void* arg);
void tareaCaidaMPU(void* arg);

void inicializarSensores();
void inicializarNRF();
void inicializarIA();

uint8_t clasificarSonido(const ei_impulse_result_t &result);
uint8_t leerPresenciaRadar();
uint8_t leerCaidaMPU(float *vibracionOut);
uint8_t leerBateria();
uint8_t analizarPeligro(uint8_t sonido, uint8_t presencia, uint8_t caida, uint8_t bateria);
void crearYEnviarPaquete(uint8_t sonido);
void imprimirPaquete(const YuraPacket &data);

String textoSonido(uint8_t sonido);
String textoSiNo(uint8_t v);
String textoPeligro(uint8_t peligro);

// ======================
// SETUP
// ======================
void setup() {
  Serial.begin(115200);
  delay(1500);

  Serial.println();
  Serial.println("INICIANDO YURAGUARD MORADO - ENVIO MINIMO");

  inicializarSensores();
  inicializarNRF();
  inicializarIA();

  Serial.print("Tamano YuraPacket: ");
  Serial.print(sizeof(YuraPacket));
  Serial.println(" bytes");
  Serial.println("Sistema listo.");
  Serial.println();
}

// ======================
// LOOP
// ======================
void loop() {
  feedGPS();

  if (!iaOk) {
    delay(500);
    return;
  }

  if (!microphone_inference_record()) {
    Serial.println("ERR: fallo al grabar audio");
    return;
  }

  signal_t signal;
  signal.total_length = EI_CLASSIFIER_SLICE_SIZE;
  signal.get_data = &microphone_audio_signal_get_data;

  ei_impulse_result_t result = {0};
  EI_IMPULSE_ERROR r = run_classifier_continuous(&signal, &result, debug_nn);

  if (r != EI_IMPULSE_OK) {
    Serial.print("ERR: clasificador fallo: ");
    Serial.println((int)r);
    return;
  }

  // Solo enviamos cuando ya hay una ventana completa de IA
  if (++print_results >= EI_CLASSIFIER_SLICES_PER_MODEL_WINDOW) {
    uint8_t sonido = clasificarSonido(result);
    crearYEnviarPaquete(sonido);
    print_results = 0;
  }
}

// ======================
// INICIALIZACION
// ======================
void inicializarSensores() {
  analogReadResolution(12);
  analogSetPinAttenuation(MIC_PIN, ADC_11db);
  pinMode(PIN_BATERIA, INPUT);
  analogSetPinAttenuation(PIN_BATERIA, ADC_11db);

  GPSSerial.begin(9600, SERIAL_8N1, PIN_GPS_RX_ESP, PIN_GPS_TX_ESP);
  RadarSerial.begin(115200, SERIAL_8N1, PIN_RADAR_RX_ESP, PIN_RADAR_TX_ESP);

  Wire.begin(PIN_MPU_SDA, PIN_MPU_SCL);
  Wire.setClock(400000);

  mpuOk = mpu.begin(0x68, &Wire);
  if (mpuOk) {
    mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
    mpu.setGyroRange(MPU6050_RANGE_500_DEG);
    mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
    Serial.println("MPU6050 OK");

    BaseType_t tareaOk = xTaskCreatePinnedToCore(
      tareaCaidaMPU,
      "TareaCaidaMPU",
      4096,
      NULL,
      1,
      NULL,
      1
    );

    if (tareaOk == pdPASS) {
      Serial.println("Deteccion mejorada de caida OK");
    } else {
      Serial.println("ERR: no se pudo iniciar tarea de caida");
    }
  } else {
    Serial.println("MPU6050 no detectado");
  }
}

void inicializarNRF() {
  SPI.begin(NRF_SCK, NRF_MISO, NRF_MOSI, NRF_CSN);

  nrfOk = radio.begin();
  if (!nrfOk) {
    Serial.println("NRF24L01 no detectado");
    return;
  }

  radio.setChannel(90);
  radio.setDataRate(RF24_250KBPS);
  radio.setPALevel(RF24_PA_LOW);
  radio.setAutoAck(true);
  radio.setRetries(5, 15);
  radio.setPayloadSize(sizeof(YuraPacket));
  radio.openWritingPipe(address);
  radio.stopListening();

  Serial.println("NRF24L01 OK");
}

void inicializarIA() {
  Serial.println("Calibrando microfono...");
  calibrar_microfono();

  Serial.print("ADC center: ");
  Serial.println(adc_center);

  run_classifier_init();

  if (!microphone_inference_start(EI_CLASSIFIER_SLICE_SIZE)) {
    Serial.println("ERR: no se pudo iniciar buffer de audio");
    iaOk = false;
    return;
  }

  iaOk = true;
  Serial.println("IA OK");
}

// ======================
// GPS
// ======================
void feedGPS() {
  while (GPSSerial.available()) {
    gps.encode(GPSSerial.read());
  }
}

// ======================
// CLASIFICACION IA
// ======================
uint8_t clasificarSonido(const ei_impulse_result_t &result) {
  float p_bosque = 0.0f;
  float p_voz = 0.0f;
  float p_actividad = 0.0f;

  for (size_t ix = 0; ix < EI_CLASSIFIER_LABEL_COUNT; ix++) {
    const char *label = result.classification[ix].label;
    float value = result.classification[ix].value;

    if (strcmp(label, "bosque") == 0 || strcmp(label, "BOSQUE") == 0) {
      p_bosque = value;
    }
    else if (
      strcmp(label, "voz_humana") == 0 ||
      strcmp(label, "voz humana") == 0 ||
      strcmp(label, "VOZ_HUMANA") == 0 ||
      strcmp(label, "VOZ HUMANA") == 0
    ) {
      p_voz = value;
    }
    else if (
      strcmp(label, "actividad_humana") == 0 ||
      strcmp(label, "actividad humana") == 0 ||
      strcmp(label, "ACTIVIDAD_HUMANA") == 0 ||
      strcmp(label, "ACTIVIDAD HUMANA") == 0
    ) {
      p_actividad = value;
    }
  }

  float p_alerta = p_voz + p_actividad;

  if (audio_peak >= 32000) {
    // Audio saturado: se conserva el resultado, pero seria bueno bajar AUDIO_GAIN.
  }

  if (p_alerta > p_bosque && audio_rms > RMS_ALERTA_MIN) {
    if (p_voz >= p_actividad) {
      return SONIDO_VOZ;
    }
    return SONIDO_ACTIVIDAD;
  }

  return SONIDO_BOSQUE;
}

// ======================
// SENSORES
// ======================
uint8_t leerPresenciaRadar() {
  uint8_t presencia = 0;

  if (RadarSerial.available()) {
    presencia = 1;
    while (RadarSerial.available()) {
      RadarSerial.read();
    }
  }

  return presencia;
}

void tareaCaidaMPU(void* arg) {
  (void)arg;

  while (true) {
    if (!mpuOk) {
      vTaskDelay(pdMS_TO_TICKS(200));
      continue;
    }

    sensors_event_t acc, gyro, temp;
    mpu.getEvent(&acc, &gyro, &temp);

    float accMag = sqrt(
      acc.acceleration.x * acc.acceleration.x +
      acc.acceleration.y * acc.acceleration.y +
      acc.acceleration.z * acc.acceleration.z
    );

    float gActual = accMag / 9.81f;
    float vibracion = fabs(accMag - 9.81f);
    unsigned long ahora = millis();

    ultimaGMPU = gActual;
    ultimaVibracionMPU = vibracion;

    // Fase 1: posible caida libre.
    if (!posibleCaidaLibre && gActual < CAIDA_LIBRE_G_MAX) {
      posibleCaidaLibre = true;
      tiempoCaidaLibre = ahora;
    }

    // Fase 2: impacto fuerte despues de la caida libre.
    if (posibleCaidaLibre &&
        gActual > CAIDA_IMPACTO_G_MIN &&
        (ahora - tiempoCaidaLibre) <= CAIDA_TIEMPO_MAX_MS) {
      caidaPendiente = true;
      posibleCaidaLibre = false;
    }

    // Si no hubo impacto a tiempo, se cancela la posible caida.
    if (posibleCaidaLibre &&
        (ahora - tiempoCaidaLibre) > CAIDA_TIEMPO_MAX_MS) {
      posibleCaidaLibre = false;
    }

    vTaskDelay(pdMS_TO_TICKS(20));
  }
}

uint8_t leerCaidaMPU(float *vibracionOut) {
  if (!mpuOk) {
    *vibracionOut = 0.0f;
    return 0;
  }

  *vibracionOut = ultimaVibracionMPU;

  // Se consume la caida pendiente para enviarla una sola vez por NRF.
  if (caidaPendiente) {
    caidaPendiente = false;
    return 1;
  }

  return 0;
}


uint8_t leerBateria() {
#if USAR_SENSOR_BATERIA == 0
  return 100;
#else
  // Pausa temporal de audio para evitar conflicto ADC con el MAX9814.
  pausarAudioADC = true;
  delay(30);

  analogSetPinAttenuation(PIN_BATERIA, ADC_11db);

  for (int i = 0; i < 10; i++) {
    analogRead(PIN_BATERIA);
    delay(2);
  }

  const int muestras = 25;
  int lecturas[muestras];

  for (int i = 0; i < muestras; i++) {
    lecturas[i] = analogRead(PIN_BATERIA);
    delay(4);
  }

  analogSetPinAttenuation(MIC_PIN, ADC_11db);
  pausarAudioADC = false;

  // Ordenar para mediana
  for (int i = 0; i < muestras - 1; i++) {
    for (int j = i + 1; j < muestras; j++) {
      if (lecturas[j] < lecturas[i]) {
        int tmp = lecturas[i];
        lecturas[i] = lecturas[j];
        lecturas[j] = tmp;
      }
    }
  }

  int lecturaMediana = lecturas[muestras / 2];
  float voltajeADC = (lecturaMediana / 4095.0f) * 3.3f;
  float voltajeBateria = voltajeADC * FACTOR_FZ0430;

  if (lecturaMediana < 20) {
    return 0;
  }

  float porcentaje = ((voltajeBateria - VOLTAJE_BATERIA_MIN) /
                     (VOLTAJE_BATERIA_MAX - VOLTAJE_BATERIA_MIN)) * 100.0f;

  if (porcentaje > 100.0f) porcentaje = 100.0f;
  if (porcentaje < 0.0f) porcentaje = 0.0f;

  return (uint8_t)porcentaje;
#endif
}

uint8_t analizarPeligro(uint8_t sonido, uint8_t presencia, uint8_t caida, uint8_t bateria) {
  bool sonidoAlerta = (sonido == SONIDO_ACTIVIDAD || sonido == SONIDO_VOZ);

#if EXIGIR_PRESENCIA_PARA_TALA == 1
  if (sonidoAlerta && presencia == 1) {
    return PELIGRO_TALA;
  }
#else
  if (sonidoAlerta) {
    return PELIGRO_TALA;
  }
#endif

  if (bateria <= 20) {
    return PELIGRO_BATERIA;
  }

  return PELIGRO_NORMAL;
}

// ======================
// CREAR Y ENVIAR PAQUETE
// ======================
void crearYEnviarPaquete(uint8_t sonido) {
  feedGPS();

  float vibracion = 0.0f;
  uint8_t caida = leerCaidaMPU(&vibracion);
  uint8_t presencia = leerPresenciaRadar();
  uint8_t bateria = leerBateria();

  float lat = 0.0f;
  float lng = 0.0f;
  float alt = 0.0f;
  float hdop = 99.99f;
  uint8_t satelites = 0;

  if (gps.location.isValid()) {
    lat = gps.location.lat();
    lng = gps.location.lng();
  }

  if (gps.altitude.isValid()) {
    alt = gps.altitude.meters();
  }

  if (gps.hdop.isValid()) {
    hdop = gps.hdop.hdop();
  }

  if (gps.satellites.isValid()) {
    satelites = gps.satellites.value();
  }

  uint8_t peligro = analizarPeligro(sonido, presencia, caida, bateria);

  YuraPacket data;
  data.contador = contador++;
  data.sonido = sonido;
  data.presencia = presencia;
  data.caida = caida;
  data.peligro = peligro;
  data.bateria = bateria;
  data.lat = lat;
  data.lng = lng;
  data.alt = alt;
  data.hdop = hdop;
  data.satelites = satelites;

  if (nrfOk) {
    bool enviado = radio.write(&data, sizeof(data));
    if (!enviado) {
      Serial.println("NRF envio: FALLO");
    }
  } else {
    Serial.println("NRF no disponible");
  }

  imprimirPaquete(data);
}

void imprimirPaquete(const YuraPacket &data) {
  Serial.println("========== PAQUETE ENVIADO ==========");

  Serial.print("Contador: ");
  Serial.println(data.contador);

  Serial.print("Sonido: ");
  Serial.println(textoSonido(data.sonido));

  Serial.print("Presencia: ");
  Serial.println(textoSiNo(data.presencia));

  Serial.print("Caida: ");
  Serial.println(textoSiNo(data.caida));

  Serial.print("Peligro: ");
  Serial.println(textoPeligro(data.peligro));

  Serial.print("Bateria: ");
  Serial.print(data.bateria);
  Serial.println("%");

  Serial.print("Latitud: ");
  Serial.println(data.lat, 6);

  Serial.print("Longitud: ");
  Serial.println(data.lng, 6);

  Serial.print("Altitud: ");
  Serial.println(data.alt, 1);

  Serial.print("Satelites: ");
  Serial.println(data.satelites);

  Serial.print("HDOP: ");
  Serial.println(data.hdop, 2);

  Serial.println("=====================================");
  Serial.println();
}

// ======================
// TEXTOS
// ======================
String textoSonido(uint8_t sonido) {
  if (sonido == SONIDO_BOSQUE) return "bosque";
  if (sonido == SONIDO_ACTIVIDAD) return "actividad humana";
  if (sonido == SONIDO_VOZ) return "voz humana";
  return "reservado";
}

String textoSiNo(uint8_t v) {
  return v ? "SI" : "NO";
}

String textoPeligro(uint8_t peligro) {
  if (peligro == PELIGRO_TALA) return "alerta / posible tala o actividad humana";
  if (peligro == PELIGRO_BATERIA) return "bateria baja";
  return "normal";
}

// ======================
// AUDIO / EDGE IMPULSE
// ======================
void calibrar_microfono() {
  long sum = 0;
  const int N = 2000;

  for (int i = 0; i < N; i++) {
    sum += analogRead(MIC_PIN);
    delayMicroseconds(100);
  }

  adc_center = sum / N;
}

bool microphone_inference_start(uint32_t n_samples) {
  inference.buffers[0] = (signed short *)malloc(n_samples * sizeof(signed short));
  if (inference.buffers[0] == NULL) return false;

  inference.buffers[1] = (signed short *)malloc(n_samples * sizeof(signed short));
  if (inference.buffers[1] == NULL) {
    free(inference.buffers[0]);
    return false;
  }

  inference.buf_select = 0;
  inference.buf_count = 0;
  inference.n_samples = n_samples;
  inference.buf_ready = 0;
  record_status = true;

  BaseType_t task_ok = xTaskCreatePinnedToCore(
    capture_samples,
    "CaptureSamples",
    1024 * 8,
    NULL,
    1,
    NULL,
    0
  );

  return task_ok == pdPASS;
}

void capture_samples(void* arg) {
  (void)arg;

  const uint32_t sample_interval_us = 1000000UL / EI_CLASSIFIER_FREQUENCY;
  int64_t sum_squares = 0;
  int32_t peak = 0;
  unsigned long next_sample_time = micros();

  while (record_status) {
    if (pausarAudioADC) {
      next_sample_time = micros() + sample_interval_us;
      vTaskDelay(1);
      continue;
    }

    while ((long)(micros() - next_sample_time) < 0) {
      // espera para mantener frecuencia de muestreo
    }

    next_sample_time += sample_interval_us;

    int raw = analogRead(MIC_PIN);
    int centered = raw - adc_center;
    int32_t sample32 = centered * AUDIO_GAIN;

    if (sample32 > 32767) sample32 = 32767;
    if (sample32 < -32768) sample32 = -32768;

    int16_t sample16 = (int16_t)sample32;
    inference.buffers[inference.buf_select][inference.buf_count++] = sample16;

    int32_t abs_sample = abs((int)sample16);
    if (abs_sample > peak) peak = abs_sample;
    sum_squares += (int64_t)sample16 * (int64_t)sample16;

    if (inference.buf_count >= inference.n_samples) {
      audio_peak = peak;
      audio_rms = sqrt((float)sum_squares / inference.n_samples);

      inference.buf_select ^= 1;
      inference.buf_count = 0;
      inference.buf_ready = 1;

      peak = 0;
      sum_squares = 0;
      vTaskDelay(1);
    }
  }

  vTaskDelete(NULL);
}

bool microphone_inference_record() {
  while (inference.buf_ready == 0) {
    delay(1);
  }

  inference.buf_ready = 0;
  return true;
}

int microphone_audio_signal_get_data(size_t offset, size_t length, float *out_ptr) {
  numpy::int16_to_float(
    &inference.buffers[inference.buf_select ^ 1][offset],
    out_ptr,
    length
  );

  return 0;
}

void microphone_inference_end() {
  record_status = false;

  if (inference.buffers[0]) free(inference.buffers[0]);
  if (inference.buffers[1]) free(inference.buffers[1]);
}

#if !defined(EI_CLASSIFIER_SENSOR) || EI_CLASSIFIER_SENSOR != EI_CLASSIFIER_SENSOR_MICROPHONE
#error "Invalid model for current sensor."
#endif
```
