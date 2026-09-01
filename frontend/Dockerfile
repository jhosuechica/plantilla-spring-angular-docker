# =============================================================================
# Frontend Angular - build multi-etapa
#
# Etapa 1: compila el bundle de produccion con Node.
# Etapa 2: sirve los archivos estaticos con Nginx.
#
# Node no existe en la imagen final: Angular compilado son solo archivos
# estaticos, no necesita un runtime de JavaScript en el servidor.
# =============================================================================

ARG NODE_VERSION=20

# ---------- Etapa 1: build ----------
FROM node:${NODE_VERSION}-alpine AS builder
WORKDIR /build

# npm ci (no npm install): instala exactamente lo que dice package-lock.json.
# Builds reproducibles: la misma imagen hoy y dentro de seis meses.
COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# La ruta de salida de Angular cambia segun la version:
#   Angular 17+  -> dist/<app>/browser
#   Angular <=16 -> dist/<app>
# En vez de exigir que se configure a mano, la localizamos buscando el
# index.html generado. Se excluye /server/ porque los proyectos con SSR
# tambien generan un index alli, y ese no es el que sirve Nginx.
RUN npm run build -- --configuration=production \
    && INDEX="$(find /build/dist -name index.html | grep -v '/server/' | head -n 1)" \
    && if [ -z "$INDEX" ]; then \
         echo "ERROR: no se encontro index.html dentro de dist/"; \
         echo "Contenido generado:"; ls -R /build/dist; exit 1; \
       fi \
    && DIST_DIR="$(dirname "$INDEX")" \
    && echo ">> Archivos estaticos tomados de: $DIST_DIR" \
    && mkdir -p /build/web \
    && cp -a "$DIST_DIR"/. /build/web/

# ---------- Etapa 2: runtime ----------
FROM nginx:1.27-alpine

RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /build/web/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q --spider http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
