# 🚦 SafeLight IoT — Sistema de Iluminación Inteligente para Seguridad Vial

> **Arquitectura de Software — Corporación Universitaria Remington**  
> Autoras: Doralis Martínez Pastrana · Julián Morales De La Ossa  
> Docente: Sonia Huérfano

---

## 📋 Descripción del Proyecto

**SafeLight IoT** es un sistema IoT de iluminación inteligente para seguridad vial que protege a los conductores del deslumbramiento nocturno mediante dos capas de protección complementarias:

| Capa | Nombre | Descripción |
|------|--------|-------------|
| Capa 1 | Protección reactiva autónoma | Sensor BH1750 detecta deslumbramiento y activa automáticamente un espejo electrocrómático en menos de 500ms |
| Capa 2 | Comunicación cooperativa V2V | Dos nodos ESP32 se comunican vía WiFi/BLE para coordinar la reducción de luces altas entre vehículos |

### Problema que resuelve
El deslumbramiento vehicular en vías de doble sentido puede dejar a un conductor prácticamente ciego entre 2 y 8 segundos — suficiente para recorrer hasta 178 metros sin visibilidad a 80 km/h.

---

## 🏗️ Arquitectura del Sistema

El sistema implementa dos estilos arquitectónicos complementarios:
- **Pub/Sub (MQTT)** — comunicación asíncrona desacoplada entre nodos y backend
- **Arquitectura en Capas** — independencia entre la capa física y la capa de comunicación

### Diagramas C4

#### Nivel 1 — Contexto
```
Conductor A ──► SafeLight IoT System ──► Conductor B
                      │
                 Broker MQTT
                      │
                  Dashboard
```

#### Nivel 2 — Contenedores
```
[ESP32 Nodo A] ──MQTT──► [Broker Mosquitto] ──► [Backend Node.js] ──► [InfluxDB]
[ESP32 Nodo B] ──MQTT──►        │                                          │
        │                  [Node-RED]                                       │
        └──V2V BLE──────►                        [Dashboard React.js] ◄────┘
```

#### Nivel 3 — Componentes (ESP32)
```
SensorManager ──► ActuatorController
     │                   │
     ▼                   ▼
EventLogger        CircuitBreaker
     │                   │
     └──► MQTTPublisher ◄┘
               │
          V2VModule
```

> 📁 Los diagramas completos en formato draw.io se encuentran en la carpeta `/docs/diagramas/`

---

## 🔧 Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Dispositivo IoT | ESP32 DevKit + BH1750 + HC-SR04 + PWM |
| Firmware | Arduino C++ · PlatformIO · FreeRTOS · PubSubClient |
| Broker MQTT | Eclipse Mosquitto 2.x (local) / HiveMQ Cloud |
| Backend | Node.js + Express · Node-RED |
| Base de datos | InfluxDB 2.7 (series temporales) |
| Frontend | React.js · MQTT.js · Recharts |
| Contenedores | Docker · Docker Compose |

---

## 🚀 Instalación y Demo

### Requisitos previos
- Docker y Docker Compose instalados
- Node.js 18+ (solo para desarrollo local)
- Git

### 1. Clonar el repositorio

```bash
git clone https://github.com/TU_USUARIO/safelight-iot.git
cd safelight-iot
```

### 2. Configurar variables de entorno

```bash
cp .env.example .env
# Editar .env con tus credenciales
```

```env
INFLUX_TOKEN=tu_token_aqui
INFLUX_PASSWORD=tu_password_aqui
```

### 3. Levantar el stack completo

```bash
docker-compose up -d
```

### 4. Verificar que todos los servicios están activos

```bash
docker-compose ps
```

Deberías ver los 5 servicios en estado `healthy`:

| Servicio | Puerto | Estado |
|----------|--------|--------|
| safelight-mqtt | 1883, 9001 | healthy |
| safelight-backend | 3001 | healthy |
| safelight-influxdb | 8086 | healthy |
| safelight-nodered | 1880 | healthy |
| safelight-dashboard | 3000 | healthy |

### 5. Acceder al Dashboard

```
http://localhost:3000
```

### 6. Monitorear tópicos MQTT en tiempo real

```bash
docker exec safelight-mqtt mosquitto_sub -h localhost -t 'safelight/#' -v
```

---

## 📡 Prueba MQTT Mínima

Para publicar un evento de prueba manualmente:

