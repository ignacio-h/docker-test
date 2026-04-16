# Bloque 0 — Fundamentos Conceptuales

## 0.1 · El problema del entorno: VMs vs Contenedores

### El problema real

Cuando desarrollas una API en .NET 8 en tu máquina Windows y la despliegas en un servidor Linux de producción, puede explotar. No por el código, sino por el **entorno**:

``` text
Tu máquina:              Servidor producción:
─────────────────        ─────────────────────
Windows 11               Ubuntu 22.04
.NET 8.0.3               .NET 6.0.1  ← versión diferente
OpenSSL 3.1              OpenSSL 1.1 ← versión diferente
Variable PATH=...        Variable PATH=...  ← diferente
Puerto 5432 libre        Puerto 5432 ocupado
```

El código es idéntico. El entorno no. Y el entorno manda.

---

### La primera solución: Máquinas Virtuales

Las VMs resuelven el problema empaquetando el entorno entero, pero con un coste brutal:

``` text
┌─────────────────────────────────────┐
│           Tu aplicación             │
├─────────────────────────────────────┤
│     Sistema Operativo Completo      │  ← Windows/Linux ENTERO
│         (4-20 GB)                   │
├─────────────────────────────────────┤
│          Hypervisor                 │  ← VMware, VirtualBox, Hyper-V
├─────────────────────────────────────┤
│       Hardware físico               │
└─────────────────────────────────────┘
```

- Una VM ocupa **4-20 GB** en disco
- Tarda **1-2 minutos** en arrancar
- Consume **RAM y CPU** del SO invitado aunque no haga nada
- Si necesitas 10 servicios, necesitas 10 SOs completos

---

### La solución de Docker: Contenedores

Docker **comparte el kernel del sistema operativo host** y solo aísla lo que necesita cada aplicación:

``` text
┌──────────────┬──────────────┬──────────────┐
│   App .NET   │  App Node.js │   App Python │
│  + libs .NET │  + libs Node │  + libs Py   │  ← Solo las diferencias
├──────────────┴──────────────┴──────────────┤
│            Docker Engine                   │  ← Capa de gestión
├────────────────────────────────────────────┤
│         Kernel Linux (compartido)          │  ← UN solo kernel
├────────────────────────────────────────────┤
│              Hardware físico               │
└────────────────────────────────────────────┘
```

Resultado práctico:

``` text
VM:                          Contenedor:
─────────────────────        ─────────────────────
Tamaño: 4-20 GB              Tamaño: 50-200 MB
Arranque: 1-2 min            Arranque: < 1 segundo
RAM idle: 512MB-2GB          RAM idle: ~10-50 MB
```

---

## 0.2 · Kernel vs Imagen: la diferencia real

Una imagen Docker **no incluye el kernel**. Nunca.

``` text
Lo que SÍ contiene una imagen:     Lo que NO contiene:
────────────────────────────────   ────────────────────
/bin, /usr, /lib, /etc             El kernel
apt, bash, curl, openssl           Drivers
Librerías del sistema              Gestión de hardware
```

El kernel siempre es el del **host**. Sin excepción.

Docker empaqueta todo lo necesario excepto el kernel.

``` text
Máquina física / VM:
├── Hardware
├── Kernel Linux        ← el "runtime" que Docker NO empaqueta
└── Docker Engine
    ├── Contenedor A
    │   └── Imagen: Ubuntu + Node.js + tu app   ← sin kernel
    └── Contenedor B
        └── Imagen: Alpine + Python + tu app    ← sin kernel
```

> 💡 Ubuntu y Alpine son distribuciones completamente distintas, con utilidades y librerías distintas, pero **ambas usan el mismo kernel del host**.

> ❓ *¿Por qué una imagen de Ubuntu pesa 80MB si Ubuntu completo pesa 2GB?*
> Porque esos 2GB incluyen kernel, drivers, interfaz gráfica... La imagen solo trae el userspace mínimo.

---

## 0.3 · Qué es el kernel de Linux y por qué Docker depende de él

El kernel es el núcleo del SO: habla directamente con el hardware y provee servicios fundamentales (memoria, procesos, sistema de archivos, red...).

Docker usa **dos características específicas del kernel Linux**:

- **Namespaces**: para el aislamiento
- **cgroups**: para el control de recursos

Ninguna existe en el kernel de Windows de forma nativa. Por eso Docker en Windows usa **WSL2**:

``` text
Windows 11
└── Hyper-V (hipervisor ligero)
    └── Kernel Linux real (WSL2)   ← Docker usa ESTE kernel
        └── Tus contenedores Linux
```

---

## 0.4 · Contenedores Windows: sí existen, pero son un mundo aparte

Docker Desktop en Windows tiene dos modos:

