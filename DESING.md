# Documento de Diseño - 15 Días (Fullstack)

## Integrantes del Grupo
- [Diego Alvarez] 
- [Matias Moraga] 

## Repositorio GitHub
[https://github.com/DiegoUC-01/Solemne2]

---

## 1. Arquitectura Fullstack

### Visión general

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐
│   Frontend       │────▶│   Backend REST    │────▶│  MongoDB     │
│   Vue 3 + Pinia  │◀────│   Express.js      │◀────│              │
│   (Game/)        │     │   (backend/)      │     │              │
└──────────────────┘     └────────┬─────────┘     └──────────────┘
                                  │
                                  │ Cache y consulta
                                  ▼
                         ┌──────────────────┐
                         │  Google Gemini   │
                         │  API (REST)      │
                         │  gemini-2.5-flash│
                         └──────────────────┘
```

### Stack tecnológico

| Capa | Tecnología | Propósito |
|------|-----------|-----------|
| Frontend | Vue.js 3 + Pinia + Vite | UI del juego, estado local, minijuegos |
| Backend | Node.js 20 + Express 5 | API REST, lógica de juego, auth |
| Base de datos | MongoDB 7 + Mongoose 8 | Persistencia de usuarios, partidas, caché IA |
| IA externa | Google Gemini API (gemini-2.5-flash) | Diálogos y eventos dinámicos |
| Testing | Vitest | Pruebas unitarias (frontend y backend) |
| Linter | oxlint | Análisis estático (frontend y backend) |
| CI/CD | GitHub Actions | Lint → Test → Build y Push a DockerHub |
| Containerización | Docker + Docker Compose | 3 servicios: frontend, backend, mongodb |

---

## 2. Mejoras y Correcciones (Solemne 2 → Solemne 3)

- **Eventos múltiples por día**: El sistema original tenía 1 evento por día. Ahora cada día puede tener hasta 3 eventos, combinando eventos fijos predefinidos con eventos generados por IA.
- **Persistencia en servidor**: Las partidas se guardan en MongoDB, permitiendo continuar en cualquier momento y desde cualquier dispositivo.
- **Autenticación de usuarios**: Registro y login con JWT para identificar jugadores.
- **Diálogos dinámicos con IA**: Eventos narrativos generados por Google Gemini que complementan los eventos fijos, enriqueciendo la experiencia sin repetir contenido.
- **Caché de IA**: Las respuestas generadas por Gemini se almacenan en MongoDB para evitar llamadas repetidas a la API externa, ahorrando tokens y latencia.

---

## 3. Servicio REST Externo: Google Gemini

### Justificación

Google Gemini API (modelo `gemini-2.5-flash`) fue elegido porque:
- Es **gratuito** en su tier gratuito (1,500 requests/día)
- Genera texto en español de alta calidad
- Responde en formato JSON estructurado, ideal para parsear eventos del juego
- Su latencia es baja (~1-2 segundos por request)

### Endpoints consumidos

| Endpoint | Método | Uso en el juego |
|----------|--------|----------------|
| `generateContent` | POST | Generar eventos narrativos dinámicos con decisiones y efectos |

### Integración en la arquitectura

El backend actúa como intermediario entre el frontend y Gemini:

1. El juego necesita un evento adicional para un día → frontend llama a `POST /api/dialogue/generate`
2. El backend calcula un hash del contexto actual (día, recursos, flags)
3. Busca en la colección `dialogues` de MongoDB por ese hash
4. Si existe → devuelve respuesta cacheada (0 tokens)
5. Si no existe → llama a Gemini, guarda respuesta en MongoDB, devuelve al frontend

### Modelo de datos de caché (dialogues)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `contextHash` | String (SHA256) | Hash del prompt + contexto |
| `prompt` | String | Prompt enviado a Gemini |
| `response` | Object | Evento generado (título, segmentos, decisiones) |
| `model` | String | Modelo usado (gemini-2.5-flash) |
| `tokensUsed` | Number | Tokens consumidos |

---

## 4. Modelo de Datos MongoDB

### Colección `users`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `_id` | ObjectId | |
| `username` | String | Nombre único |
| `email` | String | Email único |
| `password` | String | Bcrypt hasheado |
| `createdAt/updatedAt` | Date | Timestamps |

### Colección `games`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `_id` | ObjectId | |
| `userId` | ObjectId | FK a users |
| `day` | Number | Día actual (0-15) |
| `phase` | String | menu, intro, story, decision, result, minigame, victory, gameover |
| `food` | Number | Comida (0-20) |
| `water` | Number | Agua (0-20) |
| `health` | Number | Salud (0-100) |
| `morale` | Number | Moral (0-100) |
| `flags` | Object | Flags de decisiones (refugees, d1_solo, etc.) |
| `journal` | Array | Bitácora de eventos y decisiones |
| `currentEvent` | Object | Evento actual (fijo o generado por IA) |
| `currentSegment` | Number | Segmento actual del evento |
| `decisionResult` | Object | Resultado de la última decisión |
| `eventsThisDay` | Number | Eventos jugados hoy |
| `maxEventsPerDay` | Number | Máximo de eventos por día (default: 3) |
| `status` | String | active, won, lost |
| `createdAt/updatedAt` | Date | Timestamps |

---

## 5. Endpoints de la API REST

### Auth (`/api/auth`)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| POST | `/register` | No | Registro → JWT |
| POST | `/login` | No | Login → JWT |
| GET | `/me` | JWT | Datos del usuario autenticado |

### Game (`/api/games`)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| POST | `/` | JWT | Crear nueva partida |
| GET | `/user` | JWT | Listar partidas del usuario |
| GET | `/:id` | JWT | Obtener estado de partida |
| PUT | `/:id/start` | JWT | Iniciar día 1 (tras intro) |
| PUT | `/:id/advance-segment` | JWT | Avanzar segmento de texto |
| PUT | `/:id/decision` | JWT | Registrar decisión tomada |
| PUT | `/:id/continue` | JWT | Continuar tras ver resultado |
| PUT | `/:id/minigame` | JWT | Registrar resultado de minijuego |

### Diálogos IA (`/api/dialogue`)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| POST | `/generate` | JWT | Generar evento con IA (cacheado) |
| GET | `/cache/stats` | JWT | Estadísticas de caché |

---

## 6. Estructura de Carpetas (Fullstack)

```
Solemne2/
├── compose.yml
├── DESIGN.md
├── PLANNING.md
├── .github/
│   └── workflows/
│       └── main.yml
├── Game/                          # Frontend (Vue 3)
│   ├── Dockerfile
│   ├── package.json
│   ├── vite.config.js
│   ├── index.html
│   └── src/
│       ├── main.js
│       ├── App.vue
│       ├── api/
│       │   └── index.js
│       ├── stores/
│       │   ├── gameStore.js
│       │   └── serverStore.js
│       ├── components/
│       │   ├── StatBar.vue
│       │   ├── StoryText.vue
│       │   ├── DecisionButtons.vue
│       │   ├── PixelBackground.vue
│       │   ├── SceneArt.vue
│       │   └── minigames/
│       │       ├── CatchRainGame.vue
│       │       ├── FindcansGame.vue
│       │       └── EscapeGame.vue
│       ├── composables/
│       │   └── useAudio.js
│       ├── data/
│       │   └── events.js
│       ├── assets/
│       │   └── styles/
│       └── tests/
│           └── gameStore.spec.js
└── backend/                       # Backend (Express)
    ├── Dockerfile
    ├── package.json
    ├── .env.example
    ├── .gitignore
    └── src/
        ├── index.js
        ├── config/
        │   └── db.js
        ├── models/
        │   ├── User.js
        │   ├── Game.js
        │   └── Dialogue.js
        ├── middleware/
        │   └── auth.js
        ├── routes/
        │   ├── auth.js
        │   ├── games.js
        │   └── dialogue.js
        ├── services/
        │   └── gemini.js
        └── tests/
            ├── auth.spec.js
            ├── games.spec.js
            └── gemini.spec.js
```

## 7. Nuevas Pantallas (Solemne 3)

### Pantalla de Login/Registro
El usuario puede registrarse o iniciar sesión para guardar su progreso en el servidor.

### Pantalla de Selección de Partida
El usuario puede ver sus partidas guardadas y continuar o empezar una nueva.

### Indicador de eventos por día
En la UI se muestra "Evento X de Y" para indicar cuántos eventos quedan en el día actual.

### Indicador de modo online
Un icono en el header muestra si el juego está conectado al servidor (partida guardada) o en modo local sin conexión.
