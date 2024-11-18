
# Control Automatizado de Invernadero con Arduino


## Tabla de Contenidos

1. [Introducción](#introducción)
2. [Inclusión de Librerías y Definición de Variables](#inclusión-de-librerías-y-definición-de-variables)
3. [Configuración de Sensores y Pantalla OLED](#configuración-de-sensores-y-pantalla-oled)
4. [Parámetros de Control y Calibración](#parámetros-de-control-y-calibración)
5. [Funciones Auxiliares](#funciones-auxiliares)
6. [Función `setup()`](#función-setup)
7. [Bucle Principal `loop()`](#bucle-principal-loop)
8. [Lectura de Sensores](#lectura-de-sensores)
9. [Control Automático](#control-automático)
10. [Actualización de Ventiladores y Pantalla](#actualización-de-ventiladores-y-pantalla)
11. [Cambio de Modo de Operación](#cambio-de-modo-de-operación)
12. [Conclusión](#conclusión)

---

## Introducción

Este código permite la automatización de un invernadero utilizando una placa Arduino. El sistema:

- Monitorea la **temperatura** y **humedad** del aire mediante un sensor DHT22.
- Mide la **humedad del suelo** con un sensor HW-080 analógico.
- Controla la velocidad de dos **ventiladores** para regular las condiciones internas.
- Muestra información en una **pantalla OLED**.
- Puede operar en modo **automático** o **manual**.

---

## Inclusión de Librerías y Definición de Variables

Comenzamos incluyendo las librerías necesarias y definiendo las variables y constantes que utilizaremos a lo largo del código.

```cpp
#include "arduino_secrets.h"
#include "thingProperties.h"
#include "DHT.h"
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
```

### Configuración de Pines

```cpp
const int PIN_DHT = 2;           // Pin del sensor DHT22
const int PIN_VENTILADOR1 = 5;   // Pin del ventilador 1
const int PIN_VENTILADOR2 = 6;   // Pin del ventilador 2
const int PIN_SENSOR_SUELO = A0; // Pin del sensor de humedad del suelo
```

### Variables Globales

```cpp
DHT sensorDHT(PIN_DHT, DHT22);   // Inicialización del sensor DHT22
Adafruit_SSD1306 pantalla(128, 64, &Wire, -1); // Configuración de la pantalla OLED

// Variables para el promedio de humedad del suelo
const int NUM_MUESTRAS = 10;
float lecturasSuelo[NUM_MUESTRAS];
int indiceLectura = 0;

// Control de tiempo
unsigned long tiempoUltimaLectura = 0;
```

---

## Configuración de Sensores y Pantalla OLED

### Sensor DHT22

Utilizamos el sensor DHT22 para obtener lecturas de temperatura y humedad del aire.

```cpp
DHT sensorDHT(PIN_DHT, DHT22);
```

### Pantalla OLED

La pantalla OLED mostrará en tiempo real las lecturas y estados del sistema.

```cpp
Adafruit_SSD1306 pantalla(128, 64, &Wire, -1);
```

---

## Parámetros de Control y Calibración

Establecemos los parámetros que definirán el comportamiento del sistema.

### Tiempos y Temperaturas

```cpp
const int TIEMPO_MUESTREO = 2000;    // Tiempo entre lecturas (ms)
const float TEMP_MINIMA = 20.0;      // Temperatura mínima deseada (°C)
const float TEMP_MAXIMA = 35.0;      // Temperatura máxima deseada (°C)
```

### Humedad del Suelo

```cpp
const int HUMEDAD_SUELO_MINIMA = 30; // Humedad mínima del suelo (%)
```

### Calibración del Sensor de Suelo

Determinamos los valores analógicos que corresponden a suelo seco y mojado para calibrar el sensor.

```cpp
const int SUELO_SECO = 1020;   // Valor analógico cuando el suelo está seco
const int SUELO_MOJADO = 632;  // Valor analógico cuando el suelo está mojado
```

---

## Funciones Auxiliares

### Conversión de Rangos

Esta función nos permite mapear un valor de un rango a otro.

```cpp
float convertirRango(float valor, float minEntrada, float maxEntrada, float minSalida, float maxSalida) {
    return (valor - minEntrada) * (maxSalida - minSalida) / (maxEntrada - minEntrada) + minSalida;
}
```

### Configuración de la Pantalla

Inicializamos y configuramos la pantalla OLED.

```cpp
void configurarPantalla() {
    if (!pantalla.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
        while (1); // Si falla la inicialización, se detiene el programa
    }
    pantalla.setTextSize(1);
    pantalla.setTextColor(SSD1306_WHITE);
}
```

---

## Función `setup()`

En la función `setup()`, inicializamos los sensores, la pantalla y establecemos el modo de operación.

```cpp
void setup() {
    sensorDHT.begin(); // Iniciamos el sensor DHT22
    
    pinMode(PIN_VENTILADOR1, OUTPUT);
    pinMode(PIN_VENTILADOR2, OUTPUT);
    pinMode(PIN_SENSOR_SUELO, INPUT);
    
    velFan1 = velFan2 = 0;     // Velocidad inicial de los ventiladores
    automatico = true;         // Modo automático activado por defecto
    
    initProperties();          // Inicialización de propiedades para Arduino Cloud
    ArduinoCloud.begin(ArduinoIoTPreferredConnection);
    
    configurarPantalla();      // Configuramos la pantalla OLED
}
```

---

## Bucle Principal `loop()`

En el bucle `loop()`, se realizan las operaciones principales del sistema.

```cpp
void loop() {
    ArduinoCloud.update(); // Actualizamos la conexión con Arduino Cloud
    
    if (millis() - tiempoUltimaLectura >= TIEMPO_MUESTREO) {
        leerSensores();        // Leemos los sensores
        if (automatico) {
            controlAutomatico(); // Ejecutamos el control automático
        }
        actualizarPantalla();  // Actualizamos la pantalla OLED
        tiempoUltimaLectura = millis();
    }
    
    actualizarVentiladores(); // Actualizamos la velocidad de los ventiladores
}
```

---

## Lectura de Sensores

La función `leerSensores()` obtiene y procesa las lecturas de los sensores.

```cpp
void leerSensores() {
    float tempActual = sensorDHT.readTemperature(); // Leemos la temperatura
    float humedadActual = sensorDHT.readHumidity(); // Leemos la humedad del aire
    
    // Validamos que las lecturas sean correctas
    if (!isnan(tempActual) && !isnan(humedadActual)) {
        temp = tempActual;
        humedad = humedadActual;
    }
    
    // Lectura y promedio de la humedad del suelo
    lecturasSuelo[indiceLectura] = analogRead(PIN_SENSOR_SUELO);
    indiceLectura = (indiceLectura + 1) % NUM_MUESTRAS;
    
    float sumaLecturas = 0;
    for (int i = 0; i < NUM_MUESTRAS; i++) {
        sumaLecturas += lecturasSuelo[i];
    }
    float promedioSuelo = sumaLecturas / NUM_MUESTRAS;
    
    float porcentajeHumedad = convertirRango(promedioSuelo, SUELO_SECO, SUELO_MOJADO, 0, 100);
    
    // Aseguramos que el porcentaje esté entre 0 y 100
    porcentajeHumedad = constrain(porcentajeHumedad, 0, 100);
    
    humedadSuelo = porcentajeHumedad;
}
```

---

## Control Automático

En `controlAutomatico()`, ajustamos la velocidad de los ventiladores basándonos en las lecturas obtenidas.

```cpp
void controlAutomatico() {
    float velocidad;
    
    // Control basado en la temperatura
    if (temp >= TEMP_MAXIMA) {
        velocidad = 100;
    } else if (temp <= TEMP_MINIMA) {
        velocidad = 20;
    } else {
        float proporcionTemp = (temp - TEMP_MINIMA) / (TEMP_MAXIMA - TEMP_MINIMA);
        velocidad = 20 + (proporcionTemp * proporcionTemp * 80);
    }
    
    // Ajuste basado en la humedad del suelo
    if (humedadSuelo >= 95) {
        velocidad = 100;
    } else if (humedadSuelo > 80) {
        velocidad = max(velocidad * 2.0, 80.0f);
    } else if (humedadSuelo > 60) {
        velocidad = max(velocidad * 1.5, 60.0f);
    } else if (humedadSuelo > 30) {
        velocidad *= (0.8 + ((humedadSuelo - 30) / 30) * 0.7);
    } else {
        velocidad *= 0.8;
    }
    
    // Limitamos la velocidad entre 20% y 100%
    velocidad = constrain(velocidad, 20, 100);
    velFan1 = velFan2 = velocidad;
}
```

---

## Actualización de Ventiladores y Pantalla

### Control de Ventiladores

La función `actualizarVentiladores()` convierte la velocidad en porcentaje a un valor PWM y lo aplica a los pines de los ventiladores.

```cpp
void actualizarVentiladores() {
    int pwm1 = int(convertirRango(float(velFan1), 0, 100, 0, 255));
    int pwm2 = int(convertirRango(float(velFan2), 0, 100, 0, 255));
    
    pwm1 = constrain(pwm1, 0, 255);
    pwm2 = constrain(pwm2, 0, 255);
    
    analogWrite(PIN_VENTILADOR1, pwm1);
    analogWrite(PIN_VENTILADOR2, pwm2);
}
```

### Actualización de la Pantalla OLED

La función `actualizarPantalla()` muestra la información relevante en la pantalla.

```cpp
void actualizarPantalla() {
    pantalla.clearDisplay();
    pantalla.setCursor(0, 0);
    
    pantalla.println("INVERNADERO - CONTROL");
    pantalla.println("--------------------");
    pantalla.print("Temp: ");
    pantalla.print(float(temp));
    pantalla.println(" C");
    pantalla.print("Hum. Aire: ");
    pantalla.print(float(humedad));
    pantalla.println("%");
    pantalla.print("Hum. Suelo: ");
    pantalla.print(float(humedadSuelo));
    pantalla.println("%");
    pantalla.print("Vent 1: ");
    pantalla.print(float(velFan1));
    pantalla.println("%");
    pantalla.print("Vent 2: ");
    pantalla.print(float(velFan2));
    pantalla.println("%");
    pantalla.print("Modo: ");
    pantalla.println(automatico ? "Auto" : "Manual");
    
    pantalla.display();
}
```

---

## Cambio de Modo de Operación

La función `onAutomaticoChange()` detecta cuando el usuario cambia entre el modo automático y manual, permitiendo acciones adicionales si es necesario.

```cpp
void onAutomaticoChange() {
    Serial.println("Cambio de modo: " + String(automatico ? "Automatico" : "Manual"));
}
```
