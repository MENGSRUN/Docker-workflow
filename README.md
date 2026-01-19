# Docker Workflow

This project builds Laravel + Vue assets in a multi-stage Docker build, runs PHP-FPM for the app, and serves static files + proxy via Nginx. MySQL runs as a separate service with a persistent volume.

## Files (current)

### Dockerfile
```Dockerfile
# ---------------------------------------------
# Stage 1: Node.js build stage for Laravel Mix
# ---------------------------------------------
FROM node:18-alpine AS node-builder

WORKDIR /app

# Install build tools
RUN apk add --no-cache python3 make g++

# Copy package files and config
COPY package*.json ./
COPY webpack.mix.js* ./
COPY tailwind.config.js* ./

# Install dependencies
RUN npm ci

# Copy source files
COPY resources/ ./resources/
COPY public/images ./public/images
COPY public/themes ./public/themes

# Build assets
RUN npm run prod

# ---------------------------------------------
# Stage 2: PHP + Laravel (FPM)
# ---------------------------------------------
FROM php:8.2-fpm-alpine AS app

WORKDIR /var/www

# System dependencies
RUN apk add --no-cache \
    git curl libpng-dev libxml2-dev zip unzip bash \
    mysql-client sqlite netcat-openbsd

# PHP extensions installer
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/
RUN install-php-extensions pdo_mysql pdo_sqlite mbstring exif pcntl bcmath gd redis zip opcache

# Composer
COPY --from=composer:2.7.1 /usr/bin/composer /usr/bin/composer

# PHP production settings
RUN cp "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" \
    && echo "memory_limit = 512M" >> /usr/local/etc/php/conf.d/app.ini \
    && echo "upload_max_filesize = 100M" >> /usr/local/etc/php/conf.d/app.ini \
    && echo "post_max_size = 100M" >> /usr/local/etc/php/conf.d/app.ini \
    && echo "max_execution_time = 300" >> /usr/local/etc/php/conf.d/app.ini \
    && echo "opcache.enable = 1" >> /usr/local/etc/php/conf.d/app.ini \
    && echo "opcache.validate_timestamps = 0" >> /usr/local/etc/php/conf.d/app.ini

# Copy Laravel app
COPY . .

# EnvEditor backup dir for production
RUN mkdir -p resources/backups/dotenv-editor \
    && chown -R www-data:www-data resources/backups \
    && chmod -R 775 resources/backups

# Copy built assets from node-builder
COPY --from=node-builder /app/public ./public

# Laravel required directories
RUN mkdir -p bootstrap/cache storage/framework/{cache,sessions,views} storage/logs \
    && chmod -R 775 bootstrap/cache storage

# Composer dependencies
RUN composer install --optimize-autoloader --no-interaction --no-dev --prefer-dist

# Entrypoint script
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

EXPOSE 9000

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["php-fpm"]

# ---------------------------------------------
# Stage 3: Nginx for static + proxy
# ---------------------------------------------
FROM nginx:1.25-alpine AS nginx

COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=app /var/www/public /var/www/public
RUN ln -s /var/www/storage/app/public /var/www/public/storage
```

### docker-compose.yml
```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: app
    image: leng-kheang-prod-ecommerce
    container_name: leng-kheang-prod-ecommerce
    restart: unless-stopped
    working_dir: /var/www
    expose:
      - "9000:9000"
    environment:
      APP_ENV: ${APP_ENV}
      DB_HOST: mysql-prod-db
      DB_PORT: 3306
      DB_DATABASE: ${DB_DATABASE}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
    env_file:
      - .env.prod
    volumes:
      - app_storage:/var/www/storage
      - ./.env.prod:/var/www/.env
    networks:
      - kilo-net
    depends_on:
      mysql-prod-db:
        condition: service_healthy

  nginx:
    build:
      context: .
      dockerfile: Dockerfile
      target: nginx
    image: leng-kheang-prod-nginx
    container_name: leng-kheang-prod-nginx
    restart: unless-stopped
    ports:
      - "8100:80"
    volumes:
      - app_storage:/var/www/storage
    networks:
      - kilo-net
    depends_on:
      - app
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  mysql-prod-db:
    image: mysql:8.2
    container_name: mysql-prod-db
    restart: unless-stopped
    ports:
      - "3307:3306"
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - kilo-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p$${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

#   cloudflared:
#     image: cloudflare/cloudflared:latest
#     container_name: cloudflared
#     restart: unless-stopped
#     command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
#     networks:
#       - kilo-net
volumes:
  mysql_data:
  app_storage:

networks:
  kilo-net:
    external: true
```

Notes:
- `kilo-net` is an external network. Create it once if it does not exist:
  `docker network create kilo-net`
- `.env.prod` is mounted into the app container at `/var/www/.env`. This is currently RW for quick setup; switch back to `:ro` for safer production.

### .dockerignore
```
node_modules
vendor
.git
.env
storage/logs/*
bootstrap/cache/*
.DS_Store
*.log
```

