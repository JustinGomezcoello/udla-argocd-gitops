# ---------- Etapa 1: build de la aplicación React ----------
FROM node:20-alpine AS build

WORKDIR /app

# Dependencias primero para aprovechar la caché de capas
COPY package*.json ./
RUN npm install --legacy-peer-deps --no-audit --no-fund

# Código fuente y compilación de producción
COPY . .
ENV CI=false
RUN npm run build

# ---------- Etapa 2: servidor estático nginx ----------
FROM nginx:alpine

COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