```bash
docker exec safelight-mqtt mosquitto_pub \
  -h localhost \
  -t "safelight/nodoA/eventos" \
  -m '{"node_id":"safelight_vehiculo_A","timestamp":1711234567890,"lux":2800,"threshold":1000,"event_type":"DESLUMBRAMIENTO_DETECTADO","actuator":"DIMMED","layer1_ok":true,"cb_state":"CLOSED"}'
```

---

## 📐 Decisiones Arquitectónicas (ADRs)

| ID | Decisión | Alternativas descartadas | Resultado |
|----|----------|--------------------------|-----------|
| ADR-01 | MQTT en lugar de HTTP REST | HTTP REST (bloqueante), CoAP | Comunicación asíncrona no bloqueante |
| ADR-02 | Capas independientes C1/C2 | Módulo monolítico | Disponibilidad >99% garantizada |
| ADR-03 | Circuit Breaker en firmware | Reintento exponencial | Capa 1 SLA <500ms irrompible |
| ADR-04 | Gateway Config centralizado | Dashboard contacta nodos directo | Mantenibilidad OTA sin intervención física |
| ADR-05 | ESP32 WiFi/BLE integrado | NRF24, LoRa | Bajo costo, OTA nativo, comunidad amplia |

---

## 📊 Atributos de Calidad

| ID | Atributo | Medida | Estado |
|----|----------|--------|--------|
| AQ-01 | Rendimiento | Respuesta Capa 1 < 500ms | ✅ Medido: ~120ms |
| AQ-02 | Disponibilidad | Sistema > 99% operación nocturna | ✅ Circuit Breaker + degradación graciosa |
| AQ-03 | Confiabilidad | Falsos positivos < 1% | ✅ Filtro media móvil 5 muestras |
| AQ-04 | Seguridad vial | No oscurecer campo visual completo | ✅ Límites hardware en firmware |
| AQ-05 | Mantenibilidad | Firmware OTA sin intervención física | ✅ Gateway Config + MQTT |
| AQ-06 | Interoperabilidad | MQTT 3.1.1 / 5.0 estándar | ✅ JSON Schema estructurado |

---

## 🐋 Docker Compose — Comandos útiles

```bash
# Ver logs en tiempo real
docker-compose logs -f

# Reiniciar un servicio específico
docker-compose restart backend

# Ver uso de CPU y RAM
docker stats --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'

# Detener y limpiar todo
docker-compose down -v
```

---

## 📁 Estructura del Repositorio

```
safelight-iot/
├── firmware/                  # Código Arduino C++ para ESP32
│   ├── src/
│   │   ├── SensorManager.cpp
│   │   ├── ActuatorController.cpp
│   │   ├── V2VModule.cpp
│   │   ├── CircuitBreaker.cpp
│   │   ├── MQTTPublisher.cpp
│   │   └── EventLogger.cpp
│   └── platformio.ini
├── backend/                   # Node.js + Express
│   ├── src/
│   │   ├── mqttSubscriber.js
│   │   ├── eventProcessor.js
│   │   ├── gatewayConfig.js
│   │   └── dbConnector.js
│   └── package.json
├── dashboard/                 # React.js
│   ├── src/
│   │   ├── RealTimeMonitor.jsx
│   │   └── ConfigPanel.jsx
│   └── package.json
├── nodered/
│   └── flows.json
├── mosquitto/
│   └── config/mosquitto.conf
├── docs/
│   └── diagramas/
│       ├── SafeLight_C4_Nivel1_Contexto.drawio
│       ├── SafeLight_C4_Nivel2_Contenedores.drawio
│       ├── SafeLight_C4_Nivel3_Componentes.drawio
│       └── SafeLight_Vista_Despliegue.drawio
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 👥 Autores

- **Doralis Martínez Pastrana** — [@TU_USUARIO](https://github.com/DoralisM)
- **Julián Morales De La Ossa** — [@TU_USUARIO2](https://github.com/TU_USUARIO2)

---

## 📚 Referencias

- [Documentación MQTT — HiveMQ](https://www.hivemq.com/mqtt/)
- [ESP32 Arduino Core](https://github.com/espressif/arduino-esp32)
- [Simon Brown — Modelo C4](https://c4model.com/)
- [SEI — ATAM Method](https://www.sei.cmu.edu/our-work/projects/display.cfm?customel_datapageid_4050=6542)
- [Eclipse Mosquitto](https://mosquitto.org/)
