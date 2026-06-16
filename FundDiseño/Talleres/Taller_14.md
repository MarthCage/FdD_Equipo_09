# ACTIVIDAD 1:


# ACTIVIDAD 2: Control de ángulo de un servomotor con ESP32 mediante Bluetooth

## 1. Descripción general

En esta actividad se desarrolló un sistema para controlar el ángulo de giro de un servomotor utilizando un ESP32 y una aplicación móvil creada en MIT App Inventor.

La comunicación entre la aplicación y el ESP32 se realiza mediante Bluetooth clásico. Desde la aplicación, el usuario mueve un control deslizante o *slider*, el cual envía un valor numérico al ESP32. Luego, el ESP32 recibe ese valor, lo procesa y mueve el servomotor al ángulo correspondiente.

El sistema permite controlar el servomotor de manera inalámbrica, usando un teléfono Android como interfaz de control.

---

## 2. Código utilizado en el ESP32

```cpp
#include "BluetoothSerial.h"
#include <ESP32Servo.h>

BluetoothSerial SerialBT;
Servo servoMotor;

int pinServo = 13;
int angulo = 0;

void setup() {
  Serial.begin(115200);

  SerialBT.begin("ESP32_SERV0");
  Serial.println("Bluetooth iniciado como ESP32_SERVO");

  servoMotor.attach(pinServo);
  servoMotor.write(0);

  Serial.println("Servo listo");
}

void loop() {
  if (SerialBT.available()) {
    String dato = SerialBT.readStringUntil('\n');
    dato.trim();

    if (dato.length() > 0) {
      angulo = dato.toInt();

      if (angulo < 0) {
        angulo = 0;
      }

      if (angulo > 270) {
        angulo = 270;
      }

      int anguloServo = map(angulo, 0, 270, 0, 180);

      servoMotor.write(anguloServo);

      Serial.print("Ángulo recibido: ");
      Serial.println(angulo);
    }
  }
}
```

---

## 3. Explicación del código

### 3.1 Librerías utilizadas

Al inicio del programa se incluyen dos librerías:

```cpp
#include "BluetoothSerial.h"
#include <ESP32Servo.h>
```

La librería `BluetoothSerial.h` permite usar la comunicación Bluetooth clásica del ESP32. Gracias a esta librería, el ESP32 puede recibir datos enviados desde la aplicación móvil.

La librería `ESP32Servo.h` permite controlar el servomotor conectado al ESP32. Esta librería es necesaria porque el manejo de servomotores en ESP32 requiere una configuración diferente a la utilizada en placas como Arduino UNO.

---

### 3.2 Creación de objetos

Después se crean dos objetos principales:

```cpp
BluetoothSerial SerialBT;
Servo servoMotor;
```

El objeto `SerialBT` se utiliza para manejar la comunicación Bluetooth entre el celular y el ESP32.

El objeto `servoMotor` representa al servomotor que será controlado desde el programa.

---

### 3.3 Declaración de variables

Luego se declaran las variables principales del sistema:

```cpp
int pinServo = 13;
int angulo = 0;
```

La variable `pinServo` indica que el cable de señal del servomotor está conectado al pin GPIO 13 del ESP32.

La variable `angulo` almacena el valor recibido desde la aplicación móvil. Este valor corresponde a la posición seleccionada en el slider.

---

### 3.4 Configuración inicial del programa

La función `setup()` se ejecuta una sola vez al iniciar el ESP32. En esta parte se configuran la comunicación serial, el Bluetooth y el servomotor.

Primero se inicia la comunicación serial:

```cpp
Serial.begin(115200);
```

Esta instrucción permite mostrar mensajes en el monitor serial del Arduino IDE. Sirve para verificar si el ESP32 está funcionando correctamente y si recibe los datos enviados por Bluetooth.

Luego se inicia el Bluetooth del ESP32 con el nombre `ESP32_SERVO`:

```cpp
SerialBT.begin("ESP32_SERVO");
Serial.println("Bluetooth iniciado como ESP32_SERVO");
```

Este nombre será visible desde el celular al momento de buscar dispositivos Bluetooth. Por ello, en la aplicación móvil se debe seleccionar el dispositivo llamado `ESP32_SERVO`.

Después se configura el pin del servomotor:

```cpp
servoMotor.attach(pinServo);
servoMotor.write(0);
```

La instrucción `servoMotor.attach(pinServo)` indica que el servomotor será controlado desde el pin GPIO 13.

La instrucción `servoMotor.write(0)` coloca inicialmente el servomotor en la posición 0.

Finalmente, se muestra un mensaje indicando que el servo ya está listo:

```cpp
Serial.println("Servo listo");
```

---

### 3.5 Lectura de datos por Bluetooth

La función `loop()` se ejecuta de manera repetitiva mientras el ESP32 está encendido. Dentro de esta función se verifica constantemente si llegó algún dato por Bluetooth:

```cpp
if (SerialBT.available()) {
```

La instrucción `SerialBT.available()` comprueba si existe información disponible para leer. Si la aplicación móvil envía un valor desde el slider, esta condición se cumple.

