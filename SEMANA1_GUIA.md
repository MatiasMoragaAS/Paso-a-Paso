# GUÍA SEMANA 1 — Infraestructura, CI/CD y Setup

> **Fechas:** 16 al 22 de junio
> **Objetivo:** Dejar el esqueleto del proyecto listo. Al final de esta semana, si haces `git push`, GitHub Actions se ejecuta solo, Docker Compose levanta 3 contenedores, y tienes toda la documentación al día.

---

## Día 1 — Preparar el terreno

### 1.1 ¿Dónde estoy parado?

Tu proyecto actual vive en `Solemne2/` y dentro tiene la carpeta `Game/` que es el juego en Vue 3. Esa carpeta **no se toca** (por ahora). Todo lo nuevo va en una carpeta hermana llamada `backend/`.

**Estructura actual:**
```
Solemne2/
├── Game/          ← El juego Vue 3 (no lo toques)
├── DESIGN.md
├── PLANNING.md
└── PLANNING_BACKEND.md
```

**Estructura que tendrás al final de la semana 1:**
```
Solemne2/
├── .github/
│   └── workflows/
│       └── main.yml          ← CI/CD (lo creamos hoy)
├── backend/                   ← NUEVO: todo el backend
│   ├── Dockerfile
│   ├── package.json
│   ├── .env.example
│   ├── .gitignore
│   └── src/
│       ├── index.js           ← Servidor Express (vacío aún)
│       ├── config/
│       │   └── db.js          ← Conexión a MongoDB
│       ├── models/
│       │   ├── User.js        ← Modelo de usuario
│       │   ├── Game.js        ← Modelo de partida
│       │   └── Dialogue.js    ← Modelo de caché IA
│       ├── middleware/        ← (vacío, se llena semana 2)
│       ├── routes/            ← (vacío, se llena semana 2)
│       ├── services/          ← (vacío, se llena semana 2)
│       └── tests/             ← (vacío, se llena semana 2)
├── compose.yml                ← Orquesta los 3 contenedores
├── Game/                      ← (sin cambios)
├── DESIGN.md                  ← (actualizado)
├── PLANNING.md
└── PLANNING_BACKEND.md
```

---

### 1.2 Crear las carpetas del backend

Abre PowerShell en la raíz del proyecto (`Solemne2/`) y ejecuta:

```powershell
# Crear todas las carpetas de una sola vez
New-Item -ItemType Directory -Path "backend\src\config" -Force
New-Item -ItemType Directory -Path "backend\src\models" -Force
New-Item -ItemType Directory -Path "backend\src\middleware" -Force
New-Item -ItemType Directory -Path "backend\src\routes" -Force
New-Item -ItemType Directory -Path "backend\src\services" -Force
New-Item -ItemType Directory -Path "backend\src\tests" -Force
```

**¿Qué es cada carpeta?**

| Carpeta | ¿Qué va ahí? |
|---------|-------------|
| `config/` | Archivos de configuración. Por ahora solo `db.js` para conectarse a MongoDB. |
| `models/` | Los "planos" de los datos. Definen qué forma tienen los documentos en MongoDB (como tablas en SQL pero para MongoDB). |
| `middleware/` | Funciones que se ejecutan en cada petición ANTES de llegar a la ruta. Por ejemplo, verificar que el usuario tiene token JWT. |
| `routes/` | Los endpoints de la API. Cada archivo agrupa rutas relacionadas: `auth.js` para login, `games.js` para partidas, etc. |
| `services/` | Lógica de negocio que no es ni ruta ni modelo. Por ejemplo, `gemini.js` que construye prompts y llama a la IA. |
| `tests/` | Pruebas unitarias con Vitest. |

---

### 1.3 Crear `backend/package.json`

Este archivo le dice a Node.js:
- Cómo se llama el proyecto
- Qué comandos tiene (`pnpm dev`, `pnpm test`)
- Qué librerías necesita (express, mongoose, etc.)

Crea el archivo `backend/package.json` con este contenido:

