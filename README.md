# Tarea 2: Análisis del Protocolo RTMP bajo Infraestructura de Contenedores

## Descripción del proyecto

Modelamiento, despliegue y análisis promiscuo de tráfico de red para el protocolo de streaming en tiempo real RTMP (Real-Time Messaging Protocol) utilizando una arquitectura aislada basada en contenedores Docker (IaC) y la suite de análisis Wireshark.

## Índice

- [Información General](#información-general)
- [Tecnologías Utilizadas](#tecnologías-utilizadas)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Instalación y Requisitos](#instalación-y-requisitos)
- [Configuración de Archivos Dockerfile](#configuración-de-archivos-dockerfile)
- [Guía de Uso y Despliegue](#guía-de-uso-y-despliegue)
- [Análisis de Tráfico y Hallazgos](#análisis-de-tráfico-y-hallazgos)
- [Autores](#autores)

---

## Información General

### ¿Por qué y cuándo utilizar RTMP?

**¿Por qué?**
RTMP establece conexiones síncronas, persistentes y únicas sobre TCP, minimizando el overhead de negociación y saludos repetitivos. Su arquitectura basada en *chunks* permite multiplexar audio, video y comandos de control en un único canal lógico, evitando bloqueos de datos y garantizando una baja latencia nativa (3 a 5 segundos).

**¿Cuándo?**
En el ecosistema moderno del streaming, RTMP es el estándar absoluto para la fase de contribución o ingesta (*Streaming Uplink*) desde codificadores locales como OBS Studio o FFmpeg hacia los servidores centrales de ingesta de plataformas como YouTube Live, Twitch o redes CDN.

---

## Tecnologías Utilizadas

| Tecnología | Rol en la Arquitectura | Versión Sugerida |
|-----------|-------------------------|------------------|
| Docker / Engine | Virtualización y aislamiento de red enmascarada | v24.x o superior |
| Node.js (Alpine) | Entorno de ejecución para el Servidor RTMP | v18-alpine |
| Node-Media-Server | Software servidor interceptor de flujos RTMP | v2.x |
| FFmpeg (Alpine) | Codificador / cliente emisor sintético de red | v6.x (Alpine 3.19) |
| FFplay | Cliente consumidor nativo (host anfitrión) | Local del sistema |
| Wireshark | Analizador de paquetes promiscuo en la interfaz bridge | v4.x o superior |

---

## Estructura del Proyecto

La topología está diseñada bajo el principio de Infrastructure as Code (IaC) mediante recetas locales:

```text
tarea2-redes/
├── client/
│   └── Dockerfile          # Configuración del emisor automatizado con FFmpeg
├── server/
│   └── Dockerfile          # Configuración del servidor con Node-Media-Server
├── README.md               # Esta guía de documentación del repositorio
└── logo.png                # Identificador gráfico institucional
```

---

## Instalación y Requisitos

### 1. Instalación del motor Docker (Ubuntu/Debian)

Ejecute la siguiente secuencia para preparar el entorno e instalar las herramientas oficiales de Docker en su terminal:

```bash
# Actualizar repositorios e instalar prerrequisitos
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Añadir llave criptográfica oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Añadir el repositorio a las fuentes de APT
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar los binarios del runtime
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin
```

---

## Configuración de Archivos Dockerfile (Infrastructure as Code)

### A. Dockerfile del Servidor (`./server/Dockerfile`)

```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
RUN npm install -g node-media-server
EXPOSE 1935 8000
CMD ["node-media-server"]
```

### B. Dockerfile del Emisor (`./client/Dockerfile`)

```dockerfile
FROM alpine:3.19
RUN apk add --no-cache ffmpeg
CMD ffmpeg -re -f lavfi -i "testsrc=size=640x480:rate=30" -f lavfi -i "sine=frequency=1000:sample_rate=44100" -c:v libx264 -c:a aac -f flv rtmp://rtmp-server/live/prueba
```

---

## Guía de Uso y Despliegue

Siga estrictamente el orden de los comandos para asegurar que las fases lógicas del protocolo queden registradas de forma ordenada.

### Paso 1: Configurar la red aislada de pruebas

Para evitar colisiones de ejecuciones pasadas, limpie preventivamente el daemon de Docker:

```bash
sudo docker network rm red-rtmp 2>/dev/null || true
sudo docker network create red-rtmp
```

### Paso 2: Compilar las imágenes locales (IaC)

Sitúese en el directorio raíz `tarea2-redes/` y construya sus propias imágenes de aplicación:

```bash
sudo docker build -t rtmp-server-local ./server
sudo docker build -t rtmp-emisor-local ./client
```

### Paso 3: Inicializar el Servidor Central

Exponemos el puerto de streaming (`1935`) y el panel web de administración (`8000`):

```bash
sudo docker run -d --name rtmp-server --network red-rtmp -p 1935:1935 -p 8000:8000 rtmp-server-local
```

### Paso 4: Hito de Activación de Captura (Wireshark)

> Nota importante: Antes de levantar el emisor en el siguiente paso, abra Wireshark en su sistema anfitrión y comience la captura promiscuamente seleccionando la interfaz de red virtual generada por Docker (identificable en su terminal mediante el comando `ip link` o `ifconfig` como `br-xxxxxx`). Esto garantiza capturar el handshake inicial.

### Paso 5: Desplegar Emisor Sintético y Cliente Consumidor

Levante el contenedor emisor (inyectará barras de color y audio de prueba) y consuma el flujo con `ffplay`:

```bash
# Iniciar transmisión automática en el contenedor emisor
sudo docker run -d --name rtmp-emisor --network red-rtmp rtmp-emisor-local

# Visualizar el streaming en tiempo real desde el sistema anfitrión
ffplay rtmp://127.0.0.1/live/prueba
```

---

## Análisis de Tráfico y Hallazgos

Durante el desarrollo práctico del laboratorio se evidenciaron tres fases de datos mediante filtros de Wireshark (`tcp.port == 1935`):

1. **Fase de Handshake Estricto**
   - Intercambio secuencial de las tramas binarias base `C0/C1/C2` y `S0/S1/S2` entre el emisor (`172.18.0.3`) y el servidor (`172.18.0.2`).

2. **Mensajes de Control AMF0**
   - Intercepción en texto claro de las llamadas RPC de señalización clave como los comandos `publish('prueba')` y `play('prueba')`.

3. **Flujo Multimedia Dinámico**
   - Inundación masiva de tramas ya segmentadas `Video Data (H.264)` y `Audio Data (AAC)` con banderas TCP `PSH` activas para forzar el procesado directo en la aplicación de destino.

### Repercusiones de Fallo (Modificación On-the-fly)

Si un intermediario malicioso modificara en tiempo real el campo de cabecera `Message Length` en un chunk multimedia, la máquina de estados lógicos interna de decodificación de FFplay perdería la alineación de bytes. Como el reproductor depende de este campo para saber cuántos bytes componen el fotograma comprimido, leería datos aleatorios o basura, provocando artefactos visuales extremos (píxeles caóticos rotos), desincronización crítica de audio y un cuelgue definitivo de la aplicación mediante un desbordamiento de búfer (`Buffer Overflow / Crash`).

---

## Autores

- Jeremías Quezada
- Matías Droguett

Taller de Redes y Servicios - Universidad Diego Portales - Semestre 2026-1
