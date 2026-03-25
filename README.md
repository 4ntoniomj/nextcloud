# ☁️ Nextcloud en Docker

Este repositorio define una infraestructura como código (IaC) para el despliegue de una nube privada basada en Nextcloud. El objetivo es proporcionar un entorno altamente automatizado, resiliente y rigurosamente diseñado para entornos de producción.

La arquitectura sigue un enfoque estricto de microservicios (un proceso por contenedor) para maximizar la seguridad, la escalabilidad y la eficiencia de los recursos. El stack tecnológico incluye:

- **Caddy:** Proxy inverso y servidor web con gestión automatizada de certificados TLS y enrutamiento seguro.
    
- **Nextcloud (PHP-FPM):** Motor principal de la aplicación, aislado del servidor web para mayor rendimiento.
    
- **PostgreSQL:** Base de datos relacional de alto rendimiento con versiones estables fijadas.
    
- **Redis:** Almacenamiento clave-valor en memoria dedicado al bloqueo transaccional de archivos (file locking) y caché distribuida para evitar cuellos de botella en la base de datos.
    
- **DuckDNS:** Resolución de DNS dinámico para entornos sin IP estática pública.

---
## 📋 Requisitos Previos

Para desplegar esta infraestructura, el servidor anfitrión debe contar con las siguientes herramientas instaladas:

* **Git:** Para clonar este repositorio.
* **Docker Engine & Docker Compose (V2):** El motor de contenerización y su orquestador.

Si estás operando sobre un sistema basado en Debian/Ubuntu, puedes instalar todas las dependencias ejecutando el siguiente bloque de comandos:

```bash
# 1. Actualizar repositorios e instalar Git
sudo apt update && sudo apt install -y git curl

# 2. Descargar e instalar el script oficial de Docker
curl -fsSL [https://get.docker.com](https://get.docker.com) -o /tmp/get-docker.sh
sudo sh /tmp/get-docker.sh

# 3. (Opcional) Añadir tu usuario al grupo docker para no usar sudo constantemente
sudo usermod -aG docker $USER
```

También debemos crear un archivo .env donde se deben definir todas las variables que requiere Docker Compose:

```bash
cp .env.example .env
```

```txt
POSTGRES_DB=<tu base de datos>
POSTGRES_USER=<tu usuario>
POSTGRES_PASSWORD=<tu contraseña>
DUCKDNS_TOKEN=<tu token>
```

---
## 🚀 Despliegue del Stack

Una vez configurado el archivo `.env`, ejecuta el siguiente comando para levantar la infraestructura en segundo plano:

```bash
docker compose up -d
```
### 🔍 Verificación del Estado

No asumas que el sistema está listo solo porque el comando finalizó. Ejecuta una auditoría rápida de los procesos:

```bash
docker compose ps
```

**Criterio de éxito:** Todos los servicios (`Web_Nextcloud`, `Nextcloud`, `BSD_Nextcloud`, `Redis_Nextcloud`) deben mostrar el estado `Up` o `Running`. Si alguno muestra `Exit 1`, revisa los logs con `docker compose logs <servicio>`.

### 🛡️ Recomendación Estratégica: Infraestructura sobre ZFS

Implementar un stack de nube privada (Nextcloud + PostgreSQL) sobre sistemas de ficheros convencionales como **ext4, XFS o NTFS** introduce riesgos críticos de integridad y disponibilidad. Para un entorno de producción, la arquitectura debe asentarse sobre **ZFS (Zettabyte File System)** por las siguientes razones técnicas:

---
## 🦆 Gestión de Dominio y Certificados (DuckDNS)

Este proyecto integra **DuckDNS** para proporcionar un nombre de dominio gratuito y gestionar la emisión automática de certificados SSL (HTTPS). 

### 🛠️ Escenarios de Configuración

Dependiendo de cómo vayas a acceder a tu nube, la configuración varía:

#### A. Acceso vía IP Pública (Acceso desde Internet)
Si tu intención es acceder a Nextcloud desde fuera de tu casa/oficina:
1. **Activa el contenedor `duckdns`** en el `docker-compose.yml` (descomenta las líneas). Este servicio monitoriza si tu proveedor de internet cambia tu IP pública y actualiza el dominio automáticamente.
2. **Abre los puertos 80 y 443** en tu router apuntando a la IP local de tu servidor.

#### B. Acceso vía IP Privada (Solo Local / VPN)
Si solo quieres usar Nextcloud dentro de tu red local pero con HTTPS funcional:
1. **No necesitas activar el contenedor `duckdns`**. Basta con crear el subdominio una vez en la web de DuckDNS y apuntarlo a tu IP privada (ej. `192.168.1.50`).
2. **El Token es obligatorio**: Caddy usará el `DUCKDNS_TOKEN` definido en el `.env` para demostrarle a Let's Encrypt que eres el dueño del dominio sin necesidad de abrir puertos en el router (Desafío DNS-01).