```json
{
  "name": "15-dias-backend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "node --watch src/index.js",
    "start": "node src/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "oxlint src/"
  },
  "dependencies": {
    "express": "^5.1.0",
    "mongoose": "^8.9.0",
    "jsonwebtoken": "^9.0.2",
    "bcryptjs": "^2.4.3",
    "@google/generative-ai": "^0.21.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.7"
  },
  "devDependencies": {
    "vitest": "^4.0.0",
    "oxlint": "^0.15.0"
  }
}
```

**Explicación de cada dependencia:**

| Paquete | ¿Para qué sirve? |
|---------|-----------------|
| `express` | El framework web. Maneja las peticiones HTTP (GET, POST, PUT, DELETE). Es el corazón del backend. |
| `mongoose` | Librería para hablar con MongoDB desde Node.js. Define esquemas, valida datos, hace queries. |
| `jsonwebtoken` | Crea y verifica tokens JWT. El token es como un "carnet" que el usuario recibe al hacer login y debe enviar en cada petición para probar quién es. |
| `bcryptjs` | Encripta contraseñas. NUNCA guardes contraseñas en texto plano. bcrypt las transforma en un hash irreversible. |
| `@google/generative-ai` | El SDK oficial de Google para llamar a Gemini (la IA que genera los diálogos del juego). |
| `cors` | Permite que el frontend (en puerto 5173) hable con el backend (en puerto 3000). Sin esto, el navegador bloquea las peticiones. |
| `dotenv` | Carga variables de entorno desde un archivo `.env`. Ahí guardas secretos como la API key de Gemini. |
| `vitest` | Framework de testing. Misma herramienta que usa el frontend, así mantenemos consistencia. |
| `oxlint` | Linter ultrarrápido. Revisa el código en busca de errores y malas prácticas. |

**¿Qué significan los comandos en `scripts`?**

```json
"dev": "node --watch src/index.js"
```
`node --watch` ejecuta `src/index.js` y lo reinicia automáticamente si detecta cambios en cualquier archivo. Como `nodemon` pero nativo de Node.js 20+.

```json
"test": "vitest run"
```
Ejecuta todas las pruebas una vez y termina. Ideal para CI/CD.

```json
"test:watch": "vitest"
```
Ejecuta pruebas en modo watch: se quedan corriendo y re-ejecutan automáticamente al guardar archivos.

```json
"lint": "oxlint src/"
```
Revisa todo el código en `src/` en busca de errores.

> **IMPORTANTE:** `"type": "module"` le dice a Node.js que use `import`/`export` en vez de `require`. Sin esto, los `import` no funcionan.

---

### 1.4 Crear `.env.example` y `.gitignore`

**`.env.example`** — Es una plantilla de las variables de entorno que el proyecto necesita. Se sube a GitHub (sin valores reales). Cada desarrollador copia este archivo a `.env` y pone sus propios valores.

```bash
PORT=3000
MONGODB_URI=mongodb://localhost:27017/15dias
JWT_SECRET=cambia-esto-por-un-secreto-real
GEMINI_API_KEY=tu-api-key-de-gemini
```

| Variable | ¿Qué es? |
|----------|---------|
| `PORT` | Puerto donde corre el backend (3000) |
| `MONGODB_URI` | Dirección de conexión a MongoDB. `localhost:27017` es la default. `15dias` es el nombre de la base de datos. |
| `JWT_SECRET` | Clave secreta para firmar tokens. Debe ser larga y aleatoria. Puedes generar una con `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `GEMINI_API_KEY` | Tu API key de Google AI Studio. Se obtiene gratis en https://aistudio.google.com/apikey |

**`.gitignore`** — Lista de archivos/carpetas que NO se suben a GitHub:

```
node_modules/
.env
coverage/
dist/
```

- `node_modules/` — Las dependencias instaladas. Ocupan cientos de MB y se regeneran con `pnpm install`.
- `.env` — Contiene secretos reales (API keys, contraseñas). **Jamás** se sube a GitHub.
- `coverage/` — Reportes de cobertura de tests, generados automáticamente.
- `dist/` — Carpeta de build, generada automáticamente.

---

### 1.5 Instalar dependencias

```powershell
cd backend
pnpm install
```

Esto lee `package.json`, descarga todas las dependencias a `node_modules/` y genera `pnpm-lock.yaml` (que registra las versiones exactas instaladas). El lock file SÍ se sube a GitHub para que todos tengan las mismas versiones.

---

### 1.6 Crear el servidor Express mínimo (`src/index.js`)

Por ahora creamos un servidor que solo dice "estoy vivo". Lo llenaremos en la semana 2.

```javascript
import 'dotenv/config'
import express from 'express'
import cors from 'cors'
import { connectDB } from './config/db.js'

