# Nodo Inteligente de Detección de Incendios

## Identificación del Proyecto
* **Equipo:** E33
* **Proyecto:** P23
* **Asignatura:** Fundamentos de IoT
* **Integrantes:** Fiorella Bovet, Victoria Rojas, Felipe Stuardo, Arthur Urtubia y Javier Vidal

---

## 1. Autoría y Trazabilidad del Firmware (Auditoría de Contribuciones)

Para dar cumplimiento al criterio D7 (Autoría y trazabilidad) y a la retroalimentación de la Evaluación Regular 1, se detalla la contribución real de cada integrante del equipo E33.

### Hito 1: Evaluación Regular 1 — NP1 (Nodo Local)
| Integrante | Módulo o tramo desarrollado | Evidencia |
| :--- | :--- | :--- |
| **Felipe Stuardo** | Gestión inicial del repositorio, organización de carpetas y creación de entregas. | [Historial de commits](https://github.com/FelipeS901/Trabajo-prevencion-incendios/commits/main/) |
| **Arthur Urtubia** | Desarrollo del código base, apoyo en conexiones de hardware y cálculos matemáticos (calibración MQ-2 y umbrales). | [Código histórico de NP1](https://github.com/FelipeS901/Trabajo-prevencion-incendios/blob/main/CODIGOS/archivo-anterior/NP1_Codigo.ino) |
| **Javier Vidal** | Desarrollo del código base y apoyo en las conexiones de hardware. | [Código histórico de NP1](https://github.com/FelipeS901/Trabajo-prevencion-incendios/blob/main/CODIGOS/archivo-anterior/NP1_Codigo.ino) |
| **Fiorella Bovet** | Diseño de la máquina de estados, diagramas, apoyo en presentación y soldadura de componentes. | [Grafo de estados](https://github.com/FelipeS901/Trabajo-prevencion-incendios/blob/main/docs/Grafo-Estados.drawio) |
| **Victoria Rojas** | Documentación de guías, organización, soldadura, presentación de NP1 y apoyo en entregas. | [Documentación del proyecto](https://github.com/FelipeS901/Trabajo-prevencion-incendios/tree/main/docs) |

### Hito 2: Evaluación Regular 2 — NP2 (Sistema IoT Conectado y Persistido)
En NP2, el equipo extendió el nodo local hacia un sistema conectado mediante MQTT y persistencia de datos. La distribución del trabajo se documenta por tramos funcionales y se respalda con el repositorio.

| Integrante | Contribución en NP2 y correcciones de NP1 | Evidencia |
| :--- | :--- | :--- |
| **Arthur Urtubia** | Desarrollo del modelo 3D y apoyo en la integración de mejoras de hardware. | [Carpeta Proyecto 3D](https://github.com/FelipeS901/Trabajo-prevencion-incendios/tree/main/Proyecto%203D) |
| **Javier Vidal** | Apoyo en la configuración MQTT, conexiones y lógica de envío de datos al broker. | [Función `publicarDatos()`](https://github.com/FelipeS901/Trabajo-prevencion-incendios/blob/main/CODIGOS/E33_P23_Firmware.ino#L923-L1005) |
| **Fiorella Bovet** | Edición del firmware, reconexión no bloqueante y actualización del grafo FSM a seis estados. | [Firmware actualizado](https://github.com/FelipeS901/Trabajo-prevencion-incendios/blob/main/CODIGOS/E33_P23_Firmware.ino) · [Grafo FSM](https://github.com/FelipeS901/Trabajo-prevencion-incendios/blob/main/docs/Grafo-Estados.drawio) |
| **Felipe Stuardo** | Documentación de GT4, gestión del repositorio, organización de entregas y diseño 3D. | [Historial del repositorio](https://github.com/FelipeS901/Trabajo-prevencion-incendios/commits/main/) |
| **Victoria Rojas** | Integración y actualización del firmware oficial (sensor MQ-2), control de tags/releases y coordinación final. | [Commits de Victoria Rojas](https://github.com/FelipeS901/Trabajo-prevencion-incendios/commits/main/?author=vctoriaing) |

#### Correcciones de NP1 y Aclaraciones Técnicas
* **Grafo FSM:** Se confirma el uso oficial del grafo de **6 estados principales** (`PRECALENTANDO`, `CALIBRANDO`, `VIGILANDO`, `SOSPECHA`, `ALARMA_CONFIRMADA`, `ERROR_SEGURO`) alineado con el firmware real.
* **Protección de Hardware:** Montaje con filtro RC (resistencia de 1 kΩ y condensador de 100 nF) en la entrada del GPIO 34.
* **Tensión del Regulador (LM2596):** Ajustada físicamente a ~5V estables mediante multímetro por Victoria y Fiorella durante el montaje inicial, sirviendo de fundamento para la constante `VC_SENSOR_MV`.
* **Lógica de Confirmación de Alarma:** Se mantiene la exigencia de 2 sensores simultáneos en estado `SOSPECHA` para evitar falsas alarmas. Como trabajo futuro (Evaluación 3), se contempla un contador de reincidencia (escalamiento por tiempo de exposición al humo) y umbral térmico relativo a la temperatura ambiente.

---

## 2. Descripción Técnica y Operativa del Sistema

### Sensores Utilizados
* **DHT22:** Medición de temperatura y humedad relativa.
* **MQ-2:** Detección de humo y gases combustibles.
* **Sensor IR:** Detección de presencia de llama.

### Funcionamiento de la FSM
El nodo monitorea continuamente las variables ambientales. Si al menos un sensor detecta una condición de peligro, el sistema conmuta automáticamente al estado **SOSPECHA**. La alarma se confirma únicamente si al menos **dos de los tres sensores** indican peligro de forma simultánea. Al confirmarse la alarma (`ALARMA_CONFIRMADA`), se activan el buzzer y el LED de alerta de manera enclavada hasta recibir un rearme manual o remoto vía MQTT con las condiciones ambientales disipadas.

### Estados FSM Publicados (Campo Numérico `estado`)
El campo `estado` en el payload JSON se publica como **valor numérico** según el enum del firmware:
* `0`: `PRECALENTANDO`
* `1`: `CALIBRANDO`
* `2`: `VIGILANDO`
* `3`: `SOSPECHA`
* `4`: `ALARMA_CONFIRMADA`
* `5`: `ERROR_SEGURO`
* `6`: `ACTUANDO_REMOTO`

---

## 3. Umbrales, Tiempos y Criterios de Seguridad

* **Precalentamiento y Calibración:** 3 minutos (180 s) iniciales no bloqueantes para permitir el equilibrio térmico del MQ-2. Posteriormente, se promedian exactamente **60 muestras** (`N_MUESTRAS_CAL`, tomadas cada 50 ms) para establecer la línea base en aire limpio.
* **Umbral Adaptativo MQ-2 (Adimensional $Rs/R0$):** 
  * Se subsanó la interpretación previa de resistencia en kΩ; el firmware opera con la razón adimensional $Rs/R0$.
  * Se define con una caída base del 30% respecto a $R0$ ($1.0$), ajustada dinámicamente sumando 3 desviaciones estándar ($3 \cdot \sigma$) para compensar el ruido del montaje (restringido entre un `MIN: 0.50` y `MAX: 0.85`).
  * *Datos de prueba real:* Con una línea base de 584 mV, se obtuvo un umbral dinámico real de `0.67`, reflejado directamente en el JSON (`"mq2_umbral": 0.67`).
* **Umbrales Críticos:** Temperatura crítica configurada en $40.0^\circ\text{C}$ / $60.0^\circ\text{C}$ según la escala de validación para evitar falsas alarmas ambientales y facilitar la demostración física.
* **Tiempos y Persistencia del Sistema:**
  * Duración del estado de sospecha: 30 segundos (`TIMEOUT_SOSPECHA_MS`).
  * Periodo de muestreo general: 3 segundos.
  * Tolerancia a fallos de lectura: Máximo 8 lecturas consecutivas erróneas (`MAX_INVALIDAS = 8`, equivalente a 24 s) antes de transicionar a `ERROR_SEGURO`.
  * Filtro temporal para alerta: Configurado a 2 minutos para certificar en Grafana que un incremento térmico corresponde a una emergencia real y no a un pico transitorio.

---

## 4. Conectividad, Backend y Alertas (MQTT / InfluxDB)

### Tabla de Tópicos MQTT (GT4)
| Nombre del tópico | QoS | Retained | Descripción |
| :--- | :---: | :---: | :--- |
| `curso/E33/P23/nodo1` | 1 | Sí | JSON plano con temperatura, humedad, razón $Rs/R0$, llama, estado numérico y estado de sensores. |
| `curso/E33/P23/nodo1/estado` | 1 | Sí | Estado de disponibilidad MQTT del nodo: `online`/`offline`. |
| `curso/E33/P23/nodo1/cmd` | 1 | No | Recepción de comandos JSON para control remoto y solicitudes de `rearme`. |

### Esquema en InfluxDB y Grafana
* **Bucket / Measurement:** `mediciones` / `mediciones`
* **Tags:** `equipo=E33`, `proyecto=P23`, `nodo=nodo1`
* **Fields:** Métricas numéricas provenientes del payload JSON.
* **Comportamiento ante Desconexión (NoData):** Si el nodo interrumpe la telemetría durante más de 5 minutos, la consulta Flux (`range(start: -5m)`) deja de recibir registros, provocando que Grafana interprete la ausencia de datos y conmute de manera automática la regla de alerta al estado `NoData`.

---

## 5. Evidencias, Simulaciones y Registros Gráficos

### Simulación Wokwi
* **Calibración de dos puntos:** [Enlace a Wokwi](https://wokwi.com/projects/472554117536717825) (Obtención de parámetros $m = 0.443$, $b = -1.8764$).

### Registros Visuales y Capturas

#### Máquina de Estados
<img width="1531" height="615" alt="MaquinaDeEstados" src="https://github.com/user-attachments/assets/cca822cf-4a65-49e3-a47b-31532f9a3223" />

#### Montaje Físico del Hardware
<img width="1036" height="1600" alt="MontajeProyecto" src="https://github.com/user-attachments/assets/565418c7-54a0-4ea9-8792-dcc32a2bfc59" />

#### Estabilización del Sensor MQ-2 y Payload JSON
<div style="display: flex; gap: 10px;">
  <img width="48%" alt="mq-2 grafico" src="https://github.com/user-attachments/assets/79f17540-4e0d-4706-9485-e6d79fcc9719" />
  <img width="48%" alt="JSON Umbral" src="https://github.com/user-attachments/assets/79da6c68-d173-4a6b-ac76-37ecbb27a474" />
</div>

#### Pruebas de Validación GT2
* **Ítem 6: Transición a Estado de Error:**  
  <img width="826" height="280" alt="Transicion Error" src="https://github.com/user-attachments/assets/ae5123e1-3c97-4a95-a9d8-04b836fef744" />
* **Ítem 7: Fusión de Sensores:**  
  <img width="1022" height="385" alt="Fusion Sensores" src="https://github.com/user-attachments/assets/1941e115-c1e5-4e83-9dc3-21d70cef51ce" />
* **Ítem 8: Histéresis Temporal:**  
  <img width="1022" height="457" alt="Histeresis Temporal" src="https://github.com/user-attachments/assets/0e1f6cda-b810-43fd-9fa8-d3c8b8e311b2" />

---

## 6. Estructura del Repositorio y Documentación

### Archivos de Código Fuente
* **Firmware Oficial:** `CODIGOS/E33_P23_Firmware.ino`
* **Variante equivalente sin comentarios:** `CODIGOS/E33_1510`
* **Versiones históricas/desarrollo:** `CODIGOS/archivo-anterior/codigo-nodo.ino` y `CODIGOS/archivo-anterior/NP1_Codigo.ino`

### Documentación de Respaldo (`docs/`)
* [Lanzamiento del Proyecto (PDF)](docs/E33_P23_Lanzamiento_Proyecto.pdf)
* [Arquitectura Operativa por Capas (Imagen)](docs/Arquitectura%20Operativa%20por%20Capas.jpg)
* Archivos editables:
  * [`docs/Arquitectura-Capas.drawio`](docs/Arquitectura-Capas.drawio) / [`docs/Arquitectura-capas.json`](docs/Arquitectura-capas.json)
  * [`docs/Grafo-Estados.drawio`](docs/Grafo-Estados.drawio) / [`docs/Grafo-Estados.json`](docs/Grafo-Estados.json)
