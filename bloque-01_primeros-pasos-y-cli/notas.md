# Bloque 1 — Instalación y Primeros Pasos

## 1.1 · Docker Desktop vs Docker Engine

```
Docker Desktop:                      Docker Engine (solo Linux):
────────────────────────────         ────────────────────────────
Interfaz gráfica incluida            Solo CLI, sin GUI
Incluye WSL2 automáticamente         Instalación manual
Docker Compose incluido              Compose hay que instalarlo
Kubernetes local opcional            Sin Kubernetes integrado
Gratis para uso personal             Siempre gratuito
De pago para empresas >250           Para servidores de producción
empleados o >10M$ ingresos
```

> ⚠️ **Trampa comercial**: Docker Desktop es gratuito para uso personal y educativo, pero tiene licencia de pago para empresas grandes. En producción, los servidores Linux usan Docker Engine directamente, nunca Docker Desktop.

### Instalación en Linux (Docker Engine)

```bash
# Script oficial
curl -fsSL https://get.docker.com | sh

# Añadir tu usuario al grupo docker (para no necesitar sudo)
sudo usermod -aG docker $USER

# Cerrar sesión y volver a entrar para que aplique
```

### Verificar instalación

```bash
docker --version
# Docker version 27.x.x

docker compose version
# Docker Compose version v2.x.x

docker run hello-world
```

---

## 1.2 · Qué pasa exactamente con `docker run`

```
Paso 1: La CLI recibe el comando
────────────────────────────────────────────────────────
Tu terminal envía la orden al Docker Daemon via API REST.

Paso 2: El Daemon busca la imagen en local
────────────────────────────────────────────────────────
¿Tengo la imagen en caché local?
  SÍ → salta al Paso 4
  NO → continúa al Paso 3

Paso 3: Pull de la imagen desde el Registry
────────────────────────────────────────────────────────
El Daemon contacta Docker Hub y descarga la imagen capa por capa:

  Unable to find image 'hello-world:latest' locally
  latest: Pulling from library/hello-world
  719385e32844: Pull complete
  Digest: sha256:a13ec89...
  Status: Downloaded newer image for hello-world:latest

Paso 4: El Daemon crea el contenedor
────────────────────────────────────────────────────────
Apila las capas de la imagen (solo lectura)
Añade la capa R/W vacía encima
Configura namespaces y cgroups

Paso 5: containerd arranca el proceso
────────────────────────────────────────────────────────
Ejecuta el comando definido en la imagen

Paso 6: El contenedor termina y se detiene
────────────────────────────────────────────────────────
El proceso termina → contenedor pasa a estado "Exited"
La capa R/W sigue existiendo hasta que hagas docker rm
```

---

## 1.3 · Imágenes vs Contenedores

```
Mundo Docker:
──────────────────────
Imagen
Contenedor en ejecución
docker run mi-imagen
Puedes crear N contenedores
de la misma imagen
```

Cada contenedor tiene su propia capa R/W, su propio namespace de red y su propio PID 1. Son completamente independientes aunque vengan de la misma imagen.

---

## 1.4 · Ciclo de vida de un contenedor

```
                    docker run / docker start
                            │
                            ▼
              ┌─────────────────────────┐
              │         RUNNING         │◀──────────────────┐
              │   (proceso ejecutando)  │                   │
              └─────────────────────────┘                   │
                   │              │                         │
         proceso   │              │ docker stop             │ docker start
         termina   │              │ docker kill             │
         solo      │              ▼                         │
                   │   ┌─────────────────────────┐          │
                   │   │         EXITED          │──────────┘
                   └──▶│  (proceso terminado,    │
                       │   capa R/W intacta)     │
                       └─────────────────────────┘
                                   │
                                   │ docker rm
                                   ▼
                       ┌─────────────────────────┐
                       │        ELIMINADO        │
                       │  (capa R/W destruida)   │
                       └─────────────────────────┘
```

> ⚠️ `Exited` ≠ eliminado. Un contenedor detenido sigue ocupando espacio en disco.

```bash
# Eliminar automáticamente al terminar
docker run --rm hello-world
```

---

## 1.5 · `docker stop` vs `docker kill` a bajo nivel

### Señales Unix

```
Señal       Número    Significado
──────────────────────────────────────────────────
SIGTERM       15      "Por favor, termina cuando puedas"
SIGKILL        9      "Muere ahora. Sin excepciones."
SIGINT         2      Lo que ocurre cuando pulsas Ctrl+C
SIGHUP         1      "Tu terminal se cerró"
```

```
SIGTERM:                           SIGKILL:
────────────────────────           ────────────────────────
El proceso LA RECIBE               El kernel mata el proceso
El proceso DECIDE qué hacer        El proceso no se entera
Puede limpiar recursos             No puede hacer nada
Puede terminar conexiones          Conexiones cortadas en seco
Puede hacer flush de logs          Logs pendientes se pierden
Puede ignorarla (mal diseño)       No se puede ignorar ni capturar
```

`SIGKILL` no lo gestiona el proceso, lo gestiona directamente el kernel. Es imposible ignorarlo.

### `docker stop`

