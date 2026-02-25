# Controlador — Sistema de Gestión y Ventas para Locales

> Control de stock multi-local y tienda online con pago via MercadoPago.
> **Stack**: Laravel 12 + Filament 3 (backoffice) · Angular 21 (tienda) · PostgreSQL 18

---

## 📋 Descripción del Proyecto

**Controlador** es un sistema para un emprendedor que administra múltiples locales
(actualmente una tienda de bebidas y un almacén/market), diseñado para crecer o
reducirse sin reescribir nada.

| Módulo | Tecnología | Qué hace |
|---|---|---|
| **Backoffice** | Laravel + Filament 3 | Stock, productos, pedidos, usuarios, locales |
| **Tienda online** | Angular 21 (SPA) | Catálogo público, carrito, checkout con MP |
| **API** | Laravel (Sanctum) | Endpoints REST consumidos por Angular |
| **Pagos** | MercadoPago Checkout Pro | Pago + webhook de confirmación |

### Por qué este stack para MVP solo

- **Filament 3** genera el backoffice completo (CRUD, tablas, formularios, roles) con
  mínimo código PHP. El panel admin funciona en horas, no semanas.
- **Laravel** como API para Angular: un solo backend para las dos interfaces.
- **Angular** donde ya hay experiencia → no hay que aprender un framework nuevo.

---

## 🏛️ Arquitectura

```
┌─────────────────────────────────────────┐
│           TIENDA PÚBLICA                │
│         Angular 17 (SPA)               │
│  /tienda → catálogo, carrito, checkout  │
└──────────────────┬──────────────────────┘
                   │ HTTP REST (/api/v1/*)
          ┌────────▼────────────────┐
          │     Laravel 12          │
          │                         │
          │  /admin  → Filament 3   │  ← Panel backoffice
          │  /api/*  → Sanctum API  │  ← Para Angular
          │  /webhook/mp            │  ← MercadoPago
          └────────┬────────────────┘
                   │ Eloquent ORM
          ┌────────▼────────────────┐
          │     PostgreSQL 18        │
          │  (multi-local con        │
          │   local_id en tablas)    │
          └─────────────────────────┘
```

### Decisiones clave de arquitectura

1. **Un solo backend Laravel** sirve tanto al panel Filament como a la API de Angular.
   No hay microservicios, no hay complejidad extra. Un `php artisan serve` y funciona.

2. **Multi-tenancy por columna (`local_id`)**: todos los recursos tienen FK a `locales`.
   No hay schemas separados. Agregar/quitar un local es insertar/desactivar una fila.

3. **Webhooks asíncronos de MercadoPago**: el estado del pedido NUNCA se actualiza
   por el redirect del browser. Siempre por el webhook `/webhook/mp` → integridad garantizada.

4. **Filament solo en `/admin`**: los clientes jamás ven PHP. Angular consume `/api/*`
   con tokens Sanctum (SPA tokens, no JWT clásico).

---

## �️ Modelos de Base de Datos

```
locales
  id, nombre, tipo (bebidas|market), activo, created_at

categorias
  id, local_id*, nombre, descripcion

productos
  id, local_id*, categoria_id*, nombre, descripcion
  precio, precio_costo, unidad, atributos (jsonb)
  activo, created_at

stock_movimientos          ← append-only, nunca se borra
  id, producto_id*, tipo (entrada|salida|ajuste)
  cantidad, stock_resultante, motivo, usuario_id*, created_at

stock_actual               ← vista materializada o tabla de caché
  producto_id*, cantidad_actual, alerta_minima

pedidos
  id, local_id*, cliente_nombre, cliente_email
  estado (pendiente|pagado|preparando|listo|entregado|cancelado)
  total, mp_preference_id, mp_payment_id, created_at

pedido_items
  id, pedido_id*, producto_id*, cantidad, precio_unitario

usuarios
  id, nombre, email, password, rol (admin|encargado)
  local_id* nullable (null = puede ver todos)
```

> `*` = FK. `(jsonb)` = para atributos variables (ej: ABV en bebidas, peso en almacén).

---

## 📦 Módulos Filament (Backoffice `/admin`)

| Resource | Qué genera Filament |
|---|---|
| `LocalResource` | CRUD de locales, activar/desactivar |
| `ProductoResource` | CRUD con categoría, precio, stock actual visible |
| `StockMovimientoResource` | Registrar entradas/salidas, ver historial |
| `PedidoResource` | Ver estados, cambiar estado, ver items |
| `UsuarioResource` | CRUD de encargados, asignar local |

