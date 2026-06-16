# GUÍA SEMANA 3 — Integración Frontend y Entrega Final

> **Fechas:** 30 de junio al 2 de julio
> **Objetivo:** Conectar el juego Vue 3 con el backend. Al final de esta semana el juego funciona completamente: haces login en el navegador, juegas una partida que se guarda en MongoDB, y los eventos extra los genera Gemini. Todo dockerizado y con CI/CD pasando. 48 tests en total.

---

## Día 1 — Cliente HTTP para el frontend

Hasta ahora el frontend (`Game/`) funciona completamente offline. Toda la lógica está en `gameStore.js` y los eventos en `events.js`. Vamos a crear una capa que le permita hablar con el backend.

### 1.1 Crear la carpeta `api/`

```powershell
New-Item -ItemType Directory -Path "Game\src\api" -Force
```

### 1.2 Crear `Game/src/api/index.js`

Este archivo es el "puente" entre el frontend y el backend. Centraliza todas las llamadas HTTP en un solo lugar.

```javascript
// URL del backend. En desarrollo usa el proxy de Vite (localhost:5173/api → localhost:3000/api)
// En producción con Docker, usa la variable de entorno
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000/api'

let authToken = null

// ─── MANEJO DEL TOKEN JWT ──────────────────────────

export function setToken(token) {
  authToken = token
  if (token) {
    localStorage.setItem('auth_token', token)
  } else {
    localStorage.removeItem('auth_token')
  }
}

export function getToken() {
  if (!authToken) {
    // Si el token no está en memoria, intentar recuperarlo de localStorage
    authToken = localStorage.getItem('auth_token')
  }
  return authToken
}

export function isAuthenticated() {
  return !!getToken()  // !! convierte a booleano: null → false, "abc" → true
}

// ─── FUNCIÓN GENÉRICA PARA PETICIONES ──────────────

async function request(path, options = {}) {
  const url = `${API_URL}${path}`
  const headers = {
    ...options.headers,
    'Content-Type': 'application/json',
  }

  // Si hay token, agregarlo al header Authorization
  const token = getToken()
  if (token) {
    headers.Authorization = `Bearer ${token}`
  }

  const res = await fetch(url, { ...options, headers })

  // Si el servidor dice que el token expiró, hacer logout
  if (res.status === 401) {
    setToken(null)
    throw new Error('Sesión expirada')
  }

  const data = await res.json()

  // Si la respuesta no es exitosa (4xx, 5xx), lanzar error
  if (!res.ok) {
    throw new Error(data.error || 'Error en la petición')
  }

  return data
}

// ─── API DE AUTENTICACIÓN ──────────────────────────

export const api = {
  auth: {
    async register(username, email, password) {
      const data = await request('/auth/register', {
        method: 'POST',
        body: JSON.stringify({ username, email, password }),
      })
      setToken(data.token)
      return data
    },

    async login(email, password) {
      const data = await request('/auth/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      })
      setToken(data.token)
      return data
    },

    async me() {
      return request('/auth/me')
    },
  },

  // ─── API DE PARTIDAS ─────────────────────────────

  games: {
    async create() {
      return request('/games', { method: 'POST' })
    },

    async list() {
      return request('/games/user')
    },

    async get(id) {
      return request(`/games/${id}`)
    },

    async start(id) {
      return request(`/games/${id}/start`, { method: 'PUT' })
    },

    async advanceSegment(id) {
      return request(`/games/${id}/advance-segment`, { method: 'PUT' })
    },

    async makeDecision(id, decisionIndex) {
      return request(`/games/${id}/decision`, {
        method: 'PUT',
        body: JSON.stringify({ decisionIndex }),
      })
    },

    async continue(id) {
      return request(`/games/${id}/continue`, { method: 'PUT' })
    },

    async completeMinigame(id, result) {
      return request(`/games/${id}/minigame`, {
        method: 'PUT',
        body: JSON.stringify({ result }),
      })
    },
  },

  // ─── API DE DIÁLOGOS IA ──────────────────────────

  dialogue: {
    async generate(context) {
      return request('/dialogue/generate', {
        method: 'POST',
        body: JSON.stringify(context),
      })
    },

    async cacheStats() {
      return request('/dialogue/cache/stats')
    },
  },
}

export function logout() {
  setToken(null)
}
```

