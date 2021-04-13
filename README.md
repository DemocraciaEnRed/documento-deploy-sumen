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

Requerido por el modulo de excel (ver [Laravel Excel](https://laravel-excel.com/)) de la aplicación

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

### Google No Captcha

Como utilizamos Google reCaptcha, tambien es importante que creen un reCaptcha para el sitio y que en las variables de entorno ingresar los valores para `NOCAPTCHA_SECRET` y `NOCAPTCHA_SITEKEY`. Mas información en [https://www.google.com/recaptcha/](https://www.google.com/recaptcha/)


### SMTP - Mails

Se requiere hacer conexion por SMTP a un servidor de correo para hacer el envio de correos electronicos. Este paso, es puramente acorde al equipo de deployment. En las variables de entorno debe configurarse esta conexion SMTP.

---

## Preinstalaciones

### Redis

Utilizariamos las instalaciones de apt de Ubuntu. Para eso actualizamos las apt package cache y luego instalamos redis.

```
sudo apt update
sudo apt install redis-server
```

Esto va a descargar e instalar REDIS y sus dependencias. Siguiendo a esto, hay una confifuracion importante a hacer en el archivo de configuracion de REDIS, que se genera automaticamente durante la instalación.

Abrimos el siguiente archivo en `nano`:
```
sudo nano /etc/redis/redis.conf
```

Dentro del archivo, se encuentra la "supervised directive". La directiva permite declarar un init systempara que maneje Redis como un servicio, proporcionando mas control sobre su operacion. La supervised directive esta declarada como NO por default. Siendo que se esta corriendo en Ubuntu (esperemos...), que usa el systemd init system, cambiamos esto a systemd:

```
. . .

# If you run Redis from upstart or systemd, Redis can interact with your
# supervision tree. Options:
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#                        UPSTART_JOB or NOTIFY_SOCKET environment variables
# Note: these supervision methods only signal "process is ready."
#       They do not enable continuous liveness pings back to your supervisor.
supervised systemd

. . .
```

Ese es el unico cambio que tendriamos que hacer en el archivo de configuracion de Redis. Por lo tanto, guardamos y cerramos el editor una vez hecho. Luego, hay que reiniciar el servicio de Redis para que refleje los cambios hechos en el archivo de configuracion:

```
sudo systemctl restart redis.service
```

Con esto, ya estaria instalado y configurado Redis. Antes de continuar, seria bueno testear Redis.

Comenzamos chequeando que el servicio de REDIS ya se encuentra corriendo

```
sudo systemctl status redis
```

Si esta corriendo sin ningun error, este comando producirá una salida similar a la siguiente:

```
Output
● redis-server.service - Advanced key-value store
   Loaded: loaded (/lib/systemd/system/redis-server.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2018-06-27 18:48:52 UTC; 12s ago
     Docs: http://redis.io/documentation,
           man:redis-server(1)
  Process: 2421 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 2424 ExecStart=/usr/bin/redis-server /etc/redis/redis.conf (code=exited, status=0/SUCCESS)
 Main PID: 2445 (redis-server)
    Tasks: 4 (limit: 4704)
   CGroup: /system.slice/redis-server.service
           └─2445 /usr/bin/redis-server 127.0.0.1:6379
. . .
```
Aquí, puede ver que Redis se está ejecutando y ya está habilitado, lo que significa que está configurado para iniciarse cada vez que se inicia el servidor.

Para probar que Redis está funcionando correctamente, conéctese al servidor usando el cliente de línea de comandos:

```
redis-cli
```

En el CLI puede probar la conectividad con el ping command:

```
> ping
Output
PONG
```

## supervisor y config

Instalar supervisor utilizando apt

`$ apt-get install supervisor`

El prebuild viene con un script que require que el OS vuelva a iniciar. Pero podemos forzarlo haciendo

`$ service supervisor restart`

Ahora tenemos Supervisor instalado.

---

# Instalación

Clonar el repositorio

```
ubuntu:~$ cd /var/www
ubuntu:/var/www$ git clone https://github.com/DemocraciaEnRed/sumen-app.git
```

Luego entramos en la carpeta, hacemos un git pull (por las dudas) e instalamos las dependencias.

``` 
ubuntu:/var/www$ cd sumen-app
ubuntu:/var/www/sumen-app$ git pull
ubuntu:/var/www/sumen-app$ composer dump-autoload -o
ubuntu:/var/www/sumen-app$ composer install
```

#### Variables de entorno

Copiamos el archivo `.env.example` y lo llamamos `.env`

``` 
ubuntu:/var/www/sumen-app$ cp .env.example .env
ubuntu:/var/www/sumen-app$ nano .env
```

Tener en cuenta los `#####COMPLETAR######`, que son importantes. Cualquier problema en la documentacion de Laravel podran encontrar mas información: [https://laravel.com/docs/8.x/configuration#environment-configuration](https://laravel.com/docs/7.x/configuration#environment-configuration)


> **NOTA**: Por ahora, `APP_ENV=local` y `APP_DEBUG=false` que son los estados de development, por ahora dejemoslo asi, hasta verificar que la plataforma esté en funcionamiento y se puede cambiar para un entorno de produccion a `APP_ENV=production` y `APP_DEBUG=false` (`APP_DEBUG=false` habilita el Laravel Debugbar para ver algunos logs del funcionamiento)

`.env`

```
APP_NAME=Sumen
APP_ENV=local
APP_KEY=#####VER-ARTISAN-KEY:GENERATE-MAS-ABAJO######
APP_DEBUG=false
APP_URL#####COMPLETAR######

SUMEN_DISTRICTS=false

NOCAPTCHA_SECRET=#####COMPLETAR######
NOCAPTCHA_SITEKEY=#####COMPLETAR######

MAPBOX_API_KEY=#####COMPLETAR######
MAPBOX_MAP_STYLE=mapbox://styles/mapbox/light-v10

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=#####COMPLETAR######
DB_USERNAME=#####COMPLETAR######
DB_PASSWORD=#####COMPLETAR######

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=redis
SESSION_DRIVER=file
SESSION_LIFETIME=120

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_QUEUE=mailer,default

# Mailer.. si se usa mailgun, agregar lo siguiente
MAIL_MAILER=mailgun
MAILGUN_DOMAIN=#####COMPLETAR######
MAILGUN_SECRET=#####COMPLETAR######

# ENVs para Mailer
MAIL_HOST=#####COMPLETAR######
MAIL_PORT=#####COMPLETAR######
MAIL_USERNAME=#####COMPLETAR######
MAIL_PASSWORD=#####COMPLETAR######
MAIL_ENCRYPTION=#####COMPLETAR######
MAIL_FROM_ADDRESS=#####COMPLETAR######
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"

```

#### APP_KEY

Correr el siguiente:

```
ubuntu:/var/wwwsumen-app$ php artisan key:generate
```

Tecnicamente `key:generate` deberia de poner la `APP_KEY` dentro del `.env` pero por favor, verifique

#### Symlink storage (para archivos)

Ejecutar el siguiente comando. Se va a crear un symlink en /public de la carpeta "storage/public" que es donde se almacenan los archivos de la aplicacion. 

```
ubuntu:/var/wwwsumen-app$ php artisan storage:link
```

#### Base de datos / Migrations

Crear una base de datos MySQL o MariaDB. Si tiene un entorno visual como MySQL Workbench, puede crearlo haciendo conexion al DB. Elija el nombre que quiera y ese sera el valor en DB_DATABASE. Tambien configure: `'charset' => 'utf8mb4' // 'collation' => 'utf8mb4_unicode_ci'`. 

```sql

# Crear usuario -sumen- con password -sumen2020- (POR FAVOR USEN OTROS DATOS)
> CREATE USER 'sumen' IDENTIFIED BY 'sumen2020';

# Crear la base de datos -sumenDB-
> CREATE DATABASE sumenDB CHARACTER SET = 'utf8mb4' COLLATE = 'utf8mb4_unicode_ci';

# Dar permisos de ALL PRIVILEGES al user -sumen- sobre la DB -sumenDB-
> GRANT ALL PRIVILEGES ON 'sumenDB'.* TO 'sumen'@localhost;

# Aplicar los privilegios
> FLUSH PRIVILEGES;

# Puede ver los permiso (opcional)
> SHOW GRANTS FOR 'sumen'@localhost;
```

Los datos de conexion a la base de datos, deben ir en el `.env`

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=sumenDB
DB_USERNAME=sumen
DB_PASSWORD=sumen2020
```

Pasariamos a crear la base de datos, por ahora, la migracion inicial.

```
ubuntu:/var/wwwsumen-app$ php artisan migrate:fresh --force
```

##### Extra: DB_SPECIFIED_KEY_FIX=false

Si estan teniendo el siguiente error al correr el comando `php artisan migration -force`

```
[Illuminate\Database\QueryException]
SQLSTATE[42000]: Syntax error or access violation: 1071 Specified key was too long; max length is 767 bytes (SQL: alter table users add unique users_email_unique(email))

[PDOException]
SQLSTATE[42000]: Syntax error or access violation: 1071 Specified key was too long; max key length is 767 bytes
```

Entonces intentar cambiar en el .env `DB_SPECIFIED_KEY_FIX=true` y probar de nuevo. Dejo dos formas:

```
# Drop todas las bases de datos y reraliza de nuevo todas las migraciones
$ php migrate:reset
```

```
# Rollback todas las migraciones
$ php migrate:fresh
```

Cualquier cosa haciendo `php artisan` nos da mas informacion sobre el comando `migrate`

```
migrate
  migrate:fresh        Drop all tables and re-run all migrations
  migrate:install      Create the migration repository
  migrate:refresh      Reset and re-run all migrations
  migrate:reset        Rollback all database migrations
  migrate:rollback     Rollback the last database migration
  migrate:status       Show the status of each migration
```
Si no funciona, puede ver de habilitar el `InnoDB ROW_FORMAT=DYNAMIC` in MariaDB...

O instalar una version mas reciente de MySQL o MariaDB.

Some useful resources:

- https://webomnizz.com/how-to-fix-laravel-specified-key-was-too-long-error/
- https://github.com/laravel/framework/issues/17508
- https://mariadb.com/kb/en/innodb-dynamic-row-format/


#### Revisar que REDIS funciona

Por ultimo se debe hacer la conexion de las Queue a **redis**. Para eso, **redis** debe estar instalado. Comprobado que **redis** esta instalado localmente. 

#### Configurando supervisor

Una vez hecho, pasariamos a crear los procesos que **supervisor** irá monitoreando. Para eso, Laravel tiene un muy buena documentación al respecto: [https://laravel.com/docs/7.x/queues#supervisor-configuration](https://laravel.com/docs/7.x/queues#supervisor-configuration)

`sumen-app` tiene 2 colas de Jobs, una cola llamada "mailer" y otra llamada "database". Ambas son para hacer las notificaciones de la aplicacion, de forma asincronica, sin entorpecer a PHP en los procesos del servidor. Para eso se puede hacer un solo proceso para **supervisor** o dos procesos por separado, cosa de que si falla uno, el otro siga andando.

Lo que tenemos que hacer, es definir dos archivos. Uno para cada proceso. Estos se ubican dentro de la carpeta `/etc/supervisor/conf.d`

Comenzamos creando el primero, que es el proceso que se encarga de escuchar cuando hay notificaciones que entregar a usuarios (Notificaciones internas) Para eso hacemos

```
$ nano /etc/supervisor/conf.d/sumen-database.conf
```

Y copiamos y pegamos lo siguiente

```
[program:sumen-database]
command=php artisan queue:work redis --queue=database
redirect_stderr=true
autostart=true
autorestart=true
user=$$$$USUARIO$$$$
numprocs=1
directory= $$$$PATH-COMPLETO$$$$/sumen-app/
process_name=%(program_name)s_%(process_num)s
stderr_logfile=/var/log/sumen-database.err.log
stdout_logfile=/var/log/sumen-database.out.log
```

Luego vamos a hacer lo mismo pero para el proceso que escucha la cola de "mails", este es el proceso que se encargara de enviarlos. La idea es que este proceso sea el encargado de enviar emails de notificaciones, para no bloquear los hilos de ejecucion de la aplicacion web.

Hacemos entonces:

```
$ nano /etc/supervisor/conf.d/sumen-mailer.conf
```

*NOTA: Es importante que completen el valor de `$$$$PATH-COMPLETO$$$$` y `$$$$UASUARIO$$$$`*

Y copiamos y pegamos lo siguiente

```
[program:sumen-mailer]
command=php artisan queue:work redis --queue=mailer
redirect_stderr=true
autostart=true
autorestart=true
user=$$$$USUARIO$$$$$
numprocs=1
directory=$$$$PATH-COMPLETO-A-SUMEN-APP$$$$$/sumen-app/
process_name=%(program_name)s_%(process_num)s
stderr_logfile=/var/log/sumen-mailer.err.log
stdout_logfile=/var/log/sumen-mailer.out.log
```

Una vez que los archivos de configuracion fueron creados y guardados, podemos informar a Supervisor de nuestro nuevo programa utilizando el comando "supervisorctl"
Primero le decimos a Supervisor que mire por cualquier cambio o configuracion nueva en /etc/supervisor/conf.d ejecutando:

`supervisorctl reread`

Luego le decimos que aplique los cambios haciendo

`supervisorctl update`

Para ver el estatus de los procesos se pueden ingresar al command prompt de Supervisor haciendo

`supervisorctl`

Cada vez que entremos inicialmente nos va a mostrar los status de nuestros programas.

```
$ supervisorctl
mi-proceso             RUNNING    pid 12614, uptime 1:49:37
supervisor>
```

Si vemos "**RUNNING**" entonces esta funcionando correctamente, el problema es que esté constantemente en **STOPPED** o que este constantemente reiniciando


##### Extra: Ejemplos mas completos de supervisor.

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

#### Webserver

Segun Laravel, este seria lo basico para apache.

```
nano /etc/apache2/sites-available/sumen.dominio.com.conf
```

Copiar lo siguiente... **NOTA: Recuerden apuntar el dominio/subdominio a la IP del servidor en el DNS antes de hacer este paso**

```
<VirtualHost *:80>
  # CAMBIAR
  ServerAdmin sumen@municipio.gob.ar
  # CAMBIAR
  ServerName sumen.dominio.com
  # CAMBIAR
  ServerAlias www.sumen.dominio.com
  DocumentRoot /var/www/sumen-app/public
    
  <Directory /var/www/sumen-app/public>
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Order allow,deny
    allow from all
    Require all granted
  </Directory>
    
  LogLevel debug
  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Guardar el archivo y luego lo habilitamos haciendo

```
a2ensite sumen.dominio.com.conf
```

Y luego para aplicar los cambios reiniciar apache2

```
# Opcion 1 - Recarga apache2 con la nueva config
service apache2 reload

# Opcion 2 - Reinicia todo apache2
service apache2 restart
```

Si da error, se puede deshabilitar el sitio haciendo

```
a2dissite sumen.dominio.com.conf
```

Luego dependiendo de como quieran configurar el HTTPS, pueden aplicarles sus certificados (si es que tienen) o pueden instalar los de Lets Encrypt.

Para eso, sigan la guia de https://certbot.eff.org/

**NOTA: Antes de usar certbot por favor verifiquen que pueden entrar a la web por HTTP. Es necesario para el desafio que hace CERTBOT antes de instalar o expedir los formulario**

#### Creando el usuario admin por primera vez

Una vez hecho esto, intentar ingresar por browser a `https://sumen.dominio.com/start` y alli se puede crear el usuario admin para iniciar.

#### Ultimos toques

Cuando hayan creado el usuario admin inicial, deben volver a modificar el `.env` y cambiar `APP_ENV=local` a `APP_ENV=production` y `APP_DEBUG=false`

Una vez hecho eso, ejecutar:

```
ubuntu:/var/wwwsumen-app$ php artisan config:clear
```

Ante cualquier duda consultar mandando un email a [guillermo@democracyos.io](mailto:guillermo@democracyos.io)