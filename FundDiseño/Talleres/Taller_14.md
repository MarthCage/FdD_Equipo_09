## ACTIVIDAD 1:


## ACTIVIDAD 2:
Código Arduino:

Primero se cargaron las librerías que permiten el uso del bluetooth
```
#include "BluetoothSerial.h"
#include <ESP32Servo.h>

BluetoothSerial SerialBT;
Servo servoMotor;

int pinServo = 13;
int angulo = 0;

void setup() {
  Serial.begin(115200);

  SerialBT.begin("ESP32_SERVO");
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

Diseño de app en MIT App Inventor:
<img width="1071" height="762" alt="imagen" src="https://github.com/user-attachments/assets/66fcea82-3739-4835-8471-1a24bf9bd8b1" />

Programación por bloques:
<img width="925" height="617" alt="imagen" src="https://github.com/user-attachments/assets/26c1927a-6e92-462a-b7c1-44e9f8422076" />
<img width="997" height="552" alt="imagen" src="https://github.com/user-attachments/assets/2f61c134-53fe-487d-8001-5be31018ab69" />
