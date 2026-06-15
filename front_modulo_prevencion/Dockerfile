FROM node:20-alpine

WORKDIR /app

WORKDIR /app/front_modulo_prevencion

COPY front_modulo_prevencion/package*.json ./
COPY packages/ui ../packages/ui
COPY shared_components ../shared_components
RUN npm install

COPY front_modulo_prevencion ./

EXPOSE 3000

CMD ["npm", "run", "dev"]
