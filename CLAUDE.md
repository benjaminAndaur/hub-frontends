# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Flujo de Git (GitFlow)

Este repo (y los otros dos del Hub: `hub-infra`, `hub-backends`) usa **GitFlow**. Reglas:

- `main` — solo recibe merges desde `develop` o `release/*`. Representa lo desplegable/estable. **Nunca commitear directo aquí.**
- `develop` — rama de integración. Todo el trabajo nuevo (fixes, features) se commitea o mergea aquí primero.
- `feature/*`, `fix/*` — ramas de trabajo creadas desde `develop`, mergeadas de vuelta a `develop`.
- `release/*` — ramas de preparación de release creadas desde `develop`, mergeadas a `main` y de vuelta a `develop`.
- `hotfix/*` — para bugs urgentes en producción: se crean desde `main`, se mergean a **ambos** `main` y `develop`.

Antes de empezar a trabajar, verificar en qué rama se está parado (`git branch --show-current`). Si hay cambios sin commitear directo en `main`, moverlos a `develop` (o a una rama `fix/*`/`feature/*` desde `develop`) antes de commitear.

## Resumen del proyecto

Frontends (React/Vite) del Hub Empresarial. Cada módulo es una SPA independiente, servida por Nginx bajo su propio path, que consume las APIs del repo `hub-backends`. Ver `README.md` para el detalle de módulos, código compartido (`packages/ui`, `shared_components/`), arquitectura por módulo y flujo de autenticación.

Repos relacionados: [`hub-infra`](https://github.com/benjaminAndaur/hub-infra) (nginx, base de datos, `docker-compose.yml`), [`hub-backends`](https://github.com/benjaminAndaur/hub-backends).