**Explicación detallada de cada parte:**

**`localStorage`**
Es un almacenamiento del navegador que persiste aunque cierres la pestaña. Guardamos el token JWT ahí para que el usuario no tenga que hacer login cada vez que recarga la página.

**`fetch(url, options)`**
Es la API nativa del navegador para hacer peticiones HTTP. Recibe la URL y un objeto de configuración con:
- `method`: GET (por defecto), POST, PUT, DELETE
- `headers`: metadatos de la petición (Content-Type, Authorization)
- `body`: datos a enviar (solo en POST/PUT)

**`await res.json()`**
Convierte el cuerpo de la respuesta (que viene como texto) a un objeto JavaScript. Es asíncrono porque el cuerpo puede ser grande.

**Manejo de 401**
Si el servidor responde 401 (no autorizado), significa que el token expiró o es inválido. Limpiamos el token y lanzamos un error para que la UI muestre la pantalla de login.

---

## Día 1-2 — Store de autenticación (Pinia)

### 2.1 Crear `Game/src/stores/serverStore.js`

Este store maneja el estado de autenticación: si el usuario está logueado, sus datos, y las acciones de login/register/logout.

```javascript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { api, setToken, getToken, logout as apiLogout } from '../api/index.js'

// Usamos Composition API (setup stores) en vez de Options API
// porque es más moderno y flexible
export const useServerStore = defineStore('server', () => {
  const user = ref(null)        // Datos del usuario logueado
  const token = ref(getToken()) // Token actual

  // Getter: ¿está logueado?
  const isLoggedIn = computed(() => !!token.value && !!user.value)

  // Acción: registrar nuevo usuario
  async function register(username, email, password) {
    const data = await api.auth.register(username, email, password)
    token.value = data.token
    user.value = data.user
    return data
  }

  // Acción: iniciar sesión
  async function login(email, password) {
    const data = await api.auth.login(email, password)
    token.value = data.token
    user.value = data.user
    return data
  }

  // Acción: al cargar la app, verificar si hay sesión activa
  async function fetchMe() {
    try {
      if (!getToken()) return
      const data = await api.auth.me()
      user.value = data.user
      token.value = getToken()
    } catch {
      // Si falla (token expirado), limpiar
      logout()
    }
  }

  // Acción: cerrar sesión
  function logout() {
    apiLogout()
    token.value = null
    user.value = null
  }

  // Retornar todo lo que el componente necesita
  return { user, token, isLoggedIn, register, login, fetchMe, logout }
})
```

**Conceptos clave:**

`ref()` — Crea una variable reactiva. Cuando su valor cambia, Vue actualiza automáticamente cualquier parte de la UI que la use.

`computed()` — Un valor que se calcula automáticamente a partir de otros valores reactivos. `isLoggedIn` se recalcula cada vez que `token` o `user` cambian.

Setup Store vs Options Store:
```javascript
// Options Store (la que usa gameStore.js)
export const useStore = defineStore('name', {
  state: () => ({ ... }),
  getters: { ... },
  actions: { ... },
})

// Setup Store (la que usamos aquí)
export const useStore = defineStore('name', () => {
  const x = ref(0)
  function increment() { x.value++ }
  return { x, increment }
})
```

Ambos funcionan. Setup Store es más parecido a Composition API y da más control.

---

## Día 2 — Modificar gameStore para modo dual (online/offline)

### 3.1 ¿Qué cambia en gameStore.js?

El `gameStore.js` actual funciona 100% offline. Vamos a agregarle la capacidad de funcionar en modo online cuando exista un `serverGameId`. La idea es:

- **Si `serverGameId` es null** → funciona exactamente igual que antes (offline).
- **Si `serverGameId` tiene un valor** → cada acción (avanzar segmento, decidir, continuar) se envía al servidor y el estado local se sincroniza con la respuesta.

Esto se llama **"modo dual"** y permite que el juego funcione con o sin backend, lo cual es excelente para desarrollo y testing.

### 3.2 Cambios necesarios

**A) Agregar `serverGameId` al estado:**

