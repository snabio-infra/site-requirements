Dockerfile
```dockerfile
# ---------- build stage ----------
FROM node:20-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build


# ---------- runtime stage ----------
FROM nginx:alpine

# удаляем дефолтный конфиг
RUN rm /etc/nginx/conf.d/default.conf

# копируем свой конфиг
COPY nginx.conf /etc/nginx/conf.d/default.conf

# копируем build
COPY --from=build /app/build /usr/share/nginx/html
```

nginx.conf
```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # React Router
    location / {
        try_files $uri /index.html;
    }

    # кеширование
    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

docker-compose.yml
```yaml
services:
  FRONT_NAME:
    build: .
    container_name: FRONT_NAME
    restart: unless-stopped
    networks:
      - web
	environment:
      - VIRTUAL_HOST=front1.example.com
      - LETSENCRYPT_HOST=front1.example.com
      - LETSENCRYPT_EMAIL=admin@example.com

networks:
  web:
    external: true
```

`nginx` на хостовой машине живёт так же в контейнере и общается со всеми другими контейнерами в контексте проксирования внутри сети web.

Пример location - `http://front1:80` /  `http://front2:80` / `http://front3:80`

Вот как выглядит конфиг `nginx` на хостовой машине
```nginx
server {
    listen 80;
    server_name front1.example.com;

    location / {
        proxy_pass http://front1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name front2.example.com;

    location / {
        proxy_pass http://front2:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
