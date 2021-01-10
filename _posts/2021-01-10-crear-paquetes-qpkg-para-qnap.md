---
title: Crear paquetes qpkg para QNAP
date: 2021-01-10 17:00:00 +02:00
tags: [qnap]
description: Cómo crear un paquete de instalación para QNAP
---

Hace tiempo que tengo un QNAP TS-251 como copia de seguridad para las fotos y algun pequeño proyecto que otro.

Quería instalarle el agente de infrastructura de New Relic pero al parecer no hay un paquete oficial, ya que QNAP tiene su propia distro con su propio sistema de instalación.

Así que hoy hablaremos un poco del proceso para crear un paquete instalable en QNAP.

## Pre-requisitos

Antes de empezar necesitarás un QNAP con posibilidad de SSH al mismo, ya que no es posible crear un qpkg sin uno :(

A dicho QNAP deberás instalarle [QDK](https://www.qnap.com/event/dev/en/p_qdk.php) que entre otras cosas trae el comando que necesitamos, `qbuild`.

Una vez instalado en la sesión SSH deberías poder ejecutar `qbuild -h`.

En caso de que, como a mi, no te funcione yo he tenido que hacer el alias a mano `ln -s /share/CACHEDEV1_DATA/.qpkg/QDK/bin/qbuild /usr/bin/qbuild`.

## Creando la estructura

Yo he optado por trabajar bajo la carpeta `Public` principalmente porque la tengo montada en mi PC y el flujo de trabajo era algo mas sencillo.
`mkdir /share/CACHEDEV1_DATA/Public/Packages && cd /share/CACHEDEV1_DATA/Public/Packages`

Para crear el esqueleto del proyecto usamos `qbuild --create-env <Nombre de tu proyecto> && cd <Nombre de tu proyecto>`.

Una vez acabado veremos muchas carpetas y ficheros, pero solo algunas de ellas nos interesan por el momento.

En `build` es donde encontraremos el `.qpkg` final una vez creado.

`qpck.cfg` es el fichero que usaremos para configurar nuestro paquete. Y `package_routines` son hooks que puedes utilizar durante la instalación.

Luego tenemos `icons` y `config`. La primera es para guardar los iconos que se mostrarán al instalar la aplicación y la segunda no tengo ni idea :D

Por último tenemos una carpeta `shared` y varias carpetas por arquitecutra (ejemplo `x86_64`) por defectgo pondremos todo en shared. Pero si hay algun fichero que solo sea para una arquitectura concreta debemos ponerlo en dicha carpeta.

## Configuración del paquete

Como siempre la documentación deja bastante que desear así que dejaré aquí los parámetros que entiendo hasta el momento.

### qpkg.cfg

Este es el fichero principal de configuración para el paquete.

`QPKG_NAME` y `QPKG_DISPLAY_NAME`. El primero es el nombre del paquete y el segundo es un nombre "bonito" por si no quieres usar el del paquete.
Por ejemplo: `QPKG_NAME="NR-Infra-Anget`, `QPKG_DISPLAY_NAME="New Relic Infrastructure Agent"`

`QPKG_VER`, `QPKG_AUTHOR`, `QPKG_LICENSE`, `QPKG_SUMMARY` se explican por sí mismas. `QPKG_SUMMARY` no lo he visto aplicado en ningún sitio en la UI..

`QPKG_SERVICE_PROGRAM` el fichero encargado de encender o apagar el servicio, lo veremos mas adelante.
`QPKG_DISABLE_APPCENTER_UI_SERVICE` si no quieres que se pueda parar el servicio desde la UI ponlo a 1.

`QPKG_REQUIRE` y `QPKG_CONFLICT` que paquetes hacen falta para instalar este. Y que paquetes no pueden estar instalados respectivamente.

`QPKG_SERVICE_PIDFILE` el fichero que indicará si el servicio ya está funcionando. No tengo ni idea para que sirve, porque no he visto diferencia si pones una ruta erronea.

`QPKG_WEBUI`, `QPKG_WEB_PORT`, `QPKG_WEB_SSL_PORT`, `QPKG_USE_PROXY`, `QPKG_PROXY_PATH` Si quieres proveer una web UI del paquete debes configurarla aquí.
Pero de momento no he conseguido que funcione.

`QPKG_DESKTOP_APP`, `QPKG_DESKTOP_APP_WIN_WIDTH`, `QPKG_DESKTOP_APP_WIN_HEIGHT`. Si quieres que se abra la web ui dentro de la UI de qnap al hacer clic en el paquete y el tamaño de la ventana.

### package_routines

En este fichero encontramos los hooks durante la instalación y desinstalación del paquete.
Si tenemos que limpiar ficheros cuando se desinstala o descargar algo durante la instalación, aquí es dónde lo haríamos.

Al principio tenía la idea de descargar el agente durante la instalación (para que siempre instalara la última versión), pero la documentación no es clara en como referencia la carpeta donde vivirá el paquete una vez instalado...

## La carpeta shared

En esta carpeta encontramos primero un fichero sh podemos.
Este fichero se encargará de iniciar y parar el proceso del programa.

Para el caso del agente de New Relic hacemos lo siguiente para `start`

```bash
# Si ya existe el PID_FILE significa que la app ya está iniciada
if test -f "$PID_FILE"; then
  echo "Agent already running under pid $PID"
  exit
fi

# Si el fichero de configuración no existe lo creamos
# Esto se podría hacer usando los hooks pero no ha habido forma de entender los paths, así que lo he dejado aquí
if test ! -f "$AGENT_CONF"; then
echo "pid_file: $PID_FILE" > $AGENT_CONF
echo "plugin_dir: $PLUGIN_DIR" >> $AGENT_CONF
echo "agent_dir: $AGENT_DIR" >> $AGENT_CONF
echo "log_file: $LOG_FILE" >> $AGENT_CONF
echo "license_key: $LICENSE_KEY" >> $AGENT_CONF
fi

touch $LOG_FILE
chmod g+rw $AGENT_CONF

# Iniciamos el agente
$QPKG_ROOT/usr/bin/newrelic-infra -config $AGENT_CONF > /dev/null 2>&1 &
```

Y en el `stop` lo siguiente:

```bash
# Si existe el PID_FILE significa que la app está funcionando.
if test -f "$PID_FILE"; then
echo "Sending KILL signal to the agent."

# Cojemos el PID del fichero y le hacemos un kill
kill `cat $PID_FILE`

# Borramos el fichero ya que la hemos parado
rm $PID_FILE
fi
```

Si necesitamos que el fichero sea diferente por distribución lo podemos poner en las diferentes carpetas que se han creado. Pero tenemos que dar 1 para cada una de ellas.

Por último crearemos las carpetas y ficheros que queremos copiar en el sistema.
En el ejemplo de New Relic tenemos `/usr/bin/newrelic-infra` que es el agente en sí. `/var/{db, log, run}` carpetas donde el agente guardará algunos ficheros y `/etc/newrelic-infra/integrations.d` que es la carpeta donde añadir las integraciones para el agente (postgres, redis etc..)

Hay que tener en cuenta que estas carpetas no se copiarán en la raíz del sistema sino que estarán dentro de la carpeta del qpkg.

Por ejemplo, el ficherp `/usr/bin/newrelic-infra` estará en `/share/CACHEDEV1_DATA/.qpkg/NR-Infra-Agent/usr/bin/newrelic-infra`.

## Construir el paquete

Y eso es todo. Una vez hemos acabado de configurar nuestro paquete ejecutamos `qbuild` en la raíz y encontraremos el fichero `.qpkg` en la carpeta `build/`.
Si vamos a la UI de QNAP podemos instalar dicho qpkg manualmente.

## Conclusión

Hemos visto que crear un paquete para QNAP no es especialmente complicado. Hay más trabajo entendiendo como funciona la aplicación que queremos empaquetar y que recursos y carpetas necesita.

El proceso de desarrollo es simplemente hacer el build, instalar, comprobar que todo esté bien, hacer cambios, desinstalar, build, instalar, etc..

Es algo pesado y lleva su tiempo, es una de las razones por las cuales no me he puesto a investigar porque no van los iconos y como hacer funcionar la UI.