Cada resource es ~80 líneas de PHP con Filament. Sin escribir HTML ni CSS.

---

## 🌐 Endpoints API (para Angular)

```
# Auth
POST   /api/auth/login        → token Sanctum
POST   /api/auth/logout

# Catálogo público (sin auth)
GET    /api/v1/locales
GET    /api/v1/locales/{id}/productos
GET    /api/v1/productos/{id}

# Carrito y pedidos (sin auth, cliente anónimo)
POST   /api/v1/pedidos              → crea pedido + preference MP
GET    /api/v1/pedidos/{id}         → estado del pedido

# Webhook (sin auth, verificación por firma MP)
POST   /webhook/mp                  → confirma pago → actualiza pedido
```

---

## 💳 Flujo MercadoPago

```
1. Cliente hace click en "Pagar" en Angular
2. Angular → POST /api/v1/pedidos → Laravel crea pedido (estado: PENDIENTE)
3. Laravel → MP API → crea Preference → devuelve preference_id + init_point
4. Angular redirige al usuario a MP (init_point)
5. Usuario paga en MercadoPago
6. MP → POST /webhook/mp → Laravel verifica firma → actualiza pedido a PAGADO
7. MP redirige al usuario a /tienda/pedido/{id}/exito
8. Angular muestra confirmación (consultando GET /api/v1/pedidos/{id})
```

> ⚠️ El paso 6 es el que realmente cuenta. El paso 7-8 es solo UX.

---

## 🚀 Fases de Desarrollo (MVP)

```yaml
phases:
  - name: "Phase 1 — Fundamentos Laravel 12"
    est_tiempo: "3-4 días"
    tasks:
      - "Instalar Laravel 12 + configurar PostgreSQL 18"
      - "Crear migraciones (locales, categorias, productos, stock, pedidos)"
      - "Instalar Filament 3 + configurar panel /admin"
      - "Resources Filament: Locales, Categorias, Productos"
      - "Auth de Filament (admin) separada de la API"
    infra_why: >
      Filament se instala sobre Laravel existente. No hay setup separado.
      Las migraciones definen el contrato de datos para todo el proyecto.
      Empezar por acá valida el modelo antes de tocar Angular.

  - name: "Phase 2 — Stock y Pedidos (Filament)"
    est_tiempo: "3-4 días"
    tasks:
      - "Resource StockMovimiento con formulario de entrada/salida"
      - "Resource Pedido con cambio de estados"
      - "Widget Dashboard: stock bajo, pedidos del día"
      - "Roles y permisos (admin vs encargado por local)"
    infra_why: >
      Stock append-only garantiza auditoría completa.
      Los roles en Filament usan policies de Laravel → un lugar para la lógica de acceso.

  - name: "Phase 3 — API para Angular"
    est_tiempo: "2-3 días"
    tasks:
      - "Instalar Laravel Sanctum (SPA tokens)"
      - "Endpoints catálogo público (sin auth)"
      - "Endpoint crear pedido + integración MercadoPago SDK PHP"
      - "Endpoint webhook MP con verificación de firma"
      - "Tests básicos de los endpoints críticos (pedido + webhook)"
    infra_why: >
      Sanctum para SPA es más simple que JWT puro.
      El webhook requiere verificación de firma HMAC → sin esto MP puede ser spoofed.

  - name: "Phase 4 — Frontend Angular 21"
    est_tiempo: "5-7 días"
    tasks:
      - "Setup Angular 21 standalone + HttpClient + Router"
      - "Servicio ApiService (wrapper de HttpClient configurado)"
      - "Página: catálogo de productos por local"
      - "Componente: carrito (estado en localStorage)"
      - "Página: checkout → llama API → redirige a MP"
      - "Página: confirmación de pedido (polling o redirect de MP)"
    infra_why: >
      localStorage para el carrito evita gestionar sesiones server-side en MVP.
      El polling de estado del pedido evita WebSockets en esta fase.

  - name: "Phase 5 — Deploy MVP"
    est_tiempo: "1-2 días"
    tasks:
      - "Dockerizar Laravel (php-fpm + nginx)"
      - "Docker Compose: laravel + postgres + angular-build"
      - "Variables de entorno validadas (.env.example completo)"
      - "Backup automático: pg_dump + cron dentro del container"
      - "Deploy en VPS (DigitalOcean/Hetzner) o Railway"
    infra_why: >
      PHP-FPM + Nginx es el setup más probado para Laravel en producción.
      pg_dump con cron es suficiente para MVP → sin costos extra de servicio.
```

