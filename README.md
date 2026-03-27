# optimizador-envios — infra

Repositorio de infraestructura del proyecto. Centraliza el `docker-compose.yml` que levanta el stack completo usando las imágenes publicadas en GHCR por los repositorios `frontend` y `backend`.

---

## Requisitos previos

- Docker Desktop >= 24 (o Docker Engine + Compose plugin >= 2.20)
- Acceso de lectura a `ghcr.io/optimizador-de-envios` (login si los packages son privados)

```bash
echo $CR_PAT | docker login ghcr.io -u TU_USUARIO --password-stdin
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

Editá `.env` y completá los valores:

| Variable | Descripción |
|---|---|
| `BACKEND_IMAGE` | Tag completo de la imagen del backend en GHCR |
| `FRONTEND_IMAGE` | Tag completo de la imagen del frontend en GHCR |
| `ORS_API_KEY` | API key de [OpenRouteService](https://openrouteservice.org/dev/#/signup) |

### 3. Levantar el stack

```bash
docker compose up -d
```

### 4. Verificar que los servicios estén corriendo

```bash
docker compose ps
docker compose logs -f
```

### 5. Acceder

| Servicio | URL local | Descripción |
|---|---|---|
| App (frontend + proxy) | http://localhost:3000 | Entrada principal. El proxy enruta `/api/*` al backend internamente. |
| Backend (API directa) | http://localhost:8080 | Acceso directo para pruebas con curl/Postman. |

---

## Cómo actualizar las versiones de imagen

Las imágenes se controlan exclusivamente con las variables `FRONTEND_IMAGE` y `BACKEND_IMAGE` en el archivo `.env`.

Cuando CI publique un nuevo build en GHCR, actualizá el tag correspondiente:

```dotenv
# Antes
FRONTEND_IMAGE=ghcr.io/optimizador-de-envios/optimizador-envios-frontend:sha-e1b59a2

# Después (nuevo SHA generado por CI)
FRONTEND_IMAGE=ghcr.io/optimizador-de-envios/optimizador-envios-frontend:sha-XXXXXXX
```

Luego aplicá el cambio sin downtime:

```bash
docker compose pull
docker compose up -d
```

> También podés usar etiquetas semánticas (`1.2.0`, `latest`) si tu pipeline CI las publica.

---

## Arquitectura del proxy

El servicio `proxy` (nginx) es el único punto de entrada en la red Docker:

```
Navegador → localhost:3000 → nginx proxy
                                ├── /api/* → http://backend:8080/api/*
                                └── /*     → http://frontend:80
```

Esto resuelve el problema estructural de las variables Vite de build time:
- El frontend fue compilado con `VITE_ORDER_API_BASE=''` (vacío), por lo que el JS genera URLs relativas: `/api/v1/pedido`.
- El navegador resuelve esa URL relativa contra `http://localhost:3000`, que es el nginx proxy.
- Nginx enruta `/api/` al backend dentro de la red Docker, donde `backend:8080` sí resuelve.
- **La misma imagen funciona en local, staging y producción** sin recompilar.

### Requisito en el CI del frontend

Esta arquitectura funciona si el CI del frontend compila la imagen con `VITE_ORDER_API_BASE` vacío. La variable de GitHub Actions debe existir con valor vacío:

| Variable (GitHub Actions → Variables) | Valor recomendado |
|---|---|
| `VITE_ORDER_API_BASE` | *(vacío)* |

Con esto, el fallback del código (`?? 'http://localhost:8080'`) nunca se usa dentro de la imagen Docker y el routing siempre pasa por el proxy.

---

## Estructura del repo

```
infra/
├── docker-compose.yml      # Definición de todos los servicios
├── nginx/
│   └── proxy.conf          # Config del reverse proxy
├── .env.example            # Plantilla de variables de entorno (commitear)
├── .env                    # Variables reales (NO commitear)
└── README.md
```

---

## Mejoras opcionales (sin mezclar con la solución actual)

Estas mejoras pueden incorporarse en una iteración futura sin modificar la arquitectura base:

- **Nginx reverse proxy** en el propio infra: resolvería el problema de build-time Vite de forma definitiva. El browser apuntaría a `http://localhost:3000/api` y nginx haría proxy a `http://backend:8080` dentro de la red Docker.
- **Etiquetas semánticas** en CI: publicar `:latest` y `:1.x.x` además del SHA para facilitar rollbacks y actualizaciones más descriptivas.
- **`docker compose watch`**: para desarrollo local con hot-reload mapeando volúmenes del código fuente (requiere imágenes con modo dev).
- **GitHub Actions en este repo**: un workflow que valide el `.env.example` y haga smoke test contra las imágenes referenciadas.