``` text
Modo Linux (por defecto):
Windows
└── Hyper-V / WSL2
    └── Kernel Linux (instalado por Docker)
        └── Contenedores Linux  ← el 99% de la industria

Modo Windows (hay que activarlo explícitamente):
Windows
└── Kernel Windows del host  ← se usa directamente
    └── Contenedores Windows
```

Comparativa:

``` text
Contenedores Linux:              Contenedores Windows:
────────────────────             ────────────────────────
Imágenes de ~50-200MB            Imágenes de ~4-8GB
Arranque en <1 segundo           Arranque en 20-60 segundos
Millones de imágenes en Hub      Catálogo muy limitado
Toda la industria los usa        Nicho muy específico
Corren en cualquier Linux        Solo en Windows Server / Win 11
```

> ⚠️ **No puedes mezclar ambos modos simultáneamente.** Docker Desktop está en modo Linux **o** en modo Windows, nunca los dos a la vez.

Los contenedores Windows solo tienen sentido para aplicaciones legacy que dependen de APIs específicas de Windows que no puedes migrar. Para todo lo demás, incluso .NET, la industria usa contenedores Linux.

---

## 0.5 · Los tres pilares: namespaces, cgroups y union filesystems

### Pilar 1: Namespaces — El aislamiento

Un namespace es una **vista limitada del sistema**. Permite que un proceso crea que es el único en la máquina.

Docker usa varios tipos simultáneamente:

``` text
Namespace PID:
  → El contenedor tiene su propio árbol de procesos
  → El proceso principal cree que su PID es 1
  → Desde fuera, tiene otro PID completamente diferente

Namespace NET:
  → El contenedor tiene su propia interfaz de red
  → Su propia IP, sus propias tablas de rutas
  → No ve el tráfico del host ni de otros contenedores

Namespace MNT:
  → El contenedor tiene su propio sistema de archivos
  → Cree que / es su raíz, no la del host

Namespace UTS:
  → El contenedor puede tener su propio hostname
```

---

### Pilar 2: cgroups — El control de recursos

Los **control groups** permiten al kernel **limitar y monitorizar** los recursos de un grupo de procesos:

``` text
Contenedor A: máximo 512MB RAM, máximo 0.5 CPUs
Contenedor B: máximo 2GB RAM, máximo 2 CPUs
Contenedor C: sin límites (usa lo que queda)
```

Sin cgroups, un contenedor podría consumir toda la RAM del servidor y tumbar los demás.

---

### Pilar 3: Union Filesystem — Las capas

Permite **apilar múltiples sistemas de archivos** y verlos como uno solo:

``` text
Capa 4 (lectura/escritura): tus cambios en ejecución  ← única capa mutable
Capa 3 (solo lectura):      tu aplicación copiada
Capa 2 (solo lectura):      .NET runtime instalado
Capa 1 (solo lectura):      Ubuntu base
```

Si tienes 10 contenedores basados en Ubuntu, el disco almacena Ubuntu **una sola vez**:

``` text
Contenedor 1 ─┐
Contenedor 2 ─┤─→ Capa Ubuntu (compartida, ~80MB en disco)
Contenedor 3 ─┘
```

El número de capas **no es siempre 4**. Depende del Dockerfile. Cada instrucción que modifica el sistema de archivos genera una nueva capa.

```dockerfile
FROM ubuntu:24.04          # Capa 1
RUN apt-get update         # Capa 2
RUN apt-get install nginx  # Capa 3
COPY ./mi-app /app         # Capa 4
RUN chmod +x /app/start.sh # Capa 5
```

⚠️ **Trampa — Optimización de capas:**

> ```dockerfile
> # ❌ Mal: 3 capas, una guarda basura temporalmente
> RUN apt-get update
> RUN apt-get install -y nginx
> RUN apt-get clean
>
> # ✅ Bien: 1 sola capa, la basura nunca llega a persistir
> RUN apt-get update && \
>     apt-get install -y nginx && \
>     apt-get clean
> ```

---

### Copy-on-Write (CoW)

Cuando un contenedor necesita modificar un archivo de una capa compartida de solo lectura, Docker lo **copia a la capa R/W del contenedor** y lo modifica ahí. Las capas compartidas nunca se tocan.

Funciona exactamente como la **cascada CSS**:

``` text
CSS:                              Docker CoW:
──────────────────────────        ──────────────────────────
Estilo en línea          ← gana  Capa R/W contenedor  ← gana
<style> en el <head>              Capa imagen 3
stylesheet externo                Capa imagen 2
estilos del navegador   ← pierde Capa base (FROM)     ← pierde
```

Docker busca un archivo recorriendo las capas **de arriba hacia abajo** y se queda con la primera versión que encuentra.

