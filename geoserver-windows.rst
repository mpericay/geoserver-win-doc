============================================
Instalación Geoserver en Windows Server 2008
============================================

Acerca de
=========

Documento publicado bajo licencia Creative Commons reconocimiento compartir-igual (CC-by-sa). Consultar autoría en el histórico de commits. Contribuciones bienvenidas.

Este documento está basado en la instalación de Geoserver sobre Linux en https://github.com/oscarfonts/geoserver-deploy-doc/blob/master/geoserver-deploy.rst

Configuración inicial de la máquina
-----------------------------------
Instalar Oracle Java SE Runtime Envorinment 7 (Windows x64)

http://www.oracle.com/technetwork/java/javase/downloads/java-se-jre-7-download-432155.html

Instalar Apache Tomcat 7.0.57

http://tomcat.apache.org/download-70.cgi

Comprobar instalación y configurar inicio automático

http://localhost:8080

Abrir puerto 8080 en el firewall de Windows.

No hay JAI y JAI-ImageIO para JRE 64-bit (por lo que no se instalarán) en Windows (sí existen en Linux). Alternativamente, si se quisieran servir sobretodo datos raster on-the-fly,  podría instalarse JRE 32-bit con JAI y JAI-ImageIO nativos, pero se cree mejor el rendimiento de la máquina virtual java de 64-bits para la cartografía vectorial. Si el rendimiento al servir rasters no es suficiente, se recomienda hacer tiles con GeoWebCache, y la velocidad será máxima.

PostGIS
=======
Instalar PostgreSQL (9.3.5) y PostGIS (2.1.4):

http://www.enterprisedb.com/products-services-training/pgdownload (Win x86-64)

A través de Stack Builder 3.1.1, instalar:

- PostGIS v. 2.1.3 (64-bit)
- phpPgAdmin v.5.1-1 (requiere Apache/PHP v. 2.4.10-5.5-17.1)
- driver psqlODBC (64-bit) v.09.03.0400-1

Habilitar acceso local. En C:\\Program Files\\PostgreSQL\\9.3\\data\\pg_hba.conf:
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                ident
local   all             all                                     md5
host    all             all             127.0.0.1/32            md5

Y en C:\\Program Files\\PostgreSQL\\9.3\\data\\postgresql.conf, descomentar:::

listen_addresses = 'localhost'

Reiniciar para aplicar cambios

Para acceder a la consola SQL, usar pgAdmin

Crear un nuevo "usuario"::

CREATE USER usuario LOGIN PASSWORD '------' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE;

Crear una nueva BDD "geodatos" cuyo propietario sea "usuario"

Habilitar capacidades "geo" en la base de datos::

CREATE EXTENSION postgis;

Instalar funciones para poder importar datos de versiones PostGIS 1.x ejecutando:

C:\\Program Files\\PostgreSQL\\9.3\\share\\contrib\\postgis-2.1\\legacy.sql

GeoServer
=========

Instalación base
----------------

GeoServer 2.6.1 (o "latest stable"):

http://sourceforge.net/projects/geoserver/files/GeoServer/2.6.1/geoserver-2.6.1-war.zip

Mover y descomprimir archivo geoserver-2.6.1-war.zip a C:\\Program Files\\Apache Software Foundation\\Tomcat 7.0\\webapps\\

Entorno JVM
-----------

Mover el GEOSERVER_DATA_DIR fuera de los binarios:

desde C:\\Program Files\\Apache Software Foundation\\Tomcat 7.0\\webapps\\geoserver a E:\\geoserver_data

Crear directorio E:\\geowebcache_cache

Editar el fichero C:\\Program Files\\Apache Software Foundation\\Tomcat 7.0\\webapps\\geoserver\\WEB-INF\\web.xml y añadir los dirs de geoserver y la caché::

 <context-param>
 <param-name>GEOWEBCACHE_CACHE_DIR</param-name>
 <param-value>E:\\geowebcache_cache</param-value>
 </context-param> 
 
 <context-param>
 <param-name>GEOSERVER_DATA_DIR</param-name>
 <param-value>E:\\geoserver_data</param-value>
 </context-param> 
 
