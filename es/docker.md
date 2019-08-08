# Docker

Docker es un sistema de ejecución contenida de procesos que permite ejecutar "máquinas virtuales" ligeras con un impacto mínimo en el rendimiento.

**Aviso**: La implementación de Docker en Windows está aún bastante incompleta y se recomienda ejecutarlo sobre entornos Linux.

## Conceptos básicos

- **Imágenes**: Las imágenes representan un estado concreto del sistema de archivos de una "máquina virtual". Cuando se inician, será el punto de partida.
- **Contenedores**: Son las "máquinas virtuales" en ejecución (a partir de ahora nos referiremos a ellas usando este término).

## Instalación y ejecución

La mayoría de sistemas traerán una versión fácilmente instalable del servidor Docker.

```bash
# Ubuntu
sudo apt install docker.io
```

Podemos comprobar que el servidor Docker está corriendo ejecutando (en sistemas con `systemd`)

```bash
systemctl status docker.service
```

La forma de ejecutar comandos contra el servidor tiene la forma `docker [opciones] [{elemento}] {comando}` (algunos comandos no se aplican sobre ningún elemento). Ésto ejecutará el comando `{comando}` sobre el elemento `{elemento}`. Por ejemplo, si quisiéramos listar las imágenes disponibles en la máquina:

```bash
sudo docker image ls
```

Por defecto no habrá ninguna imagen instalada. Pueden obtenerse de un repositorio existente [Docker hub](https://hub.docker.com/) usando `docker pull`. Por ejemplo, para descargar una imagen con la versión más reciente de Ubuntu:

```bash
# Si se omite el tag latest, se usará ese por defecto. Se pueden especificar imágenes diferentes, como ubuntu:18.04
sudo docker pull ubuntu:latest
```

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

**TO DO**:

- Variables de entorno
- Volúmenes
- Imágenes personalizadas