```javascript
state: () => ({
  // ... todos los campos existentes ...
  serverGameId: null,  // NUEVO: null = offline, string = ID de partida en MongoDB
}),
```

**B) Agregar import del cliente API:**

```javascript
import { api, getToken } from '../api/index.js'
```

**C) Agregar el método `applyServerState()`:**

Este método toma el estado que devuelve el servidor y lo copia al estado local del store.

```javascript
applyServerState(serverState) {
  if (!serverState) return
  this.day = serverState.day ?? this.day
  this.phase = serverState.phase ?? this.phase
  this.food = serverState.food ?? this.food
  this.water = serverState.water ?? this.water
  this.health = serverState.health ?? this.health
  this.morale = serverState.morale ?? this.morale
  this.currentEvent = serverState.currentEvent ?? this.currentEvent
  this.currentSegment = serverState.currentSegment ?? this.currentSegment
  this.decisionResult = serverState.decisionResult ?? this.decisionResult
  this.flags = serverState.flags ?? this.flags
  this.journal = serverState.journal ?? this.journal
  this.gameOverReason = serverState.gameOverReason ?? this.gameOverReason
},
```

**El operador `??` (nullish coalescing):**
`a ?? b` significa: "usa `a` si no es null ni undefined, si no usa `b`". Es más seguro que `||` porque `||` también rechaza 0 y strings vacíos.

Ejemplo:
```javascript
this.food = serverState.food ?? this.food
// Si el servidor devuelve food: 0, usamos 0 (porque 0 no es null)
// Si el servidor no incluye food (undefined), mantenemos el valor actual
```

**D) Agregar métodos server:**

```javascript
async startGameServer() {
  try {
    const data = await api.games.create()
    this.serverGameId = data._id
    this.applyServerState(data)
  } catch (err) {
    console.error('Error al crear partida en servidor:', err)
    // Fallback: si el servidor no responde, iniciar en modo offline
    this.startGame()
  }
},

async advanceSegmentServer() {
  if (!this.serverGameId) return
  try {
    const data = await api.games.advanceSegment(this.serverGameId)
    this.applyServerState(data)
  } catch (err) {
    console.error('Error al avanzar segmento en servidor:', err)
  }
},

async makeDecisionServer(index) {
  if (!this.serverGameId) return
  try {
    const data = await api.games.makeDecision(this.serverGameId, index)
    this.applyServerState(data)
  } catch (err) {
    console.error('Error al tomar decisión en servidor:', err)
  }
},

async continueAfterResultServer() {
  if (!this.serverGameId) return
  try {
    const data = await api.games.continue(this.serverGameId)
    this.applyServerState(data)
  } catch (err) {
    console.error('Error al continuar en servidor:', err)
  }
},

async completeMinigameServer(result) {
  if (!this.serverGameId) return
  try {
    const data = await api.games.completeMinigame(this.serverGameId, result)
    this.applyServerState(data)
  } catch (err) {
    console.error('Error al completar minijuego en servidor:', err)
  }
},
```

**E) Modificar las acciones existentes:**

En cada acción clave, agregar al INICIO una verificación:

```javascript
advanceSegment() {
  if (this.serverGameId) {
    this.advanceSegmentServer()
    return
  }
  // ... lógica offline existente ...
},

makeDecision(decisionIndex) {
  if (this.serverGameId) {
    this.makeDecisionServer(decisionIndex)
    return
  }
  // ... lógica offline existente ...
},

continueAfterResult() {
  if (this.serverGameId) {
    this.continueAfterResultServer()
    return
  }
  // ... lógica offline existente ...
},

completeMinigame(result) {
  if (this.serverGameId) {
    this.completeMinigameServer(result)
    return
  }
  // ... lógica offline existente ...
},
```

**F) Modificar `reset()`:**

```javascript
reset() {
  this.serverGameId = null
  this.$reset()
},
```

### 3.3 ¿Por qué este diseño?

**Ventajas del modo dual:**
1. **Desarrollo:** Puedes trabajar en el frontend sin tener el backend corriendo.
2. **Testing:** Los tests existentes siguen funcionando sin cambios (no necesitan servidor).
3. **Degradación elegante:** Si el servidor se cae, el juego sigue funcionando offline.
4. **Transición suave:** No hay que reescribir todo el frontend de una vez.