const app = express()
const PORT = process.env.PORT || 3000

app.use(cors())
app.use(express.json())

// Ruta de health check — útil para saber si el servidor está corriendo
app.get('/api/health', (req, res) => {
  res.json({ status: 'ok' })
})

async function start() {
  await connectDB()
  app.listen(PORT, () => {
    console.log(`Backend corriendo en http://localhost:${PORT}`)
  })
}

start()
```

**Línea por línea:**

```javascript
import 'dotenv/config'
```
Carga el archivo `.env` y mete sus valores en `process.env`. Debe ser lo PRIMERO que se ejecuta.

```javascript
import express from 'express'
import cors from 'cors'
```
Importa express (el framework) y cors (para permitir peticiones de otros dominios).

```javascript
const app = express()
const PORT = process.env.PORT || 3000
```
Crea la aplicación express. El puerto sale de la variable de entorno PORT, o 3000 por defecto.

```javascript
app.use(cors())
app.use(express.json())
```
Middleware: cors permite peticiones de cualquier origen. `express.json()` parsea el body de las peticiones JSON automáticamente.

```javascript
app.get('/api/health', (req, res) => {
  res.json({ status: 'ok' })
})
```
Define una ruta GET en `/api/health`. Si alguien hace `GET http://localhost:3000/api/health`, recibe `{"status":"ok"}`. Esto sirve para verificar que el servidor está vivo.

```javascript
async function start() {
  await connectDB()
  app.listen(PORT, () => { ... })
}
start()
```
Función async que primero conecta a MongoDB, luego arranca el servidor. El `await` es importante: no queremos recibir peticiones si la base de datos no está lista.

---

### 1.7 Crear la conexión a MongoDB (`src/config/db.js`)

```javascript
import mongoose from 'mongoose'

export async function connectDB() {
  const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017/15dias'
  await mongoose.connect(uri)
  console.log('MongoDB conectado')
}
```

**¿Qué hace esto?**
- `mongoose.connect(uri)` — Establece la conexión a MongoDB. La URI es la dirección del servidor.
- Si la base de datos `15dias` no existe, MongoDB la crea automáticamente al insertar el primer documento.
- Si la conexión falla, lanza un error y el servidor no arranca.

---

## Día 2 — Modelos de MongoDB

Los modelos definen la ESTRUCTURA de los datos. Piensa en ellos como el equivalente a las tablas de SQL, pero con esquemas flexibles.

### 2.1 Modelo User (`src/models/User.js`)

```javascript
import mongoose from 'mongoose'
import bcrypt from 'bcryptjs'

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true        // Elimina espacios al inicio y final
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,   // Convierte a minúsculas automáticamente
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6       // Mínimo 6 caracteres
  },
}, { timestamps: true })  // Agrega createdAt y updatedAt automáticamente

// Antes de guardar, encripta la contraseña
userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next()
  this.password = await bcrypt.hash(this.password, 10)
  next()
})

// Método para comparar contraseñas
userSchema.methods.comparePassword = function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password)
}

// Al convertir a JSON, elimina la contraseña
userSchema.methods.toJSON = function () {
  const obj = this.toObject()
  delete obj.password
  return obj
}

export default mongoose.model('User', userSchema)
```

**Conceptos clave:**