```
docker stop mi-contenedor
      │
      ├─▶ Envía SIGTERM al proceso PID 1
      │   Espera hasta 10 segundos (configurable con --time)
      └─▶ Si sigue vivo tras el timeout → envía SIGKILL
```

### `docker kill`

```
docker kill mi-contenedor
      │
      └─▶ Envía SIGKILL directamente. Sin espera. Sin gracia.
```

### ¿Cuándo usar cada uno?

```
USA docker stop cuando:            USA docker kill cuando:
────────────────────────           ────────────────────────
Parada normal planificada          El contenedor no responde a stop
Deploys y mantenimiento            El proceso está colgado (deadlock)
Cualquier situación normal         Emergencia absoluta
```

> ⚠️ **Trampa**: Si tu app tarda más de 10 segundos en responder al SIGTERM, Docker la mata igualmente. La solución no es aumentar el timeout ciegamente, es revisar por qué tarda tanto en cerrar.

```bash
# Cambiar el timeout de gracia
docker stop --time 30 mi-contenedor
```

---

## 1.6 · Puertos: host, contenedor y mapeo

### Puertos well-known

```
Puerto    Protocolo    Aplicación
──────────────────────────────────
22        SSH          Conexión segura
80        HTTP         Web sin cifrar
443       HTTPS        Web cifrada
5432      PostgreSQL   Base de datos
3306      MySQL        Base de datos
6379      Redis        Cache
27017     MongoDB      Base de datos
```

- Puertos **< 1024** → requieren root en Linux → uso en producción
- Puertos **≥ 1024** → sin restricciones → uso en desarrollo (8080, 3000, 4200...)

### Namespace de red

Cada contenedor tiene su **propio espacio de puertos** (0-65535). El conflicto solo ocurre en el **host**:

```
Host (tu máquina):
├── Puerto 8080  ──map──▶  Contenedor nginx-1  puerto 80
├── Puerto 8081  ──map──▶  Contenedor nginx-2  puerto 80
└── Puerto 5432  ──map──▶  Contenedor postgres puerto 5432
```

Múltiples contenedores pueden escuchar en el mismo puerto interno sin conflicto.

```
                    ┌─────────────────────┐
                    │   TU MÁQUINA (host) │
Tu navegador        │                     │
localhost:8080 ────▶│  :8080 ─────────────┼──▶ contenedor-1 :80
                    │  :8081 ─────────────┼──▶ contenedor-2 :80
Tu navegador        │  :5432 ─────────────┼──▶ contenedor-3 :5432
localhost:8081 ────▶│                     │
                    └─────────────────────┘
```

### Sintaxis

```bash
# -p PUERTO_HOST:PUERTO_CONTENEDOR
docker run -d -p 8080:80 nginx
```

### Sin mapeo de puertos

```bash
docker run -d nginx   # sin -p
```

El puerto interno es **inaccesible desde el exterior**. Solo otros contenedores en la misma red Docker pueden acceder. Es una feature de seguridad: los contenedores no exponen nada por defecto.

---

## 1.7 · Comandos esenciales de la CLI

### Contenedores

```bash
docker run nginx                      # crear y arrancar
docker run -d nginx                   # en segundo plano (detached)
docker run -d --name mi-nginx nginx   # con nombre propio
docker run --rm nginx                 # eliminar al terminar
docker run -d -p 8080:80 nginx        # con port mapping

docker ps                             # contenedores en ejecución
docker ps -a                          # todos (incluidos detenidos)

docker stop mi-nginx                  # SIGTERM → espera → SIGKILL
docker kill mi-nginx                  # SIGKILL directo
docker start mi-nginx                 # arrancar uno detenido
docker rm mi-nginx                    # eliminar (debe estar detenido)
docker rm -f mi-nginx                 # forzar stop + rm
docker container prune                # eliminar todos los detenidos
```

### Imágenes

```bash
docker pull ubuntu:24.04              # descargar sin crear contenedor
docker images                         # ver imágenes en local
docker rmi ubuntu:24.04               # eliminar imagen
docker image prune                    # eliminar imágenes sin usar
docker system prune                   # limpiar TODO lo que no se usa
```

### Inspección y debugging

```bash
docker logs mi-nginx                  # ver logs
docker logs -f mi-nginx               # logs en tiempo real (tail -f)
docker exec mi-nginx ls /etc/nginx    # ejecutar comando dentro
docker exec -it mi-nginx bash         # terminal interactiva dentro
docker stats                          # CPU y RAM en tiempo real
docker inspect mi-nginx               # info detallada en JSON
```

> 💡 `-it` = `-i` (interactivo, mantiene STDIN) + `-t` (pseudo-TTY, emula terminal)

### Ejemplo completo

```bash
# Arrancar nginx con port mapping
docker run -d --name mi-web -p 8080:80 nginx

# Verificar → abrir http://localhost:8080 en el navegador

# Ver logs
docker logs mi-web

# Entrar dentro
docker exec -it mi-web bash
# root@a1b2c3d4:/# ls /etc/nginx/
exit

# Limpiar
docker stop mi-web && docker rm mi-web
```
