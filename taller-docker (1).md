# Taller Practico de Docker

---

## Parte 1 - Verificar instalacion

Abra una terminal y ejecute:

```
docker --version
docker compose version
```

Que debe ver: la version instalada de Docker. Si aparece un error, Docker no esta instalado correctamente.

---

## Parte 2 - Primer contenedor

Ejecute:

```
docker run hello-world
```

Que sale: un mensaje de texto confirmando que Docker funciona. La imagen se descargo automaticamente desde Docker Hub porque no existia en su maquina.

---

## Parte 3 - Contenedor interactivo

Ejecute:

```
docker run -it ubuntu bash
```

Que sale: su terminal cambia de prompt. Ahora esta dentro de un contenedor Ubuntu.

Dentro del contenedor, ejecute:

```
ls
cat /etc/os-release
exit
```

Que sale: el sistema de archivos del contenedor y la informacion del sistema operativo. Al ejecutar `exit` regresa a su maquina.

---

## Parte 4 - Ver contenedores

Para ver los contenedores que estan corriendo ahora mismo:

```
docker ps
```

Para ver todos los contenedores, incluyendo los detenidos:

```
docker ps -a
```

Que sale: una tabla con ID, imagen usada, estado y nombre de cada contenedor.

---

## Parte 5 - Correr un servidor web

Ejecute:

```
docker run -d --name servidor-web -p 8080:80 nginx
```

Que hace cada parte del comando:

- `-d` corre el contenedor en segundo plano
- `--name servidor-web` le asigna un nombre
- `-p 8080:80` conecta el puerto 8080 de su maquina con el puerto 80 del contenedor
- `nginx` es la imagen que se usa

Abra un navegador y entre a `http://localhost:8080`.

Que sale: la pagina de bienvenida de Nginx.

Verifique que el contenedor esta corriendo:

```
docker ps
```

---

## Parte 6 - Manejo del ciclo de vida

Con el contenedor del ejercicio anterior, ejecute cada comando y observe que pasa:

Detener el contenedor:

```
docker stop servidor-web
```

Verifique con `docker ps` — ya no aparece. Verifique con `docker ps -a` — si aparece, pero detenido.

Volver a iniciarlo:

```
docker start servidor-web
```

Reiniciarlo:

```
docker restart servidor-web
```

Ver los logs:

```
docker logs servidor-web
```

Ver los logs en tiempo real (refresque el navegador mientras lo ejecuta):

```
docker logs -f servidor-web
```

Presione Ctrl+C para salir.

---

## Parte 7 - Entrar a un contenedor que ya esta corriendo

Con el servidor-web corriendo, ejecute:

```
docker exec -it servidor-web bash
```

Que sale: entra al contenedor sin detenerlo.

Dentro del contenedor, cambie la pagina de bienvenida:

```
echo "<h1>Modificado desde dentro del contenedor</h1>" > /usr/share/nginx/html/index.html
exit
```

Refresque `http://localhost:8080` en el navegador.

Que sale: la pagina ahora muestra el texto que usted escribio.

---

## Parte 8 - Multiples contenedores de la misma imagen

Ejecute los tres comandos seguidos:

```
docker run -d --name web-1 -p 8081:80 nginx
docker run -d --name web-2 -p 8082:80 nginx
docker run -d --name web-3 -p 8083:80 nginx
```

Verifique con `docker ps`.

Abra en el navegador:

- `http://localhost:8081`
- `http://localhost:8082`
- `http://localhost:8083`

Que sale: tres servidores identicos corriendo al mismo tiempo, cada uno en su propio puerto, todos creados a partir de la misma imagen.

Modifique solo el primero:

```
docker exec -it web-1 bash
echo "<h1>Soy el servidor 1</h1>" > /usr/share/nginx/html/index.html
exit
```

Refresque los tres puertos en el navegador.

Que sale: solo el 8081 cambio. Los otros dos siguen igual. Cada contenedor es independiente.

Limpie los tres contenedores:

```
docker rm -f web-1 web-2 web-3
```

---

## Parte 9 - Base de datos y perdida de datos

Levante un contenedor de MariaDB:

```
docker run -d \
  --name mi-db \
  -e MYSQL_ROOT_PASSWORD=secret123 \
  -e MYSQL_DATABASE=taller \
  -p 3306:3306 \
  mariadb:11
```