- `mongoose.Schema({...})` — Define los campos y sus reglas.
- `unique: true` — No puede haber dos usuarios con el mismo email o username. MongoDB crea un índice único automáticamente.
- `timestamps: true` — Agrega `createdAt` (fecha de creación) y `updatedAt` (fecha de última modificación) a cada documento.
- `pre('save')` — Es un "hook" o "middleware de Mongoose". Se ejecuta JUSTO ANTES de guardar un usuario. Si la contraseña fue modificada (o es nueva), la encripta con bcrypt.
- `bcrypt.hash(password, 10)` — El número 10 son las "rondas de sal". Más rondas = más seguro pero más lento. 10 es el estándar.
- `comparePassword()` — bcrypt no "desencripta", compara el hash guardado con un nuevo hash de la contraseña ingresada. Si coinciden, la contraseña es correcta.
- `toJSON()` — Cuando Express envía el usuario como respuesta JSON, este método se llama automáticamente y elimina el campo `password` para que nunca se filtre.

### 2.2 Modelo Game (`src/models/Game.js`)

```javascript
import mongoose from 'mongoose'

const journalEntrySchema = new mongoose.Schema({
  day: Number,
  type: { type: String, enum: ['evento', 'decision', 'minijuego'] },
  title: String,
  decision: String,
  result: String,
  description: String,
  effects: mongoose.Schema.Types.Mixed,
  success: Boolean,
  timestamp: { type: Date, default: Date.now },
})

const gameSchema = new mongoose.Schema({
  userId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  day:         { type: Number, default: 0 },
  phase:       { type: String, default: 'intro' },
  food:        { type: Number, default: 6 },
  water:       { type: Number, default: 4 },
  health:      { type: Number, default: 80 },
  morale:      { type: Number, default: 70 },
  maxStat:     { type: Number, default: 20 },
  maxHealth:   { type: Number, default: 100 },
  maxMorale:   { type: Number, default: 100 },
  flags:       { type: mongoose.Schema.Types.Mixed, default: {} },
  journal:     [journalEntrySchema],
  currentEvent:     { type: mongoose.Schema.Types.Mixed, default: null },
  currentSegment:   { type: Number, default: 0 },
  decisionResult:   { type: mongoose.Schema.Types.Mixed, default: null },
  usedRandomEvents: [Number],
  gameOverReason:   { type: String, default: null },
  eventsThisDay:    { type: Number, default: 0 },
  maxEventsPerDay:  { type: Number, default: 3 },
  status: {
    type: String,
    enum: ['active', 'won', 'lost'],
    default: 'active'
  },
}, { timestamps: true })

export default mongoose.model('Game', gameSchema)
```

**Conceptos clave:**

- `userId: { type: ObjectId, ref: 'User' }` — Esto es una "referencia" a otro documento. Relaciona cada partida con un usuario. Es el equivalente a una foreign key en SQL.
- `flags: { type: Mixed }` — `Mixed` significa "cualquier cosa". Los flags son un objeto dinámico (`{refugees: true, d3_super: true, ...}`) que crece según las decisiones del jugador. No podemos definir campos fijos porque no sabemos qué flags existirán.
- `journal: [journalEntrySchema]` — Un array de subdocumentos. Cada entrada del diario es un objeto embebido dentro del documento Game. En SQL esto serían dos tablas separadas; en MongoDB puede ir todo junto.
- `eventsThisDay` / `maxEventsPerDay` — El sistema de eventos múltiples: cuántos eventos han ocurrido hoy y cuál es el máximo permitido (default 3).
- `status` — `active` = partida en curso, `won` = victoria, `lost` = game over.

### 2.3 Modelo Dialogue (`src/models/Dialogue.js`)

```javascript
import mongoose from 'mongoose'

const dialogueSchema = new mongoose.Schema({
  contextHash: {
    type: String,
    required: true,
    unique: true,
    index: true     // Crea un índice para búsquedas rápidas
  },
  prompt:    { type: String, required: true },
  response:  { type: mongoose.Schema.Types.Mixed, required: true },
  model:     { type: String, default: 'gemini-2.5-flash' },
  tokensUsed: { type: Number, default: 0 },
}, { timestamps: true })

export default mongoose.model('Dialogue', dialogueSchema)
```

**Conceptos clave:**

