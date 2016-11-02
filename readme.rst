===================================
Instalación GeoServer en producción
===================================


Acerca de
=========

Documento publicado bajo licencia Creative Commons reconocimiento compartir-igual (CC-by-sa). Consultar autoría en el histórico de commits. Contribuciones bienvenidas.


Prerrequisitos
==============

Hardware y SO
-------------

* Sistema Operativo: Recomendado Ubuntu 16.04 Server 64 bits
* CPU: 2-4 cores
* RAM: 1-2 GB
* Disco: Según cantidad de datos a publicar (contar con la caché de teselas)


Configuración inicial de la máquina
-----------------------------------

Configurar conexión y nombre de la máquina::

	ifconfig eth0 <public_ip> <mask>
	route add default gw <gateway>

	echo "<nombre>" > /etc/hostname
	hostname -F /etc/hostname

/etc/hosts debería contener::

	127.0.0.1    localhost.localdomain  localhost
	<public_ip>  <nombre>.example.com   <nombre>

Zona horaria::

	dpkg-reconfigure tzdata

Actualizar paquetes::

	apt-get update
	apt-get upgrade

Activar la actualización automática de paquetes::

	sudo apt-get install unattended-upgrades
	sudo dpkg-reconfigure -plow unattended-upgrades

Algunas utilidades básicas::

	apt-get install locate
	updatedb


Instalar y configurar firewall (iptables)::

	apt-get install iptables

	iptables -A INPUT -i lo -j ACCEPT
	iptables -A INPUT -p tcp -m multiport --destination-ports 22,80,443,8080,8443,5432 -j ACCEPT # Dejar los que se necesiten
	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	iptables -P INPUT DROP
	iptables -P FORWARD DROP

	iptables-save > /etc/iptables.rules

Crear /etc/network/if-pre-up.d/firewall con este contenidor::

	#!/bin/sh
	/sbin/iptables-restore < /etc/iptables.rules


Instalar fail2ban::

	apt-get install fail2ban
	service fail2ban restart


Crear usuarios SUDOers::

	adduser <name>
	adduser <name> sudo


Bloquear login de ROOT vía SSH. Cambiar esta línea /etc/ssh/sshd_config::

	PermitRootLogin no

Reiniciar servicio ssh::

	service ssh reload

Evitar el error 'too many files open'...

Añadir estas líneas a /etc/security/limits.conf ::

    *                hard    nofile          65535
    *                soft    nofile          65535
    root             hard    nofile          65535
    root             soft    nofile          65535


Tras reiniciar la máquina, comprobar límite con::

	cat /proc/<tomcat pid>/limits

Si el límite sigue siendo 1024 o 4096, tocar las siguientes líneas en /etc/init.d/tomcat::

	ulimit -Hn 65535
	ulimit -Sn 65535