> [!NOTE]
> En ambos casos, el **Token** debe estar correctamente configurado en el archivo `.env` para que Caddy pueda generar el certificado de seguridad y usar el archivo de configuración de caddy oportuno.

---
## 🚀 Aceleración y Consistencia (Redis)

Este proyecto integra **Redis 7.2** como motor de almacenamiento en caché en memoria. Su función no es solo mejorar la velocidad, sino evitar la corrupción de archivos en entornos multiusuario.

### ⚙️ Funciones Críticas en este Stack

1. **Caché Distribuida:** Redis almacena los metadatos y sesiones de usuario en la memoria RAM. Esto descarga a la base de datos **PostgreSQL** de miles de consultas repetitivas, reduciendo la latencia de la interfaz web en un **60-80%**.
    
2. **Transactional File Locking (Bloqueo de Archivos):** Es la función más importante para la seguridad de tus datos. Cuando un cliente (ej. tu móvil) sube un archivo, Redis "bloquea" esa entrada en memoria. Si otro dispositivo intenta modificar el mismo archivo simultáneamente, Redis gestiona la cola para evitar **condiciones de carrera** que corromperían el archivo.    

### 🛡️ Configuración de Seguridad y Rendimiento

A diferencia de las configuraciones por defecto, este stack implementa límites estrictos para proteger el servidor anfitrión:

- **Límite de Memoria (`512MB`):** Hemos configurado Redis con un techo de consumo de RAM. Esto evita que un uso intensivo de Nextcloud sature la memoria del servidor y provoque el cierre inesperado de otros servicios.
    
- **Política de Expulsión (`allkeys-lru`):** Cuando Redis alcanza su límite de 512MB, utiliza el algoritmo _Least Recently Used_ (Usado menos recientemente) para eliminar automáticamente los datos antiguos y dejar espacio a los nuevos sin interrumpir el servicio.

### 🧪 Verificación de Operatividad

Puedes comprobar que Nextcloud está comunicándose correctamente con Redis ejecutando:

```bash
docker exec -it Redis_Nextcloud redis-cli ping
```

_Si la respuesta es `PONG`, el sistema de caché y bloqueo de archivos está activo y funcional._

---
## 🐘 Persistencia de Datos (PostgreSQL 17)

El motor de base de datos elegido para este stack es **PostgreSQL 17-alpine**. Se ha seleccionado por su robustez, integridad referencial y su superior gestión de concurrencia comparada con otras soluciones como MariaDB o SQLite.

### 🏗️ Decisiones de Arquitectura

1. **Versión Estable (v17):** A diferencia de usar el tag `latest`, fijar la versión a la **17** garantiza que las actualizaciones automáticas de Docker no provoquen incompatibilidades en el formato de los datos binarios, lo que podría impedir que el contenedor arranque tras un reinicio.
    
2. **Optimización de Memoria Compartida (`shm_size: 256mb`):** Nextcloud realiza consultas complejas y pesadas. He aumentado el tamaño de la memoria compartida por defecto del contenedor para permitir que PostgreSQL gestione búferes más grandes, evitando errores de "out of memory" durante la indexación de archivos grandes.
    
3. **Aislamiento de Red:** El contenedor de la base de datos no expone puertos al exterior del host. Solo es accesible internamente por el servicio **Nextcloud**, minimizando la superficie de ataque ante intentos de intrusión por fuerza bruta.

### 💾 Seguridad de los Datos (Volúmenes)

La base de datos utiliza un volumen persistente llamado `PostgresNextcloudData`. Esto separa los binarios del motor de base de datos de tus datos reales.

- **Ventaja:** Puedes destruir, actualizar o cambiar el contenedor de Postgres sin perder ni un solo registro de tu nube, siempre que el volumen permanezca intacto.

### 🛠️ Comprobación de Salud (Healthcheck)

Para verificar que la base de datos está aceptando conexiones correctamente:

```bash
docker exec -it BSD_Nextcloud pg_isready -U $POSTGRES_USER
```

_Si el comando devuelve `accepting connections`, la base de datos está lista para servir a Nextcloud._

---
## ☁️ El Motor del Sistema (Nextcloud 33 FPM)

He elegido la imagen **Nextcloud-FPM (FastCGI Process Manager)** basada en **Alpine Linux**. Esta versión es la más ligera y eficiente, diseñada específicamente para entornos donde el rendimiento y la seguridad son la prioridad.

### ⚙️ Por qué FPM y no la imagen estándar (Apache)

1. **Eficiencia de Recursos:** A diferencia de la imagen con Apache, FPM no mantiene procesos pesados abiertos innecesariamente. Solo procesa el código PHP cuando recibe una petición, lo que libera RAM para el sistema.
    