- `contextHash` — Es un hash SHA256 que identifica de forma única una situación del juego (día + recursos + flags). Si dos situaciones son idénticas, generan el mismo hash y se reutiliza la respuesta cacheada.
- `index: true` — Crea un índice en MongoDB para que la búsqueda por `contextHash` sea instantánea.
- `unique: true` — Garantiza que no haya dos documentos con el mismo hash.
- `tokensUsed` — Cuántos tokens consumió la llamada a Gemini. Útil para monitorear costos (aunque Gemini tiene capa gratuita).

---

## Día 3 — Docker y Docker Compose

Docker empaqueta tu aplicación en un "contenedor" que incluye el código, las dependencias y la configuración. Así se ejecuta igual en tu computadora, en la del profesor o en un servidor.

### 3.1 Dockerfile del backend

Crea `backend/Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install --legacy-peer-deps

COPY . .

EXPOSE 3000

CMD ["node", "src/index.js"]
```

**Línea por línea:**

```dockerfile
FROM node:20-alpine
```
Usa una imagen base de Node.js 20 con Alpine Linux (una distro ultra-ligera de ~5MB). Esto minimiza el tamaño del contenedor.

```dockerfile
WORKDIR /app
```
Crea y se mueve a la carpeta `/app` dentro del contenedor. Todos los comandos siguientes se ejecutan ahí.

```dockerfile
COPY package*.json ./
```
Copia `package.json` y `pnpm-lock.yaml` al contenedor. Se hace ANTES de instalar dependencias para aprovechar la caché de Docker: si los archivos no cambiaron, Docker reutiliza la capa anterior.

```dockerfile
RUN npm install --legacy-peer-deps
```
Instala las dependencias. Usamos npm en vez de pnpm dentro del contenedor porque Alpine a veces tiene problemas con pnpm, y `--legacy-peer-deps` evita conflictos de versiones.

```dockerfile
COPY . .
```
Copia el resto del código fuente.

```dockerfile
EXPOSE 3000
```
Informa que el contenedor escucha en el puerto 3000. Es documentación, no abre el puerto realmente (eso se hace en docker-compose).

```dockerfile
CMD ["node", "src/index.js"]
```
El comando que se ejecuta cuando el contenedor arranca.

### 3.2 Dockerfile del frontend

El frontend YA tiene un Dockerfile en `Game/Dockerfile`. Pero tenemos que actualizarlo para usar **pnpm** (la rúbrica lo exige). Cámbialo por:

```dockerfile
FROM node:20-alpine

WORKDIR /app

RUN npm install -g pnpm

COPY package*.json pnpm-lock.yaml* ./

RUN pnpm install --frozen-lockfile

COPY . .

EXPOSE 5173

CMD ["pnpm", "dev", "--", "--host", "0.0.0.0"]
```

**Diferencias con el original:**
- Agrega `RUN npm install -g pnpm` para instalar pnpm globalmente.
- Usa `pnpm install --frozen-lockfile` en vez de `npm install`. Esto respeta EXACTAMENTE las versiones del lock file.
- `--host 0.0.0.0` permite que el contenedor acepte conexiones desde fuera (necesario en Docker).

### 3.3 docker-compose.yml

Crea `compose.yml` en la RAÍZ del proyecto (`Solemne2/compose.yml`):

```yaml
services:
  mongodb:
    image: mongo:7
    container_name: 15dias-mongo
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_DATABASE: 15dias

  backend:
    build: ./backend
    container_name: 15dias-backend
    ports:
      - "3000:3000"
    depends_on:
      - mongodb
    environment:
      PORT: 3000
      MONGODB_URI: mongodb://mongodb:27017/15dias
      JWT_SECRET: ${JWT_SECRET:-cambia-por-un-secreto}
      GEMINI_API_KEY: ${GEMINI_API_KEY}

  frontend:
    build: ./Game
    container_name: 15dias-frontend
    ports:
      - "5173:5173"
    depends_on:
      - backend
    environment:
      VITE_API_URL: http://localhost:3000/api

volumes:
  mongo-data:
```

**Explicación detallada:**

```yaml
services:
```
Define los 3 servicios (contenedores) que vamos a levantar.