Editar la variable de entorno JAVA_OPTS editando los parámetros de optimización en Configure Tomcat > Java::

"-Djava.awt.headless=true -Xms1560m -Xmx1560m -XX:PermSize=384m -XX:MaxPermSize=512m -XX:+UseParallelOldGC -XX:+UseParallelGC"

Reiniciar tomcat

Comprobación entorno
--------------------

Entrar a:

http://localhost:8080/geoserver/web/

En "server status", combrobar que:

- La JVM es la instalada Oracle Corporation: 1.7.0 (Java HotSpot(TM) 64-Bit Server VM)
- el data directory apunta a E:\\geoserver_data

Seguridad
---------

Seguir las notificaciones de seguridad que aparecen en la página principal de GeoServer:

- Cambiar password de "admin".
- Cambiar el master password.
- Añadir “unrestricted policy jar files” de Java: descargar de http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html y sobreescribir archivos en C:\\Program Files\\Java\\jre7\\lib\\security
 
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

Copiar el fichero de malla en E:\\geoserver_data\\user_projections

Forzar que se use también para la proyección Google Earth. Crear un fichero en user_projections llamado epsg_operations.properties, con el siguiente contenido::

4230,4258=PARAM_MT["NTv2", PARAMETER["Latitude and longitude difference file", "100800401.gsb"]]
4230,4326=PARAM_MT["NTv2", PARAMETER["Latitude and longitude difference file", "100800401.gsb"]]

Reiniciar Tomcat

Comprobar que se utiliza la malla para reproyectar entre "EPSG:4230" y "EPSG:4258", y entre "EPSG:4230" y "EPSG:4326".
Esto se puede comprobar en la web de GeoServer, bajo "Demos" => Reprojection Console.

Añadir soporte para formatos ECW y SID
--------------------------------------

Hay que seguir los pasos definidos en http://docs.geoserver.org/stable/en/user/data/raster/gdal.html

1. Instalar la extensión "GDAL" correspondiente a la versión de GeoServer: http://sourceforge.net/projects/geoserver/files/GeoServer/2.6.1/extensions/geoserver-2.6.1-gdal-plugin.zip

2. Descomprimir y copiar en C:\\Program Files\\Apache Software Foundation\\Tomcat 7.0\\webapps\\geoserver\\WEB-INF\\lib

3. Instalar las definiciones CRS (gdal_data) bajar http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.8/gdal/gdal-data.zip y descomprimir en  E:\\geoserver_data\\gdal

4. Instalar las librerías nativas de GDAL http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.10/ y copiar drivers a C:\\Windows\\System32

5. Añadir variables de entorno: editar el fichero C:\\Program Files\\Apache Software Foundation\\Tomcat 7.0\\webapps\\geoserver\\WEB-INF\\web.xml y añadir::

 <context-param>
 
   <param-name>GDAL_DATA</param-name>
   
   <param-value>$GEOSERVER_DATA_DIR/gdal/gdal-data</param-value>
   
 </context-param> 

6. Bajar los drivers específicos de ECW y MRSID de http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.10/native/gdal/windows/MSVC2010/
y copiarlos a C:\\Windows\\System32\\ 

7. Crear variable de entorno GDAL_DRIVERS_PATH y apuntarla a C:\\Windows\\System32\\gdalplugins

8. Reiniciar tomcat. 

Se puede comprobar que la instalación ha tenido éxito porque se listarán los nuevos formatos al crear un almacén de datos raster:

Hay que advertir que utilizar ECW en un servidor sin comprar una licencia a ERDAS es no es legal sin leer y aceptar esto: http://demo.geo-solutions.it/share/github/imageio-ext/releases/1.1.X/1.1.8/gdal/ECWEULA.txt
