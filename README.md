# Servidor LibreOffice Online con Nextcloud

Este proyecto despliega una solución completa de ofimática en la nube utilizando Nextcloud y Collabora Online (LibreOffice), permitiendo editar documentos de office directamente desde el navegador.

## 📋 Índice

- [¿Cómo funciona?](#-cómo-funciona)
- [Arquitectura](#-arquitectura)
- [Requisitos previos](#-requisitos-previos)
- [Configuración](#-configuración)
- [Despliegue](#-despliegue)
- [Verificación](#-verificación)
- [Solución de problemas](#-solución-de-problemas)

## 🔧 ¿Cómo funciona?

Este proyecto utiliza Docker Compose para orquestar múltiples servicios que trabajan juntos:

1. **Nextcloud**: Plataforma de almacenamiento en la nube (similar a Google Drive/Dropbox)
2. **Collabora Online**: Motor de edición de documentos basado en LibreOffice
3. **MariaDB**: Base de datos para Nextcloud
4. **Redis**:  Sistema de caché para mejorar el rendimiento
5. **Nginx**:  Proxy inverso que gestiona el tráfico HTTPS y distribuye las peticiones

### Flujo de trabajo

```
Usuario → Nginx (HTTPS) → Nextcloud (gestión de archivos)
                       → Collabora (edición de documentos)
                       → Redis (caché)
                       → MariaDB (datos)
```

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────┐
│         Navegador (Puerto 443)          │
└─────────────────┬───────────────────────┘
                  │
          ┌───────▼───────┐
          │  Nginx Proxy  │
          │  (SSL/TLS)    │
          └───┬───────┬───┘
              │       │
    ┌─────────▼──┐  ┌▼──────────────┐
    │ Nextcloud  │  │  Collabora    │
    │  : 80      │  │  :9980        │
    └─┬────────┬─┘  └───────────────┘
      │        │
  ┌───▼──┐  ┌─▼─────┐
  │Redis │  │MariaDB│
  └──────┘  └───────┘
```

## 📦 Requisitos previos

- **Docker** (versión 20.10 o superior)
- **Docker Compose** (versión 2.0 o superior)
- **Certificados SSL/TLS** (autofirmados o de Let's Encrypt)
- Al menos **4GB de RAM** disponible
- **10GB de espacio en disco** (más el espacio que quieras para almacenamiento)

## ⚙️ Configuración

### 1. Preparar certificados SSL

Crea el directorio para los certificados: 

```bash
mkdir -p certs
```

**Opción A: Certificados autofirmados (para pruebas)**

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/tienda.key \
  -out certs/tienda.crt \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=MiEmpresa/CN=192.168.16.130"
```

**Opción B: Let's Encrypt (para producción con dominio)**

```bash
# Instala certbot primero
sudo apt-get install certbot

# Genera los certificados
sudo certbot certonly --standalone -d tu-dominio.com

# Copia los certificados
sudo cp /etc/letsencrypt/live/tu-dominio.com/fullchain.pem certs/tienda.crt
sudo cp /etc/letsencrypt/live/tu-dominio.com/privkey.pem certs/tienda.key
```

### 2. Ajustar configuración

#### A. Modificar `docker-compose.yml`

Cambia los siguientes valores **OBLIGATORIAMENTE**:

```yaml
# Base de datos
environment:
  - MYSQL_ROOT_PASSWORD=TU_PASSWORD_SEGURA_AQUI  # ⚠️ Cambiar
  - MYSQL_PASSWORD=TU_PASSWORD_NEXTCLOUD_AQUI    # ⚠️ Cambiar

# Nextcloud
environment: 
  - MYSQL_PASSWORD=TU_PASSWORD_NEXTCLOUD_AQUI    # ⚠️ Debe coincidir con el anterior
  - NEXTCLOUD_TRUSTED_DOMAINS=TU_IP_O_DOMINIO   # ⚠️ Cambiar
  - OVERWRITEHOST=TU_IP_O_DOMINIO               # ⚠️ Cambiar
  - OVERWRITECLIURL=https://TU_IP_O_DOMINIO     # ⚠️ Cambiar

# Collabora
environment: 
  - aliasgroup1=https://TU_IP_O_DOMINIO:443     # ⚠️ Cambiar
  - username=admin                               # Opcional:  cambiar usuario admin
  - password=TU_PASSWORD_ADMIN_COLLABORA        # ⚠️ Cambiar
```

**Ejemplo con dominio:**
```yaml
- NEXTCLOUD_TRUSTED_DOMAINS=cloud.miempresa.com
- OVERWRITEHOST=cloud.miempresa.com
- OVERWRITECLIURL=https://cloud.miempresa.com
```

**Ejemplo con IP:**
```yaml
- NEXTCLOUD_TRUSTED_DOMAINS=192.168.1.100
- OVERWRITEHOST=192.168.1.100
- OVERWRITECLIURL=https://192.168.1.100
```

#### B. Modificar `proxy/proxy.conf`

1. Cambia el `server_name` en las líneas 30 y 37: 
   ```nginx
   server_name TU_IP_O_DOMINIO;
   ```

2. Si cambiaste los nombres de los certificados, actualiza las líneas 39-40:
   ```nginx
   ssl_certificate /etc/nginx/certs/tu-certificado.crt;
   ssl_certificate_key /etc/nginx/certs/tu-certificado.key;
   ```

#### C. Configuraciones opcionales

**Aumentar tamaño máximo de archivos** (en `proxy/proxy.conf`):
```nginx
client_max_body_size 50G;  # Cambiar de 10G a lo que necesites
```

**Cambiar puertos** (en `docker-compose.yml`):
```yaml
proxy:
  ports:
    - "8080:80"    # Puerto HTTP personalizado
    - "8443:443"   # Puerto HTTPS personalizado
```

**Agregar más idiomas a Collabora** (en `docker-compose.yml`):
```yaml
- dictionaries=es_ES en_US fr_FR de_DE pt_BR
```

## 🚀 Despliegue

### Paso 1: Clonar el repositorio

```bash
git clone https://github.com/Aragorn7372/libre-office-server.git
cd libre-office-server
```

### Paso 2: Configurar (ver sección anterior)

Asegúrate de haber: 
- ✅ Generado los certificados SSL en `certs/`
- ✅ Modificado las contraseñas en `docker-compose.yml`
- ✅ Ajustado IP/dominio en `docker-compose.yml` y `proxy/proxy.conf`

### Paso 3: Iniciar servicios

```bash
# Iniciar todos los servicios en segundo plano
docker-compose up -d

# Ver los logs en tiempo real
docker-compose logs -f

# Ver el estado de los contenedores
docker-compose ps
```

### Paso 4: Configurar Nextcloud (primera vez)

1. Abre tu navegador en `https://TU_IP_O_DOMINIO`
2. Crea una cuenta de administrador (usuario y contraseña)
3. Nextcloud se configurará automáticamente con la base de datos

### Paso 5: Instalar la aplicación Collabora Online

1. En Nextcloud, ve a **Aplicaciones** (icono de cuadrícula arriba a la derecha)
2. Busca **"Nextcloud Office"** o **"Collabora Online"**
3. Haz clic en **Descargar e instalar**
4. Ve a **Configuración** → **Administración** → **Nextcloud Office**
5. Selecciona **"Usar tu propio servidor"**
6. Introduce:  `https://TU_IP_O_DOMINIO`
7. Guarda los cambios

## ✅ Verificación

### Comprobar que todos los servicios están corriendo

```bash
docker-compose ps
```

Deberías ver 5 contenedores en estado `Up`:
- `nextcloud`
- `nextcloud-db`
- `nextcloud-redis`
- `collabora`
- `nginx-proxy`

### Verificar conectividad con Collabora

```bash
curl -k https://TU_IP_O_DOMINIO/hosting/discovery
```

Deberías recibir una respuesta XML con información sobre Collabora.

### Probar edición de documentos

1. En Nextcloud, crea un nuevo archivo (+ → Documento nuevo)
2. El documento debería abrirse en el editor de Collabora
3. Realiza cambios y guarda

## 🐛 Solución de problemas

### Error: "No se puede conectar con Collabora"

**Solución:**
```bash
# Verifica que los dominios coincidan en docker-compose.yml y proxy. conf
grep -r "192.168.16.130" . 

# Reinicia los servicios
docker-compose restart
```

### Error: "Certificado SSL inválido"

**Solución:**
```bash
# Verifica que los certificados existen
ls -lh certs/

# Verifica los permisos
chmod 644 certs/tienda.crt
chmod 600 certs/tienda.key

# Reinicia el proxy
docker-compose restart proxy
```

### Error: "Cannot write into config directory"

**Solución:**
```bash
# Cambia permisos del volumen de Nextcloud
docker-compose exec nextcloud chown -R www-data:www-data /var/www/html
```

### Nextcloud no carga estilos CSS

**Solución:**
```bash
# Reconstruye los archivos estáticos
docker-compose exec -u www-data nextcloud php occ maintenance:repair
```

### Ver logs de un servicio específico

```bash
# Nextcloud
docker-compose logs -f nextcloud

# Collabora
docker-compose logs -f collabora

# Nginx
docker-compose logs -f proxy
```

## 🔄 Comandos útiles

```bash
# Detener todos los servicios
docker-compose down

# Detener y eliminar volúmenes (⚠️ borra todos los datos)
docker-compose down -v

# Reiniciar un servicio específico
docker-compose restart nextcloud

# Ver uso de recursos
docker stats

# Actualizar imágenes
docker-compose pull
docker-compose up -d

# Backup de datos
docker run --rm -v libre-office-server_nextcloud_data:/data -v $(pwd):/backup \
  ubuntu tar czf /backup/nextcloud-backup-$(date +%Y%m%d).tar.gz /data
```

## 📚 Mantenimiento

### Actualizaciones

```bash
# Detener servicios
docker-compose down

# Actualizar imágenes
docker-compose pull

# Iniciar con nuevas versiones
docker-compose up -d
```

### Backups recomendados

1. **Datos de Nextcloud**:  Volumen `nextcloud_data`
2. **Base de datos**: Volumen `db_data`
3. **Archivos de configuración**: `docker-compose.yml` y `proxy/proxy.conf`

## 📄 Licencia

Este proyecto es de código abierto. Los componentes utilizados tienen sus propias licencias: 
- Nextcloud: AGPLv3
- Collabora Online: MPL 2.0
- MariaDB: GPL v2
- Nginx: BSD-like

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor, abre un issue o pull request para sugerencias y mejoras. 