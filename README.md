# Biblioteca · API REST y Sitio Web (`library_api`)

Proyecto de **API REST** y **Sitio Web** construido con **Next.js 16 (App Router, Route Handlers, `cacheComponents: true`, `proxy.ts`)**, **React 19**, **TypeScript**, **Firebase Admin SDK 13 (Authentication + Firestore)**, **Zod 4**, **Tailwind CSS** y **Vitest 5** para la Unidad 03 (Semanas 9 a 12 · Guía 1).

---

## 1. Enlaces en Producción (Vercel) y Repositorios

- **Sitio Web en Producción**: [https://library-api-guia1.vercel.app](https://library-api-guia1.vercel.app)
- **Documentación Interactiva (Swagger UI)**: [https://library-api-guia1.vercel.app/api/docs](https://library-api-guia1.vercel.app/api/docs)
- **Especificación OpenAPI 3.1**: [https://library-api-guia1.vercel.app/api/openapi.json](https://library-api-guia1.vercel.app/api/openapi.json)
- **Estado del Servidor (`health`)**: [https://library-api-guia1.vercel.app/api/health](https://library-api-guia1.vercel.app/api/health)
- **Repositorio GitHub (Guía 1 - API y Web)**: [https://github.com/TheStevenDog/Library-api-Guia1](https://github.com/TheStevenDog/Library-api-Guia1)
- **Repositorio GitHub (Guía 2 - App Móvil)**: [https://github.com/TheStevenDog/Library-app-Guia2](https://github.com/TheStevenDog/Library-app-Guia2)

---

## 2. Credenciales de Prueba (Estudiante y Bibliotecario)

Estas cuentas están activas en Firebase Authentication y Firestore con sus respectivos *custom claims* (`role`) para probar tanto en el **Sitio Web**, en **Swagger UI (`/api/docs`)** como en la **App Móvil (`library_app`)**:

| Rol | Correo | Contraseña | Permisos principales |
| :--- | :--- | :--- | :--- |
| **Estudiante (`student`)** | `estudiante@biblioteca.edu.co` | `Estudiante123` | Consultar catálogo, buscar/filtrar libros, solicitar préstamos y ver sus propios préstamos |
| **Bibliotecario (`librarian`)** | `bibliotecario@biblioteca.edu.co` | `Bibliotecario123` | Consultar catálogo, ver todos los préstamos, aprobar/rechazar/registrar devolución y cambiar portadas |

---

## 3. Cómo ejecutar y verificar el proyecto en local

### 3.1 Instalar dependencias y variables de entorno
```bash
pnpm install
# Copiar .env.example a .env y completar las 4 variables de Firebase
```

### 3.2 Cargar datos iniciales y gestionar usuarios
```bash
pnpm seed
pnpm create-user -- estudiante@biblioteca.edu.co "Estudiante123" "Estudiante Prueba" student
pnpm create-user -- bibliotecario@biblioteca.edu.co "Bibliotecario123" "Bibliotecario Prueba" librarian
pnpm reset-demo
```

### 3.3 Levantar el servidor de desarrollo
```bash
pnpm dev       # En localhost:3000
pnpm dev:lan   # Escuchando en 0.0.0.0:3000 para conectar el teléfono en la misma red Wi-Fi
```

### 3.4 Comandos de verificación y pruebas (Puntos de control)
```bash
pnpm typecheck
pnpm test
pnpm lint
pnpm build
pnpm smoke     # Ejecuta las 41 comprobaciones E2E contra Firebase real
```

---

## 4. Endpoints de la API REST

| Método y ruta | Acceso | Descripción | Semana |
| :--- | :--- | :--- | :---: |
| `GET /api/health` | Público | Comprueba que el servidor responde | 9 |
| `GET /api/openapi.json` | Público | Especificación OpenAPI 3.1 generada desde los esquemas Zod | Cap. 8 |
| `GET /api/docs` | Público | Documentación interactiva (Swagger UI) | Cap. 8 |
| `GET /api/books?q=&category=&updatedSince=` | Autenticado | Lista de libros, categorías y `serverTime` (cursor incremental) | 9 y 10 |
| `GET /api/books/:id` | Autenticado | Detalle de un libro | 9 y 10 |
| `POST /api/auth/register` | Público | Crea un estudiante (`role: "student"`) y devuelve tokens | 10 |
| `POST /api/auth/login` | Público | Inicia sesión con correo y contraseña | 10 |
| `POST /api/auth/refresh` | Público | Intercambia un `refreshToken` por un `idToken` nuevo | 10 |
| `GET /api/auth/me` | Autenticado | Devuelve el usuario actual y su rol | 10 |
| `GET /api/loans?status=` | Autenticado | Estudiante: solo los suyos · Bibliotecario: todos | 11 |
| `POST /api/loans` | Estudiante | Solicita un préstamo de forma idempotente por `clientRequestId` | 11 |
| `GET /api/loans/:id` | Autenticado | Detalle de un préstamo (`404` si es ajeno para un estudiante) | 11 |
| `PATCH /api/loans/:id` | Bibliotecario | Transición de estado: `approve`, `reject` o `return` | 11 |
| `GET /api/books/:id/cover` | Público | Sirve la imagen de portada con caché inmutable (`?v=`) | 12 |
| `PUT /api/books/:id/cover` | Bibliotecario | Valida *magic bytes* reales (JPEG/PNG/WebP) y guarda la portada | 12 |

---

## 5. Matriz de Autorización y Amenazas Controladas

| Amenaza | Control implementado |
| :--- | :--- |
| **Secretos de administrador filtrados** | Credenciales solo en variables de entorno (`lib/env.ts`); `.env*` y `*-firebase-adminsdk-*.json` ignorados en `.gitignore`. Reglas de Firestore en `allow read, write: if false;`. |
| **Escalada de privilegios al registrarse** | `/api/auth/register` asigna siempre `{ role: "student" }` mediante *custom claims* de Firebase Admin. El rol `librarian` solo se asigna con `scripts/create-user.mts` o `scripts/set-role.mts`. |
| **Un estudiante invoca rutas de bibliotecario** | `assertRole(user, "librarian")` verifica el claim firmado en el JWT en cada ruta (`403 FORBIDDEN`). |
| **Enumeración de correos y préstamos ajenos** | Mensaje único para credenciales inválidas (`INVALID_CREDENTIALS`) y respuesta `404 NOT_FOUND` (en lugar de `403`) cuando un estudiante consulta un préstamo de otro usuario. |
| **Archivos maliciosos disfrazados de imagen** | `sniffImageType()` en `lib/image.ts` verifica los *magic bytes* reales (`0xFFD8FF`, `0x89504E47`, `RIFF...WEBP`) antes de guardar en Firestore. |

---

## 6. Bitácora de Uso de IA Generativa (Sección 11)

| Fecha / Semana | Herramienta y prompt (resumido) | Qué respondió | Cómo lo validé | Decisión |
| :--- | :--- | :--- | :--- | :--- |
| **Semana 9** | *"Cómo envolver Route Handlers en Next.js 16 para unificar errores de Zod y de negocio sin romper el prerender"* | Sugirió capturar todas las excepciones en un bloque `try/catch` pero inicialmente omitió re-lanzar las excepciones internas de Next.js. | Al probar con `cacheComponents: true`, verifiqué que `unstable_rethrow(err)` de `next/navigation` es necesario antes de convertir el error en JSON. Lo validé con `tests/http.test.ts` y `pnpm build`. | **Aceptada con ajustes**: incluida `unstable_rethrow(err)` en `lib/http.ts`. |
| **Semana 10** | *"Cómo verificar un idToken de Firebase en Next.js y leer el rol del usuario"* | Sugirió usar `adminAuth().verifyIdToken(token)` del Admin SDK y leer el custom claim `decoded.role`. | Contrasté con la documentación de Firebase Admin SDK y escribí las pruebas unitarias en `tests/guard.test.ts` simulando tokens inválidos, falta de cabecera Bearer y roles distintos. | **Aceptada**: implementada en `lib/services/auth.ts` y `lib/guard.ts`. |
| **Semana 11** | *"Escribe pruebas unitarias para la máquina de estados de préstamos y límite de ejemplares"* | Generó los casos para `assertCanRequest` y `applyAction` (`approve`, `reject`, `return`). | Ejecuté `pnpm test` (`tests/loan-rules.test.ts`) comprobando los 12 casos, incluyendo que `return` nunca supere `totalCopies` y que `ALREADY_ACTIVE` / `NO_COPIES` devuelvan `409`. | **Aceptada**: integrada en `tests/loan-rules.test.ts`. |
| **Semana 12** | *"Cómo validar magic bytes de JPEG, PNG y WebP en TypeScript sin librerías nativas"* | Propujo inspeccionar los primeros bytes del `Uint8Array` para las firmas hexadecimales de JPEG, PNG y RIFF/WEBP. | Validé con `tests/image.test.ts` y con `pnpm smoke` enviando un HTML disfrazado de `image/png` (rechazado con `400`). | **Aceptada**: implementada en `lib/image.ts`. |
