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

También debemos crear un archivo .env donde se deben definir todas las variables que requiere docker compose:

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