2. **Seguridad por Aislamiento:** Al separar el servidor web (Caddy) del intérprete de PHP (Nextcloud), se crea una capa de seguridad adicional. Si el servidor web se ve comprometido, el atacante sigue estando fuera del entorno de ejecución de tus archivos.
    
3. **Versión Alpine:** El uso de Alpine Linux reduce el tamaño de la imagen y la superficie de ataque, eliminando librerías y herramientas innecesarias que podrían tener vulnerabilidades.

### 🛠️ Configuración para el Proxy Inverso

Para que Nextcloud funcione correctamente detrás de Caddy, he inyectado variables de entorno críticas en el archivo de configuración:

- **TRUSTED_PROXIES:** Permite que Nextcloud acepte las cabeceras de seguridad enviadas por Caddy.
    
- **OVERWRITEPROTOCOL:** Fuerza a Nextcloud a generar todos sus enlaces internos bajo `https`, evitando errores de contenido mixto o bloqueos del navegador.

### ⏳Cron

Nextcloud, por defecto, utiliza un sistema de ejecución de tareas tipo `AJAX`. Esto significa que las tareas en segundo plano solo se ejecutan cuando un usuario interactúa con la interfaz web, lo que provoca dos problemas:

1. **Latencia:** El usuario que activa el proceso sufre tiempos de carga elevados.
    
2. **Inconsistencia:** Si no hay actividad, las tareas críticas (notificaciones, limpieza, indexación) se retrasan.

La solución: contenedor de cron dedicado.
Se ha implementado un contenedor paralelo basado en la misma imagen de Nextcloud, configurado específicamente para ejecutar el planificador de tareas de forma independiente.

- **Desacoplamiento:** El servidor principal queda liberado de procesar lógica de mantenimiento durante las peticiones de los usuarios.
    
- **Eficiencia:** Al utilizar el binario de PHP-CLI en un bucle ligero, el consumo de recursos es mínimo y solo aumenta durante la ejecución efectiva de las tareas.
    
- **Rendimiento:** Garantiza que el sistema esté siempre al día, independientemente del tráfico web.

---
## 🛡️ Proxy Inverso y Seguridad (Caddy)

En este stack, **Caddy** actúa como el punto de entrada único. No es solo un servidor web; es un proxy inverso moderno diseñado para la seguridad automática.

### 🏗️ Construcción Personalizada (Dockerfile)

A diferencia de usar la imagen estándar, he construido una versión personalizada de Caddy mediante un `Dockerfile` para incluir el **módulo de DuckDNS**. Esto permite realizar el **desafío DNS-01**:

- **Ventaja:** Caddy puede obtener certificados SSL de Let's Encrypt incluso si el servidor no es accesible desde internet o si los puertos 80/443 están cerrados, validando la propiedad del dominio directamente a través de la API de DuckDNS.

### ⚙️ Configuración

Se han integrado las directivas recomendadas por Nextcloud para Caddy, asegurando el manejo correcto de los _headers_ de seguridad y la optimización de los _puntos finales_ (endpoints) de la API, fundamentales para la integridad de los datos.

### ⚡ Rendimiento (PHP-FPM)

Caddy se comunica con Nextcloud a través del protocolo **FastCGI (puerto 9000)**. Esta arquitectura es superior a la clásica de Apache porque separa el servidor web del motor de procesamiento, permitiendo una gestión de memoria mucho más eficiente y una carga de archivos optimizada gracias a la compresión `gzip`.

### 🧪 Verificación de Certificados

Puedes auditar el estado de los certificados y la configuración de Caddy revisando los logs internos:

```bash
docker compose logs caddy
```

_Si aparece el mensaje `certificate obtained successfully`, la conexión es 100% segura._

---
## 🌐 Docker Networks

Para mitigar vectores de ataque en caso de una vulnerabilidad en el servidor expuesto, se ha implementado una arquitectura de **redes segmentadas** basada en el principio de menor privilegio.

En lugar de una red única, el despliegue se divide en dos capas aisladas:

- **Red Frontend (`proxy-net`):** Una red aislada donde solo conviven **Caddy** y **Nextcloud**. Caddy actúa como único punto de entrada, derivando el tráfico exclusivamente a la aplicación.
    
- **Red Backend (`backend-net`):** Una red privada donde reside la lógica y persistencia. **Nextcloud** se comunica aquí con la **Base de Datos** y **Redis**.

**Ventajas de esta estructura:**

1. **Reducción de la superficie de ataque:** Si el servidor web (Caddy) se ve comprometido, el atacante no tiene visibilidad ni ruta de acceso directa a la base de datos, ya que pertenecen a redes lógicas distintas.
    
2. **Aislamiento de persistencia:** Los datos sensibles solo son accesibles por la aplicación Nextcloud, quedando totalmente invisibles para el tráfico externo.