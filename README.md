# Documento guia para deploy de Participes+Sumen

La plataforma está hecha en **Laravel 7.x** y utiliza una base de datos InnoDB que popularmente recomendaria **MySQL 5.6+ o MariaDB**.
Bajo el supuesto de que contamos con una maquina virtual basada en Ubuntu 18.04 LTS como servidor.

Recursos:
* [Laravel 7.x - Installation](https://laravel.com/docs/7.x/installation)
* [Laravel 7.x - Configuration](https://laravel.com/docs/7.x/configuration)
* [Laravel 7.x - Deployment](https://laravel.com/docs/7.x/deployment)

Nota: En este manual solamente nos enfocamos en la instalacion del aplicativo para un entorno LAMP. Con respecto a configuraciones del Servidor o Webserver quedan a criterio del equipo de deployment.

## Previo a comenzar...

Laravel requiere **PHP >= 7.2.5**. No es muy diferente a lo que seria un deployment de Symfony u otros frameworks de PHP. En `production`, deberian ir por un **PHP-FPM** por razones de performance.

#### Requisitos:

1. **PHP >= 7.2.5** y **PHP-FPM** (Recomendado para produccion .
2. Web server, como **Apache** o **Nginx** que apunte al directiorio `/public` del proyecto. (Doc para [nginx](https://laravel.com/docs/7.x/deployment#nginx))
3. **MySQL 5.6+ o MariaDB** como base de datos
4. **Composer** [https://getcomposer.org/](https://getcomposer.org/) para la instalacion de las dependencias del proyecto
5. **REDIS** [https://redis.io/](https://redis.io/), "an advanced key-value store" que es utilizado en paralelo con la base de datos para encolar Jobs de Laravel, importante para tener habilitado del envio de correos y notificaciones de la plataforma
6. **SUPERVISOR** [http://supervisord.org/](http://supervisord.org/), "a process control system" que controla y monitorea procesos. Importante para ejecutar los listeners de los Queues de Laravel.
7. Una cuenta en [https://mapbox.com](https://mapbox.com) para consumir los map tiles.

### Con respecto a PHP

Para que Laravel funcione, se requieren los siguientes

* BCMath PHP Extension
* Ctype PHP Extension
* Fileinfo PHP extension
* JSON PHP Extension
* Mbstring PHP Extension
* OpenSSL PHP Extension
* PDO PHP Extension
* Tokenizer PHP Extension
* XML PHP Extension

La aplicacion tambien precisas el siguiente modulo de PHP para procesar imagenes

* Imagemagick PHP Extension

Requerido por el modulo de excel de la aplicacion

* PHP extension php_zip enabled
* PHP extension php_xml enabled
* PHP extension php_gd2 enabled
* PHP extension php_iconv enabled 0
* PHP extension php_simplexml enabled
* PHP extension php_xmlreader enabled
* PHP extension php_zlib enabled

De todas formas, si la instalacion ejecutando `$ composer install` devuelve error, probablemente es por falta de alguna extension. Favor de prestar atencion a las mismas

### Mapbox

Para que los mapas funcionen, deben crear una cuenta en Mapbox.

1. Entren en www.mapbox.com
2. Crear una cuenta
3. Loguearse, y en el dashboard, hacer clic en "Create a token"
4. Darle un nombre al token y dejar los siguientes Token Scope tildados:
```
Public scopes
Styles:tiles
Styles:read
Fonts:read
Datasets:read
Vision:read
```
5. En token restrictions deberan restringir el uso bajo una URL. Para eso tendran que tener definido el dominio asignado para la plataforma. Si llevan a cabo un entorno de staging, deberan tambien agregar el mismo dentro del listado de URLs habilitados.

> **IMPORTANTE**: Revisen los pricings de Mapbox, cuenta con un tier gratuito que deberia ser suficiente para el uso que le dan a la plataforma, pero esten atentos al mismo.

### SMTP - Mails

Se requiere hacer conexion por SMTP a un servidor de correo para hacer el envio de correos electronicos. Este paso, es puramente acorde al equipo de deployment. En las variables de entorno debe configurarse esta conexion SMTP.

## Instalacion

### 1. Clonar el repositorio

```
ubuntu:~$ cd /var/www
ubuntu:/var/www$ git clone https://github.com/DemocraciaEnRed/sumen-app.git
```



```bash 
ubuntu:/var/www$ cd sumen-app
ubuntu:/var/wwwsumen-app$ git pull
ubuntu:/var/wwwsumen-app$ composer dump-autoload -o
ubuntu:/var/wwwsumen-app$ composer install
```

Crear un archivo .env

```

```

Correr el siguiente:
```
ubuntu:/var/wwwsumen-app$ php artisan key:generate
```
Copiar la `APP_KEY` y colocarla en el .env

Ejecutar el siguiente comando. Se va a crear un symlink en /public de la carpeta "storage/public" que es donde se almacenan los archivos de la aplicacion. 

```
ubuntu:/var/wwwsumen-app$ php artisan storage:link
```

Crear una base de datos Mysql. Elija el nombrer que quiera y ese sera el valor en DB_DATABASE. Debe ser: `'charset' => 'utf8mb4' // 'collation' => 'utf8mb4_unicode_ci'`. Los datos de conexion a la base de datos, deben ir en el `.env`

Pasariamos a crear la base de datos, por ahora, la migracion inicial.

```
ubuntu:/var/wwwsumen-app$ php artisan migrate:fresh --force
```

Por ultimo se debe hacer la conexion de las Queue a **redis**. Para eso, **redis** debe estar instalado. Comprobado que **redis** esta instalado localmente. Una vez hecho, pasariamos a crear los procesos que **supervisor** irá monitoreando. Para eso, Laravel tiene un muy buena documentación al respecto: [https://laravel.com/docs/7.x/queues#supervisor-configuration](https://laravel.com/docs/7.x/queues#supervisor-configuration)

`sumen-app` tiene 2 colas de Jobs, una cola llamada "mailer" y otra llamada "database". Ambas son para hacer las notificaciones de la aplicacion, de forma asincronica, sin entorpecer a PHP en los procesos del servidor. Para eso se puede hacer un solo proceso para **supervisor** o dos procesos por separado, cosa de que si falla uno, el otro siga andando.

Ejemplo de 2 procesos monitoreados por supervisor
```
[program:participes-mailer]
command=/RunCloud/Packages/php74rc/bin/php artisan queue:work redis --queue=mailer
redirect_stderr=true
autostart=true
autorestart=true
user=runcloud
numprocs=1
directory= /home/runcloud/webapps/participes
process_name=%(program_name)s_%(process_num)s

[program:participes-database]
command=/RunCloud/Packages/php74rc/bin/php artisan queue:work redis --queue=database
redirect_stderr=true
autostart=true
autorestart=true
user=runcloud
numprocs=1
directory= /home/runcloud/webapps/participes
process_name=%(program_name)s_%(process_num)s
```

Ejemplo de un solo proceso que monitorea ambas colas

```
[program:participes-queue]
command=/RunCloud/Packages/php74rc/bin/php artisan queue:work redis --queue=mailer,database
redirect_stderr=true
autostart=true
autorestart=true
user=runcloud
numprocs=1
directory= /home/runcloud/webapps/participes
process_name=%(program_name)s_%(process_num)s
```

Pueden agregar mas de `numprocs` de querer, es a criterio del equipo y del user load de la web.

Una vez hecho esto, intentar ingresar por broser a `https://dominio.com/start` y alli se puede crear el usuario admin para iniciar.

Cuando hayan creado el usuario admin inicial, deben volver a modificar el `.env` y cambiar `APP_ENV=local` a `APP_ENV=production` y `APP_DEBUG=false`}

Una vez hecho eso, ejecutar:

```
ubuntu:/var/wwwsumen-app$ php artisan config:clear
```