---

## Día 2 — Configurar el proxy de Vite

### 4.1 ¿Por qué necesito un proxy?

En desarrollo, el frontend corre en `localhost:5173` y el backend en `localhost:3000`. Si el frontend intenta hacer `fetch('http://localhost:3000/api/games')`, el navegador lo bloquea por **CORS** (Cross-Origin Resource Sharing).

Aunque tenemos `cors()` en el backend, durante desarrollo es más limpio usar un proxy: Vite redirige las peticiones a `/api/*` hacia el backend, evitando problemas de CORS y simplificando las URLs.

### 4.2 Actualizar `Game/vite.config.js`

Busca el archivo existente y agrega la sección `server.proxy`:

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],

  // NUEVO: proxy para desarrollo
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },

  test: {
    environment: 'jsdom',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      include: ['src/**/*.{js,vue}'],
      exclude: ['src/tests/**', 'src/main.js'],
    },
  },
})
```

**¿Cómo funciona?**

1. El frontend hace `fetch('/api/games')` (sin dominio, URL relativa).
2. Vite ve que empieza con `/api` y lo redirige a `http://localhost:3000/api/games`.
3. El backend responde normalmente.
4. Vite devuelve la respuesta al frontend.

El navegador cree que todo viene del mismo origen (localhost:5173), así que no hay problema de CORS.

**En producción (Docker):**
La variable `VITE_API_URL` apunta directamente al backend, y el proxy de Vite no se usa porque Vite solo sirve archivos estáticos (la app ya fue "buildeada").

---

## Día 2-3 — Verificación y tests finales

### 5.1 Instalar dependencias

Si no lo has hecho ya:

```powershell
# Backend
cd backend
pnpm install

# Frontend
cd Game
pnpm install
```

### 5.2 Ejecutar tests del backend

```powershell
cd backend
pnpm test
```

Debes ver:
```
✓ src/tests/auth.spec.js      (5 tests)
✓ src/tests/games.spec.js    (15 tests)
✓ src/tests/gemini.spec.js    (9 tests)

Test Files  3 passed (3)
     Tests  29 passed (29)
```

### 5.3 Ejecutar tests del frontend

```powershell
cd Game
pnpm test
```

Debes ver:
```
✓ src/tests/gameStore.spec.js (19 tests)

Test Files  1 passed (1)
     Tests  19 passed (19)
```

**Total combinado: 48 tests pasando.**

### 5.4 Probar el flujo completo manualmente

**Paso 1: Levantar el backend**

```powershell
cd backend
# Asegúrate de tener MongoDB corriendo
pnpm dev
```

**Paso 2: Levantar el frontend**

En otra terminal:
```powershell
cd Game
pnpm dev
```

**Paso 3: Abrir el navegador**

Ve a `http://localhost:5173`. El juego debería funcionar exactamente igual que antes.

**Paso 4: Jugar en modo online**

Para probar el modo online, necesitas que el frontend cree la partida en el servidor. Esto requiere que el componente `App.vue` llame a `game.startGameServer()` en vez de `game.startGame()`. (Este cambio en App.vue es opcional para esta semana — el modo offline sigue funcionando y es suficiente para la entrega.)

---

## Día 3 — Docker Compose y entrega final

### 6.1 Verificar Docker Compose

```powershell
# Desde la raíz del proyecto (Solemne2/)
docker compose up --build
```

Esto debería:
1. Bajar la imagen `mongo:7` (si no la tienes)
2. Construir la imagen del backend desde `backend/Dockerfile`
3. Construir la imagen del frontend desde `Game/Dockerfile`
4. Levantar los 3 contenedores

Para verificar que todo está corriendo:

```powershell
# Ver contenedores activos
docker ps

# Ver logs de un servicio
docker compose logs backend
docker compose logs frontend

# Health check del backend
curl http://localhost:3000/api/health
```

### 6.2 Verificar CI/CD en GitHub

1. Haz commit de todos los cambios:
```powershell
git add .
git commit -m "Semana 3: integración frontend-backend, 48 tests, Docker Compose"
git push
```

2. Ve a tu repo en GitHub → pestaña **Actions**
3. Deberías ver el workflow ejecutándose con 3 jobs
4. Espera a que todos estén verdes