---

## ⚠️ Riesgos y Mitigaciones

| Riesgo | Prob | Mitigación | Impacto |
|---|---|---|---|
| Webhook MP no llega / duplicado | Media | Idempotencia: verificar estado antes de actualizar pedido | Pedido no confirmado |
| Stock negativo por concurrencia | Baja | `DB::transaction()` + constraint `CHECK (cantidad >= 0)` | Venta sin stock |
| Datos de locales mezclados | Media | Siempre filtrar por `local_id`, policy en Filament | Data corruption |
| Credenciales MP en código | Baja | `.env` + validación en `AppServiceProvider::boot()` | Fraude |
| CORS bloqueando Angular→Laravel | Alta en dev | Configurar `config/cors.php` desde el día 1 | Bloqueo desarrollo |

---

## 🔑 Variables de Entorno

```env
# Base de datos
DB_CONNECTION=pgsql
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=controlador
DB_USERNAME=controlador
DB_PASSWORD=secreto

# App
APP_URL=https://tudominio.com
APP_ENV=production
APP_KEY=   # php artisan key:generate

# MercadoPago
MP_ACCESS_TOKEN=APP_USR-...
MP_PUBLIC_KEY=APP_USR-...
MP_WEBHOOK_SECRET=...
MP_SUCCESS_URL="${APP_URL}/tienda/pedido/exito"
MP_FAILURE_URL="${APP_URL}/tienda/pedido/error"
MP_PENDING_URL="${APP_URL}/tienda/pedido/pendiente"

# CORS (para Angular en prod)
FRONTEND_URL=https://tudominio.com
```

---

## �️ Comandos Útiles

```bash
# Instalar proyecto
composer install
php artisan key:generate
php artisan migrate
php artisan db:seed   # datos de prueba

# Instalar Filament
composer require filament/filament:"^3.0"
php artisan filament:install --panels
php artisan make:filament-user

# Crear un resource Filament
php artisan make:filament-resource Producto --generate

# Backup manual de la BD
pg_dump -Fc controlador > backup_$(date +%Y%m%d).dump

# Restaurar backup
pg_restore -d controlador backup_20260224.dump
```

---

## 📁 Estructura del Proyecto

```
controlador/
├── app/
│   ├── Filament/
│   │   └── Resources/          # Un archivo por modelo (Filament)
│   ├── Http/
│   │   └── Controllers/Api/    # Controllers para Angular
│   ├── Models/                 # Eloquent models
│   └── Policies/               # Autorización por recurso
├── database/
│   ├── migrations/
│   └── seeders/
├── routes/
│   ├── api.php                 # Endpoints para Angular
│   └── web.php                 # Filament usa web.php internamente
├── frontend/                   # Angular app (o repo separado)
│   └── src/
│       ├── app/
│       │   ├── tienda/         # Catálogo, carrito, checkout
│       │   └── core/           # ApiService, modelos TS
│       └── environments/
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 📎 Handover para OpenCode / Claude Code

```json
{
  "project": "controlador",
  "execute_with": "opencode",
  "stack": {
    "backend": "Laravel 12",
    "admin_panel": "Filament 3",
    "frontend": "Angular 21 standalone",
    "database": "PostgreSQL 18",
    "payments": "MercadoPago Checkout Pro"
  },
  "entrypoint": "phase1",
  "max_tokens_per_task": 4000,
  "vars": {
    "project_root": "/home/gabriel-mondino/Desktop/proyectos/controlador",
    "php_version": "8.4",
    "node_version": "22"
  },
  "constraints": [
    "Multi-tenant: siempre filtrar por local_id",
    "Stock append-only: nunca UPDATE/DELETE en stock_movimientos",
    "Webhook MP: verificar firma HMAC antes de procesar",
    "Filament solo en /admin, API en /api/v1/*"
  ]
}
```

---

*Última actualización: Febrero 2026 — Stack: Laravel 12 · Filament 3 · Angular 21 · PostgreSQL 18 · PHP 8.4 · Node 22*
