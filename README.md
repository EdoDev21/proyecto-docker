# Proyecto Docker: Nginx, Apache y Odoo

Este proyecto demuestra cómo orquestar múltiples servicios (un sitio web estático en Apache, una instancia de Odoo y una base de datos PostgreSQL) y exponerlos a través de un único proxy inverso Nginx usando Docker Compose.

## Componentes

* **Nginx (Proxy Inverso):** El punto de entrada a la aplicación. Redirige el tráfico a los servicios correctos basándose en el nombre de dominio.
    * `sitio.eduardo.com` -> Redirige al servicio `website` (Apache).
    * `odoo.eduardo.com` -> Redirige al servicio `odoo` (Odoo).
* **Apache (Servicio `website`):** Un servidor web `httpd` que sirve un sitio web estático simple (HTML/CSS/JS).
* **Odoo 17 (Servicio `odoo`):** Una instancia de la aplicación Odoo ERP.
* **PostgreSQL 16 (Servicio `db`):** La base de datos requerida por Odoo para almacenar sus datos.

## Estructura del Proyecto

Para que `docker-compose` y `Dockerfile` funcionen correctamente, los archivos deben estar organizados de la siguiente manera:
```bash
.
 ├── docker-compose.yaml 
 ├── Dockerfile 
 ├── nginx 
 │ └── nginx.conf 
 └── website 
    ├── index.html 
    ├── script.js 
    └── style.css
```

## Instalación y Uso

1.  **Modificar el archivo `hosts`**

    Para que el navegador pueda resolver los dominios `sitio.eduardo.com` y `odoo.eduardo.com`, se deben añadir al archivo `hosts` local para que apunten a `localhost`.

    * **En Linux/macOS:** Editar el archivo `/etc/hosts`
    * **En Windows:** Editar `C:\Windows\System32\drivers\etc\hosts` (se necesitarán permisos de administrador)

    Añadir la siguiente línea al final del archivo:
    ```
    127.0.0.1   sitio.eduardo.com odoo.eduardo.com
    ```

2.  **Construir y Levantar los Contenedores**

    Se abre una terminal en el directorio raíz del proyecto (donde se encuentra el `docker-compose.yaml`) y se ejecuta:
    ```bash
    docker-compose up -d --build
    ```
    * `up` inicia los contenedores.
    * `-d` los ejecuta en modo "detached" (en segundo plano).
    * `--build` fuerza la construcción de la imagen de Apache desde el `Dockerfile`.

3.  **Acceder a los Servicios**

    Una vez que los contenedores estén en funcionamiento, se puede acceder a ellos desde el navegador:

    * **Sitio Web Estático:** [http://sitio.eduardo.com](http://sitio.eduardo.com)
    * **Instancia de Odoo:** [http://odoo.eduardo.com](http://odoo.eduardo.com)

4.  **Detener los Servicios**

    Para detener todos los contenedores, se ejecuta:
    ```bash
    docker-compose down
    ```
    Si también se desea eliminar los volúmenes (¡esto borrará los datos de Odoo!), se usa:
    ```bash
    docker-compose down -v
    ```

## Explicación de los Archivos

### `docker-compose.yaml`

Este es el archivo principal que orquesta todos los servicios:

* **`services:`**
    * **`nginx`**: Construye desde la imagen `nginx:latest`. Mapea el puerto `80` del host al `80` del contenedor y monta el `nginx.conf` local dentro del contenedor. Depende de `website` y `odoo` para iniciarse.
    * **`website`**: Construye una imagen personalizada usando el `Dockerfile` en el directorio actual (`.`).
    * **`db`**: Usa la imagen oficial `postgres:16` y define las variables de entorno para el usuario, contraseña y base de datos de Odoo. Monta un volumen llamado `postgres-data` para persistencia.
    * **`odoo`**: Usa la imagen `odoo:17`. Depende del servicio `db`. Se le pasan las credenciales de la base de datos como variables de entorno y se inicia con el flag `--proxy-mode` (importante cuando está detrás de un proxy). Monta un volumen `odoo-data` para persistencia.
* **`networks:`**
    * `app-network`: Crea una red de tipo `bridge` personalizada para que todos los contenedores puedan comunicarse entre sí usando sus nombres de servicio (ej. `http://odoo`, `http://website`).
* **`volumes:`**
    * `postgres-data` y `odoo-data`: Declara los volúmenes para hacer que los datos de la base de datos y de Odoo sean persistentes, incluso si los contenedores se detienen o eliminan.

### `Dockerfile`

Este archivo crea la imagen para el servicio `website`:

* `FROM httpd:2.4`: Inicia desde la imagen oficial de Apache.
* `COPY ./website /usr/local/apache2/htdocs/`: Copia los archivos de la carpeta local `website` al directorio donde Apache sirve el contenido web dentro del contenedor.

### `nginx/nginx.conf`

Este archivo configura Nginx como un proxy inverso:

* **`upstream`**: Define grupos de servidores. `upstream website` apunta al servicio `website` en el puerto `80` (el nombre del servicio de Docker). `upstream odoo` apunta al servicio `odoo` en el puerto `8069`.
* **`server { ... }` (para `sitio.eduardo.com`)**:
    * Escucha en el puerto `80` para peticiones a `sitio.eduardo.com`.
    * `proxy_pass http://website;`: Pasa todas las peticiones (`/`) al *upstream* llamado `website`.
* **`server { ... }` (para `odoo.eduardo.com`)**:
    * Escucha en el puerto `80` para peticiones a `odoo.eduardo.com`.
    * `proxy_pass http://odoo;`: Pasa todas las peticiones (`/`) al *upstream* llamado `odoo`.
    * Las cabeceras `Upgrade` y `Connection` son necesarias para que funcionen los WebSockets de Odoo (para el chat en vivo, etc.).