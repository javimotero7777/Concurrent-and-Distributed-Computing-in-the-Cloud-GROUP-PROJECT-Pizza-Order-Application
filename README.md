# Pizza Order Application — Despliegue con Docker y CI/CD

Práctica 4 de Administración de Servidores. Modalidad A (proyecto propio): partimos de la
Pizza Order Application, una API REST que desarrollamos en otra asignatura, y la hemos
containerizado, orquestado con Docker Compose y desplegado en una VM de Azure con un pipeline
CI/CD en GitHub Actions.

## Arquitectura

Tres contenedores en la misma VM, conectados por redes Docker. Solo nginx publica un puerto.

```
Internet --(SSH :50266)--> [ VM Azure ]
                             nginx :80  -->  app :8080  -->  db :3306
                             (proxy)         (Express)       (MariaDB)
```

| Servicio | Imagen              | Puerto | Función                                            |
|----------|--------------------|--------|----------------------------------------------------|
| `proxy`  | `nginx:1.27-alpine`| 80     | Reverse proxy a la app. Único servicio publicado.  |
| `app`    | imagen propia (GHCR)| 8080  | API Express. Sin puertos publicados al host.       |
| `db`     | `mariadb:11`       | 3306   | Datos persistidos en el volumen `db_data`.         |

Dos redes: `backend` (privada, conecta app y db) y `frontend` (la que nginx publica). Así la
base de datos no queda expuesta fuera de la VM.

## Arrancar en local

Requisitos: Docker Desktop o Docker Engine + Compose v2.

```bash
cp .env.example .env        # rellenar las contraseñas
docker compose up --build   # construye y arranca los 3 servicios
```

Probar:

```bash
curl http://localhost/healthCheck             # devuelve OK
curl -X POST http://localhost/api/people/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"a@a.com","password":"password123","name":"A","lastname":"B","role":"manager"}'
```

Parar:

```bash
docker compose down       # mantiene los datos
docker compose down -v    # borra también el volumen de la BD
```

## Pipeline CI/CD

Definido en `.github/workflows/deploy.yml`. Se ejecuta en cada `push` a `main`. Tres jobs en
este orden (los tests van primero a propósito: si el código no compila, no perdemos tiempo
construyendo y subiendo la imagen):

1. **test** — `npm ci`, `node --check` sobre cada `.js` y `npm test` si existe.
2. **build-and-push** — construye la imagen multi-stage y la publica en GHCR con los tags
   `latest` y `sha-<commit>`.
3. **deploy** — entra por SSH a la VM, copia el `docker-compose.yml` + config de nginx,
   hace `docker compose pull` + `up -d` y comprueba con `curl /healthCheck`.

### Secrets configurados en el repo

| Secret             | Valor en nuestro caso                              |
|--------------------|----------------------------------------------------|
| `VPS_HOST`         | hostname de la VM (`ml-lab-….cloudapp.azure.com`)  |
| `VPS_USER`         | `usj`                                              |
| `VPS_PORT`         | `50266` (la VM no usa el 22 estándar)              |
| `VPS_SSH_KEY`      | clave privada Ed25519 generada para el CI          |
| `GHCR_PAT`         | token con permiso `read:packages`                  |
| `JWT_KEY`          | clave de firma de los JWT                          |
| `DB_PASSWORD`      | password del usuario `admin` de MariaDB            |
| `DB_ROOT_PASSWORD` | password de root de MariaDB                        |

`GITHUB_TOKEN` no hace falta crearlo, lo inyecta GitHub automáticamente.

## Preparar la VM (lo que hicimos una vez)

La VM es de Azure Lab Services (Debian 11). Pasos que seguimos:

```bash
# Instalar Docker + Compose
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # para usar docker sin sudo (requiere reabrir sesión)

# Generar una clave SSH dedicada para el CI (esto en el portátil)
ssh-keygen -t ed25519 -f ~/.ssh/azure_pizza_ci -N ""

# Copiar la pública a la VM
ssh-copy-id -i ~/.ssh/azure_pizza_ci.pub -p 50266 usj@<VPS_HOST>
```

La clave privada `~/.ssh/azure_pizza_ci` es la que va en el secret `VPS_SSH_KEY`.

## Acceder al servicio desplegado

El puerto 80 de la VM no está abierto a internet (Azure Lab Services solo expone el puerto SSH
que mapea el profesor). Para ver la app desde el navegador usamos un túnel SSH:

```bash
ssh -L 8080:localhost:80 -i ~/.ssh/azure_pizza_ci -p 50266 usj@<VPS_HOST>
```

Con el túnel abierto, `http://localhost:8080` apunta al puerto 80 dentro de la VM. Esto es
también buena práctica de seguridad (la API no queda expuesta a internet, solo accesible por
SSH). El despliegue automático funciona igual porque GitHub Actions entra por el puerto SSH.

## Buenas prácticas aplicadas

- Multi-stage build: la imagen final no lleva `npm` ni devDependencies (~150 MB en vez de ~400).
- Usuario non-root: el contenedor corre como el usuario `node` (UID 1000).
- `tini` como PID 1 para que `docker stop` haga un apagado limpio.
- Healthcheck contra `/healthCheck`; Docker reinicia el contenedor si falla.
- Red interna privada: la BD no se publica al host.
- Secrets fuera del repo: el `.env` está en `.gitignore` y se genera en la VM desde los secrets.

## Problemas que encontramos

- **`.env` con credenciales en el repo original.** Lo sacamos del control de versiones con
  `git rm --cached .env`, lo metimos en `.gitignore` y movimos las credenciales a los secrets.
- **`DB_HOST=0.0.0.0` en el código.** No funciona dentro de Compose; lo cambiamos a `db`, que
  es el nombre del servicio y se resuelve por la red interna de Docker.
- **El `test` por defecto de `npm init` rompía el CI.** El `package.json` traía
  `"test": "echo ... && exit 1"`, que siempre falla. Lo quitamos.
- **Puerto 80 cerrado en la VM.** Es una limitación del laboratorio; lo resolvimos con el túnel
  SSH descrito arriba.
- **El healthcheck del deploy fallaba al arrancar.** MariaDB tarda unos segundos en estar lista,
  así que el smoke test del pipeline reintenta varias veces antes de dar el deploy por fallido.

## Endpoints

| Recurso   | Rutas principales                          |
|-----------|--------------------------------------------|
| Health    | `GET /healthCheck`                         |
| Usuarios  | `POST /api/people/signup`, `/login`        |
| Pizzerías | `/api/pizzaPlaces`                         |
| Pizzas    | `/api/pizzas`                              |
| Cocineros | `/api/cooks`                               |
| Pedidos   | `/api/orders`                              |

La colección de Postman (`ConcurrencyFinalProject Collection.postman_collection.json`) tiene
todos los endpoints con ejemplos.

<!-- prueba de despliegue -->
