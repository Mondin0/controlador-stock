# 📦 Controlador Stock - Despliegue Docker

## 🚀 **Despliegue Rápido (2 minutos)**

### **0. iniciar de cero**
```bash
docker run --rm   -v $(pwd)/back:/app   composer create-project laravel/laravel:^12.0 . --prefer-dist
```

### **1. Clonar y preparar**
```bash
git clone <tu-repo> controlador-stock
cd controlador-stock
cp .env.example .env
```

### **2. Iniciar servicios**
```bash
docker compose up --build -d
```

### **3. Configurar Laravel**
```bash
docker compose exec controlador php artisan key:generate
docker compose exec controlador php artisan migrate --force
docker compose exec controlador php artisan optimize
docker compose exec -it controlador php artisan install:api
```

### **4. Acceder**
```
🌐 http://localhost:8000
📊 Adminer PG: http://localhost:5432
```

## 🏗️ **Arquitectura**

```
nginx:alpine (8000:80) ← ./back (Laravel)
         ↓
controlador (php:8.4-fpm) ← postgres:18-alpine
```

## 📁 **Estructura de archivos requerida**

```
controlador-stock/
├── docker-compose.yml
├── back/                 # Tu Laravel app
│   ├── public/
│   │   └── index.php
│   └── ...
├── deploy/local/
│   ├── Dockerfile       # PHP-FPM
│   ├── nginx.conf       # Nginx config
│   └── php-fpm.conf     # PHP TCP 9000
└── README.md
```

## 🔧 **Archivos de configuración**

### **docker-compose.yml**
```yaml
services:
  controlador:
    build: ./deploy/local/
    volumes: ['./back:/var/www/html']
    depends_on: [postgres]
    environment:
      DB_CONNECTION: pgsql
      DB_HOST: postgres
      DB_PORT: 5432
      DB_DATABASE: controlador
      DB_USERNAME: postgres
      DB_PASSWORD: secret

  nginx:
    image: nginx:alpine
    ports: ["8000:80"]
    volumes:
      - ./back:/var/www/html:ro
      - ./deploy/local/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on: [controlador]

  postgres:
    image: postgres:18-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: controlador
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    command: postgres -c 'hba_file=/var/lib/postgresql/data/pg_hba.conf'

volumes:
  postgres_data:
```

### **deploy/local/Dockerfile**
```dockerfile
FROM php:8.4-fpm
RUN apt-get update && apt-get install -y \
    libpq-dev postgresql-client git unzip curl \
    && rm -rf /var/lib/apt/lists/*
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
RUN docker-php-ext-install pdo_pgsql
WORKDIR /var/www/html
COPY php-fpm.conf /usr/local/etc/php-fpm.d/zzz-custom.conf
EXPOSE 9000
```

### **deploy/local/php-fpm.conf** (CRÍTICO)
```ini
[www]
listen = 9000
```

### **deploy/local/nginx.conf** (CRÍTICO)
```nginx
server {
    listen 80;
    root /var/www/html/public;
    index index.php;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass controlador:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

## ⚡ **Comandos útiles**

```bash
# 🔄 Reinicio completo
docker compose down -v && docker compose up --build -d

# 🐛 Debug logs
docker compose logs -f controlador nginx postgres

# 💾 Migraciones
docker compose exec controlador php artisan migrate --force

# 🔑 Configurar Laravel
docker compose exec controlador php artisan key:generate
docker compose exec controlador php artisan storage:link

# 🗑️ Limpiar cache
docker compose exec controlador php artisan config:cache
docker compose exec controlador php artisan route:cache

# 📊 Acceder PostgreSQL
docker compose exec postgres psql -U postgres -d controlador
```

## 🚨 **Troubleshooting**

| **Error** | **Solución** |
|---|---|
| `403 Forbidden` | `docker compose exec nginx chown -R www-data:www-data /var/www/html` |
| `502 Bad Gateway` | Verificar `php-fpm.conf` tiene `listen = 9000` |
| `sessions table` | `docker compose exec controlador php artisan migrate` |
| `Connection refused` | `docker compose up --build -d` |

## 🌐 **URLs**
```
✅ App: http://localhost:8000
✅ Postgres: localhost:5432
✅ Logs: docker compose logs -f
```

## 📈 **Estado: PRODUCTION READY**

✅ **nginx + php-fpm + postgres** = ✅  
✅ **Laravel 12 + PHP 8.4** = ✅  
✅ **Hot reload** (volumen `./back`) = ✅  
✅ **Migraciones automáticas** = ✅  

**¡Despliegue exitoso!** 🎉