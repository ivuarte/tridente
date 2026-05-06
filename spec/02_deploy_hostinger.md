# Manual de Deploy — GitHub Actions → Hostinger

Este documento explica cómo está configurada la integración entre el repositorio de GitHub y el hosting de Hostinger para el proyecto Tridente. Cualquier persona del equipo puede seguirlo para entender el flujo, replicarlo o solucionar problemas.

---

## Cómo funciona el flujo

```
Desarrollador
    │
    │  git push → main
    ▼
GitHub (repositorio: ivuarte/tridente)
    │
    │  dispara automáticamente
    ▼
GitHub Actions (runner Ubuntu)
    │  1. Clona el repo
    │  2. Instala Node.js 20
    │  3. npm ci  (instala dependencias)
    │  4. npm run build  (genera site/dist/)
    │
    │  sube archivos via FTPS
    ▼
Hostinger — servidor 46.202.199.66
    │
    └── /public_html/ → tridente.iammtechs.com
```

Cada `git push` a la rama `main` dispara el deploy automáticamente. No se necesita subir archivos manualmente ni abrir el File Manager de Hostinger.

---

## Archivos relevantes del proyecto

```
tridente/
├── .github/
│   └── workflows/
│       └── deploy.yml        ← define todo el proceso de CI/CD
├── site/
│   ├── src/                  ← código fuente (lo que se edita)
│   ├── dist/                 ← build output (generado, NO se sube al repo)
│   └── package.json
└── spec/
    └── 02_deploy_hostinger.md  ← este documento
```

---

## El workflow: `.github/workflows/deploy.yml`

```yaml
name: Deploy to Hostinger

on:
  push:
    branches: [main]       # se dispara solo en pushes a main
  workflow_dispatch:        # permite dispararlo manualmente desde GitHub

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # descarga el código

      - uses: actions/setup-node@v4        # instala Node.js 20
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: site/package-lock.json

      - name: Install dependencies
        working-directory: site
        run: npm ci                        # instala dependencias exactas del lock

      - name: Build static site
        working-directory: site
        run: npm run build                 # genera site/dist/

      - name: Deploy via FTP
        uses: SamKirkland/FTP-Deploy-Action@v4.3.5
        with:
          server: ${{ secrets.FTP_HOST }}
          username: ${{ secrets.FTP_USERNAME }}
          password: ${{ secrets.FTP_PASSWORD }}
          local-dir: ./site/dist/          # carpeta local a subir
          server-dir: ${{ secrets.FTP_SERVER_DIR }}  # destino en Hostinger
          protocol: ftps
          port: 21
          exclude: |
            **/.git*
            **/.git*/**
            **/node_modules/**
```

**Puntos clave:**
- `npm ci` (no `npm install`) garantiza que se instalen exactamente las versiones del `package-lock.json`. Más predecible en CI.
- Solo se sube el contenido de `site/dist/` — el código fuente, `node_modules` y el repo Git nunca llegan al servidor.
- `FTP-Deploy-Action` hace una sincronización incremental: solo transfiere los archivos que cambiaron respecto al deploy anterior. El primer deploy sube todo.

---

## Secrets de GitHub — configuración actual

Los secrets se almacenan cifrados en GitHub y nunca aparecen en los logs ni en el código. Se configuran en:

`https://github.com/ivuarte/tridente/settings/secrets/actions`

| Secret | Descripción | Valor de referencia |
|--------|-------------|---------------------|
| `FTP_HOST` | IP o hostname del servidor FTP | `46.202.199.66` |
| `FTP_USERNAME` | Usuario FTP de Hostinger | `u485729917.tridente.iammtechs.com` |
| `FTP_PASSWORD` | Contraseña de la cuenta FTP | *(privada)* |
| `FTP_SERVER_DIR` | Carpeta destino en el servidor | `/public_html/` |

### Dónde encontrar estos valores en Hostinger

1. Ingresa al **hPanel** → **Hosting** → tu plan → **Administrar**
2. Menú lateral → **Archivos → Cuentas FTP**
3. Selecciona la cuenta `u485729917.tridente.iammtechs.com`
4. Ahí aparecen el IP, usuario y puerto

**Notas importantes sobre los valores:**
- `FTP_HOST`: solo la IP o dominio, **sin** `ftp://` al inicio
- `FTP_SERVER_DIR`: debe empezar Y terminar con `/`. El valor `/public_html/` funciona porque la cuenta FTP está anclada a `/home/u485729917/domains/tridente.iammtechs.com/` y `public_html/` es la carpeta raíz del sitio web
- Si la contraseña se olvida, se puede cambiar desde la misma sección de Cuentas FTP en hPanel, y luego actualizar el secret `FTP_PASSWORD` en GitHub

