# Trabajando con SELinux

SELinux (Security-Enhanced Linux) es un mecanismo de seguridad, que se implementa directamente en el Kernel basado en MAC (Mandatory Access Control) que define el acceso y los derechos de transición de cada usuario, aplicación, proceso y archivo en el sistema, utilizando políticas de seguridad, es decir, indica cuando un objeto o usuario puede acceder a otro objeto.

SELinux funciona como un módulo en el Kernel y complementa a los métodos de seguridad tradiccionales de controles de acceso discrecionales, como las ACL en Linux.

En SELinux se trabajo con contexto de seguridad, que lo construyen 4 atributos de seguridad.

* Identidad de usuario

> **NOTA**: Se pueden listar con el comando `semanage user -l`
~~~
sudo semanage user -l

                    Etiquetado MLS/       MLS/                          
    Usuario SELinux  Prefijo    Nivel MCS  Rango MCS                      Roles SELinux

    guest_u         user       s0         s0                             guest_r
    root            user       s0         s0-s0:c0.c1023                 staff_r sysadm_r system_r  unconfined_r
    staff_u         user       s0         s0-s0:c0.c1023                 staff_r sysadm_r system_r  unconfined_r
    sysadm_u        user       s0         s0-s0:c0.c1023                 sysadm_r
    system_u        user       s0         s0-s0:c0.c1023                 system_r unconfined_r
    unconfined_u    user       s0         s0-s0:c0.c1023                 system_r unconfined_r
    user_u          user       s0         s0                             user_r
    xguest_u        user       s0         s0                             xguest_r
~~~

* Role
* Tipo y Dominio
* Categoria y Nivel

Hay un control de niveles en la seguridad con los roles. Desde el mas bajo **user_r** y que mas privilegios tiene **sysadm_r**.

Hay que tener en cuenta que por defecto SELinux es activado en Centos o RHEL, esto lo podemos comprobar con el comando `getenforce`.

~~~
getenforce
    Enforcing
~~~

Como vemos esta activado en el modo `Enforcing` el cual advierte de de la violación de la política de seguridad y la impone. Pero también hay un modo llamado `Permissive` que solo advierte de la violación de la política. Además esta el modo `Disabled`, donde SELinux esta completamente desactivado.

Una manera de trabajar con SELinux es con los booleanos. Los booleanos son ajustes que permiten activar o desactivar las funciones de SELinux, se pueden listar el estado de estas con el comando `getsebool -a`, podemos filtrar con `egrep`.

~~~
getsebool -a | egrep ' on' | egrep 'httpd'
    httpd_builtin_scripting --> on
    httpd_can_network_connect --> on
    httpd_can_network_connect_db --> on
    httpd_enable_cgi --> on
    httpd_execmem --> on
~~~

> **NOTA**: si quieres un listado de los booleanos con el estado y la descripción de estos utiliza `semanage boolean -l`.

Si queremos cambiar el estado de los booleanos se utiliza el comando `setsebool` seguido del nombre y el estado.

**Ejemplo**

Esta sentencia hace que deniegue la conexión de un servidor web a la base de datos. Con la opción **P** el cambio es persistente en un reinicio de la máquina.
~~~
setsebool -P httpd_can_network_connect_db off
~~~

Otra opción que trae SELinux, es la detectar las aplicaciones que estan siendo denegadas por SELinux en el sistema. Gracias a los log, SELinux detectas los servicios denegados y nos muestra una descripción del bloqueo y como solucionarlos.

En este ejemplo mostramos como solucionar un bloqueo de SELinux hacia **php-fpm**, donde no tenía acceso de escritura en el directorio del DocumentRoot de Netcloud.

