# Docker

Docker es un sistema de ejecución contenida de procesos que permite ejecutar "máquinas virtuales" ligeras con un impacto mínimo en el rendimiento.

**Aviso**: La implementación de Docker en Windows está aún bastante incompleta y se recomienda ejecutarlo sobre entornos Linux.

## Conceptos básicos

- **Imágenes**: Las imágenes representan un estado concreto del sistema de archivos de una "máquina virtual". Cuando se inician, será el punto de partida.
- **Contenedores**: Son las "máquinas virtuales" en ejecución (a partir de ahora nos referiremos a ellas usando este término).

## Instalación

La mayoría de sistemas traerán una versión fácilmente instalable del servidor Docker.

```bash
# Ubuntu
sudo apt install docker.io
```

Podemos comprobar que el servidor Docker está corriendo ejecutando (en sistemas con `systemd`)

```bash
systemctl status docker.service
```

## Sintaxis general

La forma de ejecutar comandos contra el servidor tiene la forma `docker [opciones] [{elemento}] {comando}` (algunos comandos no se aplican sobre ningún elemento). Ésto ejecutará el comando `{comando}` sobre el elemento `{elemento}`. Por ejemplo, si quisiéramos listar las imágenes disponibles en la máquina:

```bash
sudo docker image ls
```

## Descarga de imágenes

Por defecto no habrá ninguna imagen instalada. Pueden obtenerse de un repositorio existente [Docker hub](https://hub.docker.com/) usando `docker pull`. Por ejemplo, para descargar una imagen con la versión más reciente de Ubuntu:

```bash
# Si se omite el tag latest, se usará ese por defecto. Se pueden especificar imágenes diferentes, como ubuntu:18.04
sudo docker pull ubuntu:latest
```

## Lanzar un contenedor

Para levantar un nuevo contenedor a partir de esta imagen de Ubuntu, usaremos el comando `docker container run` (se puede omitir `container`):

```bash
sudo docker run ubuntu uname -a
```

Si, por ejemplo, quisiéramos levantar ese contenedor con un proceso `bash`, tendríamos que indicar además que queremos que sea interactivoy con un terminal, usaremos las opciones `-i` y `-t`. Si además queremos que al terminar el proceso raíz, se borre el contenedor automáticamente, usaremos la opción `--rm`. Y para usar un nombre conocido y no uno autogenerado por Docker, le especificaremos uno con `--name`:

```bash
sudo docker run -it --rm --name ubuntu_bash ubuntu /bin/bash

# Desde el contenedor recien abierto:
ps auxf
# Se verá el proceso bash con PID 1, y el proceso ps lanzado desde el terminal como hijo de éste.
```

El proceso con PID 1 siempre será el _entrypoint_ del contenedor. Cuando este proceso termine, se cerrará el contenedor. Si el contenedor se detiene desde el servidor (`docker stop`), se enviará la señal de cierre contra este proceso.

El código de retorno de `docker run` será el devuelto por este proceso al cerrarse. Ésto es particularmente útil para ciertas tareas como la ejecución automatizada de pruebas en un entorno controlado (por ejemplo en _pipelines_ de integración contínua).

Los procesos del contenedor son realmente procesos del anfitrión pero "enjaulados" en un entorno controlado (el contenedor). Si abrimos un nuevo terminal en el anfitrión antes de cerrar el contenedor anterior, y examinamos los procesos, veremos que aparece nuestro `bash` del contenedor listado como un proceso más, hijo de un proceso `containerd-shim`, y éste hijo de `containerd`. Los procesos de los contenedores están, por tanto, utilizando el kernel de la máquina anfitriona.

## Ejecutar un comando sobre un contenedor activo

Si queremos ejecutar un comando sobre un contenedor en ejecución podemos utilizar `docker exec`. Es muy útil por ejemplo cuando el contenedor va a ejecutar un servicio permanentemente y queremos acceder a un terminal en su interior. Por ejemplo, si queremos ejecutar un servidor web Apache con la aplicación del directorio actual, y después abrir un terminal dentro del contenedor:

```bash
sudo docker run -dit --rm --name apache -p 8080:80 -v "$PWD":/usr/local/apache2/htdocs/ httpd:2.4
sudo docker exec -it apache /bin/bash
```

## Detener y borrar contenedores activos

Para detener un contenedor en ejecución se usa `docker stop nombre_del_contenedor`:

```bash
sudo docker stop apache
```

Esto lo detendrá pero no lo borrará si éste no se ha lanzado con el parámetro `--rm`. Para borrarlo de la lista de contenedores se usará de una forma similar `docker rm nombre_del_contenedor`:

```bash
sudo docker rm apache
```

## Obtener información de imágenes y contenedores

Para obtener información de imágenes instaladas se usará el comando `docker image inspect tag_de_imagen`. Por ejemplo:

```bash
sudo docker image inspect ubuntu:latest
```

Para obtener la información de un contenedor, la sintaxis es similar: `docker [container] inspect nombre_del_contenedor`:

```bash
sudo docker inspect ubuntu_bash
```

## Definir variables de entorno en el contenedor

Una forma de parametrizar la ejecución de una imagen concreta al levantar un contenedor a partir de ella es definir variables de entorno.

Si por ejemplo, ejecutamos

```bash
sudo docker run --rm --name ubuntu_test ubuntu /usr/bin/env
```

podremos ver las variables de entorno definidas por defecto al lanzar un contenedor de la imagen ubuntu.

Para definir nuevas variables de entorno, usaríamos el parámetro `-e 'Variable=Valor'` tantas veces como sea necesario. Por ejemplo:

```bash
sudo docker run --rm -e 'TEXT1=Hola Docker!' -e 'TEXT2=Lorem ipsum' --name ubuntu_test ubuntu /usr/bin/env
```

## Volúmenes

Cuando queremos pasar datos persistentes entre el anfitrión y el contenedor, lo haremos normalmente usando volúmenes. Como cuando se montan volúmenes en un entorno Linux, aquí mapeamos una ruta en el anfitrión con una ruta en el contenedor.

Para el ejemplo, crearemos la siguiente estructura de ficheros:

```bash
mkdir -p /tmp/{,read_only_}data_dir
touch /tmp/{,read_only_}data_dir/file
ls /tmp/*data_dir

# /tmp/data_dir:
# file

# /tmp/read_only_data_dir:
# file
```

Vamos a abrir un terminal en un contenedor de ubuntu, mapeando esos dos directorios a unos homólogos, con el sufijo `_docker` en el contenedor. Para ello definimos los volúmenes con el parámetro `-v ruta_anfitrion:ruta_contenedor`:

```bash
sudo docker run -it --rm -v /tmp/data_dir:/tmp/data_dir_docker -v /tmp/read_only_data_dir:/tmp/read_only_data_dir_docker:ro --name ubuntu_bash ubuntu /bin/bash
```

Atención, porque en el caso de `/tmp/read_only_data_dir_docker` hemos añadido un tercer valor `:ro`. Se pueden añadir _flags_ como se hace en la tabla de montajes de volúmenes `fstab`, en este caso indicando que queremos que monte el volumen como de solo lectura.

Podemos comprobar su funcionamiento:

```bash
# Dentro del contenedor
echo DATA1 > /tmp/data_dir_docker/file # OK
echo DATA2 > /tmp/read_only_data_dir_docker/file # Error

# Fuera del contenedor, en el anfitrión
cat /tmp/data_dir/file # DATA1
cat /tmp/read_only_data_dir/file # <vacío>
```

**TO DO**:

- Puertos
- Imágenes personalizadas
- docker-compose