Espere 15 segundos y revise los logs para confirmar que inicio:

```
docker logs mi-db
```

Busque la linea que dice `ready for connections`.

Conectese a la base de datos:

```
docker exec -it mi-db mariadb -u root -psecret123
```

Dentro de MariaDB ejecute:

```sql
USE taller;
CREATE TABLE productos (id INT AUTO_INCREMENT PRIMARY KEY, nombre VARCHAR(100));
INSERT INTO productos (nombre) VALUES ('Laptop'), ('Mouse'), ('Teclado');
SELECT * FROM productos;
EXIT;
```

Que sale: una tabla con tres registros.

Ahora elimine el contenedor:

```
docker rm -f mi-db
```

Vuelva a crearlo con el mismo comando de antes y conektese de nuevo. Ejecute:

```sql
USE taller;
SELECT * FROM productos;
```

Que sale: un error porque la tabla no existe. Los datos se perdieron al eliminar el contenedor.

---

## Parte 10 - Persistencia con volumenes

Levante MariaDB con un volumen:

```
docker run -d \
  --name mi-db \
  -e MYSQL_ROOT_PASSWORD=secret123 \
  -e MYSQL_DATABASE=taller \
  -v datos-mariadb:/var/lib/mysql \
  -p 3306:3306 \
  mariadb:11
```

La diferencia es `-v datos-mariadb:/var/lib/mysql`. Eso conecta una carpeta de su maquina con la carpeta donde MariaDB guarda los datos.

Espere 15 segundos, luego cree los datos:

```
docker exec -it mi-db mariadb -u root -psecret123 taller -e "
  CREATE TABLE productos (id INT AUTO_INCREMENT PRIMARY KEY, nombre VARCHAR(100));
  INSERT INTO productos (nombre) VALUES ('Laptop'), ('Mouse'), ('Teclado');
"
```

Elimine el contenedor:

```
docker rm -f mi-db
```

Vuelva a crearlo con el mismo comando (con el mismo volumen) y consulte los datos:

```
docker exec -it mi-db mariadb -u root -psecret123 taller -e "SELECT * FROM productos;"
```

Que sale: los tres registros siguen ahi. El volumen conservo los datos aunque el contenedor fue eliminado.

Ver los volumenes existentes:

```
docker volume ls
```

---

## Parte 11 - Construir su propia imagen

Cree una carpeta y entre a ella:

```
mkdir mi-imagen
cd mi-imagen
```

Cree el archivo `index.html`:

```
echo "<h1>Imagen construida por mi</h1>" > index.html
```

Cree el archivo `Dockerfile` con el siguiente contenido:

```
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Construya la imagen:

```
docker build -t mi-imagen:v1 .
```

Que sale: Docker ejecuta cada instruccion del Dockerfile y al final crea la imagen.

Verifique que la imagen existe:

```
docker images
```

Corra un contenedor con su imagen:

```
docker run -d --name mi-contenedor -p 9090:80 mi-imagen:v1
```

Abra `http://localhost:9090`.

Que sale: la pagina que usted definio en el `index.html`.

---

## Referencia de comandos

**Imagenes**

| Comando | Para que sirve |
|---|---|
| `docker images` | Listar imagenes descargadas |
| `docker pull nombre` | Descargar una imagen |
| `docker rmi nombre` | Eliminar una imagen |
| `docker build -t nombre .` | Construir imagen desde Dockerfile |

**Contenedores**

| Comando | Para que sirve |
|---|---|
| `docker run -d --name X -p H:C imagen` | Crear y correr contenedor |
| `docker ps` | Ver contenedores activos |
| `docker ps -a` | Ver todos los contenedores |
| `docker stop X` | Detener contenedor |
| `docker start X` | Iniciar contenedor detenido |
| `docker restart X` | Reiniciar contenedor |
| `docker rm X` | Eliminar contenedor detenido |
| `docker rm -f X` | Forzar eliminacion |
| `docker logs X` | Ver logs |
| `docker logs -f X` | Ver logs en tiempo real |
| `docker exec -it X bash` | Entrar al contenedor |

**Volumenes**

| Comando | Para que sirve |
|---|---|
| `docker volume ls` | Listar volumenes |
| `docker volume rm nombre` | Eliminar volumen |

**Limpieza general**

| Comando | Para que sirve |
|---|---|
| `docker system prune` | Eliminar todo lo que no esta en uso |
