# BautistaGo — Paso a paso: creación de repositorios y proyectos base

Este documento ejecuta la estrategia definida en `00-estrategia-repositorios.md`: **3 repositorios** en GitHub (`IngOscar19` o la cuenta/organización que uses), confirmando los frameworks exactos que marca la ERS (sección 2.1 "Perspectiva del producto" y 3.1.3 "Interfaces de software"):

| Repo | Apps que contiene | Framework (según ERS) |
|---|---|---|
| `BautistaGo-Backend` | API REST + Socket.io (ya existe) | Node.js + Express |
| `BautistaGo-Web` | Admin + Cocina | **Next.js / React** (PWA) |
| `BautistaGo-Mobile` | Cliente + Repartidor | **React Native con Expo** |

> Nota sobre el documento de requerimientos: la ERS (1.4 y 2.1) especifica que el **Cliente** se construye como **PWA (Next.js/React)**, al igual que Cocina y Admin, y que **solo el Repartidor** es la app móvil híbrida (React Native/Expo, empaquetada como APK). Se decidió conscientemente **desviarse de ese punto de la ERS**: Cliente se construye como app móvil nativa (Expo), igual que Repartidor, conforme a tu planteamiento original de "dos aplicaciones móviles". Si en algún momento esto debe justificarse ante el cliente o el docente evaluador, conviene documentar el cambio como un adendum a la ERS (nueva revisión del documento, sección 2.1).

---

## Paso 0 — Prerrequisitos locales

```bash
# Verifica que tienes git, node (LTS 20+), pnpm y GitHub CLI
git --version
node --version
corepack enable        # habilita pnpm sin instalarlo aparte
pnpm --version
gh --version
gh auth login           # si no has autenticado el CLI de GitHub
```

---

## Paso 1 — Crear los repositorios vacíos en GitHub

Usa el mismo owner que ya tienes en `BautistaGo-Backend` (`IngOscar19`), o cámbialo por tu organización si prefieres centralizar ahí los 3 repos.

```bash
gh repo create IngOscar19/BautistaGo-Web \
  --public \
  --description "BautistaGo — Panel Administrador y Pantalla de Despacho (Next.js PWA)" \
  --clone

gh repo create IngOscar19/BautistaGo-Mobile \
  --public \
  --description "BautistaGo — Apps móviles (Cliente y Repartidor, Expo/React Native)" \
  --clone
```

Usa `--private` en vez de `--public` si el repo del backend también es privado (verifica con `gh repo view IngOscar19/BautistaGo-Backend`). Esto deja dos carpetas locales, `BautistaGo-Web/` y `BautistaGo-Mobile/`, ya clonadas y conectadas a `origin`.

---

## Paso 2 — Inicializar cada monorepo con pnpm workspaces + Turborepo

Repite esto dentro de **cada** carpeta (`BautistaGo-Web` y `BautistaGo-Mobile`):

```bash
cd BautistaGo-Web   # (o BautistaGo-Mobile)

pnpm init
pnpm add -D turbo -w

mkdir -p apps packages
```

Crea `pnpm-workspace.yaml` en la raíz:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