```yaml
  mongodb:
    image: mongo:7
```
Usa la imagen oficial de MongoDB versión 7 desde Docker Hub. No necesita Dockerfile porque la imagen ya existe.

```yaml
    ports:
      - "27017:27017"
```
Mapea el puerto 27017 del contenedor al 27017 de tu máquina. Así puedes conectarte con MongoDB Compass o desde el backend.

```yaml
    volumes:
      - mongo-data:/data/db
```
`/data/db` es donde MongoDB guarda los datos dentro del contenedor. Al mapearlo a un volumen llamado `mongo-data`, los datos PERSISTEN aunque borres el contenedor. Sin esto, cada vez que haces `docker compose down` pierdes la base de datos.

```yaml
  backend:
    build: ./backend
```
Construye la imagen desde el Dockerfile en la carpeta `./backend`.

```yaml
    depends_on:
      - mongodb
```
El backend no arranca hasta que mongodb esté listo. OJO: `depends_on` solo espera a que el contenedor arranque, no a que MongoDB esté realmente listo para aceptar conexiones. En producción usarías `healthcheck`, pero para desarrollo es suficiente.

```yaml
    environment:
      MONGODB_URI: mongodb://mongodb:27017/15dias
```
Dentro de Docker, los contenedores se comunican por el NOMBRE del servicio. `mongodb` es el nombre del contenedor de MongoDB, no `localhost`. Esto es crucial: si pones `localhost` no va a funcionar porque cada contenedor tiene su propio `localhost`.

```yaml
      JWT_SECRET: ${JWT_SECRET:-cambia-por-un-secreto}
```
`${JWT_SECRET:-valor}` significa: usa la variable de entorno `JWT_SECRET` de tu sistema, y si no existe, usa `cambia-por-un-secreto` como fallback.

```yaml
volumes:
  mongo-data:
```
Declara el volumen nombrado al final. Docker lo crea automáticamente.

---

## Día 4 — CI/CD con GitHub Actions

GitHub Actions ejecuta tareas automáticas cada vez que haces `git push`. Nuestro pipeline tiene 3 etapas:

1. **Lint y test del frontend** — Revisa código y corre los 19 tests del juego Vue.
2. **Lint y test del backend** — Revisa código y corre los 29 tests del servidor.
3. **Build y push a DockerHub** — Solo si los tests pasan, construye las imágenes Docker y las publica.

Crea la carpeta y el archivo:

```powershell
New-Item -ItemType Directory -Path ".github\workflows" -Force
```

Crea `.github/workflows/main.yml`:

```yaml
name: CI/CD Fullstack

on:
  push:
    branches: [main, master]

jobs:
  lint-and-test-frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./Game
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
          cache-dependency-path: './Game/pnpm-lock.yaml'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test

  lint-and-test-backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./backend
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
          cache-dependency-path: './backend/pnpm-lock.yaml'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test

  build-and-push:
    needs: [lint-and-test-frontend, lint-and-test-backend]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build and push frontend
        uses: docker/build-push-action@v6
        with:
          context: ./Game
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/15dias-frontend:latest

      - name: Build and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/15dias-backend:latest
```

**Conceptos clave:**

```yaml
on:
  push:
    branches: [main, master]
```
El workflow se dispara en cada push a main o master. No se ejecuta en otras ramas.

```yaml
jobs:
```
Cada job es un conjunto de pasos que se ejecutan en una máquina virtual limpia de Ubuntu.

```yaml
    runs-on: ubuntu-latest
```
La máquina virtual usa Ubuntu (Linux). Es lo más común para Node.js.

```yaml
    defaults:
      run:
        working-directory: ./Game
```
Todos los comandos `run` en este job se ejecutan desde la carpeta `./Game`.

```yaml
      - uses: actions/checkout@v4
```
Clona tu repositorio en la máquina virtual. Es siempre el primer paso.

```yaml
      - uses: pnpm/action-setup@v4
        with:
          version: 9
```
Instala pnpm versión 9 en la VM.

```yaml
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
          cache-dependency-path: './Game/pnpm-lock.yaml'
```
Instala Node.js 20 y configura la caché de pnpm para que `pnpm install` sea más rápido en futuras ejecuciones.

