# Plan de Implementación — Tridente Landing Page

## Decisiones Confirmadas

| Parámetro | Valor |
|-----------|-------|
| Dominio producción | `tridente.iammtechs.com` |
| Hosting | Hostinger Premium Web Hosting (compartido) |
| Pasarela de pago | Sin definir — CTA de lista de espera + WhatsApp |
| Imágenes | Placeholders en desarrollo |
| LinkedIn | https://www.linkedin.com/in/iván-ramiro-duarte-aguirre-7901571ba/ |
| Redes sociales | @protocolo_duarte (IG · FB · TikTok) |

---

## Stack

**Astro + Tailwind CSS** → sitio estático puro

- Astro genera `dist/` con HTML/CSS/JS puros al hacer build
- Ese `dist/` se sube directamente a Hostinger vía FTP o File Manager
- Sin base de datos, sin servidor, sin plugins que mantener
- Docker solo en desarrollo local para aislar la versión de Node

---

## Estructura de Carpetas

```
tridente/
├── spec/
│   ├── 00_init.md
│   └── 01_plan.md
│
├── site/                        # Código fuente
│   ├── public/
│   │   ├── images/
│   │   │   ├── hero-bg.jpg          # placeholder
│   │   │   ├── product-tridente.png # placeholder
│   │   │   └── profile-duarte.jpg   # placeholder
│   │   └── favicon.ico
│   │
│   ├── src/
│   │   ├── components/
│   │   │   ├── Nav.astro
│   │   │   ├── Hero.astro
│   │   │   ├── About.astro
│   │   │   ├── Product.astro
│   │   │   ├── CTA.astro
│   │   │   └── Footer.astro
│   │   ├── layouts/
│   │   │   └── Base.astro
│   │   ├── pages/
│   │   │   └── index.astro
│   │   └── styles/
│   │       └── global.css
│   │
│   ├── astro.config.mjs
│   ├── tailwind.config.mjs
│   └── package.json
│
├── docker/
│   └── docker-compose.yml       # Solo para desarrollo local
│
└── .gitignore
```

---

## Fase 1 — Entorno Local

### Opción A (recomendada): Node directo
```bash
cd site
npm install
npm run dev   # http://localhost:4321 con live reload
```

### Opción B: Docker (si se prefiere aislar Node del SO)
```yaml
# docker/docker-compose.yml
services:
  site:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ../site:/app
    ports:
      - "4321:4321"
    command: sh -c "npm install && npm run dev -- --host 0.0.0.0"
```
```bash
docker compose -f docker/docker-compose.yml up
# → http://localhost:4321
```

> Traefik no es necesario: no hay subdominios en local que necesiten enrutamiento. Un puerto directo es suficiente.

---

## Fase 2 — Estructura de la Página

Embudo de conversión de una sola página, scroll vertical.

### `<Nav>`
- Logo / nombre a la izquierda: **"Protocolo Duarte"**
- Links ancla: `Sobre mí` · `El Tridente` · `Adquirir`
- Sticky en scroll, fondo semitransparente al bajar

### `<Hero>`
- **Headline:** "El rendimiento no se improvisa. Se construye."
- **Sub-headline:** Negocios de alto nivel, disciplina física y mentalidad ejecutiva — los tres pilares no son opcionales.
- **CTA:** botón "Conoce el Tridente" → scroll a sección Product
- Fondo: imagen oscura de alto contraste (placeholder de entrenamiento)

### `<About>`
- Foto de perfil (placeholder)
- Párrafo de marca personal: empresario tech, lifestyle disciplinado
- 3 pilares en cards:
  - **Negocios** — Sector tecnológico, ejecución y decisiones
  - **Cuerpo** — Entrenamiento con pesas, disciplina física
  - **Mente** — Lectura constante, filosofía, autodescubrimiento
- Links sociales: LinkedIn · Instagram · Facebook · TikTok

### `<Product>` — El Tridente
- Título: **"Las 3 herramientas del rendimiento"**
- 3 cards individuales:
  - **Proteína** — Recuperación muscular y síntesis proteica
  - **Creatina** — Fuerza, explosividad y rendimiento cognitivo
  - **Omega-3** — Inflamación, salud cerebral y cardiovascular
