# hub-frontends

Frontends (React/Vite) del Hub Empresarial. Cada módulo es una SPA independiente, servida por Nginx bajo su propio path, que consume las APIs del repo [`hub-backends`](https://github.com/benjaminAndaur/hub-backends).

Repos relacionados:
- [`hub-backends`](https://github.com/benjaminAndaur/hub-backends) — microservicios Python/Quart
- [`hub-infra`](https://github.com/benjaminAndaur/hub-infra) — nginx, base de datos y `docker-compose.yml`

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
