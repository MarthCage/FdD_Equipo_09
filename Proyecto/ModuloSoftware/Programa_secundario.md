# Dispositivo receptor y envío de datos a la aplicación:

```
// =========================================================
// RECEPTOR / PUENTE YURAGUARD
// Recibe datos del dispositivo YuraGuard por medio del nRF24L01
// y los envía a la app por BLE en formato JSON.
// =========================================================

#include <SPI.h>
#include <RF24.h>

#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>


// ======================
// CONFIGURACIÓN GENERAL
// ======================
#define DEVICE_NAME "YuraGuard"

// True si se desea que la app reciba datos ni bien se inicia, false para poder usar el botón de encendido de la app jahsja (esto ya no sería necesario)
#define SISTEMA_ACTIVO_INICIAL true

// Tiempo para mostrar estado en Serial cuando no llegan paquetes
#define TIEMPO_MENSAJE_ESTADO_MS 3000UL

// Tiempo para reintentar iniciar nRF si no fue detectado
#define TIEMPO_REINTENTO_NRF_MS 5000UL

// ======================
// PINES nRF24L01
// ======================
#define NRF_CE   14
#define NRF_CSN  10
#define NRF_SCK  12
#define NRF_MOSI 11
#define NRF_MISO 13

RF24 radio(NRF_CE, NRF_CSN);
const byte address[6] = "YURA1";

bool nrfOk = false;
unsigned long ultimoReintentoNRF = 0;

// ======================
// BLE - APP INVENTOR
// UUID tipo UART BLE
// ======================
#define SERVICE_UUID        "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHARACTERISTIC_RX   "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"  // App -> ESP32 negro
#define CHARACTERISTIC_TX   "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"  // ESP32 negro -> App

BLEServer *bleServer = nullptr;
BLECharacteristic *txCharacteristic = nullptr;

bool appConectada = false;
bool sistemaActivo = SISTEMA_ACTIVO_INICIAL;

// Declaraciones adelantadas usadas por los callbacks BLE
void enviarEstadoBLE(const char *evento);

// ======================
// CÓDIGOS RECIBIDOS DE YURAGUARD
// ======================
#define SONIDO_BOSQUE       0
#define SONIDO_ACTIVIDAD    1   // motosierra + maquinaria agrupadas
#define SONIDO_RESERVADO    2
#define SONIDO_VOZ          3

#define PELIGRO_NORMAL      0
#define PELIGRO_TALA        1   // actividad humana / posible tala
#define PELIGRO_BATERIA     2

// ======================
// PAQUETE RECIBIDO DE YURAGUARD
// Tamaño esperado: 26 bytes.
// ======================
struct __attribute__((packed)) YuraPacket {
  uint32_t contador;

  uint8_t sonido;      // 0 bosque, 1 actividad_humana, 2 reservado, 3 voz_humana
  uint8_t presencia;   // 0 no, 1 si
  uint8_t caida;       // 0 no, 1 si
  uint8_t peligro;     // 0 normal, 1 tala/actividad, 2 bateria
  uint8_t bateria;     // 0 a 100

  float lat;
  float lng;
  float alt;
  float hdop;

  uint8_t satelites;
};

unsigned long ultimoMensajeEstado = 0;
unsigned long ultimoPaqueteMs = 0;
YuraPacket ultimoPaquete;
bool hayPaqueteValido = false;

// ======================
// BLE CALLBACKS
// ======================
class ServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer *pServer) override {
    appConectada = true;
    Serial.println("BLE: app conectada.");
  }

  void onDisconnect(BLEServer *pServer) override {
    appConectada = false;
    Serial.println("BLE: app desconectada. Reiniciando advertising...");
    delay(100);
    pServer->startAdvertising();
  }
};

class RXCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *pCharacteristic) override {
    String comando = String(pCharacteristic->getValue().c_str());
    comando.trim();
    comando.toUpperCase();

    Serial.print("BLE comando recibido: ");
    Serial.println(comando);

    if (comando == "ON" || comando == "1" || comando == "START") {
      sistemaActivo = true;
      Serial.println("Sistema activado desde app.");
      enviarEstadoBLE("sistema_activado");
    }
    else if (comando == "OFF" || comando == "0" || comando == "STOP") {
      sistemaActivo = false;
      Serial.println("Sistema apagado desde app.");
      enviarEstadoBLE("sistema_apagado");
    }
    else if (comando == "PING" || comando == "ESTADO") {
      enviarEstadoBLE("estado");
    }
    else {
      Serial.println("Comando no reconocido. Usa ON, OFF, PING o ESTADO.");
      enviarEstadoBLE("comando_no_reconocido");
    }
  }
};

// ======================
// CONVERSIÓN A TEXTO
// ======================
const char* textoSonido(uint8_t sonido) {
  switch (sonido) {
    case SONIDO_BOSQUE:
      return "bosque";
    case SONIDO_ACTIVIDAD:
      return "actividad_humana";   // motosierra + maquinaria agrupadas
    case SONIDO_RESERVADO:
      return "reservado";
    case SONIDO_VOZ:
      return "voz_humana";
    default:
      return "desconocido";
  }
}

const char* textoSiNo(uint8_t valor) {
  return valor ? "si" : "no";
}

const char* textoPeligro(uint8_t peligro) {
  switch (peligro) {
    case PELIGRO_NORMAL:
      return "normal";
    case PELIGRO_TALA:
      return "tala";
    case PELIGRO_BATERIA:
      return "bateria";
    default:
      return "desconocido";
  }
}

// ======================
// CREAR JSON PARA APP INVENTOR
// Mantiene las claves antiguas y agrega contador, satelites, hdop y activo.
// ======================
String crearJSON(const YuraPacket &data) {
  String json;
  json.reserve(220);

  json += "{";
  json += "\"contador\":" + String(data.contador) + ",";
  json += "\"sonido\":\"" + String(textoSonido(data.sonido)) + "\",";
  json += "\"presencia\":\"" + String(textoSiNo(data.presencia)) + "\",";
  json += "\"caida\":\"" + String(textoSiNo(data.caida)) + "\",";
  json += "\"peligro\":\"" + String(textoPeligro(data.peligro)) + "\",";
  json += "\"bateria\":" + String(data.bateria) + ",";
  json += "\"lat\":" + String(data.lat, 6) + ",";
  json += "\"lon\":" + String(data.lng, 6) + ",";
  json += "\"alt\":" + String(data.alt, 1) + ",";
  json += "\"satelites\":" + String(data.satelites) + ",";
  json += "\"hdop\":" + String(data.hdop, 2) + ",";
  json += "\"activo\":\"" + String(sistemaActivo ? "si" : "no") + "\"";
  json += "}";

  return json;
}

String crearJSONEstado(const char *evento) {
  String json;
  json.reserve(120);

  json += "{";
  json += "\"evento\":\"" + String(evento) + "\",";
  json += "\"ble\":\"" + String(appConectada ? "conectado" : "desconectado") + "\",";
  json += "\"sistema\":\"" + String(sistemaActivo ? "activo" : "apagado") + "\",";
  json += "\"nrf\":\"" + String(nrfOk ? "ok" : "no_detectado") + "\",";
  json += "\"hay_datos\":\"" + String(hayPaqueteValido ? "si" : "no") + "\"";
  json += "}";

  return json;
}

// ======================
// ENVIAR A LA APP POR BLE
// ======================
void enviarBLE(const String &json, bool forzarEnvio = false) {
  if (!appConectada) {
    Serial.println("BLE: no se envia, app no conectada.");
    return;
  }

  if (!sistemaActivo && !forzarEnvio) {
    Serial.println("BLE: no se envia paquete, sistema apagado.");
    return;
  }

  if (txCharacteristic == nullptr) {
    Serial.println("BLE: TX characteristic no inicializada.");
    return;
  }

  txCharacteristic->setValue(json.c_str());
  txCharacteristic->notify();

  Serial.print("BLE JSON enviado: ");
  Serial.println(json);
}

void enviarEstadoBLE(const char *evento) {
  String json = crearJSONEstado(evento);
  enviarBLE(json, true);
}

// ======================
// IMPRIMIR PAQUETE EN SERIAL
// ======================
void imprimirPaquete(const YuraPacket &data) {
  Serial.println("========== PAQUETE RECIBIDO ==========");

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
}

// ======================
// INICIAR NRF24L01
// No bloquea el programa si falla.
// ======================
bool iniciarNRF() {
  SPI.begin(NRF_SCK, NRF_MISO, NRF_MOSI, NRF_CSN);

  if (!radio.begin()) {
    Serial.println("ERROR: NRF24L01 no detectado en receptor negro.");
    Serial.println("Revisa VCC 3.3V, GND, CE, CSN, SCK, MOSI y MISO.");
    nrfOk = false;
    return false;
  }

  radio.setChannel(90);
  radio.setDataRate(RF24_250KBPS);
  radio.setPALevel(RF24_PA_LOW);

  radio.setAutoAck(true);
  radio.setRetries(5, 15);
  radio.setPayloadSize(sizeof(YuraPacket));

  radio.openReadingPipe(1, address);
  radio.startListening();
  radio.flush_rx();

  nrfOk = true;

  Serial.println("NRF24L01 receptor detectado.");
  Serial.print("Tamano paquete receptor: ");
  Serial.print(sizeof(YuraPacket));
  Serial.println(" bytes");

  if (sizeof(YuraPacket) <= 32) {
    Serial.println("Paquete compatible con nRF24L01.");
  } else {
    Serial.println("ERROR: paquete mayor a 32 bytes. nRF24L01 no puede enviarlo.");
  }

  Serial.println("Receptor nRF listo.");
  return true;
}

void reintentarNRFSiHaceFalta() {
  if (nrfOk) return;

  if (millis() - ultimoReintentoNRF >= TIEMPO_REINTENTO_NRF_MS) {
    ultimoReintentoNRF = millis();
    Serial.println("Reintentando iniciar nRF24L01...");
    iniciarNRF();
  }
}

// ======================
// INICIAR BLE
// ======================
void iniciarBLE() {
  BLEDevice::init(DEVICE_NAME);
  BLEDevice::setMTU(185);

  bleServer = BLEDevice::createServer();
  bleServer->setCallbacks(new ServerCallbacks());

  BLEService *service = bleServer->createService(SERVICE_UUID);

  txCharacteristic = service->createCharacteristic(
    CHARACTERISTIC_TX,
    BLECharacteristic::PROPERTY_NOTIFY
  );
  txCharacteristic->addDescriptor(new BLE2902());

  BLECharacteristic *rxCharacteristic = service->createCharacteristic(
    CHARACTERISTIC_RX,
    BLECharacteristic::PROPERTY_WRITE
  );
  rxCharacteristic->setCallbacks(new RXCallbacks());

  service->start();

  BLEAdvertising *advertising = BLEDevice::getAdvertising();
  advertising->addServiceUUID(SERVICE_UUID);
  advertising->setScanResponse(true);
  advertising->setMinPreferred(0x06);
  advertising->setMinPreferred(0x12);

  BLEDevice::startAdvertising();

  Serial.print("BLE listo. Busca el dispositivo: ");
  Serial.println(DEVICE_NAME);
}

// ======================
// LEER PAQUETE NRF
// ======================
bool leerPaqueteNRF(YuraPacket &data) {
  if (!nrfOk) return false;
  if (!radio.available()) return false;

  // Si llegaron varios paquetes, nos quedamos con el último.
  while (radio.available()) {
    radio.read(&data, sizeof(data));
  }

  return true;
}

// ======================
// MENSAJE DE ESTADO SERIAL
// ======================
void imprimirEstadoPeriodico() {
  if (millis() - ultimoMensajeEstado < TIEMPO_MENSAJE_ESTADO_MS) return;
  ultimoMensajeEstado = millis();

  Serial.println("---------- ESTADO RECEPTOR ----------");
  Serial.print("nRF: ");
  Serial.println(nrfOk ? "OK" : "NO DETECTADO");

  Serial.print("BLE conectado: ");
  Serial.println(appConectada ? "SI" : "NO");

  Serial.print("Sistema activo: ");
  Serial.println(sistemaActivo ? "SI" : "NO");

  Serial.print("Hay paquete recibido: ");
  Serial.println(hayPaqueteValido ? "SI" : "NO");

  if (hayPaqueteValido) {
    Serial.print("Ultimo paquete hace: ");
    Serial.print((millis() - ultimoPaqueteMs) / 1000.0, 1);
    Serial.println(" s");
  } else {
    Serial.println("Esperando paquetes del ESP32 morado...");
  }

  Serial.println("--------------------------------------");
  Serial.println();
}

// ======================
// SETUP
// ======================
void setup() {
  Serial.begin(115200);
  delay(1500);

  Serial.println();
  Serial.println("========================================");
  Serial.println("ESP32 NEGRO - RECEPTOR PUENTE YURAGUARD");
  Serial.println("========================================");

  Serial.print("Sistema activo inicial: ");
  Serial.println(sistemaActivo ? "SI" : "NO");

  iniciarNRF();
  iniciarBLE();

  Serial.println("Listo. Esperando paquetes del ESP32 morado...");
  Serial.println();
}

// ======================
// LOOP
// ======================
void loop() {
  reintentarNRFSiHaceFalta();

  YuraPacket data;

  if (leerPaqueteNRF(data)) {
    ultimoPaquete = data;
    hayPaqueteValido = true;
    ultimoPaqueteMs = millis();

    imprimirPaquete(data);

    String json = crearJSON(data);

    Serial.print("JSON creado: ");
    Serial.println(json);

    enviarBLE(json);

    Serial.println("======================================");
    Serial.println();
  }

  imprimirEstadoPeriodico();
}

```
