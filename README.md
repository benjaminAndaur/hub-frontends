# hub-frontends

Frontends (React/Vite) del Hub Empresarial. Cada módulo es una SPA independiente, servida por Nginx bajo su propio path, que consume las APIs de [`hub-backends`](https://github.com/benjaminAndaur/hub-backends), [`hub-ms-operacion`](https://github.com/benjaminAndaur/hub-ms-operacion) y [`hub-ms-facturacion`](https://github.com/benjaminAndaur/hub-ms-facturacion).

Repos relacionados:
- [`hub-backends`](https://github.com/benjaminAndaur/hub-backends) — microservicios Python/Quart
- [`hub-infra`](https://github.com/benjaminAndaur/hub-infra) — nginx, base de datos y `docker-compose.yml`
- [`hub-ms-operacion`](https://github.com/benjaminAndaur/hub-ms-operacion) — microservicio de viajes, BD propia
- [`hub-ms-facturacion`](https://github.com/benjaminAndaur/hub-ms-facturacion) — microservicio de facturación, BD propia
- [`hub-bff`](https://github.com/benjaminAndaur/hub-bff) — BFF (NestJS) que agrega Operación + Facturación; aún sin módulo frontend propio que lo consuma (ver nota abajo)

## Módulos incluidos

| Módulo | Path en gateway | Dominio |
|---|---|---|
| `front_modulo_login` | `/login/` | Login centralizado |
| `front_modulo_administracion` | `/` | Usuarios y permisos |
| `front_modulo_rrhh` | `/rrhh/` | Personal / RRHH |
| `front_modulo_mantencion` | `/mantencion/` | Mantención de vehículos |
| `front_modulo_acreditacion` | `/acreditacion/` | Acreditación de clientes |
| `front_modulo_operacion` | `/operacion/` | Viajes / Operaciones |
| `front_modulo_bodega` | `/bodega/` | Bodega / Inventario |
| `front_modulo_facturacion` | `/facturacion/` | Facturación |
| `front_modulo_prevencion` | `/prevencion/` | Prevención / Incidentes |
| `front_modulo_watchdog` | `/watchdog/` | Monitoreo de servicios |

> No existe (todavía) un módulo frontend para el BFF (`hub-bff`). El endpoint `GET /api/v1/bff/dashboard` está disponible para consumo programático en `http://localhost:8080/api/v1/bff/dashboard`, pero no tiene una UI dedicada — pendiente si se decide construir un dashboard agregado.

## Código compartido

| Paquete | Contenido | Usado por |
|---|---|---|
| `packages/ui` (`@hub/ui`) | `AppShell`, `ModuleHeader`, `Tabs`, preset de Tailwind, `styles.css` | rrhh, mantencion, operacion, acreditacion |
| `shared_components/` | `Header.jsx`, `Sidebar.jsx`, `LoginForm.jsx`, `apiClient.js`, `auth.js` | todos los módulos |

`shared_components/auth.js` expone `checkModuleAccess('nombre_modulo', '/ruta/')`, que cada `App.jsx` llama al montar para leer el JWT desde `localStorage` y redirigir si el usuario no tiene permiso sobre ese módulo.

## Cómo ejecutar un módulo individual

```bash
cd front_modulo_rrhh   # o cualquier otro front_modulo_*
npm install
npm run dev
```

> Requiere que el backend correspondiente y el gateway Nginx estén corriendo (ver [`hub-infra`](https://github.com/benjaminAndaur/hub-infra) para levantar el stack completo con `docker-compose up --build`).

## Variables de entorno

| Variable | Valor en dev |
|---|---|
| `VITE_API_URL` | `/api/v1` (inyectado en build; Nginx lo proxifica al microservicio correspondiente) |

## Arquitectura de cada módulo

```
front_modulo_x/
├── package.json
├── Dockerfile
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
└── src/
    ├── App.jsx               # navegación por pestañas (useState); checkModuleAccess() al montar
    ├── main.jsx
    ├── components/
    │   ├── {Entidad}Form.jsx
    │   ├── {Entidad}List.jsx
    │   └── {Entidad}StatusManage.jsx
    ├── services/
    │   └── {entidad}Service.js   # fetch con Authorization: Bearer {token} hacia VITE_API_URL
    └── __tests__/
```

- No se usa React Router: la navegación es por `activeTab` con renderizado condicional.
- Los botones de acción se muestran u ocultan según `permiso === 'edit'` (leído del JWT).
- `Sidebar`, `Header` y `LoginForm` provienen de `shared_components/`.

## Flujo de autenticación (lado frontend)

1. El usuario hace login en `front_modulo_login` → `POST /api/v1/administracion/login`.
2. El backend retorna un JWT con `{sub, email, rol, permisos: {modulo: "none"|"view"|"edit"}}`.
3. El frontend guarda `token` y `userData` en `localStorage`.
4. Cada request posterior incluye `Authorization: Bearer {token}` (vía `apiClient.js`).
5. Al montar, cada módulo llama `checkModuleAccess()` y redirige si `permisos[modulo] === "none"`.

No hay tests automatizados end-to-end; algunos módulos incluyen tests unitarios con Vitest (`vitest.config.js`, `src/__tests__/`).

---

## Despliegue en AWS — DevOps (ISY1101)

### Arquitectura en producción

```
Internet → ALB (puerto 80)
  /* (default) → ECS Fargate Task: hub-frontend (puerto 80)  ← este servicio
  /api/*       → ECS Fargate Task: hub-bff (puerto 3000)
```

- **Cluster:** `hub-empresarial-cluster` (AWS ECS Fargate)
- **Imagen:** `720243276279.dkr.ecr.us-east-1.amazonaws.com/hub-frontend:<sha>` (26.05 MB)
- **Task Definition:** 256 CPU units / 512 MB RAM, role = `LabRole`
- **Dockerfile:** multietapa — `node:20-alpine` (build Vite) → `nginx:alpine` (serve SPA)
- **URL pública:** `http://hub-empresarial-alb-1969847223.us-east-1.elb.amazonaws.com`

### Pipeline CI/CD (GitHub Actions)

Push a `main` → `.github/workflows/deploy.yml`. **Iteración real:**

| Run | Estado | Causa | Duración |
|-----|--------|-------|----------|
| #1 | ❌ Fallido | No existía `package-lock.json` — `npm ci` falló | 13s |
| #2 | ✅ Exitoso | Fix: cambiar `npm ci` por `npm install` en Dockerfile | 3m 52s |

El run fallido y su corrección demuestran el ciclo real de desarrollo iterativo con CI/CD.

### Dockerfile multietapa

```dockerfile
# Etapa 1: build de producción
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Etapa 2: servidor estático optimizado
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

Resultado: imagen de 26 MB sirviendo la SPA compilada — sin Node.js en producción.

### Verificar despliegue

```bash
# Frontend carga desde ECS
curl -I http://hub-empresarial-alb-1969847223.us-east-1.elb.amazonaws.com/
# HTTP/1.1 301 → /login/ (correcto)

curl -I http://hub-empresarial-alb-1969847223.us-east-1.elb.amazonaws.com/login/
# HTTP/1.1 200 OK
```