Crea `turbo.json` en la raíz:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "type-check": {}
  }
}
```

Añade un `.gitignore` raíz (Node + Next + Expo):

```
node_modules/
.turbo/
.next/
dist/
.expo/
.env
.env.local
*.log
```

Haz el primer commit de la base del monorepo:

```bash
git add -A
git commit -m "chore: inicializa monorepo con pnpm workspaces + turborepo"
git push origin main
```

---

## Paso 3 — Proyectos base dentro de `BautistaGo-Web`

### 3.1 App Admin (Next.js)

```bash
cd apps
pnpm create next-app@latest admin \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --use-pnpm
```

### 3.2 App Cocina (Next.js)

```bash
pnpm create next-app@latest kitchen \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --use-pnpm
```

### 3.3 Paquetes compartidos

```bash
cd ../packages
mkdir ui api-client ws-client config
```

En cada uno, un `package.json` mínimo, por ejemplo `packages/ui/package.json`:

```json
{
  "name": "@bautistago/ui",
  "version": "0.0.0",
  "main": "./index.ts",
  "types": "./index.ts",
  "license": "UNLICENSED"
}
```

Repite el patrón (nombre `@bautistago/<paquete>`) para `api-client`, `ws-client` y `config`. Luego, desde cada app (`apps/admin`, `apps/kitchen`), agrégalos como dependencia de workspace:

```bash
cd ../../apps/admin
pnpm add @bautistago/ui@workspace:* @bautistago/api-client@workspace:* @bautistago/ws-client@workspace:*
```

### 3.4 Instalar dependencias clave de cada app
```bash
# en apps/admin y apps/kitchen
pnpm add socket.io-client axios zustand
pnpm add -D @types/node
```
Para `apps/admin` agrega además una librería de gráficas (RF5/CDU_8):
```bash
pnpm add recharts
```

### 3.5 Commit y push
```bash
cd ../..
git add -A
git commit -m "feat: scaffolding inicial de admin y kitchen (Next.js)"
git push origin main
```

---

## Paso 4 — Proyectos base dentro de `BautistaGo-Mobile`

### 4.1 App Repartidor (Expo) — obligatoria según ERS

```bash
cd apps
pnpm dlx create-expo-app@latest driver --template blank-typescript
cd driver
pnpm add expo-location expo-task-manager expo-sqlite socket.io-client
pnpm add react-native-maps   # o @maplibre/maplibre-react-native para OSM/MapLibre
cd ..
```

### 4.2 App Cliente (Expo)

```bash
# dentro de BautistaGo-Mobile/apps
pnpm dlx create-expo-app@latest customer --template blank-typescript
cd customer
pnpm add expo-location socket.io-client
pnpm add react-native-maps   # o @maplibre/maplibre-react-native
cd ..
```

### 4.3 Paquetes compartidos (igual que en Web)
```bash
cd ../packages
mkdir ui api-client ws-client geo config
```
`geo` es exclusivo de Mobile: aquí vive la lógica de MapLibre/OSM + el patrón Outbox con SQLite (ver `03-app-repartidor.md`, sección 6, tareas 5–7).

### 4.4 Commit y push
```bash
cd ../..
git add -A
git commit -m "feat: scaffolding inicial de driver (y customer si aplica) con Expo"
git push origin main
```

---

## Paso 5 — Conectar cada frontend al backend ya existente

1. En `BautistaGo-Backend`, confirma/crea un archivo `.env.example` con las variables que los frontends necesitan conocer (puerto de la API, URL base de WS). No subas `.env` reales a ningún repo.
2. En cada app frontend, crea `.env.local` (Next.js) o `.env` + `app.config.ts` (Expo) con:
   ```
   NEXT_PUBLIC_API_URL=http://localhost:4000/api
   NEXT_PUBLIC_WS_URL=http://localhost:4000
   ```
   (equivalente con prefijo `EXPO_PUBLIC_` en las apps Expo).
3. Verifica CORS en el backend: debe permitir origin de `localhost:3000` (admin), `localhost:3001` (kitchen) si corres varias apps Next.js a la vez — ajusta puertos con `-p` en el script `dev` de cada `package.json`.

---

## Paso 6 — Automatizar con OpenCode / Antigravity

Para que el agente de codificación no tenga que redescubrir la estructura en cada sesión, crea un archivo de contexto en la raíz de cada repo (`BautistaGo-Web/AGENTS.md` y `BautistaGo-Mobile/AGENTS.md`) con:
- La tabla de apps/paquetes de este documento.
- La paleta de colores y tipografías (sección 3.1.1 de la ERS, ya resumida en `00-estrategia-repositorios.md`).
- Los contratos de API/WS copiados del backend (rutas de `src/http/routes/*.ts`, eventos de `TelemetryGateway.ts`).
- Instrucción explícita: "trabaja una app a la vez, un caso de uso (CDU) a la vez; no reestructures carpetas sin pedirlo".

Luego abre OpenCode/Antigravity apuntando a la carpeta del repo correspondiente (`BautistaGo-Web/apps/kitchen`, por ejemplo) y usa como primer prompt el contenido del plan de esa app (`01-web-cocina.md`, tarea por tarea de la tabla de la sección 6).

---

## Checklist final de esta sesión de setup
- [ ] `BautistaGo-Web` creado, con `admin/` y `kitchen/` corriendo en local (`pnpm --filter admin dev`, `pnpm --filter kitchen dev`).
- [ ] `BautistaGo-Mobile` creado, con `driver/` corriendo en Expo Go (`pnpm --filter driver start`).
- [ ] Decisión tomada sobre dónde vive Cliente (Opción A o B del paso 4.2) y `02-app-cliente.md` actualizado si cambiaste de opción.
- [ ] `.env.example` del backend compartido con los frontends (sin secretos reales).
- [ ] `AGENTS.md` creado en `BautistaGo-Web` y `BautistaGo-Mobile`.