Cuando hay información disponible, el programa lee el dato recibido:

```cpp
String dato = SerialBT.readStringUntil('\n');
dato.trim();
```

La instrucción `readStringUntil('\n')` lee el texto recibido hasta encontrar un salto de línea. Este salto de línea es enviado desde MIT App Inventor junto con el valor del slider.

La instrucción `dato.trim()` elimina espacios o caracteres innecesarios al inicio y al final del dato recibido. Esto ayuda a evitar errores al convertir el texto en número.

---

### 3.6 Validación del dato recibido

Antes de usar el dato, se verifica que no esté vacío:

```cpp
if (dato.length() > 0) {
```

Esta condición evita que el programa intente procesar un dato vacío. Si el dato tiene contenido, se convierte de texto a número entero:

```cpp
angulo = dato.toInt();
```

Esto es necesario porque los datos enviados por Bluetooth llegan como texto. Para controlar el servomotor, el valor debe convertirse a un número entero.

Por ejemplo, si la aplicación envía el texto `"90"`, el ESP32 lo convierte al número `90`.

---

### 3.7 Límite del ángulo permitido

Luego, el programa verifica que el ángulo recibido esté dentro del rango permitido:

```cpp
if (angulo < 0) {
  angulo = 0;
}

if (angulo > 270) {
  angulo = 270;
}
```

Estas condiciones evitan que el valor del ángulo sea menor que 0 o mayor que 270.

Si el valor recibido es menor que 0, el programa lo corrige a 0.

Si el valor recibido es mayor que 270, el programa lo corrige a 270.

Esto permite proteger el funcionamiento del sistema y mantener el movimiento dentro del rango definido para la actividad.

---

### 3.8 Conversión del rango del slider al rango del servo

El slider de la aplicación trabaja con valores de 0 a 270. Sin embargo, la función `servoMotor.write()` normalmente trabaja con valores de 0 a 180.

Por esa razón, se utiliza la función `map()`:

```cpp
int anguloServo = map(angulo, 0, 270, 0, 180);
```

Esta instrucción convierte proporcionalmente el valor recibido desde la aplicación.

Ejemplos:

| Valor enviado por la app | Valor usado por el servo |
|---|---|
| 0 | 0 |
| 135 | 90 |
| 270 | 180 |

De esta manera, el valor del slider se adapta al rango que puede interpretar correctamente la librería del servomotor.

---

### 3.9 Movimiento del servomotor

Después de convertir el valor, el servomotor se mueve al ángulo calculado:

```cpp
servoMotor.write(anguloServo);
```

Esta instrucción envía la señal al servomotor para que gire a la posición correspondiente.

Por ejemplo, si desde la aplicación se envía el valor 135, el programa lo convierte aproximadamente a 90, y el servomotor se mueve a esa posición.

---

### 3.10 Visualización en el monitor serial

Finalmente, el programa muestra el valor recibido en el monitor serial:

```cpp
Serial.print("Ángulo recibido: ");
Serial.println(angulo);
```

Esto permite comprobar que el ESP32 está recibiendo correctamente los datos desde la aplicación móvil.

Por ejemplo, si el usuario mueve el slider a 90, en el monitor serial se mostrará:

```text
Ángulo recibido: 90
```

Esta parte es útil para realizar pruebas y verificar que la comunicación Bluetooth funciona correctamente.

---

## 4. Funcionamiento general del sistema

El funcionamiento general del sistema es el siguiente:

1. El usuario abre la aplicación móvil creada en MIT App Inventor.
2. La aplicación se conecta al ESP32 mediante Bluetooth.
3. El usuario mueve el slider para seleccionar un ángulo.
4. La aplicación envía el valor del slider al ESP32.
5. El ESP32 recibe el dato por Bluetooth.
6. El programa convierte el dato recibido en un número entero.
7. El valor se limita entre 0 y 270.
8. El valor se convierte al rango de 0 a 180.
9. El servomotor gira a la posición correspondiente.

---

## 5. Conexión del servomotor

La conexión utilizada para el servomotor fue la siguiente:

| Cable del servomotor | Conexión en el ESP32 |
|---|---|
| Rojo | 5V |
| Marrón o negro | GND |
| Naranja o amarillo | GPIO 13 |

El cable rojo alimenta el servomotor, el cable marrón o negro se conecta a tierra y el cable naranja o amarillo recibe la señal de control desde el ESP32.

## 6. Diseño de app en MIT App Inventor:
<img width="1071" height="762" alt="imagen" src="https://github.com/user-attachments/assets/66fcea82-3739-4835-8471-1a24bf9bd8b1" />

## 7. Programación por bloques:
<img width="925" height="617" alt="imagen" src="https://github.com/user-attachments/assets/26c1927a-6e92-462a-b7c1-44e9f8422076" />
<img width="997" height="552" alt="imagen" src="https://github.com/user-attachments/assets/2f61c134-53fe-487d-8001-5be31018ab69" />
