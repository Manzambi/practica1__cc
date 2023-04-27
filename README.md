## Nombre: Manzambi Antonio Kimbungu Doge

### Conceptos 
      
#### Ownclod
     Owncloud es un servicio de almacenamiento y sincronización de archivos multiplataforma que se puede instalar en nuestro servidor.
Con él, cualquier usuario con una cuenta puede subir información y se sincronizará con los demás usuarios en cualquiera de sus dispositivos.

   La diferencia principal entre Owncloud y otros servicios de almacenamiento en la nube es que al estar instalado en nuestro propio hosting nos da gran libertad para diseñar nuestro propio sistema de almacenamiento en la nube y nos permite tener el control de muchos otros aspectos relativos a la privacidad.


#### Redis
  Redis es un motor de base de datos en memoria, basado en el almacenamiento en tablas de hashes (clave/valor) pero que opcionalmente puede ser usada como una base de datos durable o persistente. Está escrito en ANSI C por Salvatore Sanfilippo, quien es patrocinado por Redis Labs. Está liberado bajo licencia BSD por lo que es considerado software de código abierto.

#### LDAP
Lightweight Directory Access Protocol, o LDAP por sus siglas, es uno de los principales protocolos de autenticación que se desarrolló por los servicios de directorio. LDAP históricamente se ha usado como una base de datos de información, principalmente para almacenar información como:
         
            Usuarios
            Atributos acerca de esos usuarios
            Privilegios de la membresía de grupos 
            … y más
            
            
 #### Mariadb
   MariaDB es una base de datos. Es muy similar a MySQL, que es un sistema de gestión de bases de datos, y, de hecho, es un fork (o bifurcación) de MySQL. La base de datos MariaDB se utiliza para diversos fines, como el almacenamiento de datos, el comercio electrónico, funciones a nivel empresarial y las aplicaciones de registro. 

   MariaDB te permite llevar a cabo tus proyectos de manera eficiente. Funciona en cualquier base de datos en la nube y a cualquier escala, grande o pequeña.


## Entornos de desarrollo/produccion utilizadas
   
  para esta practica utilizaremos 4 servios: 
        1 - OWCLOUD -como el fronted
        2-  MYSQL/MARIADB Ccomo mi base de datos
        3 - Redis que trabaja como la memoria cache 
        4 - Integracion LDAP -  para la creacion y autenticacion de los usuarios
        
 Para el desarrollo y el despliegue del mismo usé windonws 11, y Docker version 20.10.24, build 297e128, donde despliegue los contenedores
 define un archivo docker-compose.yml version 3.8, y en ella define 4 servicios.
 Para el servicio owncloud creé un contenedor llamado owncloud, donde define su imagen, para este contenedor usamos el imagen owncloud/server:10.12
 y sus variables de entornos, como vemos abajo:

         owncloud:
            image: owncloud/server:10.12.0
            container_name: owncloud
            restart: always
            environment:
              OWNCLOUD_DOMAIN: localhost        # usamos para dar un dominio a nuestro contenedor que por defecto es localhost
              OWNCLOUD_DB_TYPE: mysql           # usamos para definir el tipo de base de datos a usar
              OWNCLOUD_DB_NAME: owclouddb       # el nombre de nuestro contenedor
              OWNCLOUD_DB_USERNAME: mydb_user   # usamos para definir el nombre del segundo usuario de nuestra base de datos.
              OWNCLOUD_DB_PASSWORD: mydb_pwd    # usamos para definir el password de este usuario db tiene que coincidir con la configurada en bd
              OWNCLOUD_DB_HOST: mariadb-master  # esta variable del entorno apunto al host que ejecuta nuestra bd
              OWNCLOUD_ADMIN_USERNAME: antonio      # Usuario admin Owncloud
              OWNCLOUD_ADMIN_PASSWORD: antonio      # Contraseña admin Owncloud 
              OWNCLOUD_MYSQL_UTF8MB4: true      # 
              OWNCLOUD_REDIS_ENABLED: true
              OWNCLOUD_REDIS_HOST: redis
            depends_on:
              - mariadb-master           # depnds_on usamos para decir que nuestro servicio oncloud va a depender del otro servicio mariadb, redis y openldap
              - redis
              - openldap
            volumes:
              - files_owncloud:/mnt/data     # Los volumenes para la persistencia de los datos, para eso este volumen estará en nuestro gestor de contenedor
   
   Para el desarrollo de nuestra bd usamos la imagen de mysql version 8: las demas configuraciones se ve abajo:
                
                 mariadb-master:
                    image: mysql:8.0
                    container_name: mariadb-master
                    restart: always
                    env_file:
                     - ./mariadb/master/mariadb_master.env
                    volumes:
                      - data_mariadb_master:/var/lib/mysql
                      - ./mariadb/master/mariadb.conf.cnf:/etc/mysql/conf.d/mysql.conf.cnf
                    networks:
                      pratica1:
                      
   Para Redis use la version 7 de imagen Redis 
   y para LDAP use la version bitnami/openldap:2.5
          algunas de mis configuraciones LDAP son:
          
                openldap:
                    image: bitnami/openldap:2.5
                    container_name: openldap
                    restart: always
                    environment:
                      LDAP_ORGANISATION: antoniogomez
                      LDAP_DOMAIN: antoniogomez.org
                      LDAP_ROOT: dc=antoniogomez,dc=org   # 
                      LDAP_ADMIN_USERNAME: antonio     # administrador
                      LDAP_ADMIN_PASSWORD: antonio    #  administrador 
 
  Aunque por si docker compose crea una red en caso de no especificar, creamos una red para la comuniacion entre ellas que llamamos de practica1.
  y tambien creamos volumes para cada sericio:
                        
                      volumes:
                          haproxy_conf:
                            driver: local
                          data_mariadb_master:
                            driver: local
                          data_mariadb_slave:
                            driver: local
                          redis_cache:
                            driver: local
                          openldap_data:
                            driver: local
                          files_owncloud:
                            driver: local

