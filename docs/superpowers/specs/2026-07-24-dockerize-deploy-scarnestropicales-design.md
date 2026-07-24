# Dockerización y despliegue de scarnestropicales.codewave.com.bo

## Contexto

El proyecto (backend Node/Express + frontend React/Vite + MariaDB) es una adaptación de
un sistema previo ("salybrasas") que se despliega en el mismo VPS vía PM2 + nginx nativo
(sin Docker). Ahora se necesita desplegar esta instancia (cliente "solocarnestropicales")
en el mismo VPS, bajo el subdominio `scarnestropicales.codewave.com.bo`, usando Docker,
sin interferir con el sistema salybrasas que ya corre ahí (puerto 3007, PM2).

Docker y Docker Compose ya están instalados en el VPS.

## Arquitectura

Tres servicios en `docker-compose.yml` (raíz del proyecto):

| Servicio   | Imagen base           | Puerto contenedor | Puerto host | Notas |
|------------|------------------------|--------------------|-------------|-------|
| `db`       | `mariadb:11`           | 3306               | (no expuesto) | Volumen nombrado `db_data`; semillado inicial desde `bd/bd_solocarnestropicales.sql` vía `docker-entrypoint-initdb.d` |
| `backend`  | `node:20-alpine`       | 3001               | 3008        | `npm ci --omit=dev`; se conecta a `db` por nombre de servicio Docker |
| `frontend` | build multi-stage → `nginx:alpine` | 80     | 8081        | Stage 1 compila con Vite (`VITE_API_URL` como build-arg); Stage 2 sirve el `dist/` resultante con nginx interno (solo SPA fallback + gzip, sin TLS) |

El nginx **nativo del host** (ya gestiona otros subdominios como salybrasas) recibe un
nuevo server block para `scarnestropicales.codewave.com.bo`:
- `/` → `proxy_pass http://localhost:8081` (contenedor frontend)
- `/api`, `/socket.io`, `/webhooks`, `/uploads` → `proxy_pass http://localhost:3008` (contenedor backend)

Certbot (ya instalado en el VPS) emite el certificado SSL para el nuevo subdominio,
igual que para los demás.

`print-agent` queda fuera de Docker: corre localmente en las máquinas cliente para
impresión térmica, no se despliega en el VPS.

## Variables de entorno

Nuevo `.env.production` en la raíz del proyecto, consumido por `docker-compose.yml`
(usando `env_file` / interpolación de Compose). Reemplaza los valores de desarrollo
(`DB_HOST=localhost`, `DB_USER=root`, `DB_PASS=` vacío, `JWT_SECRET=secretas`,
`ADMIN_PASSWORD=admin123`, claves CodePay de sandbox) por:

- `DB_HOST=db` (nombre del servicio Docker, no `localhost`)
- `DB_NAME`, `DB_USER`, `DB_PASS` — usuario dedicado no-root con contraseña fuerte generada
- `JWT_SECRET`, `JWT_REFRESH_SECRET` — 64 caracteres aleatorios generados con `openssl rand -hex 32`
- `ADMIN_PASSWORD` — contraseña fuerte generada, distinta a la de desarrollo
- `CORS_ORIGIN=https://scarnestropicales.codewave.com.bo`
- `CODEPAY_*` — placeholders a completar por el usuario con las claves reales de producción
  (las actuales en `backend/.env` son de sandbox/test, no válidas para cobros reales)
- `VITE_API_URL=https://scarnestropicales.codewave.com.bo/api/v1` (usado como build-arg del frontend)

`.env.production` se añade a `.gitignore` (no se versiona, igual que `backend/.env`).
Se provee `.env.production.example` documentando cada variable sin valores reales.

## Despliegue

1. Se inicializa git en el repo local (actualmente no es un repositorio git); el usuario
   lo sube a un remoto (GitHub/GitLab).
2. En el VPS: `git clone`/`git pull` del repo dentro de una carpeta dedicada
   (ej. `/home/ubuntu/SISTEMAS/SOLOCARNESTROPICAL`, paralela a la de salybrasas).
3. Primer arranque: `docker compose --env-file .env.production up -d --build`
   (crea y semilla `db`, construye e inicia `backend` y `frontend`).
4. Se instala el server block de nginx del host (`deploy/nginx-scarnestropicales.conf`)
   y se corre `certbot --nginx` para el subdominio.
5. Actualizaciones futuras: `deploy/deploy-docker.sh` — hace `git pull`, reconstruye
   las imágenes que cambiaron (`docker compose build`), reinicia los contenedores
   (`docker compose up -d`) y recarga nginx del host si el server block cambió.

## Fuera de alcance

- No se migra salybrasas a Docker; sigue en PM2 tal cual está.
- No se dockeriza `print-agent`.
- No se automatiza la renovación de certificados más allá de lo que certbot ya hace
  por cron/systemd timer en el VPS (asumido ya configurado, dado que otros subdominios
  ya usan HTTPS).
- No se configura CI/CD (GitHub Actions, etc.) — el despliegue es manual vía script en el VPS.
