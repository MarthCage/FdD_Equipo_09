# ACTIVIDAD 1:

No se pudo completar

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

En esta sección se explican los bloques utilizados en MIT App Inventor para controlar el servomotor mediante Bluetooth. La aplicación permite conectarse al ESP32, enviar el valor del slider y actualizar el estado de conexión en pantalla.

Los bloques principales utilizados son:

- `Screen1.Initialize`
- `btn_connect.BeforePicking`
- `btn_connect.AfterPicking`
- `slider.PositionChanged`
- `Clock1.Timer`

--- 

<img width="925" height="617" alt="imagen" src="https://github.com/user-attachments/assets/26c1927a-6e92-462a-b7c1-44e9f8422076" />
<img width="997" height="552" alt="imagen" src="https://github.com/user-attachments/assets/2f61c134-53fe-487d-8001-5be31018ab69" />

---

### 7.1 Bloque `Screen1.Initialize`

El bloque `Screen1.Initialize` se ejecuta automáticamente cuando se abre la aplicación.

```text
when Screen1.Initialize
do
    set lbl_estado.Text to "Estado: Sin conectar"
    set lbl_angulo.Text to "0"
    set slider.MinValue to 0
    set slider.MaxValue to 270
    set slider.ThumbPosition to 0
```

Este bloque se encarga de establecer los valores iniciales de la aplicación.

Primero, la etiqueta `lbl_estado` muestra el mensaje:

```text
Estado: Sin conectar
```

Esto indica que al iniciar la aplicación todavía no existe conexión Bluetooth con el ESP32.

Luego, la etiqueta `lbl_angulo` muestra el valor inicial:

```text
0
```

Este valor representa la posición inicial del slider.

Después, se configura el rango del slider. El valor mínimo se establece en `0` y el valor máximo en `270`. Esto permite que el usuario seleccione un valor dentro del rango definido para la actividad.

Finalmente, la posición inicial del slider se coloca en `0`, para que al abrir la aplicación el control empiece desde el valor mínimo.

---

### 7.2 Bloque `btn_connect.BeforePicking`

El bloque `btn_connect.BeforePicking` se ejecuta antes de que el usuario seleccione un dispositivo Bluetooth.

```text
when btn_connect.BeforePicking
do
    set btn_connect.Elements to BluetoothClient1.AddressesAndNames
```

Este bloque carga en el botón de conexión la lista de dispositivos Bluetooth disponibles o previamente emparejados con el celular.

La propiedad `BluetoothClient1.AddressesAndNames` obtiene los nombres y direcciones de los dispositivos Bluetooth. Luego, esa información se asigna a los elementos del botón `btn_connect`.

Gracias a este bloque, cuando el usuario presiona el botón de conexión, la aplicación muestra una lista de dispositivos Bluetooth disponibles para conectarse.

En esta actividad, el dispositivo que debe seleccionarse es:

```text
ESP32_SERVO
```

Ese es el nombre asignado al ESP32 dentro del código cargado en Arduino IDE.

---

### 7.3 Bloque `btn_connect.AfterPicking`

El bloque `btn_connect.AfterPicking` se ejecuta después de que el usuario selecciona un dispositivo Bluetooth de la lista.

```text
when btn_connect.AfterPicking
do
    if BluetoothClient1.Connect address btn_connect.Selection
    then
        set lbl_estado.Text to "Estado: Conectado"
    else
        set lbl_estado.Text to "Estado: Error D:"
```

Este bloque intenta conectar la aplicación con el dispositivo Bluetooth seleccionado.

La instrucción:

```text
BluetoothClient1.Connect address btn_connect.Selection
```

usa el dispositivo elegido por el usuario en `btn_connect`. Si la conexión se realiza correctamente, la etiqueta `lbl_estado` cambia a:

```text
Estado: Conectado
```

Esto indica que el celular ya se encuentra conectado al ESP32.

Si la conexión falla, la aplicación muestra:

```text
Estado: Error D:
```

Este mensaje indica que no se pudo establecer la conexión Bluetooth. Esto puede suceder si el ESP32 está apagado, si no fue emparejado previamente con el celular o si se seleccionó un dispositivo incorrecto.

---

### 7.4 Bloque `slider.PositionChanged`

El bloque `slider.PositionChanged` se ejecuta cada vez que el usuario mueve el slider.

```text
when slider.PositionChanged thumbPosition
do
    set lbl_angulo.Text to round slider.ThumbPosition

    if BluetoothClient1.IsConnected
    then
        call BluetoothClient1.SendText
            text join round slider.ThumbPosition "\n"
```

Este bloque tiene dos funciones principales. La primera es mostrar en pantalla el valor actual del slider. La segunda es enviar ese valor al ESP32 mediante Bluetooth.

Primero, se actualiza la etiqueta `lbl_angulo`:

```text
set lbl_angulo.Text to round slider.ThumbPosition
```

La función `round` redondea el valor del slider para que se muestre como un número entero. Esto evita que aparezcan valores decimales en la pantalla.

Por ejemplo, si el usuario mueve el slider hacia el valor 90, la etiqueta mostrará:

```text
90
```

Luego, el bloque verifica si existe conexión Bluetooth:

```text
if BluetoothClient1.IsConnected
```

Esta condición es importante porque la aplicación solo debe enviar datos cuando está conectada al ESP32.

Si la conexión está activa, se envía el valor del slider usando:

```text
BluetoothClient1.SendText
```