### Descripción de la práctica
En esta practica la vamos a dividir en dos secciones que explicaremos a continuacion:
1-La primera seccion  será crear un orquestrador de docker-compose doonde estarán 4 servicios.
2-La segunda seccion será crear una replica del frontend onde una sera la maestra y la otra esclava en caso de que una se caya el servicio continua, 
y  una haproxy que se encarga de hacer el enrutamiento y balanceo de carga, y una replica de nuestra base de datos
donde una sera maestra y otra esclava. 

 ## Seccion 1          
   Diseño y despliegue de un servicio Owncloud basado en contenedores según la arquitectura descrita en el Escenario 1 (Ver sección Tipos de arquitecturas de cloud propuestas). En particular, se requiere que este servicio incluya, al menos, 4 microservicios:

        Servicio web ownCloud
        MariaDB
        Redis
        LDAP (autenticación de usuarios)
    
  Para esta seccion no entraremo mucho en detalle porque ya casi todo hemos explicado en el primer apartado: Entornos de desarrollo
  A continuacion, procedemos a explicar de como se comunican los 4 contenedores, para que haya comunicacion entre ellas es necesario en primera instancia que 
  estén en la misma red, y es lo que hicimos, metemos los 4 servicios en una sola red, y hicimos la persistencia de los datos para cada servicio, una vez que 
  borremos el contenedor nuestros archivos si mantienen intactos, es decir, una vez que construimos otro contenedor solo basta hacer referencia al volumen, con
  eso recuperamos la informacion que teniamos antes, los datos son sicronos, se cambiamos en el contenedor refleja en el volumen y vice-versa.
  
  Para la autenticacion de los usuarios no basta con solo usar el contenedor y hacer la refencia de dependencia en el servicio owncloud, es necesario tambien
  hacer la integracion desde la propia interface owncloud, para eso ingresamos como el administrador, apartir de la seccion, Marktet buscamos la integracion 
  LDAP lo instalamos y hacemos la autenticacion de los usuarios. El DN de mi organizacion es:
               Cn: Nombre comun = antonio
               Dominio: antoniogomez.org
               dc: Dominio del component = antoniogomez,org
  Por defecto el servicio openldap usa el puerto 389 o 1389. una vez puesto todos los parametros y definiciones(grupos, tc.), procedemos a hacer la detectacion 
  de nuestros Usuarios, una vez que los detecte ya estamos listo para hacer autentificacion.
  
  Para recordar que en nuestra base de datos habiamos definido como owclouddb en esta base de datos va a contener una entidad responsable de almacenar los usuarios 
  autenticados, es decir que en un primer inicio solo va a contener un usario registrado que el admistrador owncloud.
  
  Para gestionar nuestra base de datos creamos dos usuarios un root que es el principal que tiene acceso a todo, y otro usuario secundario, que solo tiene acceso
  a visualizar las informaciones de los onwcloud que se crean desde la integracion LDAP, hay que tener en cuenta que tanto el admin de bd como el user secundario
  solo sirven para visualizacion y gestionamiento de nuestra base de datos, no podemos acceder con ello a owncloud, de igual modo los usuarios que acceden a owncloud
  no tienen permisos para visualizar las informaciones contenidas en nuestra bd.
  
  Para visualizacion grafica de nuestra base de datos usamos un servicio que llamamos de phpmyadmin con image: phpmyadmin version 5, como vemos abajo:     
                        
                 phpmyadmin:
                    image: phpmyadmin:5
                    container_name: phpmyadmin
                    restart: always
                    ports:
                      - 8080:80
                    environment:
                      - PMA_ARBITRARY=1
                    networks:
                      pratica1:
                
  Para el caso de Redis sera como una memoria cache de nuestro servicio owncloud, donde almacenará las informaciones mas recorrentes para evitar muchas demoras.
  

