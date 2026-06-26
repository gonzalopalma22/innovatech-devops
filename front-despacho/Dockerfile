# Etapa 1: Build
FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --silent

COPY . .

ARG VITE_API_VENTAS
ARG VITE_API_DESPACHOS
ENV VITE_API_VENTAS=$VITE_API_VENTAS
ENV VITE_API_DESPACHOS=$VITE_API_DESPACHOS

RUN npm run build

# Etapa 2: Serve con Nginx
FROM nginx:1.25-alpine

RUN rm /etc/nginx/conf.d/default.conf

COPY nginx.conf /etc/nginx/conf.d/default.conf

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]