# optimizador-envios — infra

Repositorio de infraestructura del proyecto. Centraliza el `docker-compose.yml` que levanta el stack completo usando las imágenes publicadas en GHCR para `frontend`, `user-service` y `shipment-service`.

---

## Requisitos previos

- Docker Desktop >= 24, o Docker Engine + Compose plugin >= 2.20.
- Acceso de lectura a `ghcr.io/optimizador-de-envios` si los packages son privados.

```bash
echo "$CR_PAT" | docker login ghcr.io -u TU_USUARIO --password-stdin
```

---

## Uso rápido

### 1. Clonar el repo

```bash
git clone https://github.com/Optimizador-de-Envios/infra.git
cd infra
```

### 2. Crear el archivo `.env`

```bash
cp .env.example .env
```

Editá `.env` y completá los valores reales:

| Variable | Descripción |
|---|---|
| `FRONTEND_IMAGE` | Tag completo de la imagen del frontend en GHCR. |
| `USER_SERVICE_IMAGE` | Tag completo de la imagen del servicio de usuarios en GHCR. |
| `SHIPMENT_SERVICE_IMAGE` | Tag completo de la imagen del servicio de envíos en GHCR. |
| `JWT_SECRET` | Secreto compartido para firmar/validar JWT entre `user-service` y `shipment-service`. |
| `JWT_EXPIRATION_SECONDS` | Tiempo de expiración del JWT emitido por `user-service`. |
| `USER_DB_NAME`, `USER_DB_USERNAME`, `USER_DB_PASSWORD` | Base dedicada para `user-service`. |
| `SHIPMENT_DB_NAME`, `SHIPMENT_DB_USERNAME`, `SHIPMENT_DB_PASSWORD` | Base dedicada para `shipment-service`. |
| `JPA_DDL_AUTO`, `JPA_SHOW_SQL` | Configuración JPA usada por los microservicios. |
| `ORS_API_KEY` | API key de [OpenRouteService](https://openrouteservice.org/dev/#/signup). |

### 3. Levantar el stack

```bash
docker compose up -d
```

### 4. Verificar servicios

```bash
docker compose ps
docker compose logs -f
```

### 5. Acceder

| Servicio | URL local | Descripción |
|---|---|---|
| App vía proxy | http://localhost:3000 | Entrada principal. Sirve el frontend y enruta APIs internamente. |
| Shipment-Service | http://localhost:8080 | API directa de envíos para pruebas con curl/Postman. |
| User-Service | http://localhost:8081 | API directa de usuarios para pruebas con curl/Postman. |

---

## Imágenes actuales

```dotenv
FRONTEND_IMAGE=ghcr.io/optimizador-de-envios/frontend:sha-d7763cd
USER_SERVICE_IMAGE=ghcr.io/optimizador-de-envios/user-service:sha-90a4163
SHIPMENT_SERVICE_IMAGE=ghcr.io/optimizador-de-envios/shipment-service:sha-e1ba8e1
```

Cuando CI publique nuevos builds en GHCR, actualizá el tag correspondiente en `.env` y aplicá el cambio:

```bash
docker compose pull
docker compose up -d
```

---

## Topología

El servicio `proxy` (nginx) es el único origen HTTP que debería consumir el navegador:

```text
Navegador -> localhost:3000 -> nginx proxy
                                  |-- /api/users/*     -> http://user-service:8081/api/users/*
                                  |-- /api/v1/pedido*  -> http://shipment-service:8080/api/v1/pedido*
                                  |-- /*               -> http://frontend:80
```

Cada microservicio tiene su propia base de datos PostgreSQL:

```text
user-service     -> postgres-users
shipment-service -> postgres-shipment
```

No hay base compartida ni consultas entre servicios para validar identidad: `shipment-service` valida localmente el JWT emitido por `user-service` usando `JWT_SECRET`.

---

## Requisito del build del frontend

La imagen del frontend debe compilarse con rutas relativas para que el proxy decida el destino en runtime. En GitHub Actions del repo `frontend`, configurá estas variables con valor vacío:

| Variable | Valor recomendado |
|---|---|
| `VITE_AUTH_API_BASE` | *(vacío)* |
| `VITE_ORDER_API_BASE` | *(vacío)* |

Si esas variables no existen, el fallback del código puede apuntar a `http://localhost:8081` y `http://localhost:8080` dentro del bundle. Si existen con valor vacío, las llamadas quedan como `/api/users/...` y `/api/v1/pedido/...`, y pasan por `http://localhost:3000`.

---

## CI

El workflow de este repo valida:

- `docker compose config --quiet` con variables stub.
- Sintaxis de `nginx/proxy.conf` con upstreams `frontend`, `user-service` y `shipment-service`.
- Smoke test opcional/automático en `main` que levanta el stack real, verifica `http://localhost:3000` y comprueba que `/api/users/register` y `/api/v1/pedido` llegan a un backend HTTP real sin depender de `/actuator/health`.

Para el smoke test, el workflow usa por defecto los tags documentados en este README. Si querés sobreescribirlos sin editar el repo, definí estas GitHub Actions Variables:

```text
FRONTEND_IMAGE
USER_SERVICE_IMAGE
SHIPMENT_SERVICE_IMAGE
```

También puede usar estos secrets si están configurados:

```text
JWT_SECRET
ORS_API_KEY
```

---

## Estructura del repo

```text
infra/
├── .github/workflows/ci.yml
├── docker-compose.yml
├── nginx/
│   └── proxy.conf
├── .env.example
├── .env
└── README.md
```

`.env` contiene valores reales y no debe commitearse.