~~~
sudo sealert -a /var/log/audit/audit.log
.
.
.
    SELinux está negando a /usr/sbin/php-fpm de write el acceso a carpeta /usr/share/nginx/html/nextcloud/  nextcloud-data.

    *****  El complemento httpd_write_content (92.2 confidence) sugiere***********

    Si desea permitir que php-fpm tenga write acceso al nextcloud-data directory
    Entoncesnecesita modificar la etiqueta a '/usr/share/nginx/html/nextcloud/nextcloud-data'
    Hacer
    # semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/html/nextcloud/nextcloud-data'
    # restorecon -v '/usr/share/nginx/html/nextcloud/nextcloud-data'

    *****  El complemento catchall_boolean (7.83 confidence) sugiere**************

    Si quiere allow httpd to unified
    Entoncesdebe informar a SELinux de ello activando el indicador 'httpd_unified'.

    Hacer
    setsebool -P httpd_unified 1

    *****  El complemento catchall (1.41 confidence) sugiere**********************

    Si cree que de manera predeterminada se debería permitir a php-fpm el acceso write sobre    nextcloud-data directory.     
    Entoncesdebería reportar esto como un error.
    Puede generar un módulo de política local para permitir este acceso.
    Hacer
    permita el acceso temporalmente ejecutando:
    # ausearch -c 'php-fpm' --raw | audit2allow -M mi-phpfpm
    # semodule -X 300 -i mi-phpfpm.pp


    Información adicional:
    Contexto de origen            system_u:system_r:httpd_t:s0
    Contexto Destino              unconfined_u:object_r:httpd_sys_content_t:s0
    Objetos Destino               /usr/share/nginx/html/nextcloud/nextcloud-data [
                                  dir ]
    Origen                        php-fpm
    Dirección de origen           /usr/sbin/php-fpm
    Puerto                        <Unknown>
    Nombre de Equipo              <Unknown>
    Paquetes RPM Fuentes          php-
                                  fpm-7.2.11-2.module_el8.1.0+209+03b9a8ff.x86_64
    Paquetes RPM Destinos         
    RPM de Políticas              selinux-policy-3.14.3-20.el8.noarch
    SELinux activado              True
    Tipo de política              targeted
    Modo impositivo               Enforcing
    Nombre de equipo              salmorejo
    Plataforma                    Linux salmorejo 4.18.0-80.11.2.el8_0.x86_64 #1 SMP
                                  Tue Sep 24 11:32:19 UTC 2019 x86_64 x86_64
    Cantidad de alertas           13
    Visto por primera vez         2020-01-16 08:49:25 UTC
    Visto por última vez          2020-02-15 16:43:30 UTC
    ID local                      5bc050d1-eb1d-4e92-b41c-d2142923c3b3

    Mensajes raw de aviso
    type=AVC msg=audit(1581785010.587:6309): avc:  denied  { write } for  pid=19537 comm="php-fpm"  name="nextcloud-data" dev="vda1" ino=1124277 scontext=system_u:system_r:httpd_t:s0   tcontext=unconfined_u:object_r:httpd_sys_content_t:s0 tclass=dir permissive=1


    type=SYSCALL msg=audit(1581785010.587:6309): arch=x86_64 syscall=access success=yes exit=0  a0=7ff57e081358 a1=2 a2=0 a3=656e2f6c6d74682f items=0 ppid=19535 pid=19537 auid=4294967295 uid=997   gid=994 euid=997 suid=997 fsuid=997 egid=994 sgid=994 fsgid=994 tty=(none) ses=4294967295 comm=php-fpm    exe=/usr/sbin/php-fpm subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=access AUID=unset    UID=nginx GID=nginx EUID=nginx SUID=nginx FSUID=nginx EGID=nginx SGID=nginx FSGID=nginx

    Hash: php-fpm,httpd_t,httpd_sys_content_t,dir,write
.
.
.
~~~

Tal como nos muestra en la ayuda, nos sugiere que utilicemos el comando `semanage fcontext` para cambiar el contexto por defecto de algunos fichero o directorios.

~~~
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/html/nextcloud/nextcloud-data'
restorecon -v '/usr/share/nginx/html/nextcloud/nextcloud-data'
~~~

Como podemos ver ahora la dirección por defecto es `/usr/share/nginx/html/nextcloud/nextcloud-data` con el contexto `httpd_sys_rw_content_t`.

~~~
sudo semanage fcontext -l | grep httpd_sys_rw_content_t | grep nginx
    /usr/share/nginx/html/nextcloud/nextcloud-data     all files          system_u:object_r:httpd_sys_rw_content_t:s0 
~~~

> **NOTA**: Hay muchos comandos para la administración del contexto de archivos de SELinux a parte de `semanage fcontext`, como por ejemplo `chcon` que es igual pero los cambios son temporales.

Ahora vamos a activar políticas de seguridad para los diferentes servicios que tenemos funcionando en la máquina Salmorejo.

**Wordpress**

Vamos a permitir la conexión de Nginx a una base de datos, además de permitir el acceso a una red o puerto remoto. Estas opciónes es muy importante ya que es necesaria también para Nextcloud, phpMyAdmin, Mezzanine

Antes de configurarlos, tenemos que ver si el puerto TCP/80 del http esta activado, para eso vamos a ejecutar el siguiente comando:

