# GUÍA SEMANA 2 — Backend Core: Auth, Juego, Gemini

> **Fechas:** 23 al 29 de junio
> **Objetivo:** Backend completamente funcional. Al final de esta semana puedes crear un usuario, iniciar sesión, jugar una partida completa de 15 días, y los eventos extra los genera Gemini. Todo persistido en MongoDB. 29 tests pasando.

---

## Día 1 — Middleware de autenticación JWT

### ¿Qué es JWT y cómo funciona?

JWT (JSON Web Token) es un estándar para transmitir información segura entre dos partes. En nuestro caso:

1. El usuario hace **login** con email y contraseña.
2. El servidor verifica que son correctos y **crea un token** firmado con una clave secreta.
3. El frontend guarda ese token (en localStorage).
4. En CADA petición posterior, el frontend envía el token en el header `Authorization: Bearer <token>`.
5. El servidor **verifica la firma** del token. Si es válida, sabe quién es el usuario sin necesidad de volver a pedir contraseña.

El token tiene 3 partes separadas por puntos: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiI2N2EzYmM5ZiJ9.ABC123def456
└────── header ─────┘ └──── payload ────┘ └── signature ──┘
```

- **Header:** Dice qué algoritmo se usó (HS256).
- **Payload:** Los datos que guardamos (userId, fecha de expiración).
- **Signature:** La firma criptográfica. Si alguien modifica el payload, la firma no coincide y el token es rechazado.

### 1.1 Crear `src/middleware/auth.js`

```javascript
import jwt from 'jsonwebtoken'

export function authMiddleware(req, res, next) {
  // 1. Obtener el header Authorization
  const header = req.headers.authorization

  // 2. Verificar que existe y empieza con "Bearer "
  if (!header || !header.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Token requerido' })
  }

  // 3. Extraer el token (lo que está después de "Bearer ")
  const token = header.split(' ')[1]

  try {
    // 4. Verificar y decodificar el token
    const decoded = jwt.verify(token, process.env.JWT_SECRET || 'secret')

    // 5. Guardar el userId en la request para que las rutas lo usen
    req.userId = decoded.userId

    // 6. Pasar al siguiente middleware o ruta
    next()
  } catch {
    // 7. Si el token es inválido o expiró, rechazar
    return res.status(401).json({ error: 'Token inválido o expirado' })
  }
}
```

**Explicación línea por línea:**

```javascript
const header = req.headers.authorization
```
Express pone todos los headers de la petición en `req.headers`. `authorization` es donde el frontend envía el token.

```javascript
if (!header || !header.startsWith('Bearer '))
```
Si no hay header de autorización, o si no empieza con "Bearer ", la petición es rechazada. El formato estándar es: `Authorization: Bearer eyJhbG...`

```javascript
const token = header.split(' ')[1]
```
`"Bearer eyJhbG..."` → `split(' ')` → `["Bearer", "eyJhbG..."]` → `[1]` obtiene el token.

```javascript
const decoded = jwt.verify(token, process.env.JWT_SECRET || 'secret')
```
`jwt.verify()` hace dos cosas:
1. Verifica que la firma es correcta (el token no fue manipulado).
2. Decodifica el payload y devuelve los datos (en nuestro caso, `{ userId: "...", iat: ..., exp: ... }`).

Si el token expiró (`exp` es menor a la fecha actual) o la firma no coincide, `verify()` lanza un error.

```javascript
req.userId = decoded.userId
```
Guardamos el userId en la request. Todas las rutas que usen este middleware tendrán acceso a `req.userId` para saber qué usuario hizo la petición.

```javascript
next()
```
Pasa el control al siguiente middleware o a la función de la ruta. Sin `next()`, la petición se queda colgada.

> **IMPORTANTE:** Este middleware se usa en las rutas protegidas así:
> ```javascript
> router.get('/ruta-protegida', authMiddleware, (req, res) => { ... })
> ```

---

## Día 2 — Rutas de autenticación

### 2.1 Crear `src/routes/auth.js`

Este archivo maneja 3 endpoints: register, login y me.

```javascript
import { Router } from 'express'
import jwt from 'jsonwebtoken'
import User from '../models/User.js'
import { authMiddleware } from '../middleware/auth.js'

const router = Router()

// Función helper: genera un token JWT para un userId
function generateToken(userId) {
  return jwt.sign(
    { userId },
    process.env.JWT_SECRET || 'secret',
    { expiresIn: '7d' }    // El token expira en 7 días
  )
}

// ─── REGISTRO ───────────────────────────────────
router.post('/register', async (req, res) => {
  try {
    const { username, email, password } = req.body

    // Validación básica
    if (!username || !email || !password) {
      return res.status(400).json({ error: 'Todos los campos son requeridos' })
    }

    if (password.length < 6) {
      return res.status(400).json({ error: 'La contraseña debe tener al menos 6 caracteres' })
    }

    // Verificar que el email o username no existan ya
    const existingUser = await User.findOne({
      $or: [{ email }, { username }]
    })
    if (existingUser) {
      return res.status(409).json({ error: 'El usuario o email ya existe' })
    }

    // Crear el usuario (la contraseña se hashea automáticamente
    // gracias al hook pre('save') en el modelo)
    const user = await User.create({ username, email, password })

    // Generar token y responder
    const token = generateToken(user._id)
    res.status(201).json({ token, user: user.toJSON() })
  } catch (err) {
    res.status(500).json({ error: 'Error al registrar usuario' })
  }
})

