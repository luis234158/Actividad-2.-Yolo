# Detección de Objetos con YOLO + Control de LEDs en ESP32

Sistema que detecta un **carro** y una **moto de juguete** en tiempo real usando YOLOv8 desde la cámara de un computador, y enciende un LED en un ESP32 según el objeto detectado (LED rojo = carro, LED verde = moto), comunicándose por puerto serial (USB).

## Arquitectura del sistema

```
Cámara del PC → YOLOv8 (Python) → detecta objeto →
   envía comando por Serial USB →
   ESP32 (MicroPython) → enciende el LED correspondiente
```

El ESP32 no ejecuta YOLO directamente (no tiene la capacidad de cómputo necesaria). El modelo corre en el computador; el ESP32 solo recibe una señal de texto simple (`C`, `M` o `N`) y actúa sobre los LEDs.

## Hardware utilizado

- ESP32
- 2 pulsadores (usados en la etapa de prueba inicial del circuito)
- 1 LED rojo + resistencia
- 1 LED verde + resistencia
- Cable USB (ESP32 ↔ PC)
- Cámara web
- Carro y moto de juguete (objetos a detectar)

## Paso 1 — Prueba del circuito en Wokwi

Antes de integrar YOLO, se validó el circuito físico (botones y LEDs) simulándolo en [Wokwi](https://wokwi.com/), con este código en MicroPython:

```python
from machine import Pin
import time

btn_rojo = Pin(4, Pin.IN, Pin.PULL_UP)
btn_azul = Pin(5, Pin.IN, Pin.PULL_UP)
led_rojo = Pin(18, Pin.OUT)
led_verde = Pin(19, Pin.OUT)

print("Sistema listo")

while True:
    if btn_rojo.value() == 0:
        led_rojo.value(1)
    else:
        led_rojo.value(0)

    if btn_azul.value() == 0:
        led_verde.value(1)
    else:
        led_verde.value(0)

    time.sleep(0.05)
```

Esto confirmó que el cableado y los pines del ESP32 funcionaban correctamente antes de pasar a la lógica final.

## Paso 2 — Entendimiento de YOLO

Se revisó la explicación de arquitectura de YOLO (división de la imagen en grid, predicción de bounding boxes y clases en una sola pasada de la red). Se usó el modelo preentrenado `yolov8n.pt` de la librería `ultralytics`, que ya viene entrenado sobre el dataset **COCO**, el cual incluye las clases:

- `car` → id de clase `2`
- `motorcycle` → id de clase `3`

Por eso no fue necesario reentrenar el modelo: solo se filtraron esas dos clases en la inferencia.

## Paso 3 — Código final del ESP32 (MicroPython)

Este código queda cargado como `main.py` en el ESP32 y corre de forma autónoma, escuchando el puerto serial:

```python
import sys, select, time
from machine import Pin

led_rojo = Pin(18, Pin.OUT)
led_verde = Pin(19, Pin.OUT)

poll = select.poll()
poll.register(sys.stdin, select.POLLIN)

print("ESP32 listo, esperando señales...")

while True:
    if poll.poll(0):
        linea = sys.stdin.readline().strip()
        if linea == 'C':          # Carro detectado
            led_rojo.value(1)
            led_verde.value(0)
        elif linea == 'M':        # Moto detectada
            led_verde.value(1)
            led_rojo.value(0)
        elif linea == 'N':        # Nada detectado
            led_rojo.value(0)
            led_verde.value(0)
    time.sleep(0.05)
```

## Paso 4 — Código final en el PC (Python + YOLO + Serial)

Este script corre en el computador, procesa el video de la cámara con YOLO y envía la señal correspondiente al ESP32:

```python
import cv2
from ultralytics import YOLO
import serial
import time

# Ajustar el puerto COM según el Administrador de dispositivos
ser = serial.Serial('COM5', 115200, timeout=1)
time.sleep(2)  # esperar a que el ESP32 termine de reiniciar

model = YOLO('yolov8n.pt')
cap = cv2.VideoCapture(0)

CARRO = 2   # id de clase "car" en COCO
MOTO = 3    # id de clase "motorcycle" en COCO

ultimo_enviado = None

print("Presiona 'q' para salir")

while True:
    ret, frame = cap.read()
    if not ret:
        break

    results = model(frame, verbose=False, classes=[CARRO, MOTO])
    annotated = results[0].plot()

    detectado = None
    for box in results[0].boxes:
        cls = int(box.cls[0])
        if cls == CARRO:
            detectado = 'C'
            break
        elif cls == MOTO:
            detectado = 'M'
            break

    # Solo enviar cuando cambia el estado, para no saturar el puerto serial
    if detectado != ultimo_enviado:
        senal = detectado if detectado else 'N'
        ser.write((senal + '\n').encode())
        ultimo_enviado = detectado

    cv2.imshow('Deteccion YOLO', annotated)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
ser.close()
```

## Cómo ejecutar el proyecto

1. Instalar dependencias en el entorno virtual del PC:
   ```
   pip install opencv-python ultralytics pyserial
   ```
2. Cargar `main.py` en el ESP32 (con Thonny o la extensión MicroPython de VS Code).
3. **Cerrar Thonny** (o cualquier programa que tenga el puerto COM abierto) para liberar el puerto.
4. Verificar el número de puerto COM del ESP32 en el Administrador de dispositivos de Windows.
5. Ajustar el puerto en el script de Python (`ser = serial.Serial('COMx', ...)`).
6. Ejecutar el script de Python desde VS Code. Se abrirá la cámara y, al detectar el carro o la moto de juguete, el LED correspondiente se encenderá en el ESP32.

## Problemas comunes y solución

| Error | Causa | Solución |
|---|---|---|
| `could not open port 'COMx'` | El puerto ya está siendo usado por otro programa (ej. Thonny) | Cerrar el otro programa/monitor serial antes de correr el script |
| `ModuleNotFoundError: No module named 'cv2'` | Falta instalar OpenCV en el venv | `pip install opencv-python` |
| `ModuleNotFoundError: No module named 'machine'` | El código se está corriendo en Python del PC en vez de en el ESP32 | Seleccionar el intérprete "MicroPython (ESP32)" en Thonny, o subir el archivo al dispositivo |

## Extensión: control por voz

Como variante adicional, se implementó un control por voz que envía los mismos comandos (`C`, `M`, `N`) al ESP32 usando reconocimiento de voz en el PC, reemplazando la detección de YOLO como fuente de la señal — el código del ESP32 no requiere ningún cambio, ya que interpreta el mismo protocolo serial.