El dato enviado está formado por el valor redondeado del slider unido con un salto de línea:

```text
join round slider.ThumbPosition "\n"
```

El salto de línea `\n` sirve para indicar el final del dato enviado. Esto permite que el ESP32 lea correctamente la información usando la instrucción:

```cpp
readStringUntil('\n')
```

Por ejemplo, si el usuario mueve el slider al valor 90, la aplicación envía:

```text
90\n
```

El ESP32 recibe ese dato, lo convierte a número entero y mueve el servomotor según el valor recibido.

---

### 7.5 Bloque `Clock1.Timer`

El bloque `Clock1.Timer` se ejecuta de manera repetitiva cada cierto intervalo de tiempo.

```text
when Clock1.Timer
do
    if BluetoothClient1.IsConnected
    then
        set lbl_estado.Text to "Estado: Conectado"
    else
        set lbl_estado.Text to "Estado: Sin conectar"
        set btn_connect.Text to "Conectar"
```

Este bloque sirve para verificar constantemente si la aplicación sigue conectada al ESP32.

Primero, se revisa el estado de la conexión Bluetooth mediante:

```text
BluetoothClient1.IsConnected
```

Si la conexión continúa activa, la etiqueta `lbl_estado` muestra:

```text
Estado: Conectado
```

Esto permite que el usuario sepa que la comunicación con el ESP32 sigue funcionando.

Si la conexión se pierde o todavía no se ha conectado ningún dispositivo, la etiqueta cambia a:

```text
Estado: Sin conectar
```

Además, el texto del botón vuelve a mostrarse como:

```text
Conectar
```

Esto permite que el usuario pueda intentar conectarse nuevamente.

El uso del componente `Clock1` es útil porque mantiene actualizada la interfaz de la aplicación y permite supervisar el estado de la conexión Bluetooth.

---

### 7.6 Funcionamiento general de los bloques

El funcionamiento de los bloques de la aplicación sigue el siguiente proceso:

1. Al abrir la aplicación, el bloque `Screen1.Initialize` configura los valores iniciales.
2. La aplicación muestra el estado inicial como `Estado: Sin conectar`.
3. El slider se configura con un valor mínimo de `0`, un valor máximo de `270` y una posición inicial de `0`.
4. Cuando el usuario presiona el botón de conexión, el bloque `btn_connect.BeforePicking` carga la lista de dispositivos Bluetooth disponibles.
5. El usuario selecciona el dispositivo `ESP32_SERVO`.
6. El bloque `btn_connect.AfterPicking` intenta establecer la conexión Bluetooth con el ESP32.
7. Si la conexión es correcta, la aplicación muestra `Estado: Conectado`.
8. Cuando el usuario mueve el slider, el bloque `slider.PositionChanged` muestra el valor seleccionado en la etiqueta `lbl_angulo`.
9. Si la aplicación está conectada, el valor del slider se envía al ESP32 mediante Bluetooth.
10. El bloque `Clock1.Timer` verifica constantemente si la conexión sigue activa.

---

### 7.7 Relación entre la aplicación y el ESP32

La aplicación móvil y el ESP32 trabajan juntos mediante comunicación Bluetooth.

La aplicación envía valores numéricos desde el slider. Estos valores representan el ángulo seleccionado por el usuario. El ESP32 recibe esos valores como texto, los convierte a número entero y los utiliza para mover el servomotor.

Por ejemplo:

| Acción en la aplicación | Dato enviado al ESP32 | Resultado esperado |
|---|---|---|
| Slider en 0 | `0\n` | El servo se mantiene en la posición inicial |
| Slider en 90 | `90\n` | El servo gira a una posición intermedia |
| Slider en 180 | `180\n` | El servo gira hacia una posición mayor |
| Slider en 270 | `270\n` | El servo llega al valor máximo configurado |

El salto de línea `\n` permite que el ESP32 identifique correctamente el final de cada dato recibido.

---

### 7.8 Importancia del componente `BluetoothClient`

El componente `BluetoothClient1` es el encargado de realizar la comunicación Bluetooth entre la aplicación y el ESP32.

En este proyecto, sus funciones principales son:

- Mostrar los dispositivos Bluetooth disponibles.
- Conectarse al ESP32 seleccionado.
- Verificar si la conexión está activa.
- Enviar el valor del slider al ESP32.

Sin este componente, la aplicación no podría comunicarse de forma inalámbrica con el ESP32.

---

### 7.9 Importancia del componente `Clock1`

El componente `Clock1` permite ejecutar acciones de manera repetitiva cada cierto tiempo.

En esta aplicación se utiliza para verificar si el Bluetooth sigue conectado. Gracias a esto, la aplicación puede actualizar automáticamente el mensaje de estado en la pantalla.

Esto mejora la experiencia del usuario, ya que permite saber si la aplicación está conectada o desconectada del ESP32.

---

## 7.10 Conclusión de los bloques de la aplicación

Los bloques desarrollados en MIT App Inventor permiten que la aplicación móvil funcione como una interfaz de control para el servomotor. Al iniciar la aplicación, se configuran los valores principales; luego, el usuario puede conectarse al ESP32 mediante Bluetooth y enviar el valor del slider.

El componente `BluetoothClient1` permite establecer la comunicación inalámbrica, mientras que el componente `Clock1` permite supervisar el estado de conexión. En conjunto, estos bloques permiten controlar el movimiento del servomotor desde el celular de forma sencilla y en tiempo real.