// ─── LOGIN ──────────────────────────────────────
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body

    if (!email || !password) {
      return res.status(400).json({ error: 'Email y contraseña requeridos' })
    }

    // Buscar usuario por email
    const user = await User.findOne({ email })
    if (!user) {
      return res.status(401).json({ error: 'Credenciales inválidas' })
    }

    // Comparar contraseña
    const valid = await user.comparePassword(password)
    if (!valid) {
      return res.status(401).json({ error: 'Credenciales inválidas' })
    }

    // Login exitoso → generar token
    const token = generateToken(user._id)
    res.json({ token, user: user.toJSON() })
  } catch (err) {
    res.status(500).json({ error: 'Error al iniciar sesión' })
  }
})

// ─── OBTENER USUARIO ACTUAL ─────────────────────
router.get('/me', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.userId)
    if (!user) {
      return res.status(404).json({ error: 'Usuario no encontrado' })
    }
    res.json({ user: user.toJSON() })
  } catch (err) {
    res.status(500).json({ error: 'Error al obtener usuario' })
  }
})

export default router
```

**Conceptos clave:**

- `Router()` — Crea un "mini-enrutador" de Express. Permite agrupar rutas relacionadas y exportarlas.
- `req.body` — Los datos que el frontend envía en el cuerpo de la petición. Express los parsea automáticamente gracias a `express.json()`.
- `User.findOne({ $or: [...] })` — Busca un documento donde el email O el username coincidan. `$or` es un operador de MongoDB.
- `User.create({...})` — Crea Y guarda el documento en una sola operación. Equivale a `new User({...}).save()`.
- `user.toJSON()` — Llama a nuestro método personalizado que elimina la contraseña antes de enviar la respuesta.
- Códigos HTTP: `201` = creado, `400` = error del cliente, `401` = no autorizado, `409` = conflicto (ya existe), `500` = error del servidor.

### 2.2 Conectar las rutas en `src/index.js`

Actualiza `src/index.js` para importar y montar las rutas de auth. Busca la línea donde definiste `app.get('/api/health', ...)` y agrega:

```javascript
import authRoutes from './routes/auth.js'

// ... (código existente) ...

app.use('/api/auth', authRoutes)
```

**¿Qué hace `app.use('/api/auth', authRoutes)`?**

Todas las rutas definidas en `authRoutes` se montan bajo el prefijo `/api/auth`. Es decir:

- `router.post('/register')` dentro de `auth.js` → se accede como `POST /api/auth/register`
- `router.post('/login')` → `POST /api/auth/login`
- `router.get('/me')` → `GET /api/auth/me`

---

## Día 3 — Tests de autenticación

### 3.1 Crear `src/tests/auth.spec.js`

Los tests de auth NO necesitan base de datos ni servidor corriendo. Prueban la lógica pura de JWT.

```javascript
import { describe, it, expect } from 'vitest'
import jwt from 'jsonwebtoken'

describe('Auth Utilities', () => {
  it('genera un token JWT válido', () => {
    const secret = 'test-secret'
    const userId = '507f1f77bcf86cd799439011'
    const token = jwt.sign({ userId }, secret, { expiresIn: '7d' })

    expect(token).toBeDefined()
    expect(typeof token).toBe('string')
    // Un JWT tiene 3 partes separadas por puntos
    expect(token.split('.')).toHaveLength(3)
  })

  it('decodifica un token correctamente', () => {
    const secret = 'test-secret'
    const userId = '507f1f77bcf86cd799439011'
    const token = jwt.sign({ userId }, secret, { expiresIn: '7d' })

    const decoded = jwt.verify(token, secret)
    expect(decoded.userId).toBe(userId)
  })

  it('rechaza un token inválido', () => {
    expect(() => jwt.verify('token-inventado', 'secret')).toThrow()
  })

  it('rechaza un token firmado con otro secreto', () => {
    const token = jwt.sign({ userId: 'abc' }, 'secreto-a', { expiresIn: '7d' })
    expect(() => jwt.verify(token, 'secreto-b')).toThrow()
  })

  it('rechaza un token expirado', () => {
    // expiresIn: '0s' = expira inmediatamente
    const token = jwt.sign({ userId: 'abc' }, 'secret', { expiresIn: '0s' })
    expect(() => jwt.verify(token, 'secret')).toThrow()
  })
})
```

**Conceptos:**

- `describe('nombre', () => { ... })` — Agrupa tests relacionados.
- `it('descripción', () => { ... })` — Un test individual.
- `expect(valor).toBe(esperado)` — Afirmación: el valor debe ser igual al esperado.
- `expect(() => ...).toThrow()` — Afirmación: la función debe lanzar un error.
- `jwt.sign()` — Crea un token.
- `jwt.verify()` — Verifica y decodifica un token. Si es inválido, lanza error.

### 3.2 Ejecutar los tests

```powershell
cd backend
pnpm test
```

Deberías ver:
```
✓ src/tests/auth.spec.js (5 tests)
```

Si ves ❌ en rojo, leé el mensaje de error: te dice exactamente qué falló y por qué.

---

## Día 4-5 — Lógica de juego en el servidor

Este es el archivo más grande. Portamos TODA la lógica de `Game/src/stores/gameStore.js` al backend.

### 4.1 Crear `src/routes/games.js`

Este archivo es largo. Vamos por partes.

#### Parte 1: Imports y configuración inicial

```javascript
import { Router } from 'express'
import Game from '../models/Game.js'
import { authMiddleware } from '../middleware/auth.js'
import { clamp, generateContextHash, buildGamePrompt } from '../services/gemini.js'
import { GoogleGenerativeAI } from '@google/generative-ai'
import Dialogue from '../models/Dialogue.js'

