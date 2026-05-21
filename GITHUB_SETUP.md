# GITHUB_SETUP.md — Configuración inicial del repositorio

Sigue estos pasos UNA SOLA VEZ al crear el repo central o un nuevo proyecto cliente.

---

## 1. Crear el repositorio como PRIVADO

En GitHub → New repository → **Private**

---

## 2. Configurar Branch Protection en `main`

GitHub repo → Settings → Branches → Add rule:

- Branch name pattern: `main`
- ✅ Require a pull request before merging
- ✅ Require approvals: 1
- ✅ Dismiss stale pull request approvals
- ✅ Require status checks to pass before merging (seleccionar el job `deploy`)
- ✅ Restrict who can push to matching branches → agregar solo tu usuario
- ✅ Do not allow bypassing the above settings

Esto garantiza que **solo tú** puedes aprobar y hacer merge a main.

---

## 3. Agregar GitHub Secrets

GitHub repo → Settings → Secrets and variables → Actions → New repository secret

| Secret | Valor |
|---|---|
| `DATABASE_URL` | URL completa de PostgreSQL en AlwaysData |
| `NEXTAUTH_SECRET` | String aleatorio seguro (genera con `openssl rand -base64 32`) |
| `NEXTAUTH_URL` | URL pública del sitio (ej: `https://micliente.com`) |
| `ALWAYSDATA_SSH_KEY` | Contenido completo de la clave privada SSH |
| `ALWAYSDATA_HOST` | `ssh-USER.alwaysdata.net` |
| `ALWAYSDATA_USER` | Usuario SSH de AlwaysData |
| `ALWAYSDATA_PROJECT_PATH` | Ruta absoluta del proyecto en el servidor (ej: `/home/USER/www/proyecto`) |

---

## 4. Generar la clave SSH para deploy

En tu máquina local:

```bash
ssh-keygen -t ed25519 -C "deploy-key-PROYECTO" -f ~/.ssh/deploy_PROYECTO
```

- Copia el contenido de `~/.ssh/deploy_PROYECTO` (clave privada) → GitHub Secret `ALWAYSDATA_SSH_KEY`
- Copia el contenido de `~/.ssh/deploy_PROYECTO.pub` → AlwaysData panel → SSH Keys

---

## 5. Configurar Node.js en AlwaysData

AlwaysData panel → Web → Sites → Agregar sitio:

- Type: **Node.js**
- Command: `npm run start`
- Working directory: ruta del proyecto
- Port: el que asigne AlwaysData (usar en `NEXTAUTH_URL`)

---

## 6. Variables de entorno en el servidor

En AlwaysData, el archivo `.env` NO debe existir en producción.
Las variables se configuran en el panel de AlwaysData → Environment variables,
o se inyectan desde GitHub Actions durante el deploy.

---

## 7. Flujo de trabajo para mejoras al skeleton

```
1. Crea una rama: git checkout -b mejora/nombre
2. Haz los cambios
3. Abre un Pull Request hacia main
4. Solo tú puedes aprobarlo y hacer merge
5. El deploy corre automáticamente
```