``` text
Contenedor A ve:          Contenedor B ve:
─────────────────         ─────────────────
nginx.conf [R/W] ← usa   nginx.conf [R/W] ← usa su propia copia
nginx.conf [R/O]          nginx.conf [R/O]
     ↑                         ↑
  (tapada)                  (tapada)
```

> ⚠️ Cuando el contenedor muere, su capa R/W desaparece con él. Por eso los contenedores son **efímeros por diseño**, y por eso existen los volúmenes (Bloque 4).

---

## 0.6 · Arquitectura de Docker: Engine, Daemon, CLI y Registry

``` text
┌─────────────────────────────────────────────────────┐
│  TU TERMINAL                                        │
│  $ docker run nginx                                 │
│          │                                          │
│    Docker CLI (cliente)                             │
│          │  HTTP REST API                           │
└──────────┼──────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────┐
│  DOCKER ENGINE                                      │
│                                                     │
│  ┌─────────────────┐     ┌──────────────────────┐   │
│  │  Docker Daemon  │     │   containerd         │   │
│  │  (dockerd)      │───▶│   (runtime real)     │   │
│  │                 │     │                      │   │
│  │  Gestiona:      │     │  Gestiona:           │   │
│  │  - Imágenes     │     │  - Ciclo de vida     │   │
│  │  - Redes        │     │    del contenedor    │   │
│  │  - Volúmenes    │     │  - Namespaces        │   │
│  └─────────────────┘     │  - cgroups           │   │
│                          └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
           │
           │ (si la imagen no está en local)
           ▼
┌─────────────────────────────────────────────────────┐
│  DOCKER REGISTRY (Docker Hub u otro)                │
│  Almacén de imágenes públicas y privadas            │
│  hub.docker.com                                     │
└─────────────────────────────────────────────────────┘
```

| Componente | Qué es |
| --- | --- |
| **Docker CLI** | El cliente que usas. Envía órdenes al daemon via API REST. Puede conectarse a daemons remotos. |
| **Docker Daemon (`dockerd`)** | Proceso en background. Gestiona imágenes, contenedores, redes y volúmenes. |
| **containerd** | El runtime real. Habla directamente con el kernel. Kubernetes también lo usa directamente. |
| **Registry** | Almacén de imágenes. Docker Hub es el público por defecto. |

---

## 0.7 · Docker CLI conectado a daemons remotos

La CLI habla con el daemon mediante una **API REST**. Por defecto apunta al daemon local, pero puede apuntar a cualquier daemon remoto:

```bash
# Daemon local (por defecto)
docker ps

# Daemon remoto
DOCKER_HOST=ssh://usuario@servidor-produccion.com docker ps
```

### Casos de uso reales

**Gestión de servidores remotos:**

``` text
Tu portátil                    Servidor de producción
────────────────               ────────────────────────
Docker CLI ──── SSH/TLS ────▶  Docker Daemon
                               └── contenedor-api
                               └── contenedor-db
```

**CI/CD:**

``` text
GitHub Actions Runner          Tu servidor
──────────────────────         ──────────────────
docker build .        ──────▶  Docker Daemon
docker push           ──────▶  Registry
docker run            ──────▶  Docker Daemon
```

**Docker contexts** (la forma moderna):

```bash
# Crear contextos
docker context create produccion --docker "host=ssh://user@prod.server.com"
docker context create staging    --docker "host=ssh://user@staging.server.com"

# Ver todos los contextos
docker context ls
# NAME          DOCKER ENDPOINT
# default   *   unix:///var/run/docker.sock
# produccion    ssh://user@prod.server.com
# staging       ssh://user@staging.server.com

# Cambiar de contexto
docker context use produccion

# Ahora todos los comandos van al daemon de producción
docker ps
```

> 💡 Es como los connection strings en .NET. Tu aplicación no sabe ni le importa si la base de datos está en local o en Azure, solo cambia el string de conexión.

---

## Resumen del Bloque 0

| Concepto | Clave |
| --- | --- |
| Contenedor vs VM | Comparte kernel, no lo virtualiza. Mucho más ligero. |
| Kernel vs Imagen | La imagen es solo userspace. El kernel siempre es del host. |
| Namespaces | Aíslan procesos, red, filesystem y hostname por contenedor. |
| cgroups | Limitan CPU, RAM e I/O por contenedor. |
| Union Filesystem | Capas apiladas. Compartidas entre contenedores (solo lectura). |
| Copy-on-Write | Modificar un archivo lo copia a la capa R/W del contenedor. Como la cascada CSS. |
| Capas efímeras | La capa R/W muere con el contenedor. Para persistir datos → Volúmenes (Bloque 4). |
| CLI/Daemon | Separación cliente/servidor. La CLI puede apuntar a daemons remotos. |
