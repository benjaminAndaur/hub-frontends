FROM node:20-alpine

WORKDIR /app

WORKDIR /app/front_modulo_mantencion

COPY front_modulo_mantencion/package.json ./
COPY packages/ui ../packages/ui
COPY shared_components ../shared_components
RUN npm install

COPY front_modulo_mantencion ./

EXPOSE 3000

CMD ["npm", "run", "dev"]
