# LABORATORIO--7-SENSORES

# Guía 7 — Medición de volumen con sensores óptico y de ultrasonido

Instrumento que mide el **volumen de arroz o fríjol** dentro de un recipiente cilíndrico sin tocar el producto. Lo hace con dos tecnologías: un **sensor óptico infrarrojo** (Sharp GP2Y0A21YK0F) y un **sensor de ultrasonido** (HC-SR04). El volumen se muestra en una LCD 16×2, y con un solo pulsador se cambia de sensor y de producto.

**Curso:** Sensores y Laboratorio, Ingeniería Mecatrónica, Universidad Militar Nueva Granada (UMNG), 2026-2
**Autoras:** Ana Sofía Aguirre Londoño · María José Salazar

---

## Contenido

- [Cómo funciona](#cómo-funciona)
- [Hardware](#hardware)
- [Conexiones](#conexiones)
- [Acondicionamiento del sensor óptico](#acondicionamiento-del-sensor-óptico)
- [Instalación](#instalación)
- [Uso](#uso)
- [Calibración por Monitor Serie](#calibración-por-monitor-serie)
- [Ajuste a marca](#ajuste-a-marca)
- [Caracterización](#caracterización)
- [Estructura del código](#estructura-del-código)
- [Solución de problemas](#solución-de-problemas)

---

## Cómo funciona

Los dos sensores están en la tapa del recipiente, mirando hacia abajo, y miden la distancia **d** hasta la superficie del grano. A partir de esa distancia se obtiene la altura del producto **h** y luego el volumen:

```
V = A · h        A = π · D² / 4 = π · (7.5 cm)² / 4 ≈ 44.18 cm²
```

| Parámetro | Valor |
|---|---|
| Diámetro interior del recipiente | 7.5 cm |
| Distancia sensores → fondo | 29 cm (óptico) · 28 cm (ultrasonido) |
| Rango de medida (nivel) | 3 cm (LRV) a 15 cm (URV) |
| Rango de medida (volumen) | 132.5 cm³ a 662.7 cm³ |
| Resolución del display | 0.5 cm³ |

- **Sensor óptico:** emite luz infrarroja y, por triangulación, entrega un voltaje que **aumenta** cuando el producto se acerca. La relación no es lineal (varía como 1/d), así que el programa interpola la tabla de calibración usando la variable `1/(29 − h)`, en la que el sensor es casi lineal.
- **Sensor de ultrasonido:** emite un pulso de 40 kHz y mide cuánto tarda el eco en volver. El tiempo **disminuye** de forma lineal cuando sube el nivel (t = 2d/v). Para descartar ecos erróneos, en cada medida se toma la mediana de 5 disparos.

Cada producto (arroz y fríjol) tiene su propia tabla de calibración para cada sensor, guardada en la memoria del ESP32.

---

## Hardware

| Componente | Detalle |
|---|---|
| Microcontrolador | ESP32 DevKit (32 bits) |
| Sensor óptico | Sharp GP2Y0A21YK0F (10 a 80 cm, salida analógica) |
| Sensor de ultrasonido | HC-SR04 (40 kHz) |
| Acondicionamiento | Amplificador operacional LM324 |
| Pantalla | LCD 16×2 con módulo I²C (PCF8574) |
| Pulsador | SW1, normalmente abierto |
| Divisor del Echo | Resistencias de 10 kΩ y 20 kΩ |

---

## Conexiones

| Señal | Pin del ESP32 | Nota |
|---|---|---|
| Sharp → LM324 → V_B | GPIO34 | Entrada analógica (ADC1), atenuación de 11 dB |
| HC-SR04 Trig | GPIO26 | Salida digital |
| HC-SR04 Echo | GPIO27 | Pasa por el divisor 10 kΩ / 20 kΩ (5 V → 3.3 V) |
| Pulsador SW1 | GPIO25 | Al otro lado a GND (usa pull-up interno) |
| LCD SDA | GPIO21 | I²C |
| LCD SCL | GPIO22 | I²C, dirección 0x27 (si no aparece nada, probar 0x3F) |

> El Sharp y el HC-SR04 se alimentan con 5 V. Todas las tierras deben ser comunes.

---

## Acondicionamiento del sensor óptico

La salida del Sharp (V_s) solo varía entre 0.90 V y 1.79 V en el rango de medida. El LM324 le resta un nivel de referencia y la amplifica para que use casi todo el rango del ADC:

```
V_B = G · (V_s − V_z)        G ≈ 3.09     V_z ≈ 0.82 V
```

| Punto | V_s (sensor) | V_B (GPIO34) |
|---|---|---|
| h = 3 cm | 0.90 V | 0.25 V |
| h = 15 cm | 1.79 V | 3.00 V |

Resistencias ajustadas en este montaje (03-nov-2026): **R9 = 18 kΩ**, **R8 = 10 kΩ**, **R2 = 39 kΩ**, **R3 = 4.7 kΩ**.

---

## Instalación

1. Instalar **Arduino IDE 2.x** y el paquete de placas **esp32** de Espressif.
2. Instalar la librería **LiquidCrystal_I2C** (Frank de Brabander) desde el gestor de librerías. `Wire` y `Preferences` vienen incluidas con el paquete del ESP32.
3. Abrir el archivo `.ino`, seleccionar **ESP32 Dev Module** y el puerto COM.
4. Subir el programa. Si se queda en *Connecting…*, mantener presionado el botón BOOT de la placa.

---

## Uso

### Pantalla

```
ARROZ  OPT   OK          ← producto · tecnología · estado
309.5cm3 h 7.0cm         ← volumen y altura del grano
```

| Estado | Significado |
|---|---|
| `OK` | Medida dentro del rango de 3 a 15 cm |
| `BAJO` | Nivel por debajo de 3 cm |
| `ALTO` | Nivel por encima de 15 cm |
| `ECO?` | El ultrasonido no recibió eco |

### Pulsador SW1

| Pulsación | Acción |
|---|---|
| Corta | Cambia de tecnología: **óptico ↔ ultrasonido** |
| Larga (≥ 2 s) | Cambia de producto: **arroz ↔ fríjol** |
| Muy larga (≥ 5 s) | Entra o sale de la **vista de calibración**, que muestra el valor crudo (mV o µs) y la altura real |

### Monitor Serie

A 115200 baudios, el programa imprime una línea por cada actualización de la pantalla:

```
producto,modo,raw,h_real_cm,h_lcd_cm,v_lcd_cm3
ARROZ,OPT,817.0,7.02,7.0,309.5
```

`raw` es el valor crudo: mV en GPIO34 para el óptico y µs de eco para el ultrasonido.

---

## Calibración por Monitor Serie

Abrir el Monitor Serie a **115200 baudios**, con el fin de línea en **«Nueva línea»**.

| Comando | Acción |
|---|---|
| `p` | Muestra el producto activo |
| `p a` / `p f` | Selecciona arroz o fríjol |
| `c <h>` | Mide **los dos sensores** y guarda el punto de altura `h` del producto activo (niveles válidos: 3, 5, 7, 9, 11, 13 y 15) |
| `t` | Imprime las tablas de los dos productos en formato CSV, listas para copiar al informe |
| `r` | Borra la tabla del producto activo y vuelve a la tabla teórica |
| `s <cm>` | Cambia la tolerancia del ajuste a marca (0 lo desactiva) |

**Procedimiento:**

1. Marcar en el recipiente las alturas 3, 5, 7, 9, 11, 13 y 15 cm.
2. Seleccionar el producto con `p a` (arroz) o `p f` (fríjol).
3. Llenar hasta la marca de 3 cm, nivelar la superficie con cuidado y escribir `c 3`.
4. Repetir con las demás alturas: `c 5`, `c 7`, …, `c 15`.
5. Escribir `t` para ver y copiar la tabla.
6. Repetir todo con el otro producto.

Las tablas quedan guardadas en la memoria del ESP32 y no se pierden al apagarlo. Si un punto se calibra mal, se puede volver a medir con el mismo comando `c <h>`.

---

## Ajuste a marca

Si la altura medida queda a **±0.5 cm** de una marca calibrada, la LCD muestra la altura exacta de la marca y su volumen. Por ejemplo, una lectura de 6.5 cm aparece como 7.0 cm y 309.5 cm³. Fuera de esa banda se muestra la altura real. Tiene una histéresis de 0.15 cm para que el valor no parpadee en el borde.

> Para caracterizar los sensores o calcular errores, **no uses los valores de la pantalla con el ajuste activo**. Usa la vista de calibración (pulsación ≥ 5 s), la columna `h_real_cm` del Monitor Serie, o desactiva el ajuste con `s 0`.

---

## Caracterización

| Sensor óptico | Sensor de ultrasonido |
|---|---|
| ![Óptico](img/grafica_optico_final.png) | ![Ultrasonido](img/grafica_ultrasonido_final.png) |
| Lecturas del multímetro entre 3 y 7 cm, con el modelo ajustado `V = 32.91/(29 − h) + 0.367` | Tiempos de eco teóricos del programa (17 °C): `t = −58.55·h + 1639.8` µs |

---

## Estructura del código

| Función | Qué hace |
|---|---|
| `leerMilivoltios()` | Promedia 64 lecturas del ADC en GPIO34 |
| `dispararEco()` | Envía el pulso de Trig de 10 µs y mide el ancho del Echo con `pulseIn` |
| `mediana()` | Calcula la mediana de los disparos del ultrasonido |
| `mvANivel()` | Convierte mV en altura, interpolando en `1/(29 − h)` |
| `usANivel()` | Convierte µs en altura, con interpolación lineal |
| `calcularVolumen()` | Calcula el volumen, aplica un filtro de promedio y redondea a 0.5 cm³ |
| `ajustarAMarca()` | Aplica el ajuste a marca con histéresis |
| `capturarPunto()` | Mide los dos sensores y guarda un punto de calibración |
| `leerBoton()` | Antirrebote y detección de pulsación corta, larga y muy larga |
| `mostrar()` | Actualiza la LCD y envía la línea CSV por el puerto serie |
| `cargarTablas()` / `guardarTablas()` | Leen y guardan las tablas en la memoria del ESP32 (`Preferences`) |

---

## Solución de problemas

| Síntoma | Qué revisar |
|---|---|
| La LCD enciende pero no muestra texto | Dirección I²C: cambiar `0x27` por `0x3F`. Ajustar el potenciómetro de contraste del módulo |
| Aparece `ECO?` | Conexión de Trig/Echo y del divisor. Que el HC-SR04 apunte derecho al producto, sin tocar las paredes |
| La lectura óptica salta mucho | Tierra común, cables cortos y un capacitor de desacople junto al Sharp |
| Aparece `BAJO` o `ALTO` con el recipiente en rango | Recalibrar ese producto con `c <h>`, o reiniciar la tabla con `r` |
| El aviso «la tabla aún no es monótona» | Es normal mientras faltan puntos; desaparece cuando se calibran todas las alturas en orden |
| El volumen no cambia entre marcas cercanas | Es el ajuste a marca; desactivarlo con `s 0` |