~~~
sudo semanage port -l | grep '^http_port_t'
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
~~~

Como podemos ver lo tenemos configurado. En el caso de que no tengamos configurado el puerto 80 en `http_port_t`, lo activariamos de la siguiente forma:

~~~
semanage port -a -t  http_port_t -p tcp 80
~~~

* Para la conexión a una red o puerto remoto.
~~~
sudo setsebool -P httpd_can_network_connect 1
~~~
* Para la conexión de servidores de base de datos.
~~~
sudo setsebool -P httpd_can_network_connect_db 1
~~~

Comprobamos que estan activados
~~~
sudo semanage boolean -l | egrep 'httpd_can_network_connect_db '
    httpd_can_network_connect_db   (encendido,encendido)  Allow httpd to can network connect db
sudo semanage boolean -l | egrep 'httpd_can_network_connect '
    httpd_can_network_connect      (encendido,encendido)  Allow httpd to can network connect
~~~

Cambiamos el contexto de los directorios de Wordpress, estos nos lo ha notificado SELinux con el comando `sealert -a /var/log/audit/audit.log`.
~~~
sudo semanage fcontext -a -t httpd_sys_content_t /usr/share/nginx/html/wordpress
sudo semanage fcontext -a -t httpd_sys_rw_content_t /usr/share/nginx/html/wordpress/wp-config.php
sudo semanage fcontext -a -t httpd_sys_rw_content_t /usr/share/nginx/htmlwordpress/wp-content
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/html/wordpress/wp-content/languages/plugins/akismet-es_ES.po'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/html/wordpress/wp-admin/includes/update-core.php'
~~~

**NextCloud**

> **NOTA**:Tenemos que tener activado los booleanos **httpd** activados en la parte de Wordpress. 

Para Nextcloud vamos a habilitar que httpd pueda ejecutar programas que requieren direcciones de memoria.
~~~
sudo setsebool -P httpd_execmem 1
~~~

Comprobamos que esta activado
~~~
sudo semanage boolean -l | egrep 'httpd_execmem '
    httpd_execmem                  (encendido,encendido)  Allow httpd to execmem
~~~

Cambiamos el contexto de los directorios de Nextcloud.
~~~
sudo semanage fcontext -a -t httpd_sys_rw_content_t /usr/share/nginx/html/nextcloud/nextcloud-data
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/html/nextcloud/nextcloud-data/nextcloud.log'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/nginx/html/nextcloud/nextcloud-data/appdata_ocgvo78yv4ja/appstore/apps.json'
~~~

**phpMyAdmin**

> **NOTA**:Tenemos que tener activado los booleanos **httpd** activados en la parte de Wordpress. 

Cambiamos el contexto de los directorios de phpMyAdmin.
~~~
sudo semanage fcontext -a -t httpd_sys_content_t /usr/share/nginx/html/phpmyadmin
sudo semanage fcontext -a -t httpd_sys_content_t /usr/share/nginx/html/phpmyadmin
~~~

**Mezzanine**

> **NOTA**:Tenemos que tener activado los booleanos **httpd** activados en la parte de Wordpress. 

Cambiamos el contexto de los directorios de Mezzanine.
~~~
sudo semanage fcontext -a -t httpd_sys_rw_content_t /usr/share/nginx/html/iaw_mezzanine
~~~

Para mezzanine tenemos que habilitar un política mas general, para que permita un conexión con Unicorn.
~~~
sudo ausearch -c 'nginx' --raw | audit2allow -M mi-nginx
semodule -X 300 -i mi-nginx.pp
~~~

Esto lo que hace es, primero con `ausearch` busca archivos de registro de auditoria para eventos específicos, en nuestro caso nginx. Luego crea una serie de reglas a partir de auditoria anterior, donde permite las acciones que han sido denegadas.

Y el comando `semodule` activa el paquete de reglas creado anteriormente con la prioridad indicada con **-X**.

**Cliente Bacula**

Tambien en el cliente de bacula habilitamos el puerto 9102 con el comando `firewall-cmd`.
~~~
firewall-cmd --zone=public --permanent --add-port 9102/tcp
firewall-cmd --reload
~~~

Y un vez habiliado, tenemos que comprobar que en SELinux lo tenemos disponible.

~~~
sudo semanage port -l | grep 9102
    hplip_port_t                   tcp      1782, 2207, 2208, 8290, 8292, 9100, 9101, 9102, 9220, 9221, 9222, 9280, 9281, 9282, 9290, 9291, 50000, 50002
~~~