### webpack.mix.js
```js
const mix = require('laravel-mix');
/*
 |--------------------------------------------------------------------------
 | Mix Asset Management
 |--------------------------------------------------------------------------
 |
 | Mix provides a clean, fluent API for defining some Webpack build steps
 | for your Laravel applications. By default, we are compiling the CSS
 | file for the application as well as bundling up all the JS files.
 |
 */

mix.setPublicPath('public');

mix.js('resources/js/app.js', 'js')
    .vue()
    .postCss('resources/css/app.css', 'css', [require("tailwindcss")])
    .webpackConfig({
        output: {
            publicPath: '/',
            chunkFilename: 'js/[name].js',
        },
    });
```

### entrypoint.sh
```sh
#!/bin/sh
set -e

# Wait for external DB if needed
if [ "$DB_HOST" != "127.0.0.1" ] && [ "$DB_HOST" != "localhost" ]; then
    echo "Waiting for database at $DB_HOST:$DB_PORT..."
    while ! nc -z "$DB_HOST" "$DB_PORT"; do
        sleep 1
    done
    echo "Database is ready!"
fi

# Setup .env
if [ ! -f .env ]; then
    if [ "$APP_ENV" = "production" ]; then
        echo "Error: .env missing in production!"
        exit 1
    fi
    if [ -f .env.example ]; then
        cp .env.example .env
        echo "Created .env from .env.example"
    else
        echo "Error: No .env.example found!"
        exit 1
    fi
fi

# Generate app key if missing
if ! grep -q "^APP_KEY=.\+" .env; then
    if [ "$APP_ENV" = "production" ]; then
        echo "Error: APP_KEY missing in production!"
        exit 1
    fi
    php artisan key:generate --force
    echo "Generated application key"
fi

# Ensure storage symlink exists
if [ ! -e public/storage ]; then
    mkdir -p storage/app/public storage/framework/cache storage/framework/sessions storage/framework/views storage/logs
    php artisan storage:link || true
fi

# Clear caches in dev mode
if [ "$APP_ENV" = "local" ] || [ "$APP_ENV" = "development" ]; then
    echo "Development mode: clearing caches..."
    php artisan config:clear || true
    php artisan route:clear || true
    php artisan view:clear || true
    php artisan cache:clear || true
else
    echo "Production mode: caching configs..."
    php artisan config:cache
    php artisan route:cache
    php artisan view:cache
fi

# Fix permissions
chown -R www-data:www-data storage bootstrap/cache 2>/dev/null || true
chmod -R 775 storage bootstrap/cache 2>/dev/null || true

echo "Laravel application ready!"

exec "$@"
```

### .env.prod (example, secrets redacted)
Use your real `.env.prod` file for production secrets. Do not commit secrets.
```
APP_NAME="KiloIT - PWA eCommerce"
APP_ENV=production
APP_KEY=base64:REDACTED
APP_DEBUG=false
APP_URL=http://192.168.1.210:8100
APP_TIMEZONE=UTC
LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug
DB_CONNECTION=mysql
DB_HOST=mysql-prod-db
DB_PORT=3306
DB_DATABASE=lk_ecommerce_db
DB_USERNAME=lkecom
DB_PASSWORD=YOUR_DB_PASSWORD
DB_ROOT_PASSWORD=YOUR_ROOT_PASSWORD
BROADCAST_DRIVER=log
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
MEMCACHED_HOST=127.0.0.1
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
MAIL_MAILER=smtp
MAIL_HOST=
MAIL_PORT=0
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=
MAIL_FROM_ADDRESS=
MAIL_FROM_NAME=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false
AWS_ROOT=
AWS_URL=
PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=mt1
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
VITE_PUSHER_PORT="${PUSHER_PORT}"
VITE_PUSHER_SCHEME="${PUSHER_SCHEME}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
FCM_SECRET_KEY=REDACTED
FCM_TOPIC=
DEMO=false
TIMEZONE=Asia/Dhaka
MIX_HOST="${APP_URL}"
MIX_API_KEY=REDACTED
MIX_DEMO="${DEMO}"
CURRENCY=USD
CURRENCY_SYMBOL=$
CURRENCY_POSITION=5
CURRENCY_DECIMAL_POINT=2
DATE_FORMAT=d-m-Y
TIME_FORMAT="h:i A"
DISPLAY_TYPE=fashion
NON_PURCHASE_QUANTITY=100
MEDIA_DISK=public
```

## Workflow

### Build and run
```sh
docker compose --env-file .env.prod build --no-cache
docker compose --env-file .env.prod up -d
```

### Rebuild quickly
```sh
docker compose --env-file .env.prod up -d --build
```

### View logs
```sh
docker compose --env-file .env.prod logs -f app
docker compose --env-file .env.prod logs -f nginx
```

### Exec into containers
```sh
docker compose --env-file .env.prod exec app sh
docker compose --env-file .env.prod exec nginx sh
```

## Laravel tasks

```sh
docker compose --env-file .env.prod exec app php artisan migrate --force
docker compose --env-file .env.prod exec app php artisan db:seed --force
docker compose --env-file .env.prod exec app php artisan optimize:clear
```

`entrypoint.sh` already ensures `storage:link` when `public/storage` is missing.

## Storage and uploads

- `app_storage` volume maps to `/var/www/storage` and persists uploads.
- Nginx serves `/public/storage` via the symlink created in the image and/or entrypoint.

## Temporary RW .env

The compose file currently mounts `.env.prod` RW for quick setup:
```
- ./.env.prod:/var/www/.env
```
For production, switch to read-only:
```
- ./.env.prod:/var/www/.env:ro
```
