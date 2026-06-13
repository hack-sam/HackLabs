# CLAUDE.md — HackLabs

Plataforma web monolítica de laboratorios intencionalmente vulnerables (OWASP Top 10, vulns web, ataques a IA). Flask + SQLite + Tailwind (CDN) + Jinja2. Bilingüe ES/EN, sistema de dificultad y progreso por cuenta.

**Todo es inseguro a propósito.** No "arregles" vulnerabilidades de los labs. `secret_key`, cookies sin HttpOnly/Secure, SQLi, RCE, etc. son el producto.

## Estructura

- `app.py` (~5500 líneas) — toda la lógica: rutas Flask, labs, flags, DB, servicios TCP simulados. Archivo único.
- `templates/base.html` — layout global: navbar, sidebar, footer, modal de resolución, FAB. Todo lab lo extiende.
- `templates/_lab_header.html` — partial: breadcrumb + título + badges (dificultad/riesgo).
- `templates/labs/<id>.html` — un template por lab.
- `static/js/main.js` (~1577 líneas) — i18n (objeto `T`), tema, sidebar, dificultad, progreso, modal.
- `static/css/style.css` — estilos custom (variables `--hl-*`, navbar responsive, dropdowns).
- `database/schema.sql` — schema + seed (users, products, comments, orders).
- `data/hacklabs.db` — SQLite en runtime (se crea si no existe).

## Ejecutar

```bash
python3 app.py            # crea DB si falta, levanta HTTP + FTP(21)/SSH(22)/SMB(445) simulados
```
Puerto: `APP_PORT` env (default **80** en local, **5000** en docker-compose). Docker: `deploy.sh` (estilo DockerLabs, macvlan) o `docker-compose.yml`. Sin build step.

## Conceptos núcleo (en `app.py`)

- **`get_lab_list()`** (~L1454) — fuente de verdad: lista de dicts `{id, title, category, risk}`. Categorías: `OWASP Top 10`, `Vulnerabilidades`, `IA Attacks`. `risk`: `critical|high|medium|low`.
- **`lab(lab_id)`** (~L1396) — `dict dedicated` mapea `lab_id → ruta propia` (redirige). Si no tiene ruta dedicada, renderiza `labs/<id>.html`.
- **`inject_labs()`** context_processor (~L1507) — inyecta en TODAS las plantillas: `all_labs` (ordenado por título), `current_lab_id` (vía `path_to_lab`), `difficulty`, `target_ip/base/hydra` (host real), progreso. El `dict path_to_lab` mapea cada ruta → lab_id para el estado activo del sidebar.
- **`get_lab_flag_map()`** (~L765) — flags esperadas por lab. `root_flag = HL{r00t_pr1v3sc_succ3ss}` se acepta como fallback global salvo en `explicit_screen_flag_labs`. `_expand_with_leet()` acepta variante leet automáticamente. `_validate_lab_flag_coverage()` falla al arrancar si un lab no tiene flag.
- **Dificultad** — `session['difficulty']` (`easy|medium|hard|nightmare`). `nightmare` se desbloquea al 100%. Cada ruta de lab lee `session.get('difficulty','easy')` y varía filtros/comportamiento server-side (ej. `ssti()` ~L3340). El front la cambia vía `POST /set-difficulty`.

## i18n (ES/EN)

- Objeto `T` en `main.js`: `T.es` / `T.en`, claves planas. `applyTranslations()` recorre `[data-i18n]`, `[data-i18n-placeholder]`.
- Títulos de labs: clave `lab_title_<id>` en `T.es` y `T.en`.
- Idioma: `localStorage` + `setLang()`; default ES.
- **Resolución del lab**: NO usa `T`. Va en el HTML del template dentro de `<div id="resolution-data">` con dos bloques `<div class="lang-content" data-lang="es|en">`. El FAB "Ver resolución" abre el modal con ese contenido.

## Añadir un lab nuevo (checklist)

1. `get_lab_list()` — añadir dict `{id, title, category, risk}`.
2. `get_lab_flag_map()` — añadir flags. Si la flag se muestra en pantalla, añadir el id a `explicit_screen_flag_labs` (bloquea root fallback).
3. Ruta(s) Flask del lab + lógica vulnerable (lee `difficulty` si aplica).
4. `lab()` → `dedicated` y `inject_labs()` → `path_to_lab`: mapear rutas ↔ id (estado activo sidebar).
5. `templates/labs/<id>.html` — `{% extends "base.html" %}`, `{% include '_lab_header.html' %}`, UI con `data-i18n`, y `#resolution-data` (es/en). Ver `labs/ssti.html` como referencia.
6. `base.html` → dict `sidebar_icons` — icono Phosphor para el id.
7. `main.js` `T.es`/`T.en` — `lab_title_<id>` + cualquier `data-i18n` nuevo del template.

## Convenciones

- Flags: formato `HL{...}` en leet (algunos labs usan `HackLabs{...}`).
- Color primario `--hl-primary: #CEFF00` (lima). Iconos: Phosphor (`ph ph-*`). Highlight.js para código.
- Tokens de placeholder en templates: `TARGET_IP`, `localhost:5000` se reemplazan en cliente por el host real (script al final de `base.html`).
- Progreso/certificados/recompensas: solo para cuentas propias (`app_user_type == 'account'`), tabla `user_progress`. Usuarios "lab" (de los propios labs) no guardan progreso.
- Tras editar código: `graphify update .` para mantener el grafo (`graphify-out/`).
