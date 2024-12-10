# the prebuilder compiles only what's needed to install deps
# we want them to be cached for more efficient deploys when only modifying code
FROM node:lts-alpine AS prebuild

WORKDIR /app
COPY ["package.json", "yarn.lock", "./"]
COPY apps apps
COPY packages packages

# TODO: we should delete empty directories here too. it should be more flexible than -mindepth. until this is fixed it's possible that the dependency layer is recomputed when directory structure changes occur in directories without a package.json
RUN find packages apps -type d -name "thirdparty" -prune -o -type f -not -name "package.json" -print | xargs rm -rf

FROM node:lts-alpine AS builder

# install deps
WORKDIR /app
COPY --from=prebuild /app .
RUN yarn install  --immutable --immutable-cache --network-timeout 1000000


# bring in src and restore workspace symlinks
COPY . .
RUN yarn install --immutable --immutable-cache

ENV NEXT_PUBLIC_ENTE_ENDPOINT=DOCKER_RUNTIME_REPLACE_ENDPOINT
ENV NEXT_PUBLIC_ENTE_ALBUMS_ENDPOINT=DOCKER_RUNTIME_REPLACE_ALBUMS_ENDPOINT

# build
RUN yarn build

FROM nginx:alpine

COPY --from=builder /app/apps/photos/out /usr/share/nginx/html

COPY <<EOF /etc/nginx/conf.d/default.conf
server {
  listen 80 default_server;
  root /usr/share/nginx/html;
  location / {
      try_files \$uri \$uri.html \$uri/ =404;
  }
  error_page 404 /404.html;
  location = /404.html {
      internal;
  }
}
EOF

ARG ENDPOINT="http://localhost:8080"
ENV ENDPOINT="$ENDPOINT"

ARG ALBUMS_ENDPOINT="http://localhost:8082"
ENV ALBUMS_ENDPOINT="$ALBUMS_ENDPOINT"

COPY <<EOF /docker-entrypoint.d/replace_ente_endpoints.sh
echo "Replacing endpoints"
echo "  Endpoint: \$ENDPOINT"
echo "  Albums Endpoint: \$ALBUMS_ENDPOINT"

replace_enpoints() {
  sed -i -e 's,DOCKER_RUNTIME_REPLACE_ENDPOINT,'"\$ENDPOINT"',g' \$1
  sed -i -e 's,DOCKER_RUNTIME_REPLACE_ALBUMS_ENDPOINT,'"\$ALBUMS_ENDPOINT"',g' \$1
}
for jsfile in `find '/usr/share/nginx/html' -type f -name '*.js'`
do
    replace_enpoints "\$jsfile"
done
EOF

RUN chmod +x /docker-entrypoint.d/replace_ente_endpoints.sh