## Seccion 2
    
   Para la segunda seccion vamos a crear un una replica de del servicio owncloud, una para mi base de datos.
   en este caso una Replica mas del servicio owncloud que lo llamamos de owncloud2, en el caso de este servicio va a depender la mariadb-esclava.
   
   En este ecenario los puertos lo definimos desde el servicio haproxy, que se encarga de hacer el balaceo, es decir, si va a responsablizar de hacer 
   el enrutamiendos de las peticiones, usando un unico puerto de escucha, la configuracion de este servicio podemos ver abajo:
   
                  haproxy:
                    image: haproxy:2.7
                    container_name: haproxy
                    restart: always
                    depends_on:
                      - owncloud
                    environment:
                      BALANCE: source
                      OWNCLOUD_PORT: 8080
                    ports:
                      - '80:80'
                    volumes:
                      - haproxy_conf:/usr/local/etc/haproxy
                      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
                    networks:
                      pratica1:
    
   Este contenedor dependera de ownclou master como vimos, pero viene la pregunta, que pasa si este servicio owncloud master se cae? pues nada 
   el servicio sigue, pero viene otra pregunta donde se hace este balanceo de tal modo que persiste el servicio? esto lo configuramos desde haproxy.cfg
    
    para que el sercicio persista tengo que apuntar para los dos servicios, como uso uno solo puerto en caso de que uno falle, el otro esta disponoble, esta
    configuracion podemos ver en la figura de abajo:
                
                frontend http
                    bind *:80
                    default_backend owncloud

                backend owncloud
                    server owncloud1 owncloud:8080 check
                    server owncloud2 owncloud2:8080 check
     
      El codigo de arriba es una porcion de la configuracion haproxy.cfg. la primera instruccion fronted http hace referencia al nombre que le demos: http. bind *:80 
      significa que todas las peticiones se van a escuchar desde el puerto 80, nuestro backend lo definimos con el nombre owncloud de tal modo que desde la primera
      configuracion hicimos referencia del mismo default_backend owncloud. 
      
      La configuracion en Backend owncloud, es donde hacemos los enrutamientos, es decir tengo dos serviodres que trabajan internamente, desde haproxy y que apuntan
      para el mismo puerto, eso desde alli donde persiste nuestro servidor, porq si uno cae el otro esta en funcionamiento
      

## Conclusiones
    
Despues de mucha investigacion solo puedo concluir, que usar el balanceo y replicas de los servicios es muy util, para evitar fallos en el sistema o de accidentes
ocasionales, pues aydan a tener una mejor eficacia y funcionamiento de la aplicacion.

para acceder al [docker-compose completo presiona aqui....](https://github.com/Manzambi/practica1__cc/blob/main/Practica1/docker-compose.yml) 

## Referencias bibliográficas y recursos utilizados

https://hub.docker.com/r/bitnami/owncloud
https://github.com/ccano/cc2223/tree/main/practice1
https://doc.owncloud.com/server/next/admin_manual/configuration/user/user_auth_ldap.html
https://doc.owncloud.com/server/next/admin_manual/installation/docker/
https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts
https://www.haproxy.com/documentation/hapee/latest/configuration/acls/syntax/
https://hub.docker.com/
https://www.webempresa.com/blog/que-es-owncloud-y-para-que-sirve.html
https://es.wikipedia.org/wiki/Redis
   