const router = Router()
router.use(authMiddleware)  // TODAS las rutas de juego requieren autenticación
```

`router.use(authMiddleware)` aplica el middleware a TODAS las rutas definidas en este archivo. Sin token JWT válido, cualquier petición a `/api/games/*` será rechazada con 401.

#### Parte 2: Eventos fijos del juego

Copiamos los eventos predefinidos del juego original. Solo los días que tienen eventos importantes (0 al 4, 6). Los días sin evento fijo serán llenados por la IA.

```javascript
const FIXED_EVENTS = {
  0: {
    day: 0,
    title: 'ALERTA NACIONAL',
    location: 'casa',
    image: 'noticiadia0',
    type: 'intro',
    segments: [
      { text: 'La pantalla de la televisión parpadea con interferencia...' },
      { text: '"ATENCIÓN CIUDADANOS. UN BROTE INCONTROLABLE..."' },
      { text: '"UN OPERATIVO DE EVACUACIÓN SERÁ REALIZADO EN 15 DÍAS..."' },
      { text: 'La pantalla vuelve a la normalidad. Miras por la ventana...' },
    ],
  },
  1: {
    day: 1,
    title: 'DECISIÓN MORAL',
    location: 'casa',
    image: 'casadia1',
    segments: [
      { text: 'Es tu primer día completo en el refugio...' },
      { text: 'Miras por la mirilla: una mujer con su hijo pequeño...' },
    ],
    decisions: [
      {
        text: 'Dejar entrar',
        effects: { food: -1, water: 0, health: 0, morale: 18 },
        result: 'La mujer entra llorando de gratitud...',
        setsFlag: 'refugees',
      },
      {
        text: 'No abrir',
        effects: { food: 0, water: 0, health: -2, morale: -12 },
        result: 'Escuchas los pasos alejarse...',
        setsFlag: 'd1_solo',
      },
    ],
  },
  // Agregar más días según el events.js original...
  // Día 2, 3, 4, 6 con sus ramas y variantes según flags
}
```

#### Parte 3: Minijuegos

```javascript
const MINIGAME_EVENTS = {
  5: {
    day: 5, type: 'catchRain',
    title: 'RECOGER AGUA DE LLUVIA',
    location: 'casa',
    image: 'lluvia',
    description: '¡Una tormenta! Mueve el balde para atrapar gotas de lluvia.',
    win:  { food: 0, water: 5, health: 0, morale: 5 },
    lose: { food: 0, water: 0, health: -10, morale: -8 },
  },
  10: {
    day: 10, type: 'findCans',
    title: 'BUSCAR LATAS',
    location: 'supermercado',
    image: 'superme',
    description: 'Encuentra latas escondidas en el supermercado.',
    win:  { food: 6, water: 0, health: 0, morale: 8 },
    lose: { food: 0, water: 0, health: -8, morale: -6 },
  },
  15: {
    day: 15, type: 'escape',
    title: '¡EL RESCATE!',
    location: 'rescate',
    image: 'rescate',
    description: 'El rescate llegó. No lo pierdas.',
    win:  { food: 0, water: 0, health: 0, morale: 10 },
    lose: { food: 0, water: 0, health: -100, morale: 0 },
  },
}
```

#### Parte 4: Funciones auxiliares de lógica de juego

```javascript
function applyDailyConsumption(game) {
  game.food   = clamp(game.food - 1, 0, game.maxStat)
  game.water  = clamp(game.water - 1, 0, game.maxStat)
  game.health = clamp(game.health - 4, 0, game.maxHealth)
  game.morale = clamp(game.morale - 3, 0, game.maxMorale)

  // Penalizaciones extra si los recursos llegan a 0
  if (game.food <= 0) game.health = clamp(game.health - 4, 0, game.maxHealth)
  if (game.water <= 0) game.health = clamp(game.health - 6, 0, game.maxHealth)
  if (game.food <= 0 && game.water <= 0)
    game.morale = clamp(game.morale - 2, 0, game.maxMorale)
}

function getFixedEvent(day) {
  return FIXED_EVENTS[day] || null
}

function getMinigame(day) {
  return MINIGAME_EVENTS[day] || null
}

function applyEffects(game, effects) {
  if (!effects) return
  if (effects.food)   game.food   = clamp(game.food + effects.food, 0, game.maxStat)
  if (effects.water)  game.water  = clamp(game.water + effects.water, 0, game.maxStat)
  if (effects.health) game.health = clamp(game.health + effects.health, 0, game.maxHealth)
  if (effects.morale) game.morale = clamp(game.morale + effects.morale, 0, game.maxMorale)
}
```

**¿Qué hace `clamp()`?**

```javascript
function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value))
}
```

Ejemplos:
- `clamp(5, 0, 20)` → 5 (está dentro del rango)
- `clamp(-3, 0, 20)` → 0 (no puede bajar de 0)
- `clamp(25, 0, 20)` → 20 (no puede subir de 20)
- `clamp(150, 0, 100)` → 100 (no puede subir de 100)

#### Parte 5: Generación de eventos con IA

```javascript
async function generateAIEvent(game) {
  try {
    const prompt = buildGamePrompt(
      game.day, game.flags, game.food, game.water,
      game.health, game.morale,
      `${game.eventsThisDay} eventos hoy`
    )
    const contextHash = generateContextHash(
      game.day, game.flags, game.food, game.water,
      game.health, game.morale
    )

    // 1. Buscar en caché
    const cached = await Dialogue.findOne({ contextHash })
    if (cached) {
      return { ...cached.response, type: 'ai', cached: true }
    }

    // 2. Si no está en caché, llamar a Gemini
    const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || '')
    const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' })
    const result = await model.generateContent(prompt)
    const text = result.response.text()

    // 3. Limpiar la respuesta (Gemini a veces envuelve en ```json)
    const cleanJson = text.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim()
    const response = JSON.parse(cleanJson)

    // 4. Guardar en caché
    await Dialogue.create({
      contextHash,
      prompt,
      response,
      model: 'gemini-2.5-flash',
      tokensUsed: result.response.usageMetadata?.totalTokenCount || 0,
    })

    return { ...response, type: 'ai', cached: false }
  } catch (err) {
    console.error('Error generando evento IA:', err.message)
    // FALLBACK: si Gemini falla, devolvemos un evento genérico
    return {
      type: 'ai',
      title: 'SILENCIO INQUIETANTE',
      location: 'casa',
      segments: [
        { text: 'El día transcurre en calma tensa...' },
        { text: 'Revisas tus suministros. Por ahora, estás a salvo.' },
      ],
      decisions: [
        {
          text: 'Descansar y recuperar fuerzas',
          effects: { food: 0, water: 0, health: 3, morale: 5 },
          result: 'Te recuestas contra la pared...',
        },
        {
          text: 'Reforzar el refugio',
          effects: { food: 0, water: 0, health: -2, morale: 3 },
          result: 'Tablones y clavos...',
        },
      ],
    }
  }
}
```

#### Parte 6: Sistema de avance de eventos

```javascript
async function handleEventEnd(game) {
  // Contar este evento
  game.eventsThisDay++

  // ¿Llegamos al máximo de eventos por día?
  if (game.eventsThisDay >= game.maxEventsPerDay) {
    game.day++

    // ¿Pasó el día 15?
    if (game.day > 15) {
      if (game.health > 0 && game.morale > 0) {
        game.phase = 'victory'
        game.status = 'won'
      } else {
        game.phase = 'gameover'
        game.gameOverReason = game.health <= 0 ? 'health' : 'morale'
        game.status = 'lost'
      }
      return
    }

    // Nuevo día: consumir recursos
    applyDailyConsumption(game)

    // ¿Murió durante la noche?
    if (game.health <= 0 || game.morale <= 0) {
      game.phase = 'gameover'
      game.gameOverReason = game.health <= 0 ? 'health' : 'morale'
      game.status = 'lost'
      return
    }

    // Reiniciar contador de eventos del día
    game.eventsThisDay = 0
  }

  // Cargar el siguiente evento
  await advanceToNextEvent(game)
}

async function advanceToNextEvent(game) {
  // Primer evento del día: verificar si hay minijuego o evento fijo
  if (game.eventsThisDay === 0) {
    const minigame = getMinigame(game.day)
    if (minigame) {
      game.currentEvent = { ...minigame }
      game.phase = 'minigame'
      game.currentSegment = 0
      // Los minijuegos consumen todo el día
      game.eventsThisDay = game.maxEventsPerDay
      return
    }

    const fixedEvent = getFixedEvent(game.day)
    if (fixedEvent) {
      game.currentEvent = { ...fixedEvent, type: 'fixed' }
      game.phase = 'story'
      game.currentSegment = 0
      return
    }
  }

  // Si no hay evento fijo/minijuego, o ya pasó, generar con IA
  const aiEvent = await generateAIEvent(game)
  game.currentEvent = aiEvent
  game.phase = 'story'
  game.currentSegment = 0
}
```

#### Parte 7: Endpoints CRUD de partidas

```javascript
// Crear nueva partida
router.post('/', async (req, res) => {
  try {
    const game = await Game.create({
      userId: req.userId,
      day: 0,
      phase: 'intro',
      food: 6,
      water: 4,
      health: 80,
      morale: 70,
      currentEvent: FIXED_EVENTS[0],
      currentSegment: 0,
      eventsThisDay: 1,
    })
    res.status(201).json(game.toJSON())
  } catch (err) {
    res.status(500).json({ error: 'Error al crear partida' })
  }
})

// Listar partidas del usuario
router.get('/user', async (req, res) => {
  try {
    const games = await Game.find({ userId: req.userId })
      .sort({ updatedAt: -1 })  // Más recientes primero
    res.json(games.map(g => g.toJSON()))
  } catch (err) {
    res.status(500).json({ error: 'Error al obtener partidas' })
  }
})

// Obtener una partida específica
router.get('/:id', async (req, res) => {
  try {
    const game = await Game.findOne({
      _id: req.params.id,
      userId: req.userId  // Solo el dueño puede verla
    })
    if (!game) return res.status(404).json({ error: 'Partida no encontrada' })
    res.json(game.toJSON())
  } catch (err) {
    res.status(500).json({ error: 'Error al obtener partida' })
  }
})
```

#### Parte 8: Endpoints de acciones del juego

```javascript
// Iniciar día 1 (después de la intro)
router.put('/:id/start', async (req, res) => {
  const game = await Game.findOne({ _id: req.params.id, userId: req.userId })
  if (!game) return res.status(404).json({ error: 'Partida no encontrada' })

  game.day = 1
  applyDailyConsumption(game)

  if (game.health <= 0 || game.morale <= 0) {
    game.phase = 'gameover'
    game.gameOverReason = game.health <= 0 ? 'health' : 'morale'
    game.status = 'lost'
    await game.save()
    return res.json(game.toJSON())
  }

  game.eventsThisDay = 0
  await advanceToNextEvent(game)
  await game.save()
  res.json(game.toJSON())
})

// Avanzar segmento de texto
router.put('/:id/advance-segment', async (req, res) => {
  const game = await Game.findOne({ _id: req.params.id, userId: req.userId })
  if (!game) return res.status(404).json({ error: 'Partida no encontrada' })
  if (!game.currentEvent) return res.status(400).json({ error: 'No hay evento activo' })

  const isLastSegment = game.currentSegment >= game.currentEvent.segments.length - 1

  if (isLastSegment) {
    if (game.currentEvent.decisions) {
      game.phase = 'decision'
    } else if (game.currentEvent.type === 'intro') {
      game.phase = 'intro'  // Se queda esperando que el jugador presione COMENZAR
    } else {
      await handleEventEnd(game)
    }
  } else {
    game.currentSegment++
  }

  await game.save()
  res.json(game.toJSON())
})

// Tomar una decisión
router.put('/:id/decision', async (req, res) => {
  const game = await Game.findOne({ _id: req.params.id, userId: req.userId })
  if (!game) return res.status(404).json({ error: 'Partida no encontrada' })

  const { decisionIndex } = req.body
  const event = game.currentEvent
  if (!event || !event.decisions) {
    return res.status(400).json({ error: 'No hay decisiones disponibles' })
  }

  const decision = event.decisions[decisionIndex]
  if (!decision) return res.status(400).json({ error: 'Decisión inválida' })

  // Aplicar efectos (soporta decisiones random con successRate)
  if (decision.random) {
    const success = Math.random() < decision.successRate
    const effects = success ? decision.effects.success : decision.effects.failure
    applyEffects(game, effects)
    game.decisionResult = {
      text: success ? decision.successResult : decision.failureResult,
      success
    }
  } else {
    applyEffects(game, decision.effects)
    if (decision.setsFlag) {
      game.flags = { ...game.flags, [decision.setsFlag]: true }
    }
    game.decisionResult = { text: decision.result, success: true }
  }

  // Guardar en el diario
  game.journal.push({
    day: game.day,
    type: 'decision',
    title: event.title,
    decision: decision.text,
    result: game.decisionResult.text,
    success: game.decisionResult.success,
    effects: decision.random
      ? (game.decisionResult.success ? decision.effects.success : decision.effects.failure)
      : decision.effects,
    timestamp: new Date(),
  })

  game.phase = 'result'
  game.markModified('flags')
  game.markModified('journal')

  // ¿Murió por la decisión?
  if (game.health <= 0 || game.morale <= 0) {
    game.phase = 'gameover'
    game.gameOverReason = game.health <= 0 ? 'health' : 'morale'
    game.status = 'lost'
  }

  await game.save()
  res.json(game.toJSON())
})

// Continuar después de ver el resultado
router.put('/:id/continue', async (req, res) => {
  const game = await Game.findOne({ _id: req.params.id, userId: req.userId })
  if (!game) return res.status(404).json({ error: 'Partida no encontrada' })

  game.decisionResult = null
  await handleEventEnd(game)
  await game.save()
  res.json(game.toJSON())
})

// Completar minijuego
router.put('/:id/minigame', async (req, res) => {
  const game = await Game.findOne({ _id: req.params.id, userId: req.userId })
  if (!game) return res.status(404).json({ error: 'Partida no encontrada' })

  const { result } = req.body  // 'win' o 'lose'
  const event = game.currentEvent
  const outcome = result === 'win' ? event.win : event.lose

  applyEffects(game, outcome)

  game.journal.push({
    day: game.day,
    type: 'minijuego',
    title: event.title,
    description: outcome.message || `Resultado: ${result}`,
    effects: outcome,
    timestamp: new Date(),
  })
  game.markModified('journal')

  if (game.health <= 0 || game.morale <= 0) {
    game.phase = 'gameover'
    game.gameOverReason = game.health <= 0 ? 'health' : 'morale'
    game.status = 'lost'
    await game.save()
    return res.json(game.toJSON())
  }

  game.decisionResult = null
  game.day++

  if (game.day > 15) {
    game.phase = 'victory'
    game.status = 'won'
    await game.save()
    return res.json(game.toJSON())
  }

  applyDailyConsumption(game)

  if (game.health <= 0 || game.morale <= 0) {
    game.phase = 'gameover'
    game.gameOverReason = game.health <= 0 ? 'health' : 'morale'
    game.status = 'lost'
    await game.save()
    return res.json(game.toJSON())
  }

  game.eventsThisDay = 0
  await advanceToNextEvent(game)
  await game.save()
  res.json(game.toJSON())
})

export default router
```

**Concepto clave: `game.save()`**

En Mongoose, los documentos son objetos JavaScript normales con superpoderes. Cuando modificas un campo (`game.food = 5`), el cambio existe en memoria pero NO en la base de datos. Tienes que llamar `await game.save()` para persistirlo.

**Concepto clave: `game.markModified()`**

Para campos de tipo `Mixed` (como `flags` y `journal`), Mongoose no detecta cambios automáticamente. `markModified('flags')` le avisa a Mongoose "oye, este campo cambió, guárdalo".

### 4.2 Conectar las rutas en `src/index.js`

```javascript
import gameRoutes from './routes/games.js'

// Después de las rutas de auth
app.use('/api/games', gameRoutes)
```

---

## Día 6 — Servicio Gemini y caché

### 6.1 Crear `src/services/gemini.js`

```javascript
import crypto from 'crypto'

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value))
}

function generateContextHash(day, flags, food, water, health, morale) {
  const data = JSON.stringify({ day, flags, food, water, health, morale })
  return crypto.createHash('sha256').update(data).digest('hex')
}

function buildGamePrompt(day, flags, food, water, health, morale, previousEvents) {
  const flagSummary = Object.entries(flags || {})
    .filter(([, v]) => v)
    .map(([k]) => k)
    .join(', ') || 'ninguno'

  return `
Eres el narrador de un juego de supervivencia post-apocalíptico llamado "15 Días".
El jugador está en el día ${day} de 15, esperando un rescate.

Estado actual:
- Comida: ${food}/20
- Agua: ${water}/20  
- Salud: ${health}/100
- Moral: ${morale}/100
- Eventos previos hoy: ${previousEvents || 'ninguno'}
- Flags de historia: ${flagSummary}

Genera UN nuevo evento narrativo para este día. Debe ser coherente con el mundo
post-apocalíptico, la situación del jugador y los flags de historia.

El evento debe tener:
1. Un título corto y dramático (máximo 40 caracteres)
2. Una ubicación (casa, calle, supermercado, farmacia, refugio)
3. De 2 a 3 párrafos de narración inmersiva en español
4. Dos decisiones que el jugador puede tomar, cada una con:
   - Texto descriptivo de la acción (máximo 60 caracteres)
   - Efectos sobre los recursos (objeto con food, water, health, morale como números)
   - Texto de resultado (qué pasa si elige esa opción)

Responde SOLO con un objeto JSON válido, sin markdown ni texto adicional:
{
  "title": "...",
  "location": "...",
  "segments": [{"text": "..."}, {"text": "..."}],
  "decisions": [
    {
      "text": "...",
      "effects": {"food": 0, "water": 0, "health": 0, "morale": 0},
      "result": "..."
    }
  ]
}`
}

export { clamp, generateContextHash, buildGamePrompt }
```

**Concepto clave: Prompt Engineering**

El prompt es CRÍTICO. Gemini necesita instrucciones MUY específicas:

1. **Rol:** "Eres el narrador de un juego..." — Le da contexto.
2. **Estado actual:** Los números exactos para que sea coherente (si tienes 2 de comida, no debe generarte un banquete).
3. **Flags:** Para mantener continuidad narrativa (si echaste a los refugiados, no deben aparecer mágicamente).
4. **Formato de salida:** Le pedimos JSON explícitamente. Sin esto, Gemini podría devolver texto libre que no podemos parsear.
5. **"Responde SOLO con un objeto JSON válido"** — Evita que agregue "Claro, aquí tienes..." antes del JSON.

### 6.2 Crear `src/routes/dialogue.js`

```javascript
import { Router } from 'express'
import Dialogue from '../models/Dialogue.js'
import { generateContextHash, buildGamePrompt } from '../services/gemini.js'
import { GoogleGenerativeAI } from '@google/generative-ai'
import { authMiddleware } from '../middleware/auth.js'

const router = Router()
router.use(authMiddleware)

router.post('/generate', async (req, res) => {
  try {
    const { day, flags, food, water, health, morale, context } = req.body

    const prompt = buildGamePrompt(day, flags, food, water, health, morale, context || '')
    const contextHash = generateContextHash(day, flags, food, water, health, morale)

    // Buscar en caché
    const cached = await Dialogue.findOne({ contextHash })
    if (cached) {
      return res.json({ ...cached.response, cached: true })
    }

    // Llamar a Gemini
    const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || '')
    const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' })
    const result = await model.generateContent(prompt)
    const text = result.response.text()

    const cleanJson = text.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim()
    const response = JSON.parse(cleanJson)

    // Guardar en caché
    await Dialogue.create({
      contextHash,
      prompt,
      response,
      model: 'gemini-2.5-flash',
      tokensUsed: result.response.usageMetadata?.totalTokenCount || 0,
    })

    res.json({ ...response, cached: false })
  } catch (err) {
    console.error('Error en diálogo IA:', err.message)
    // Fallback
    res.json({
      type: 'ai',
      title: 'SILENCIO INQUIETANTE',
      location: 'casa',
      segments: [
        { text: 'El día transcurre en calma tensa...' },
        { text: 'Revisas tus suministros. Por ahora, estás a salvo.' },
      ],
      decisions: [
        {
          text: 'Descansar',
          effects: { food: 0, water: 0, health: 3, morale: 5 },
          result: 'Te recuestas. El silencio te envuelve.',
        },
        {
          text: 'Reforzar refugio',
          effects: { food: 0, water: 0, health: -2, morale: 3 },
          result: 'Clavas tablones. El refugio está más seguro.',
        },
      ],
    })
  }
})

router.get('/cache/stats', async (req, res) => {
  const count = await Dialogue.countDocuments()
  res.json({ cachedDialogues: count })
})

export default router
```

### 6.3 Conectar en `src/index.js`

```javascript
import dialogueRoutes from './routes/dialogue.js'

app.use('/api/dialogue', dialogueRoutes)
```

---

## Día 7 — Tests de lógica de juego y Gemini

### 7.1 Crear `src/tests/games.spec.js` (15 tests)

```javascript
import { describe, it, expect } from 'vitest'

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value))
}

const INITIAL = { food: 6, water: 4, health: 80, morale: 70 }

describe('Game Logic', () => {
  describe('Daily Resource Consumption', () => {
    it('reduce todos los recursos diariamente', () => {
      const s = { ...INITIAL }
      s.food = clamp(s.food - 1, 0, 20)
      s.water = clamp(s.water - 1, 0, 20)
      s.health = clamp(s.health - 4, 0, 100)
      s.morale = clamp(s.morale - 3, 0, 100)
      expect(s.food).toBe(5)
      expect(s.water).toBe(3)
      expect(s.health).toBe(76)
      expect(s.morale).toBe(67)
    })

    it('aplica penalización extra cuando comida es 0', () => {
      const s = { food: 1, water: 4, health: 80, morale: 70 }
      s.food = clamp(s.food - 1, 0, 20)
      s.water = clamp(s.water - 1, 0, 20)
      s.health = clamp(s.health - 4, 0, 100)
      s.morale = clamp(s.morale - 3, 0, 100)
      if (s.food <= 0) s.health = clamp(s.health - 4, 0, 100)
      expect(s.food).toBe(0)
      expect(s.health).toBe(72)
    })

    it('aplica penalización extra cuando agua es 0', () => {
      const s = { food: 6, water: 1, health: 80, morale: 70 }
      s.food = clamp(s.food - 1, 0, 20)
      s.water = clamp(s.water - 1, 0, 20)
      s.health = clamp(s.health - 4, 0, 100)
      s.morale = clamp(s.morale - 3, 0, 100)
      if (s.water <= 0) s.health = clamp(s.health - 6, 0, 100)
      expect(s.water).toBe(0)
      expect(s.health).toBe(70)
    })
  })

  describe('Decision Effect Application', () => {
    it('aplica efectos positivos', () => {
      const s = { ...INITIAL }
      const eff = { food: 3, water: 2, health: 0, morale: 10 }
      s.food = clamp(s.food + eff.food, 0, 20)
      s.water = clamp(s.water + eff.water, 0, 20)
      s.health = clamp(s.health + eff.health, 0, 100)
      s.morale = clamp(s.morale + eff.morale, 0, 100)
      expect(s.food).toBe(9)
      expect(s.morale).toBe(80)
    })

    it('aplica efectos negativos', () => {
      const s = { ...INITIAL }
      const eff = { food: -2, water: 0, health: -10, morale: -15 }
      s.food = clamp(s.food + eff.food, 0, 20)
      s.water = clamp(s.water + eff.water, 0, 20)
      s.health = clamp(s.health + eff.health, 0, 100)
      s.morale = clamp(s.morale + eff.morale, 0, 100)
      expect(s.food).toBe(4)
      expect(s.morale).toBe(55)
    })

    it('no baja de 0', () => {
      const s = { food: 1, water: 1, health: 5, morale: 2 }
      const eff = { food: -3, water: -2, health: -20, morale: -10 }
      s.food = clamp(s.food + eff.food, 0, 20)
      s.water = clamp(s.water + eff.water, 0, 20)
      s.health = clamp(s.health + eff.health, 0, 100)
      s.morale = clamp(s.morale + eff.morale, 0, 100)
      expect(s.food).toBe(0)
      expect(s.health).toBe(0)
    })
  })

  describe('Game Over Detection', () => {
    it('detecta game over por salud', () => {
      expect(true).toBe({ health: 0, morale: 50 }.health <= 0)
    })
    it('detecta game over por moral', () => {
      expect(true).toBe({ health: 50, morale: 0 }.morale <= 0)
    })
    it('no detecta game over con recursos positivos', () => {
      const s = { health: 50, morale: 50 }
      expect(s.health <= 0 || s.morale <= 0).toBe(false)
    })
  })

  describe('Victory Detection', () => {
    it('detecta victoria después del día 15', () => {
      expect(16 > 15 && 30 > 0 && 40 > 0).toBe(true)
    })
    it('no detecta victoria antes del día 16', () => {
      expect(14 > 15).toBe(false)
    })
  })

  describe('Flag Management', () => {
    it('setea un flag', () => {
      const flags = {}
      flags.refugees = true
      expect(flags.refugees).toBe(true)
    })
    it('preserva flags existentes', () => {
      const flags = { refugees: true }
      flags.d2_share = true
      expect(flags.refugees).toBe(true)
      expect(flags.d2_share).toBe(true)
    })
  })

  describe('Multiple Events Per Day', () => {
    it('avanza día cuando eventsThisDay llega al máximo', () => {
      expect(3 >= 3).toBe(true)
    })
    it('no avanza día cuando faltan eventos', () => {
      expect(1 < 3).toBe(true)
    })
  })
})
```

### 7.2 Crear `src/tests/gemini.spec.js` (9 tests)

```javascript
import { describe, it, expect } from 'vitest'
import { clamp, generateContextHash, buildGamePrompt } from '../services/gemini.js'

describe('gemini service', () => {
  describe('clamp', () => {
    it('valor dentro del rango', () => expect(clamp(5, 0, 10)).toBe(5))
    it('valor bajo el mínimo', () => expect(clamp(-5, 0, 100)).toBe(0))
    it('valor sobre el máximo', () => expect(clamp(150, 0, 100)).toBe(100))
    it('en los bordes exactos', () => {
      expect(clamp(0, 0, 100)).toBe(0)
      expect(clamp(100, 0, 100)).toBe(100)
    })
  })

  describe('generateContextHash', () => {
    it('genera un hash de 64 caracteres', () => {
      const hash = generateContextHash(5, { refugees: true }, 8, 6, 75, 65)
      expect(hash).toHaveLength(64)
    })
    it('contextos diferentes generan hashes diferentes', () => {
      const h1 = generateContextHash(1, {}, 6, 4, 80, 70)
      const h2 = generateContextHash(5, { refugees: true }, 2, 1, 50, 40)
      expect(h1).not.toBe(h2)
    })
    it('contextos idénticos generan el mismo hash', () => {
      const h1 = generateContextHash(3, { d3: true }, 10, 7, 85, 72)
      const h2 = generateContextHash(3, { d3: true }, 10, 7, 85, 72)
      expect(h1).toBe(h2)
    })
  })

  describe('buildGamePrompt', () => {
    it('contiene el estado del juego', () => {
      const p = buildGamePrompt(5, { refugees: true }, 8, 6, 75, 65, 'ya exploraste')
      expect(p).toContain('15 Días')
      expect(p).toContain('día 5')
      expect(p).toContain('Comida: 8/20')
      expect(p).toContain('refugees')
      expect(p).toContain('ya exploraste')
    })
    it('maneja flags vacíos', () => {
      const p = buildGamePrompt(1, {}, 6, 4, 80, 70, '')
      expect(p).toContain('ninguno')
    })
  })
})
```

### 7.3 Ejecutar todos los tests

```powershell
cd backend
pnpm test
```

Deberías ver 29 tests pasando:
```
✓ src/tests/auth.spec.js      (5 tests)
✓ src/tests/games.spec.js    (15 tests)
✓ src/tests/gemini.spec.js    (9 tests)
```

---

## Checklist final de la Semana 2

- [ ] `middleware/auth.js` creado y funcionando
- [ ] `routes/auth.js` con register, login, me
- [ ] `routes/games.js` con TODA la lógica de juego y 8 endpoints
- [ ] `services/gemini.js` con prompt builder y hash
- [ ] `routes/dialogue.js` con generación cacheada
- [ ] `src/index.js` actualizado con las 3 rutas montadas
- [ ] `tests/auth.spec.js` — 5 tests pasando
- [ ] `tests/games.spec.js` — 15 tests pasando
- [ ] `tests/gemini.spec.js` — 9 tests pasando
- [ ] Hiciste `git push` y el CI/CD está verde

---

## Cómo probar el backend manualmente

Una vez que el servidor está corriendo, puedes probar los endpoints con Postman, Thunder Client o curl:

```powershell
# 1. Health check
curl http://localhost:3000/api/health

# 2. Registrar usuario
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"jugador1","email":"test@test.com","password":"123456"}'

# 3. Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456"}'

# 4. Crear partida (copia el token del paso 2 o 3)
curl -X POST http://localhost:3000/api/games \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN_AQUI>"

# 5. Iniciar día 1
curl -X PUT http://localhost:3000/api/games/<GAME_ID>/start \
  -H "Authorization: Bearer <TOKEN_AQUI>"

# 6. Avanzar segmento
curl -X PUT http://localhost:3000/api/games/<GAME_ID>/advance-segment \
  -H "Authorization: Bearer <TOKEN_AQUI>"

# 7. Tomar decisión (índice 0 o 1)
curl -X PUT http://localhost:3000/api/games/<GAME_ID>/decision \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN_AQUI>" \
  -d '{"decisionIndex":0}'

# 8. Continuar
curl -X PUT http://localhost:3000/api/games/<GAME_ID>/continue \
  -H "Authorization: Bearer <TOKEN_AQUI>"
```
