# ☁️ Nextcloud High-Performance Stack (Docker)

Una arquitectura de despliegue profesional para Nextcloud, optimizada para entornos de producción utilizando contenedores ligeros Alpine, separación de servicios (FPM + NGINX) y base de datos PostgreSQL.

---
## 🏗️ Análisis de la Arquitectura

A diferencia de las instalaciones monolíticas o basadas en Apache, este stack sigue el principio de separación de responsabilidades de los microservicios, garantizando mayor seguridad y rendimiento:

* **Nextcloud FPM (Alpine):** Actúa exclusivamente como procesador FastCGI. Al usar la versión basada en Alpine Linux, reducimos drásticamente la superficie de ataque y el peso de la imagen.
* **NGINX (Alpine):** Servidor web de alto rendimiento que maneja las peticiones HTTP y sirve los archivos estáticos directamente, descargando de trabajo al procesador PHP.
* **PostgreSQL:** Motor de base de datos relacional robusto, configurado con memoria compartida optimizada (`shm_size: 256mb`) para operaciones intensivas.

---

## 🚀 Requisitos Previos

* [Docker](https://docs.docker.com/get-docker/).
* [Docker Compose](https://docs.docker.com/compose/install/) (v2+).
* Git.

Instalación:
```bash
curl -fsSL https://get.docker.com -o /tmp/get-docker.sh
sudo sh /tmp/get-docker.sh && rm /tmp/get-docker.sh
sudo apt update && sudo apt install -y git
```

---

## ⚙️ Instrucciones de Despliegue

### 1. Clonar el repositorio
```bash
git clone git@github.com:TuUsuario/nextcloud.git
cd nextcloud
```
### 2. Configuración de Seguridad (Variables de Entorno)

**CRÍTICO:** Nunca subas tus credenciales al repositorio. Debes crear un archivo local `.env` en la raíz del proyecto para definir las contraseñas de la base de datos.

Crea el archivo .env
```bash
touch .env
```

Añade el siguiente contenido, modificando los calores con contraseñas seguras:
```txt
POSTGRES_USER=nextcloud_db_user
POSTGRES_PASSWORD=tu_contraseña_super_segura_aqui
POSTGRES_DB=nextcloud_database
```
### 3. Iniciar la infraestructura
Una vez configurada las credenciales, levanta los servicios en segundo plano:
```bash
docker compose up -d
```
### 4. Acceso Inicial
El servicio estará disponible en el puerto 80 de tu máquina host.
Abre tu navegador y navega a: http://localhost:80 (o la IP de tu servidor).