- Frase ancla: *"No es un suplemento. Es un protocolo."*
- Imagen del producto (placeholder)

### `<CTA>` — Conversión (versión pre-lanzamiento)
Dado que aún no hay pasarela de pago ni precio definido, la sección tendrá **dos acciones**:

1. **Lista de espera** — Formulario (nombre + email) via Formspree (gratis, sin backend)
   - Texto: "Sé el primero en acceder al Tridente cuando esté disponible."
2. **Contacto directo** — Botón de WhatsApp con mensaje pre-llenado
   - Texto: "Hablar con Protocolo Duarte"

> Cuando el precio y la pasarela estén definidos, esta sección se reemplaza por el botón de pago directo.

### `<Footer>`
- @protocolo_duarte en IG · FB · TikTok
- LinkedIn
- Email de contacto
- Aviso legal mínimo

---

## Paleta Visual

| Elemento            | Valor                    |
|---------------------|--------------------------|
| Fondo base          | `#0a0a0a`                |
| Texto principal     | `#f0f0f0`                |
| Acento primario     | `#c9a84c` (dorado mate)  |
| Acento secundario   | `#4a7c59` (verde militar)|
| Tipografía títulos  | Space Grotesk Bold       |
| Tipografía cuerpo   | Inter Regular            |

---

## Fase 3 — Despliegue en Hostinger (cuando esté listo)

### Crear subdominio
1. hPanel → **Subdominios** → Crear: `tridente` en `iammtechs.com`
2. Hostinger crea automáticamente la carpeta `public_html/tridente.iammtechs.com/`

### Build y subida
```bash
cd site
npm run build
# Genera site/dist/
```
Subir el contenido de `dist/` a `public_html/tridente.iammtechs.com/` via:
- **hPanel File Manager** (drag & drop)
- O FTP con credenciales del panel

### SSL
hPanel → **SSL** → **Let's Encrypt** → seleccionar `tridente.iammtechs.com` → Instalar.
(Hostinger lo hace en 1 clic, sin comandos)

---

## Integración de Pagos (fase futura)

Cuando el precio esté definido, la ruta más directa para Colombia es:
- **Wompi** (Bold) — Para clientes en COP, sin costo mensual, 2.99% + IVA por transacción
- **Stripe** — Para pagos internacionales o con tarjeta

Flujo de automatización (cuando se active):
```
Compra confirmada → webhook Wompi/Stripe
      ↓
  n8n Cloud (free tier) o Make.com
      ↓
  Google Sheets (registro) + Email/Telegram al dueño
```

---

## Checklist de Implementación

### Fase 1 — Configuración inicial
- [ ] Inicializar Git en `/home/ivan/tridente`
- [ ] Crear proyecto Astro en `site/`
- [ ] Configurar Tailwind CSS
- [ ] Levantar dev server y verificar en localhost

### Fase 2 — Página
- [ ] Layout base (`Base.astro`) con fuentes y estilos globales
- [ ] `Nav.astro` — sticky, links ancla
- [ ] `Hero.astro` — headline + CTA scroll
- [ ] `About.astro` — perfil + 3 pilares + redes
- [ ] `Product.astro` — 3 cards del Tridente
- [ ] `CTA.astro` — formulario lista de espera + WhatsApp
- [ ] `Footer.astro`
- [ ] Ajuste responsive (mobile-first)
- [ ] Revisar en mobile, tablet y desktop

### Fase 3 — Deploy
- [ ] Crear subdominio en Hostinger
- [ ] `npm run build` → revisar `dist/`
- [ ] Subir a Hostinger
- [ ] Activar SSL Let's Encrypt
- [ ] Verificar `https://tridente.iammtechs.com`

### Fase 4 — Pagos (cuando estén listos)
- [ ] Definir precio del Tridente
- [ ] Crear cuenta Wompi o Stripe
- [ ] Generar payment link
- [ ] Reemplazar CTA de lista de espera por botón de pago
- [ ] Conectar webhook a n8n/Make