---

## Cómo hacer un deploy

### Deploy automático (flujo normal)
```bash
# 1. Hacer cambios en el código
# 2. Commitear y subir
git add .
git commit -m "feat: descripción del cambio"
git push
```
El deploy inicia automáticamente. Tarda ~2 minutos.

### Deploy manual (sin hacer cambios)
1. Ir a `https://github.com/ivuarte/tridente/actions`
2. Click en el workflow **"Deploy to Hostinger"**
3. Botón **"Run workflow"** → seleccionar rama `main` → **"Run workflow"**

---

## Cómo verificar el estado del deploy

1. `https://github.com/ivuarte/tridente/actions`
2. El workflow más reciente aparece con un ícono:
   - 🟡 Amarillo: en ejecución
   - ✅ Verde: deploy exitoso
   - ❌ Rojo: falló algo
3. Click en el workflow para ver los logs paso a paso

---

## Solución de errores comunes

### `ENOTFOUND` (no encuentra el servidor)
**Causa:** `FTP_HOST` tiene un formato incorrecto.
**Solución:** Verificar que el secret `FTP_HOST` sea solo la IP (`46.202.199.66`) sin `ftp://` ni espacios.

### `server-dir should be a folder (must end with /)`
**Causa:** `FTP_SERVER_DIR` no termina con `/`.
**Solución:** Editar el secret y agregar `/` al final. Ejemplo: `/public_html/`.

### `Failed to connect` / `only supports SFTP`
**Causa:** Problema de protocolo o puerto. Hostinger compartido usa FTPS en puerto 21, no SFTP.
**Solución:** Verificar en `deploy.yml` que `protocol: ftps` y `port: 21` estén correctos. No cambiar a SFTP.

### `530 Login incorrect`
**Causa:** Usuario o contraseña FTP incorrectos.
**Solución:** Verificar los secrets `FTP_USERNAME` y `FTP_PASSWORD`. Si la contraseña fue cambiada en Hostinger, actualizar el secret en GitHub.

### El sitio se ve sin estilos (CSS 404)
**Causa:** Los archivos se subieron a una subcarpeta en lugar de la raíz del subdominio.
**Solución:** Verificar que `FTP_SERVER_DIR` apunte directamente a `public_html/` del subdominio. Revisar también el File Manager de Hostinger para confirmar que `index.html` está en la raíz (no dentro de una carpeta extra).

### Build falla en `npm ci`
**Causa:** El `package-lock.json` no está commiteado o está desincronizado.
**Solución:** En local, correr `npm install` dentro de `site/` y commitear el `package-lock.json` actualizado.

---

## Estructura del servidor Hostinger

```
/home/u485729917/
└── domains/
    └── tridente.iammtechs.com/
        └── public_html/              ← raíz del subdominio (FTP_SERVER_DIR apunta aquí)
            ├── index.html            ← página principal
            ├── _astro/
            │   └── index.[hash].css  ← estilos de Tailwind (purgados)
            └── images/
                ├── hero-bg.jpg
                └── profile-duarte.jpg
```

El hash en el CSS cambia con cada build. Esto fuerza al navegador a descargar el CSS nuevo en lugar de usar el caché.

---

## Agregar un nuevo miembro al equipo

Para que otra persona pueda hacer deploys:

1. Invitarla como colaboradora en GitHub: `https://github.com/ivuarte/tridente/settings/collaborators`
2. Darle acceso de rol **Write** o superior
3. Los secrets de GitHub ya están configurados — no necesita verlos ni conocerlos para hacer push y disparar el deploy
4. Compartirle este documento y el `spec/01_plan.md` para contexto del proyecto

---

## Re-configurar desde cero (si se migra de servidor)

Si Hostinger cambia el servidor o se migra a otro hosting:

1. Obtener las nuevas credenciales FTP del panel de control del nuevo hosting
2. Actualizar los 4 secrets en `https://github.com/ivuarte/tridente/settings/secrets/actions`:
   - `FTP_HOST` → nueva IP o hostname
   - `FTP_USERNAME` → nuevo usuario FTP
   - `FTP_PASSWORD` → nueva contraseña
   - `FTP_SERVER_DIR` → nueva ruta de destino
3. Disparar un deploy manual para verificar que funciona
4. Actualizar este documento con los nuevos valores de referencia

No se necesita tocar el código ni el `deploy.yml`.