```yaml
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test
```
Los 3 pasos que ejecuta: instalar dependencias, revisar código, correr tests.

```yaml
  build-and-push:
    needs: [lint-and-test-frontend, lint-and-test-backend]
```
Este job SOLO se ejecuta si los dos jobs anteriores pasaron. Si algún test falla, no se publican imágenes rotas.

```yaml
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
```
`${{ secrets.DOCKER_USERNAME }}` — Esto NO se escribe en el código. Se configura en GitHub → Settings → Secrets and variables → Actions. Así las credenciales de DockerHub nunca quedan expuestas.

```yaml
      - uses: docker/build-push-action@v6
        with:
          context: ./Game
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/15dias-frontend:latest
```
Construye la imagen desde `./Game`, le pone el tag `tunombre/15dias-frontend:latest` y la sube a DockerHub.

### Configurar los secrets en GitHub

1. Ve a tu repo en GitHub → Settings → Secrets and variables → Actions
2. Agrega dos secrets:
   - `DOCKER_USERNAME` = tu usuario de DockerHub
   - `DOCKER_TOKEN` = un token de acceso de DockerHub (se crea en DockerHub → Account Settings → Security → New Access Token)

---

## Día 5 — Documentación

Ya tienes `DESIGN.md` y `PLANNING_BACKEND.md` actualizados (los creamos antes). Revisa que:

### DESIGN.md debe incluir:
- [x] Diagrama de arquitectura fullstack
- [x] Stack tecnológico completo (frontend, backend, DB, IA)
- [x] Mejoras respecto a la Solemne 2
- [x] Servicio externo (Gemini): qué endpoints, cómo se integra, caché
- [x] Modelo de datos (3 colecciones con sus campos)
- [x] Endpoints de la API REST
- [x] Estructura de carpetas actualizada
- [x] Flujo de juego con eventos múltiples
- [x] Nuevas pantallas (login, selección de partida)

### PLANNING_BACKEND.md debe incluir:
- [x] Tareas organizadas por semana
- [x] Cada tarea marcada como `[x]` (completada) o `[ ]` (pendiente)
- [x] Objetivo de cada semana
- [x] Lo que se logró y lo que no
- [x] Resumen de tests
- [x] Instrucciones de ejecución

---

## Checklist final de la Semana 1

Antes de pasar a la semana 2, verifica que tienes TODO esto:

- [ ] Carpetas del backend creadas (`config/`, `models/`, `middleware/`, `routes/`, `services/`, `tests/`)
- [ ] `backend/package.json` con todas las dependencias
- [ ] `pnpm install` ejecutado (existe `pnpm-lock.yaml`)
- [ ] `.env.example` y `.gitignore` creados
- [ ] `src/index.js` con servidor Express mínimo
- [ ] `src/config/db.js` con conexión a MongoDB
- [ ] `src/models/User.js` con bcrypt y toJSON
- [ ] `src/models/Game.js` con journal, flags, eventos múltiples
- [ ] `src/models/Dialogue.js` con contextHash único
- [ ] `backend/Dockerfile` listo
- [ ] `Game/Dockerfile` actualizado con pnpm
- [ ] `compose.yml` con 3 servicios (mongodb, backend, frontend)
- [ ] `.github/workflows/main.yml` con CI/CD completo
- [ ] Secrets de DockerHub configurados en GitHub
- [ ] `DESIGN.md` actualizado con arquitectura fullstack
- [ ] `PLANNING_BACKEND.md` creado
- [ ] Hiciste `git push` y el CI/CD se ejecutó (verde o falló por falta de tests — normal en semana 1)

---

## Comandos útiles para esta semana

```powershell
# Instalar dependencias del backend
cd backend
pnpm install

# Probar que el servidor arranca (necesita MongoDB corriendo)
pnpm dev

# Construir y levantar todo con Docker
docker compose up --build

# Ver logs de un servicio específico
docker compose logs backend

# Detener todo
docker compose down

# Verificar que el CI/CD funciona (después de git push)
# Ve a GitHub → Actions → mirá el workflow ejecutándose
```