Si algún job falla, haz clic en él para ver los logs y encontrar el error.

### 6.3 Revisar documentación

Antes de la entrega, verifica:

**DESIGN.md:**
- [x] Diagrama de arquitectura
- [x] Stack tecnológico completo
- [x] Mejoras respecto a Solemne 2
- [x] Servicio externo documentado (Gemini)
- [x] Modelo de datos
- [x] Endpoints REST
- [x] Estructura de carpetas actualizada
- [x] Flujo de eventos múltiples

**PLANNING_BACKEND.md:**
- [x] Tareas de las 3 semanas completadas
- [x] Objetivos, logros, notas por semana
- [x] Resumen de tests
- [x] Instrucciones de ejecución

**README.md** (en `Game/`):
- [x] Título y descripción
- [x] Instrucciones de instalación local
- [x] Instrucciones con Docker Compose
- [ ] Links a DockerHub (requiere haber hecho push de imágenes)

### 6.4 Entrega final

Según la rúbrica (solemne_3.pdf):

> "Se evaluará la última versión disponible en el repositorio de GitHub al cierre del jueves 2 de julio de 2026. No es necesario enviar un correo adicional para la entrega final."

Asegúrate de que:
- [ ] El último commit en GitHub es antes de la fecha límite
- [ ] CI/CD pasó completo (lint + test + build + push)
- [ ] Las imágenes están en DockerHub
- [ ] DESIGN.md, PLANNING_BACKEND.md y README.md están actualizados

---

## Resumen de archivos creados esta semana

| Archivo | Propósito |
|---------|-----------|
| `Game/src/api/index.js` | Cliente HTTP unificado para backend |
| `Game/src/stores/serverStore.js` | Store de autenticación (login/register/logout) |
| `Game/src/stores/gameStore.js` | Modificado para modo dual online/offline |
| `Game/vite.config.js` | Modificado con proxy a backend |

---

## Checklist final de la Semana 3

- [ ] `src/api/index.js` creado con todos los métodos
- [ ] `src/stores/serverStore.js` creado
- [ ] `src/stores/gameStore.js` modificado con serverGameId y modo dual
- [ ] `vite.config.js` con proxy configurado
- [ ] Tests backend: 29/29 pasando
- [ ] Tests frontend: 19/19 pasando
- [ ] Docker Compose levanta los 3 servicios
- [ ] CI/CD verde en GitHub Actions
- [ ] Documentación completa y actualizada
- [ ] Último push antes de la fecha límite

---

## Troubleshooting

### Error: "MongoDB conectado" pero después crashea

**Causa:** MongoDB no está corriendo.
**Solución:** Instala MongoDB localmente o usa Docker:
```powershell
docker run -d -p 27017:27017 --name mongodb mongo:7
```

### Error: `ECONNREFUSED :::3000`

**Causa:** El frontend intenta llamar al backend pero no está corriendo.
**Solución:** Asegúrate de que el backend está corriendo en otra terminal con `pnpm dev`.

### Error: CORS bloqueando las peticiones

**Causa:** El proxy de Vite no está funcionando.
**Solución:** Revisa que `vite.config.js` tenga la sección `server.proxy` correctamente. Si estás en producción (Docker), asegúrate de que `VITE_API_URL` apunte al backend correcto.

### Error: `GEMINI_API_KEY` no configurada

**Causa:** Falta la variable de entorno.
**Solución:** Crea el archivo `backend/.env` copiando `.env.example` y agrega tu API key real de Google AI Studio. Sin esto, el backend usará el fallback (eventos genéricos predefinidos).

### Tests fallan con `Cannot find module`

**Causa:** No instalaste las dependencias.
**Solución:**
```powershell
cd backend && pnpm install
cd Game && pnpm install
```

### Docker Compose: el backend no puede conectar a MongoDB

**Causa:** La URI de MongoDB usa `localhost` en vez del nombre del servicio.
**Solución:** En `compose.yml`, `MONGODB_URI` debe ser `mongodb://mongodb:27017/15dias` (no `localhost`). Dentro de Docker, cada contenedor tiene su propio localhost y no se ven entre sí por IP — se comunican por el nombre del servicio.
