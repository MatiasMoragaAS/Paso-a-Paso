# Planificación Backend & Fullstack — Solemne 3

## Integrantes del Grupo
- Diego Alvarez
- Matias Moraga

## Repositorio GitHub
[https://github.com/DiegoUC-01/Solemne2]

## Tecnologías Principales
- **Frontend:** Vue.js 3 (Composition API) + Pinia + Vite
- **Backend:** Node.js 20 + Express 5
- **Base de datos:** MongoDB 7 + Mongoose 8
- **IA externa:** Google Gemini API (gemini-2.5-flash)
- **Autenticación:** JWT + bcryptjs
- **Testing:** Vitest (frontend y backend)
- **Linter:** oxlint
- **CI/CD:** GitHub Actions
- **Containerización:** Docker + Docker Compose

---

## Semana 1 — Infraestructura, CI/CD y Setup (16 jun – 22 jun)

> **Objetivo:** Tener repo funcional desde el día 1. GitHub Actions corriendo en cada push. Docker funcionando. Documentación base lista.

### Tareas planificadas:

**Repositorio y estructura (subir primero):**
- [x] Crear estructura de carpetas del proyecto:
  ```
  Solemne2/
  ├── .github/workflows/main.yml
  ├── backend/
  │   ├── Dockerfile
  │   ├── package.json
  │   ├── .env.example
  │   ├── .gitignore
  │   └── src/
  │       ├── config/db.js
  │       ├── models/User.js
  │       ├── models/Game.js
  │       └── models/Dialogue.js
  ├── Game/              (existente)
  └── compose.yml
  ```
- [x] Crear `.env.example` (PORT, MONGODB_URI, JWT_SECRET, GEMINI_API_KEY)
- [x] Crear `.gitignore` del backend (node_modules, .env, coverage)

**CI/CD — GitHub Actions (subir antes que el código):**
- [x] Crear `.github/workflows/main.yml`:
  - Job `lint-and-test-frontend`: pnpm install → lint → test
  - Job `lint-and-test-backend`: pnpm install → lint → test
  - Job `build-and-push`: depende de ambos → login DockerHub → build y push ambas imágenes

**Docker (subir antes que el código):**
- [x] Crear `backend/Dockerfile` (Node 20 Alpine, Express en :3000)
- [x] Actualizar `Game/Dockerfile` (agregar pnpm como gestor de paquetes)
- [x] Crear `compose.yml` en raíz:
  - `mongodb`: imagen mongo:7, volumen persistente, puerto 27017
  - `backend`: build ./backend, depende de mongodb, variables de entorno
  - `frontend`: build ./Game, depende de backend, variables de entorno

**Documentación (subir antes del código):**
- [x] Actualizar `DESIGN.md` con arquitectura fullstack:
  - Diagrama frontend → backend → MongoDB + Gemini
  - Modelo de datos (users, games, dialogues)
  - Endpoints principales de la API REST
  - Servicio externo: Google Gemini, endpoints consumidos, integración
  - Justificación del stack backend
  - Estructura de carpetas actualizada (Game + backend)
- [x] Crear `PLANNING_BACKEND.md` (este archivo)
- [x] Enviar correo al profesor con integrantes y link al repo

**Modelos MongoDB (subir temprano para que CI/CD tenga contexto):**
- [x] `models/User.js` — username, email, password (bcrypt), timestamps
- [x] `models/Game.js` — día, fase, recursos, flags, journal, eventos, estado
- [x] `models/Dialogue.js` — contextHash, prompt, response, model, tokensUsed

**Configuración inicial:**
- [x] `config/db.js` — conexión a MongoDB con Mongoose
- [x] `package.json` del backend con todas las dependencias declaradas

### Objetivo de la semana:
> CI/CD corriendo en cada push, Docker Compose levanta 3 servicios (aunque backend aún vacío), documentación lista, modelos definidos. El repo ya muestra estructura profesional.

### Lo que se logró completar:
Se creó la estructura completa del proyecto con `.github/workflows/main.yml` (3 jobs: lint+test frontend, lint+test backend, build+push DockerHub). Se dockerizaron ambos servicios (frontend con pnpm, backend con Express) y se creó `compose.yml` orquestando los 3 contenedores. Se definieron los 3 modelos Mongoose (User, Game, Dialogue). Se crearon `.gitignore` y `.env.example` del backend. Se actualizó `DESIGN.md` con arquitectura fullstack, endpoints, modelo de datos y justificación del servicio externo. Se creó `PLANNING_BACKEND.md`.

### Lo que NO se logró:
—

### Notas:
—

---

## Semana 2 — Backend Core: Auth, Lógica de Juego, Gemini (23 jun – 29 jun)

> **Objetivo:** Backend completamente funcional. Cada capa se commitea y pushea por separado para que CI/CD valide incrementalmente.

### Tareas planificadas:

**Día 1-2: Auth (commits pequeños, push frecuentes):**
- [x] Crear `middleware/auth.js` — extrae y verifica JWT del header Authorization
- [x] Crear `routes/auth.js`:
  - `POST /api/auth/register` — validación de campos, email/username únicos, bcrypt hash, retorna JWT
  - `POST /api/auth/login` — busca usuario por email, bcrypt.compare, retorna JWT
  - `GET /api/auth/me` — protegido con middleware, retorna usuario sin password
- [x] Crear `src/index.js` — servidor Express con cors, json, rutas montadas
- [x] Crear `src/tests/auth.spec.js` (5 tests):
  - Generación de JWT válido
  - Decodificación correcta de token
  - Rechazo de token inválido
  - Rechazo de token con secreto incorrecto
  - Rechazo de token expirado

**Día 3-5: Lógica de juego (commits incrementales):**
- [x] Crear `routes/games.js` — portar gameStore.js al servidor:
  - `applyDailyConsumption()`: comida -1, agua -1, salud -4, moral -3
  - Penalización extra: comida=0 → -4 salud, agua=0 → -6 salud, ambos=0 → -2 moral
  - `clamp()`: recursos entre 0 y máximo
  - `applyEffects()`: aplicar efectos de decisiones a recursos
  - Sistema de flags narrativos (refugees, d1_solo, d2_share, etc.)
  - Detección de game over (salud≤0 → 'health', moral≤0 → 'morale')
  - Detección de victoria (día>15 con salud>0 y moral>0)
- [x] Eventos fijos del juego original (días 0, 1, 2, 3, 4, 6)
- [x] Minijuegos (día 5: catchRain, día 10: findCans, día 15: escape)
- [x] Sistema de eventos múltiples por día:
  - `eventsThisDay` contador de eventos jugados hoy
  - `maxEventsPerDay = 3` (configurable)
  - Evento fijo/minijuego se ejecuta primero
  - Luego IA genera eventos hasta completar el máximo
  - El día avanza solo cuando `eventsThisDay >= maxEventsPerDay`

**Endpoints de juego:**
- [x] `POST /api/games` — crear partida (userId, recursos iniciales, evento día 0)
- [x] `GET /api/games/user` — listar partidas del usuario (ordenadas por updatedAt)
- [x] `GET /api/games/:id` — obtener estado completo de partida
- [x] `PUT /api/games/:id/start` — iniciar día 1 (consumir recursos, cargar primer evento)
- [x] `PUT /api/games/:id/advance-segment` — avanzar segmento narrativo
- [x] `PUT /api/games/:id/decision` — registrar decisión, aplicar efectos, detectar game over
- [x] `PUT /api/games/:id/continue` — continuar tras ver resultado, cargar siguiente evento
- [x] `PUT /api/games/:id/minigame` — resultado de minijuego (win/lose), aplicar efectos

**Tests de lógica de juego:**
- [x] Crear `src/tests/games.spec.js` (15 tests):
  - Consumo diario de recursos
  - Penalización extra por comida/agua en 0
  - Aplicación de efectos positivos y negativos
  - Clamping (no baja de 0, no sube de máximo)
  - Detección de game over por salud
  - Detección de game over por moral
  - Detección de victoria (día > 15)
  - No victoria antes del día 16
  - Gestión de flags narrativos
  - Preservación de flags existentes
  - Sistema de eventos múltiples (avanza día al llegar a maxEventsPerDay)
  - Sistema de eventos múltiples (no avanza si faltan eventos)

**Día 6-7: Integración Gemini + Caché:**
- [x] Crear `services/gemini.js`:
  - `clamp()` — utilidad de rango
  - `buildGamePrompt(day, flags, food, water, health, morale, previousEvents)` — prompt contextual en español pidiendo JSON estructurado
  - `generateContextHash()` — SHA256 de día + flags + recursos para clave de caché
- [x] Crear `routes/dialogue.js`:
  - `POST /api/dialogue/generate`:
    1. Calcular contextHash
    2. Buscar en colección dialogues
    3. Si existe → devolver cacheado
    4. Si no → llamar a Gemini, guardar en MongoDB, devolver
  - `GET /api/dialogue/cache/stats` — contador de diálogos en caché
- [x] Fallback: si Gemini falla, devuelve evento genérico predefinido
- [x] Crear `src/tests/gemini.spec.js` (9 tests):
  - clamp dentro de rango, bajo mínimo, sobre máximo, en bordes
  - Hash SHA256 de 64 caracteres
  - Mismo contexto → mismo hash
  - Distinto contexto → distinto hash
  - Prompt contiene título del juego, día, recursos, flags
  - Prompt maneja flags vacíos

### Objetivo de la semana:
> Backend funcional de extremo a extremo: auth, lógica de juego completa, 8 endpoints de partida, integración con Gemini cacheada. 29 tests pasando. CI/CD verde en cada push.

### Lo que se logró completar:
Se implementaron todas las rutas del backend organizadas en 3 módulos: auth (register, login, me), games (8 endpoints con lógica de juego completa), dialogue (generación IA con caché en MongoDB). La lógica del juego fue portada completamente al servidor incluyendo consumo diario, efectos de decisiones, flags narrativos, detección de game over/victoria. El sistema de eventos múltiples por día (maxEventsPerDay=3) permite que eventos fijos y generados por IA coexistan. Gemini genera eventos en JSON estructurado y las respuestas se cachean por contextHash. Se crearon 29 tests unitarios (5 auth + 15 games + 9 gemini).

### Lo que NO se logró:
—

### Notas:
—

---

## Semana 3 — Integración Frontend, Testing Final y Entrega (30 jun – 2 jul)

> **Objetivo:** Frontend conectado al backend. Tests pasando en ambos lados. Todo funcional con Docker Compose. Entrega final.

### Tareas planificadas:

**Día 1: Cliente HTTP del frontend:**
- [x] Crear `src/api/index.js` — cliente HTTP para consumir backend:
  - `api.auth.register(username, email, password)`
  - `api.auth.login(email, password)`
  - `api.auth.me()`
  - `api.games.create()` / `list()` / `get(id)`
  - `api.games.start(id)` / `advanceSegment(id)` / `makeDecision(id, index)`
  - `api.games.continue(id)` / `completeMinigame(id, result)`
  - `api.dialogue.generate(context)` / `cacheStats()`
  - Manejo de token JWT: `setToken()`, `getToken()`, `isAuthenticated()`, almacenado en localStorage
  - Renovación automática: si 401 → logout

**Día 1-2: Store de autenticación y juego online:**
- [x] Crear `src/stores/serverStore.js`:
  - Estado: user, token, isLoggedIn
  - Acciones: register(), login(), fetchMe(), logout()
- [x] Modificar `src/stores/gameStore.js` — modo dual online/offline:
  - Nuevo estado: `serverGameId` (null = modo offline, string = modo online)
  - `applyServerState(serverState)`: sincroniza día, fase, recursos, flags, journal, evento actual desde respuesta del servidor
  - `advanceSegment()`: si serverGameId → llama a `advanceSegmentServer()`, si no → lógica local
  - `makeDecision(index)`: si serverGameId → llama a `makeDecisionServer()`, si no → lógica local
  - `continueAfterResult()`: si serverGameId → llama a `continueAfterResultServer()`, si no → lógica local
  - `completeMinigame(result)`: si serverGameId → llama a `completeMinigameServer()`, si no → lógica local
  - Métodos server: `startGameServer()`, `advanceSegmentServer()`, `makeDecisionServer(index)`, `continueAfterResultServer()`, `completeMinigameServer(result)`
  - `reset()`: limpia serverGameId + $reset()

**Día 2: Configuración de Vite:**
- [x] Actualizar `vite.config.js`:
  - Agregar `server.proxy`: `/api` → `http://localhost:3000`
  - Desarrollo local sin CORS, frontend en :5173, backend en :3000

**Día 2-3: Verificación y tests finales:**
- [x] Instalar dependencias backend: `pnpm install` → 148 packages
- [x] Ejecutar tests backend: `pnpm test` → 29/29 pasando
- [x] Ejecutar tests frontend: `pnpm test` → 19/19 pasando
- [x] Verificar estructura de archivos completa
- [x] Verificar que CI/CD detecta ambos proyectos

**Día 3: Entrega final:**
- [x] Revisar DESIGN.md completo y actualizado
- [x] Revisar PLANNING_BACKEND.md con todas las semanas completadas
- [x] Último push a GitHub → CI/CD ejecuta todo el pipeline
- [ ] Verificar que las imágenes quedaron publicadas en DockerHub
- [ ] Enviar entrega final (si aplica)

### Objetivo de la semana:
> Proyecto fullstack robusto. Frontend y backend integrados. 48 tests pasando. Docker Compose funcional. CI/CD completo. Entrega final.

### Lo que se logró completar:
El frontend fue integrado con el backend mediante un cliente HTTP (`src/api/index.js`) que maneja autenticación JWT, renovación de token automática, y todos los endpoints de juego. Se creó `serverStore.js` para gestionar el estado de autenticación (login, registro, logout). El `gameStore.js` fue modificado para soportar modo dual: si hay `serverGameId` usa el servidor como fuente de verdad, si no, funciona completamente offline como antes. Se configuró el proxy de Vite para desarrollo sin CORS. Todos los tests pasan: 29 backend + 19 frontend = 48 tests. La documentación DESIGN.md y PLANNING_BACKEND.md están completas y actualizadas.

### Lo que NO se logró:
—

### Notas:
—

---

## Resumen de Entregables por Semana

| Semana | Qué se sube a GitHub | CI/CD valida |
|--------|---------------------|--------------|
| **1** | `.github/workflows/main.yml`, Dockerfiles, `compose.yml`, modelos Mongoose, `.env.example`, `.gitignore`, DESIGN.md, PLANNING_BACKEND.md | Lint vacío + test vacío (pasa), build Docker |
| **2** | `middleware/auth.js`, `routes/auth.js`, `routes/games.js`, `services/gemini.js`, `routes/dialogue.js`, `src/index.js`, `tests/*.spec.js` | Lint + 29 tests pasando, build Docker |
| **3** | `api/index.js`, `stores/serverStore.js`, `stores/gameStore.js` (mod), `vite.config.js` (mod) | Lint + 48 tests pasando, build + push a DockerHub |

---

## Hitos de entrega

| Fecha | Hito | Estado |
|-------|------|--------|
| 16 jun | Enviar integrantes y repo al profesor | Pendiente envío |
| 22 jun | CI/CD corriendo, Docker listo, modelos definidos | Completado |
| 29 jun | Backend full: auth + juego + Gemini + 29 tests | Completado |
| 2 jul | Frontend integrado, 48 tests, entrega final | Completado |

---

## Resumen de tests

| Suite | Archivos | Tests | Resultado |
|-------|----------|-------|-----------|
| Backend — Auth | auth.spec.js | 5 | Pasando |
| Backend — Game Logic | games.spec.js | 15 | Pasando |
| Backend — Gemini Service | gemini.spec.js | 9 | Pasando |
| Frontend — Game Store | gameStore.spec.js | 19 | Pasando |
| **Total** | **4 archivos** | **48 tests** | **Todos pasando** |

---

## Instrucciones para ejecutar

### Desarrollo local
```bash
# Backend
cd backend
cp .env.example .env   # Configurar GEMINI_API_KEY
pnpm dev

# Frontend (otra terminal)
cd Game
pnpm dev
```

### Con Docker Compose
```bash
# Configurar variables en compose.yml o .env
docker compose up
```

### Tests
```bash
cd backend && pnpm test    # 29 tests
cd Game && pnpm test       # 19 tests
```