(fuente: https://www.jayway.com/2012/02/11/how-to-really-fix-the-too-many-open-files-problem-for-tomcat-in-ubuntu/)

Instalar fuentes de Microsoft::

	apt-get install ttf-mscorefonts-installer

Instalar OpenJDK JRE 8 y Tomcat 8::

	apt-get install openjdk-8-jre
	apt-get install tomcat8


Añadir JAI y JAI-ImageIO nativos::

	cd /usr/lib/jvm/java-8-openjdk-amd64
	wget http://download.java.net/media/jai/builds/release/1_1_3/jai-1_1_3-lib-linux-amd64-jdk.bin
	sh jai-1_1_3-lib-linux-amd64-jdk.bin

	wget http://download.java.net/media/jai-imageio/builds/release/1.1/jai_imageio-1_1-lib-linux-amd64-jdk.bin
	export _POSIX2_VERSION=199209
	sh jai_imageio-1_1-lib-linux-amd64-jdk.bin


Reiniciar server	

	service tomcat8 restart


Instalar GDAL (1.11)::

	apt-get install gdal-bin


PostGIS
=======

Instalar PostgreSQL y PostGIS::

	apt-get install postgresql postgis
	apt-get install postgresql-9.5-postgis-2.2


Habilitar acceso local. En /etc/postgresql/9.5/main/pg_hba.conf::

	# TYPE  DATABASE        USER            ADDRESS                 METHOD
	local   all             postgres                                ident
	local   all             all                                     md5
	host    all             all             127.0.0.1/32            md5

Y en /etc/postgresql/9.5/main/postgresql.conf, descomentar::

    listen_addresses = 'localhost'

Reiniciar para aplicar cambios::

	service postgresql restart

Para acceder a la consola SQL::

	sudo -u postgres psql


Crear un nuevo "usuario"::

	CREATE USER usuario LOGIN PASSWORD '------' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE;


Crear una nueva BDD "geodatos" cuyo propietario sea "usuario"::

	sudo -u postgres createdb -O usuario geodatos


Habilitar capacidades "geo" en la base de datos::

	sudo -u postgres psql -d geodatos -c "CREATE EXTENSION postgis;"


Acceso remoto abriendo puerto
-----------------------------

En caso de tener que abrir directamente un puerto (opción menos segura):

  1. En /etc/postgresql/9.5/main/postgresql.conf::

       listen_addresses = '*' # O mejor, una lista de IPs, si son fijas.

  2. En /etc/postgresql/9.5/main/pg_hba.conf, añadir una línea específica de acceso para una combinación de IP, BDD y usuario determinados (a ser posible, no usar comodines o "all" para el acceso remoto).


Oracle Instant Client
=====================

Descargar el Oracle Instant Client del sitio web de Oracle:

    http://www.oracle.com/technetwork/database/features/instant-client/index-097480.html

Además del cliente "basic", se recomienda descargar las extensiones "jdbc" y "sqlplus". Por ejemplo:

* `oracle-instantclient11.2-basic-11.2.0.3.0-1.x86_64.rpm`
* `oracle-instantclient11.2-jdbc-11.2.0.3.0-1.x86_64.rpm`
* `oracle-instantclient11.2-sqlplus-11.2.0.3.0-1.x86_64.rpm`

En Ubunu, hará falta convertir los paquetes rpm a deb::

  sudo apt-get install alien
  sudo alien <paquete>.rpm
  sudo dpkg -i <paquete>.deb

Una vez instalados los tres paquetes, el cliente estará instalado en la ruta `/usr/lib/oracle/11.2/client64/`.

Crearemos un subdirectorio `tns` para añadir los ficheros con las configuraciones de conexión::

  mkdir /usr/lib/oracle/11.2/client64/tns/

En este directorio crearemos un fichero `tnsnames.ora` con la cadena de conexión::

  <NOMBRE_TNS> =
    (DESCRIPTION =
      (ADDRESS_LIST =
        (ADDRESS = (PROTOCOL = TCP)(HOST = <HOST_ORACLE>)(PORT = 1521))
      )
      (CONNECT_DATA =
        (SID = <SID_ORACLE>)
        (SERVER = DEDICATED)
      )
    )

Se puede comprobar la conexión con `sqlplus64` mediante los siguientes comandos::

  export LD_LIBRARY_PATH=/usr/lib/oracle/11.2/client64/lib/
  export TNS_ADMIN=/usr/lib/oracle/11.2/client64/tns/
  sqlplus64 <SCHEMA>/<PASSWORD>@<NOMBRE_TNS>

Tras instalar GeoServer, instalar también la extensión oficial de Oracle. Recordar copiar `ojdbc?.jar` en `WEB-INF/lib`.


Configuración de SSL (https) en tomcat 8
========================================

1. Autogenerar certificado (para pruebas; usar certificado real en producción)::

	cd /var/lib/tomcat8
	keytool -genkey -alias admin -keypass adminpass -keystore certificate.bin -storepass adminpass -keyalg RSA
	chown tomcat8:tomcat8 certificate.bin

2. Configurar `/var/lib/tomcat8/conf/server.xml` para usar los puertos 80 y 443::

    <Connector port="80" protocol="HTTP/1.1"
               connectionTimeout="20000"
               URIEncoding="UTF-8"
               redirectPort="443" />

    <Connector port="443" protocol="HTTP/1.1" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS"
               sslEnabledProtocols="v1.2,TLSv1.1,TLSv1"
               keystoreFile="certificate.bin" keystorePass="adminpass" />

3. Permitir a Tomcat usar puertos estándard, por debajo de 1024, usando authbind::

	apt-get install authbind

	touch /etc/authbind/byport/80
	chmod 500 /etc/authbind/byport/80
	chown tomcat8 /etc/authbind/byport/80

	touch /etc/authbind/byport/443
	chmod 500 /etc/authbind/byport/443
	chown tomcat8 /etc/authbind/byport/443

4. Editar /etc/default/tomcat8 y editar la directiva AUTHBIND::

	AUTHBIND=yes

5. Si sólo se quiere usar HTTPS, forzar su uso para todas las aplicaciones, inhabilitando el puerto HTTP. Añadir este contenido a /var/lib/tomcat8/conf/web.xml::

    <security-constraint>
        <web-resource-collection>
            <web-resource-name>Protected Context</web-resource-name>
            <url-pattern>/*</url-pattern>
        </web-resource-collection>
        <user-data-constraint>
            <transport-guarantee>CONFIDENTIAL</transport-guarantee>
        </user-data-constraint>
    </security-constraint>

6. Reiniciar tomcat::
	
	service tomcat8 restart


GeoServer
=========

Instalación base
----------------

GeoServer 2.10.0 (o "latest stable")::

	cd /var/lib/tomcat8/webapps/
	wget http://sourceforge.net/projects/geoserver/files/GeoServer/2.10.0/geoserver-2.10.0-war.zip
	apt-get install unzip
	unzip geoserver-2.10.0-war.zip
	rm -rf target/ *.txt geoserver-2.10.0-war.zip


Entorno JVM
-----------

Mover el GEOSERVER_DATA_DIR fuera de los binarios::

	mv /var/lib/tomcat8/webapps/geoserver/data /var/local/geoserver
	mkdir /var/local/geowebcache
	chown tomcat8:tomcat8 /var/local/geowebcache


Editar el fichero /etc/default/tomcat8 y añadir al final las rutas a Java, los datos, la caché, y parámetros de optimización::

	JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64

	GEOSERVER_DATA_DIR=/var/local/geoserver
	GEOWEBCACHE_CACHE_DIR=/var/local/geowebcache

	JAVA_OPTS="-server -Djava.awt.headless=true -Xms512m -Xmx1536m -XX:+UseConcMarkSweepGC -XX:NewSize=48m -DGEOSERVER_DATA_DIR=$GEOSERVER_DATA_DIR -DGEOWEBCACHE_CACHE_DIR=$GEOWEBCACHE_CACHE_DIR"

Reiniciar tomcat::

	service tomcat8 restart


Comprobación entorno
....................

Entrar a::

	http://<maquina>:8080/geoserver/web/

En "server status", combrobar que:
  * El Data directory apunta a /var/lib/geoserver_data
  * La JVM es la instalada (OpenJDK 1.8 64 bits)
  * Native JAI y Native JAI ImageIO están a "true"


Habilitar CORS
--------------

En `/var/lib/tomcat8/webapps/geoserver/WEB-INF/web.xml`, añadir::

    <filter>
        <filter-name>CorsFilter</filter-name>
        <filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>CorsFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <init-param>
        <param-name>cors.support.credentials</param-name>
        <param-value>true</param-value>
    </init-param>


Seguridad
---------

Seguir las notificaciones de seguridad que aparecen en la página principal de GeoServer:

  * Cambiar password de "admin".
  * Cambiar el master password.



Configuración Web
-----------------

Bajo "About & Status":

* Editar la información de contacto. Esto aparecerá en los servicios WMS públicos: dejar a "Claudius Ptolomaeus" es indecente.

Bajo "Data":

* Borrar todos los espacios de trabajo (workspaces) existentes.
* Borrar todos los estilos existentes (dirá que hay 4 que no los puede borrar, esto es correcto).

Bajo "Services":

* WCS: Deshabilitar si no va a usarse.
* WFS: Cambiar el nivel de servicio a "Básico" (a menos que queramos permitir la edición remota de datos vectoriales).
* WMS: En "Limited SRS list", poner sólo las proyecciones que deseamos anunciar en nuestro servicio WMS. Esto reduce el tamaño del GetCapabilities. Por ejemplo: **23029, 23030, 23031, 25829, 25830, 25831, 4230, 4258, 4326, 3857, 900913**.

Bajo "Settings":

* Global: Cambiar el nivel de logging a PRODUCTION_LOGGING.

Bajo "Tile Caching":

* Caching Defaults: Activar los formatos "image/png8" para capas vectoriales, "image/jpeg" para capas ráster, y ambas para los grupos de capas.

* Disk Quota: Habilitar la cuota de disco. Tamaño máximo algo por debajo de la capacidad que tenga la unidad de Tile Caché.


Cambio de datum con malla NTv2
------------------------------

Descargar el fichero de malla de:

  https://github.com/oscarfonts/gt-datumshift/blob/master/icc-tests/src/test/resources/org/geotools/referencing/factory/gridshift/100800401.gsb?raw=true

Copiar el fichero de malla en user_projections::

  cp 100800401.gsb /var/lib/geoserver_data/user_projections/
  chown tomcat8:tomcat8 100800401.gsb

Forzar que se use también para la proyección Google Earth. Crear un fichero en user_projections llamado epsg_operations.properties, con el siguiente contenido::

  4230,4258=PARAM_MT["NTv2", PARAMETER["Latitude and longitude difference file", "100800401.gsb"]]
  4230,4326=PARAM_MT["NTv2", PARAMETER["Latitude and longitude difference file", "100800401.gsb"]]

Cambiar el owner::

  chown tomcat8:tomcat8 epsg_operations.properties

Reiniciar GeoServer::

  service tomcat8 restart

Comprobar que se utiliza la malla para reproyectar entre "EPSG:4230" y "EPSG:4258", y entre "EPSG:4230" y "EPSG:4326".

Esto se puede comprobar en la web de GeoServer, bajo "Demos" => Reprojection Console.


Añadir soporte para formatos ECW y SID
--------------------------------------

1. Instalar la extensión "GDAL" correspondiente a la versión de GeoServer: http://sourceforge.net/projects/geoserver/files/GeoServer%20Extensions/

::

	cd /var/lib/tomcat8/webapps/geoserver/WEB-INF/lib/
	wget http://sourceforge.net/projects/geoserver/files/GeoServer%20Extensions/2.9.0/geoserver-2.9.0-gdal-plugin.zip
	unzip geoserver-2.9.0-gdal-plugin.zip
	rm *.txt *.TXT *.zip
	chown tomcat8:tomcat8 *.jar

2. Instalar las definiciones CRS (gdal_data)::

	cd /var/lib/geoserver_data
	mkdir gdal
	cd gdal
	wget http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.8/gdal/gdal-data.zip
	unzip gdal-data.zip


3. Instalar las librerías nativas de GDAL::

	mkdir lib
	cd lib
	wget http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.8/gdal/linux/gdal192-Ubuntu12-gcc4.6.3-x86_64.tar.gz
	tar -xvf gdal192-Ubuntu12-gcc4.6.3-x86_64.tar.gz

4. Añadir variables de entorno, a /etc/default/tomcat8::

	export GDAL_DATA=$GEOSERVER_DATA_DIR/gdal/gdal-data
	export LD_LIBRARY_PATH=$GEOSERVER_DATA_DIR/gdal/lib

5. Cambiar permisos y reiniciar tomcat::

	chown -R tomcat8:tomcat8 /var/lib/geoserver_data/
	service tomcat8 restart

Se listarán los nuevos formatos al crear un almacén de datos raster.

.. warning::
   Utilizar ECW en un servidor sin comprar una licencia a ERDAS es ilegal.

   Para usar el formato ECW en un servidor de mapas, es necesario leer y aceptar esto: http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.7/native/gdal/linux/ECWEULA.txt


Extensiones Oficiales
---------------------

CSS. Simbolizar más fácil que con SLD::

	https://sourceforge.net/projects/geoserver/files/GeoServer/2.9.0/extensions/geoserver-2.9.0-css-plugin.zip

Importer. Crear capas de un conjunto de tablas PostGIS o de ficheros ráster sin tener que ir una a una::

	https://sourceforge.net/projects/geoserver/files/GeoServer/2.9.0/extensions/geoserver-2.9.0-importer-plugin.zip

Control Flow. Evita sobresaturar el servidor::

	https://sourceforge.net/projects/geoserver/files/GeoServer/2.9.0/extensions/geoserver-2.9.0-control-flow-plugin.zip

	http://docs.geoserver.org/latest/en/user/extensions/controlflow/index.html

LibJPEG Turbo. Acelera salida en JPEG::

	http://sourceforge.net/projects/libjpeg-turbo/files/1.3.0/libjpeg-turbo-official_1.3.0_amd64.deb

	dpkg -i libjpeg-turbo-official_1.3.0_amd64.deb

	Añadir /opt/libjpeg-turbo/lib64 a LD_LIBRARY_PATH en /etc/default/tomcat8.

	https://sourceforge.net/projects/geoserver/files/GeoServer/2.9.0/extensions/geoserver-2.9.0-libjpeg-turbo-plugin.zip

Printing (a partir de GS 2.6.0; si se instala una versión anterior, ver siguiente párrafo "Extensiones community")::

	wget https://sourceforge.net/projects/geoserver/files/GeoServer/2.9.0/extensions/geoserver-2.9.0-printing-plugin.zip
	
	unzip en WEB-INF/lib y cambiar permisos
	
Para que se pueda imprimir en diferentes formatos (gif, png, tiff) después de instalar la extensión printing hay que añadir la librería fontbox::
	
	sudo wget https://archive.apache.org/dist/pdfbox/1.6.0/fontbox-1.6.0.jar
	sudo chown tomcat8. fontbox-1.6.0.jar
	sudo service tomcat8 restart
	

Extensiones "community"
-----------------------

Cómo compilarlas
................

No están mantenidas oficialmente, y no forman parte del "build" oficial. Hay que compilarlos desde las fuentes::

	git clone git@github.com:geoserver/geoserver.git
	cd geoserver
	# git tag -l
	git checkout -b tags/2.9.0
	cd src/community
	mvn clean install -PcommunityRelease,proxy -DskipTests
	mvn assembly:single
	# Proxy jar generated in: proxy/target/gs-proxy-2.9.0.jar
	# Printing extension generated in: target/release/geoserver-2.9.0-printing-plugin.zip


Cómo configurarlas
..................

Ejemplo de configuración para la extensión de printing (copiar en /var/lib/geoserver_data/printing/):

https://dl.dropboxusercontent.com/u/2368219/geoserver/config.yaml



Esquemas de teselado
--------------------

Aumentar resolución para EPSG:4326
...................................

Si se quiere mayor resolución en los KML superoverlays autogenerados por el servicio GWC, hay que sobreescribir la definición del gridset "EPSG:4326" editando directamente el fichero en disco. En este caso, añadiremos los niveles 23, 24 y 25, que aumentan la resolución máxima en un orden de magnitud. Localizar el fichero $GEOWEBCACHE_CACHE_DIR/geowebcache.xml, y añadir el siguiente gridset::

	<gridSet>
      <name>EPSG:4326</name>
      <description>A default WGS84 tile matrix set where the first zoom level covers the world with two tiles on the horizonal axis and one tile over the vertical axis and each subsequent zoom level is calculated by half the resolution of its previous one.</description>
      <srs>
        <number>4326</number>
      </srs>
      <extent>
        <coords>
          <double>-180.0</double>
          <double>-90.0</double>
          <double>180.0</double>
          <double>90.0</double>
        </coords>
      </extent>
      <alignTopLeft>false</alignTopLeft>
      <resolutions>
        <double>0.703125</double>
        <double>0.3515625</double>
        <double>0.17578125</double>
        <double>0.087890625</double>
        <double>0.0439453125</double>
        <double>0.02197265625</double>
        <double>0.010986328125</double>
        <double>0.0054931640625</double>
        <double>0.00274658203125</double>
        <double>0.001373291015625</double>
        <double>6.866455078125E-4</double>
        <double>3.433227539062E-4</double>
        <double>1.716613769531E-4</double>
        <double>8.58306884766E-5</double>
        <double>4.29153442383E-5</double>
        <double>2.14576721191E-5</double>
        <double>1.07288360596E-5</double>
        <double>5.3644180298E-6</double>
        <double>2.6822090149E-6</double>
        <double>1.3411045074E-6</double>
        <double>6.705522537E-7</double>
        <double>3.352761269E-7</double>
        <double>1.676380634E-7</double>
        <double>8.38190317E-8</double>
        <double>4.19095159E-8</double>
      </resolutions>
      <metersPerUnit>111319.49079327358</metersPerUnit>
      <pixelSize>2.8E-4</pixelSize>
      <scaleNames>
        <string>EPSG:4326:0</string>
        <string>EPSG:4326:1</string>
        <string>EPSG:4326:2</string>
        <string>EPSG:4326:3</string>
        <string>EPSG:4326:4</string>
        <string>EPSG:4326:5</string>
        <string>EPSG:4326:6</string>
        <string>EPSG:4326:7</string>
        <string>EPSG:4326:8</string>
        <string>EPSG:4326:9</string>
        <string>EPSG:4326:10</string>
        <string>EPSG:4326:11</string>
        <string>EPSG:4326:12</string>
        <string>EPSG:4326:13</string>
        <string>EPSG:4326:14</string>
        <string>EPSG:4326:15</string>
        <string>EPSG:4326:16</string>
        <string>EPSG:4326:17</string>
        <string>EPSG:4326:18</string>
        <string>EPSG:4326:19</string>
        <string>EPSG:4326:20</string>
        <string>EPSG:4326:21</string>
        <string>EPSG:4326:22</string>
        <string>EPSG:4326:23</string>
        <string>EPSG:4326:24</string>
      </scaleNames>
      <tileHeight>256</tileHeight>
      <tileWidth>256</tileWidth>
      <yCoordinateFirst>false</yCoordinateFirst>
    </gridSet>


Teselado del ICC
................

La Tile Caché del ICC sigue un esquema de teselado particular, distinto al utilizado habitualmente por la mayoría de aplicaciones de web mapping. Por tanto, debe definirse en GeoServer este esquema particular de teselado:

* Sistema de coordenadas: EPSG:23031
* Límites:

   * Min X:  258000
   * Min Y: 4485000
   * Máx X:  536000
   * Máx Y: 4752000

* Ancho y alto tesela: 256 x 256 px.


.. image:: img/icc_gridset.png
   :width: 70%
   :align: center


Matriz de teselas, defiida a partir de resolución en m/px:

===== ================ ======================
Nivel Tamaño del píxel Nombre
===== ================ ======================
0     1100             Catalunya en 1 tile
1     550              Catalunya en 2x2 tiles
2     275              Catalunya en 4x4 tiles
3     100              Escala 1:1 000 000
4     50               Escala 1:500 000
5     25               Escala 1:250 000
6     10               Escala 1:100 000
7     5                Escala 1:50 000
8     2                Escala 1:20 000
9     1                Escala 1:10 000
10    0.5              Escala 1:5 0000
11    0.25             Escala 1:2 500
12    0.1              Escala 1:1 000
===== ================ ======================


Ajustes GeoServer en producción
-------------------------------

Nota de autoría de este apartado (copiado del proyecto GeoTalleres):

.. note::

	================  ===================================================
	Fecha              Autores
	================  ===================================================             
	6 Feb 2014          * Víctor González (victor.gonzalez@geomati.co) 
	                    * Fernando González (fernando.gonzalez@fao.org)
	================  ===================================================	

	©2014 FAO Forestry 

	Excepto donde quede reflejado de otra manera, la presente documentación se halla bajo licencia : Creative Commons (Creative Commons - Attribution - Share Alike: http://creativecommons.org/licenses/by-sa/3.0/deed.es)

Existen varias optimizaciones a tener en cuenta para poner GeoServer en producción. Aquí tendremos en cuenta únicamente la limitación del servicio WMS y la configuración del nivel de *logging*. Para una optimización más completa se puede consultar `este whitepaper <http://boundlessgeo.com/whitepaper/geoserver-production-2/#limit>`_ (en inglés). En la presente documentación asumimos que GeoServer se está ejecutando sobre el contenedor Tomcat, por lo que también veremos cómo limitar el número máximo de conexiones simultáneas en Tomcat.


Nivel de *logging*
..................

Para realizar las optimizaciones, primero tenemos que abrir interfaz web de administración y acceder a la configuración global de GeoServer:

.. image:: img/gs_global.png
    :align: center

Una vez allí, únicamente hay que cambiar el *Perfil de registro* a *PRODUCTION_LOGGING* y pulsar *Enviar* al final de la página:

.. image:: img/gs_logging.png
    :align: center

También es posible cambiar la *Ubicación del registro* desde aquí, aunque se recomienda mantener la ubicación por defecto.


Limitación del servicio WMS
...........................

En cuanto al servicio WMS, vamos a limitar las peticiones recibidas en dos niveles. Por un lado limitaremos el tiempo y la memoria necesarios para procesar una petición de la llamada GetMap, y por otro lado el número de peticiones simultáneas que acepta el dicho servicio.


**Tiempo y memoria**


Para limitar el tiempo y la memoria requeridos por una única petición WMS en GeoServer, deberemos acceder a *WMS* en la interfaz web:

.. image:: img/gs_wms.png
    :align: center

Una vez aquí, buscaremos el apartado *Límites de consumo de recursos*, donde podremos modificar tanto la memoria como el tiempo máximos de renderizado:

.. image:: img/gs_wms_render_limits.png
    :align: center


**Número de llamadas concurrentes**


Por otro lado, es interesante limitar el número de peticiones simultáneas que ha de manejar GeoServer. El número recomendado de peticiones simultáneas para GeoServer es 20. 

La manera más sencilla de conseguir esto es limitar el número de peticiones en Tomcat.

Para limitar el número de peticiones simultáneas en Tomcat hay que modificar el fichero *$TOMCAT/conf/server.xml*. Aquí buscaremos el conector con el puerto 8080 y añadiremos el parámetro *maxThreads* para determinar el número máximo de peticiones::

    <Server port="8005" shutdown="SHUTDOWN">
      ...
      <Connector port="8080" protocol="HTTP/1.1"
        ConnectionTimeout="20000" redirectPort="8443"
        maxThreads="20" minSpareThreads="20" />
      ...
    </Server>

En el caso de que se esté utilizando Tomcat dentro del servidor Apache y se esté utilizando el conector AJP, el parámetro *maxThreads* se deberá añadir en el conector adecuado::

    <Server port="8005" shutdown="SHUTDOWN">
      ...
      <Connector port="8009" protocol="AJP/1.3"
        connectionTimeout="60000" redirectPort="8443"
        maxThreads="20" minSpareThreads="20" />
      ...
    </Server>

.. note::
	En caso de no saber si se está utilizando el conector AJP, se recomienda establecer los límites igualmente.

.. warning::
	Es **MUY** importante especificar el valor de *connectionTimeout*, ya que para el conector AJP por defecto es infinito, lo cual puede resultar en un bloqueo del servidor si se reciben demasiadas peticiones simultáneamente.

Además, también es posible controlar el número de peticiones simultáneas desde GeoServer. Para ello hay que utilizar el módulo **control-flow**, que no se encuentra instalado por defecto en GeoServer. 

Para instalarlo primero hay que descargarlo de la web de GeoServer, en la sección de descargas tras seleccionar la versión de GeoServer en el apartado *Extensiones*. El fichero comprimido que se descarga contiene otro fichero llamado *control-flow-<version>.jar* que hay que copiar en *$TOMCAT/webapps/geoserver/WEB-INF/lib*. 

Una vez instalado el módulo, para configurarlo hay que crear un fichero de configuración en *$TOMCAT/webapps/geoserver/data* con el nombre *controlflow.properties*. En dicho fichero escribiremos el siguiente contenido para limitar el número de peticiones simultáneas de imágenes para el servicio WMS::

	ows.wms.getmap=16

El número de peticiones que asignamos al servicio WMS depende del uso que se vaya a hacer de nuestro servidor. La configuración anterior de Tomcat únicamente admite 20 peticiones simultáneas en total. En el caso de que usemos el servidor principalmente para WMS podemos, como en el ejemplo, dedicar 16 al servicio WMS y dejar 4 peticiones simultáneas para cualquier otro servicio o petición a GeoServer.

En la `documentación oficial de GeoServer <http://docs.geoserver.org/stable/en/user/extensions/controlflow/index.html>`_ (en inglés) se puede encontrar mayor detalle sobre la configuración del módulo *control-flow*.
