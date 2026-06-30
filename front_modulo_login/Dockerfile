# Etapa 1: build (contexto: raíz de hub-frontends/)
FROM node:20-alpine AS builder
WORKDIR /app

# Instalar dependencias del módulo
COPY front_modulo_login/package*.json ./front_modulo_login/
COPY packages/ ./packages/
COPY shared_components/ ./shared_components/

WORKDIR /app/front_modulo_login
RUN npm ci

COPY front_modulo_login/ .
RUN npm run build

# Etapa 2: servir con Nginx
FROM nginx:alpine
COPY --from=builder /app/front_modulo_login/dist /usr/share/nginx/html/login
COPY front_modulo_login/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
