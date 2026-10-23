# Laboratorio: Docker Compose con redes aisladas

Infraestructura y Cómputo en la Nube · Universidad Anáhuac Mayab
Alumno: Diego Kamil Olea Ruiz · Profesor: Fabricio A. Suárez Domínguez

Levantar **WordPress** con una base de datos **MariaDB** usando Docker Compose y comprobar que las
redes aíslan a los contenedores, que los datos sobreviven y cómo se depura un error de conexión.
Como reto se agrega **Adminer** para administrar la base desde el navegador.

## Estado

- [x] **Parte 1. Construir el compose**: `compose.yaml` listo (falta levantarlo y capturar).
- [ ] **Parte 2. Aislamiento**: `getent hosts db` desde wordpress y `ping db` desde la red frontend.
- [ ] **Parte 3. Persistencia**: publicar un post → `down` / `up` → `down -v`.
- [ ] **Parte 4. Romper y arreglar**: `WORDPRESS_DB_HOST: localhost` → error en logs → regresar a `db`.
- [ ] **Reto: Adminer**: servicio agregado al compose; falta entrar y ver el post en `wp_posts`.

Las capturas y respuestas de cada parte se agregan en `evidencias/` y en este README conforme avance.

## Estructura

```
lab-compose/
├── compose.yaml     servicios db, wordpress y adminer; redes frontend/backend; volumen db-data
├── .env.example     plantilla de contraseñas (el .env real no se sube)
├── evidencias/      capturas de cada parte
└── README.md
```

La carpeta se llama `lab-compose` a propósito: Compose usa el nombre de la carpeta como prefijo,
así que las redes quedan como `lab-compose_frontend` y `lab-compose_backend`, que son los nombres que
usan los comandos de la práctica.

## Cómo levantarlo

```bash
cp .env.example .env        # y poner contraseñas propias
docker compose up -d
docker compose ps
```

- WordPress: http://localhost:8080
- Adminer: http://localhost:8081 (servidor `db`, usuario `wp`, base `wordpress`)

## Diseño del compose.yaml

### Base de datos (`db`)
- Imagen `mariadb:11` con `MARIADB_DATABASE=wordpress` y `MARIADB_USER=wp`.
- Volumen con nombre `db-data` en `/var/lib/mysql`: los datos viven fuera del contenedor.
- **Solo** en la red `backend` y **sin puertos publicados**: desde fuera del host no se puede llegar a ella.
- `healthcheck` con el script `healthcheck.sh` que trae la imagen oficial, para saber cuándo está lista.

### WordPress (`wordpress`)
- Puerto `8080` del host → `80` del contenedor.
- `WORDPRESS_DB_HOST: db`: dentro de la red, el **nombre del servicio** funciona como dirección (DNS interno de Docker).
- En `frontend` (para que lo vea el navegador) y en `backend` (para hablar con la base).
- `depends_on` con `condition: service_healthy`: no arranca hasta que MariaDB responde.

### Redes
- `frontend`: red bridge normal.
- `backend`: `internal: true`, sin salida a internet. La base no puede comunicarse hacia afuera.

### Reto: Adminer
- Puerto `8081` del host → `8080` del contenedor.
- En `backend` para alcanzar a `db`, y en `frontend` para poder publicar su puerto hacia el navegador
  (una red `internal` no permite publicar puertos).

### Extras
- Las contraseñas están en `.env` y el compose las lee con `${DB_PASSWORD}` y `${DB_ROOT_PASSWORD}`.
- `restart: unless-stopped` en los tres servicios.
