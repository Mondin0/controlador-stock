# Fases de Desarrollo MVP — Controlador

> Stack: **Laravel 12** + **Filament 3** (backoffice) · **Angular 21** (tienda) · **PostgreSQL 18** · **MercadoPago Checkout Pro**
> Objetivo: MVP funcional con el menor tiempo posible para un desarrollador solo.

---

## Resumen de Fases

```
Phase 1 — Fundamentos Laravel         [3-4 días]  ████████░░░░░░░░░░░░
Phase 2 — Stock y Pedidos (Filament)  [3-4 días]  ████████░░░░░░░░░░░░
Phase 3 — API para Angular            [2-3 días]  ██████░░░░░░░░░░░░░░
Phase 4 — Frontend Angular            [5-7 días]  ████████████░░░░░░░░
Phase 5 — Deploy MVP                  [1-2 días]  ████░░░░░░░░░░░░░░░░
                                                   ─────────────────────
Total estimado                        [14-20 días] trabajo real, parte time
```

---

## Phase 1 — Fundamentos Laravel 12

**Objetivo:** Proyecto corriendo localmente con Filament funcional y modelos de BD creados.

### Prerrequisitos

```bash
php --version   # >= 8.4
composer --version
psql --version  # >= 18 (o usar Docker)
node --version  # >= 22 (para más adelante)
```

### Pasos

#### 1.1 Crear proyecto Laravel 12

```bash
composer create-project laravel/laravel controlador
cd controlador

# Configurar .env
cp .env.example .env
php artisan key:generate
```

#### 1.2 Configurar PostgreSQL en `.env`

```env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=controlador
DB_USERNAME=controlador
DB_PASSWORD=secreto
```

```bash
# Crear la base (desde psql como superuser)
createdb controlador
createuser controlador
psql -c "GRANT ALL PRIVILEGES ON DATABASE controlador TO controlador;"

# Verificar conexión
php artisan db:show
```

#### 1.3 Crear migraciones

Orden importante (por FKs):

```bash
php artisan make:migration create_locales_table
php artisan make:migration create_categorias_table
php artisan make:migration create_productos_table
php artisan make:migration create_stock_movimientos_table
php artisan make:migration create_stock_actual_table
php artisan make:migration create_pedidos_table
php artisan make:migration create_pedido_items_table
```

**Contenido clave de cada migración:**

```php
// locales
Schema::create('locales', function (Blueprint $table) {
    $table->id();
    $table->string('nombre');
    $table->enum('tipo', ['bebidas', 'market', 'mixto'])->default('mixto');
    $table->boolean('activo')->default(true);
    $table->timestamps();
});

// productos (ejemplo con JSONB y tenant)
Schema::create('productos', function (Blueprint $table) {
    $table->id();
    $table->foreignId('local_id')->constrained('locales');
    $table->foreignId('categoria_id')->constrained('categorias');
    $table->string('nombre');
    $table->text('descripcion')->nullable();
    $table->decimal('precio', 10, 2);
    $table->decimal('precio_costo', 10, 2)->nullable();
    $table->string('unidad')->default('unidad'); // kg, litro, unidad, etc.
    $table->jsonb('atributos')->nullable();       // atributos variables
    $table->boolean('activo')->default(true);
    $table->timestamps();
});

// stock_movimientos (append-only)
Schema::create('stock_movimientos', function (Blueprint $table) {
    $table->id();
    $table->foreignId('producto_id')->constrained('productos');
    $table->enum('tipo', ['entrada', 'salida', 'ajuste']);
    $table->decimal('cantidad', 10, 3);           // puede ser negativo
    $table->decimal('stock_resultante', 10, 3);   // snapshot post-movimiento
    $table->string('motivo')->nullable();
    $table->foreignId('usuario_id')->nullable()->constrained('users');
    $table->timestamps();
    // Sin softDeletes, sin updates. Solo INSERT.
});

// pedidos
Schema::create('pedidos', function (Blueprint $table) {
    $table->id();
    $table->foreignId('local_id')->constrained('locales');
    $table->string('cliente_nombre');
    $table->string('cliente_email');
    $table->string('cliente_telefono')->nullable();
    $table->enum('estado', [
        'pendiente', 'pagado', 'preparando', 'listo', 'entregado', 'cancelado'
    ])->default('pendiente');
    $table->decimal('total', 10, 2);
    $table->string('mp_preference_id')->nullable();
    $table->string('mp_payment_id')->nullable();
    $table->timestamps();
});
```

```bash
php artisan migrate
```

#### 1.4 Crear Modelos Eloquent

```bash
php artisan make:model Local
php artisan make:model Categoria
php artisan make:model Producto
php artisan make:model StockMovimiento
php artisan make:model StockActual
php artisan make:model Pedido
php artisan make:model PedidoItem
```

#### 1.5 Instalar y configurar Filament 3

```bash
composer require filament/filament:"^3.0" -W
php artisan filament:install --panels

# Crear usuario admin
php artisan make:filament-user
# → nombre, email, password
```

Verificar: `http://localhost:8000/admin` → panel funcionando.

#### 1.6 Crear Resources Filament básicos

```bash
php artisan make:filament-resource Local --generate
php artisan make:filament-resource Categoria --generate
php artisan make:filament-resource Producto --generate
```

> `--generate` auto-detecta las columnas del modelo y genera el form + table completo.

### ✅ Criterio de salida Phase 1

- [ ] `php artisan migrate` sin errores
- [ ] `/admin` muestra panel Filament
- [ ] CRUD de Locales, Categorias y Productos funcionando desde el panel
- [ ] Se puede crear un producto con atributos JSONB

---

## Phase 2 — Stock y Pedidos (Filament)

**Objetivo:** El backoffice completo: stock, pedidos, roles y dashboard.

### 2.1 Resource de Stock

```bash
php artisan make:filament-resource StockMovimiento --generate
```

El formulario de StockMovimiento debe:
- Seleccionar producto (filtrado por local)
- Elegir tipo (entrada / salida / ajuste)
- Ingresar cantidad y motivo
- Al guardar: calcular y actualizar `stock_actual`

```php
// En StockMovimientoResource::form()
Forms\Components\Select::make('producto_id')
    ->relationship('producto', 'nombre')
    ->searchable()
    ->required(),
Forms\Components\Select::make('tipo')
    ->options(['entrada' => 'Entrada', 'salida' => 'Salida', 'ajuste' => 'Ajuste'])
    ->required(),
Forms\Components\TextInput::make('cantidad')
    ->numeric()->required(),
Forms\Components\TextInput::make('motivo'),
```

### 2.2 Resource de Pedidos

```bash
php artisan make:filament-resource Pedido --generate
```

Agregar acción para cambiar estado:

```php
// En PedidoResource::table() → actions()
Tables\Actions\Action::make('cambiar_estado')
    ->form([
        Forms\Components\Select::make('estado')
            ->options([
                'preparando' => 'Preparando',
                'listo'      => 'Listo para retirar',
                'entregado'  => 'Entregado',
                'cancelado'  => 'Cancelado',
            ])
    ])
    ->action(fn (Pedido $pedido, array $data) => $pedido->update(['estado' => $data['estado']])),
```

### 2.3 Roles con Filament Shield (o manual)

Opción simple sin paquete extra:

```php
// En FilamentPanelProvider, restringir acceso:
->authMiddleware([Authenticate::class])
->authorization(fn () => auth()->user()->esAdmin() || auth()->user()->esEncargado())
```

```php
// En User model:
public function esAdmin(): bool    { return $this->rol === 'admin'; }
public function esEncargado(): bool { return $this->rol === 'encargado'; }
```

### 2.4 Widget de Dashboard

```bash
php artisan make:filament-widget StockBajoWidget --stats-overview
php artisan make:filament-widget PedidosHoyWidget --stats-overview
```

### ✅ Criterio de salida Phase 2

- [ ] Se puede registrar una entrada de stock y el `stock_actual` se actualiza
- [ ] Los pedidos se pueden ver y cambiar de estado desde el panel
- [ ] Un encargado solo ve los datos de su local
- [ ] Dashboard muestra pedidos del día y productos con stock bajo

---

## Phase 3 — API para Angular

**Objetivo:** Endpoints REST listos para ser consumidos por Angular.

### 3.1 Instalar Laravel Sanctum (SPA Auth)

```bash
php artisan install:api   # Laravel 12 incluye Sanctum via este comando
```

Configurar CORS en `config/cors.php`:

```php
'paths' => ['api/*', 'sanctum/csrf-cookie'],
'allowed_origins' => [env('FRONTEND_URL', 'http://localhost:4200')],
'supports_credentials' => true,  // necesario para SPA auth con cookies
```

### 3.2 Crear Controllers API

```bash
php artisan make:controller Api/LocalController --api
php artisan make:controller Api/ProductoController --api
php artisan make:controller Api/PedidoController --api
php artisan make:controller Api/WebhookController
```

### 3.3 Definir rutas en `routes/api.php`

```php
// Rutas públicas (sin auth, para la tienda)
Route::prefix('v1')->group(function () {
    Route::get('/locales', [LocalController::class, 'index']);
    Route::get('/locales/{local}/productos', [ProductoController::class, 'indexByLocal']);
    Route::get('/productos/{producto}', [ProductoController::class, 'show']);
    Route::post('/pedidos', [PedidoController::class, 'store']);
    Route::get('/pedidos/{pedido}', [PedidoController::class, 'show']);
});

// Webhook MP (sin auth, verificación por firma)
Route::post('/webhook/mp', [WebhookController::class, 'handle']);
```

### 3.4 Integrar MercadoPago SDK

```bash
composer require mercadopago/dx-php
```

```php
// En PedidoController::store()
use MercadoPago\Client\Preference\PreferenceClient;
use MercadoPago\MercadoPagoConfig;

public function store(Request $request): JsonResponse
{
    // 1. Validar request
    $data = $request->validate([...]);

    // 2. Crear pedido en BD (estado: pendiente)
    $pedido = Pedido::create([..., 'estado' => 'pendiente']);

    // 3. Crear Preference en MP
    MercadoPagoConfig::setAccessToken(config('services.mercadopago.access_token'));
    $client = new PreferenceClient();
    $preference = $client->create([
        'items' => $pedido->items->map(fn($item) => [
            'title'       => $item->producto->nombre,
            'quantity'    => $item->cantidad,
            'unit_price'  => (float) $item->precio_unitario,
        ])->toArray(),
        'external_reference' => (string) $pedido->id,
        'notification_url'   => config('app.url') . '/api/webhook/mp',
        'back_urls' => [
            'success' => config('services.mercadopago.success_url'),
            'failure' => config('services.mercadopago.failure_url'),
            'pending' => config('services.mercadopago.pending_url'),
        ],
        'auto_return' => 'approved',
    ]);

    // 4. Guardar preference_id en el pedido
    $pedido->update(['mp_preference_id' => $preference->id]);

    return response()->json([
        'pedido_id'   => $pedido->id,
        'init_point'  => $preference->init_point,   // URL de pago MP
    ], 201);
}
```

### 3.5 Webhook Handler (crítico)

```php
// En WebhookController::handle()
public function handle(Request $request): Response
{
    // 1. Verificar firma HMAC de MercadoPago
    $signature = $request->header('x-signature');
    $requestId = $request->header('x-request-id');
    // ... lógica de verificación de firma (ver docs MP)

    // 2. Solo procesar eventos de pago aprobado
    if ($request->input('type') !== 'payment') {
        return response()->noContent();
    }

    // 3. Obtener detalles del pago desde MP API
    $paymentId = $request->input('data.id');
    // ... llamar a MP para obtener external_reference

    // 4. Actualizar pedido (idempotente)
    $pedido = Pedido::find($externalReference);
    if ($pedido && $pedido->estado === 'pendiente') {
        $pedido->update([
            'estado'        => 'pagado',
            'mp_payment_id' => $paymentId,
        ]);
    }

    return response()->noContent(); // MP espera 200
}
```

> ⚠️ **El `if ($pedido->estado === 'pendiente')` es el guard de idempotencia.** MP puede enviar el webhook más de una vez.

### ✅ Criterio de salida Phase 3

- [ ] `GET /api/v1/locales` devuelve JSON con los locales activos
- [ ] `GET /api/v1/locales/1/productos` devuelve productos del local 1
- [ ] `POST /api/v1/pedidos` crea pedido y devuelve `init_point` de MP
- [ ] Simular webhook con `curl` actualiza el estado del pedido a "pagado"
- [ ] CORS configurado para `http://localhost:4200`

---

## Phase 4 — Frontend Angular 21

**Objetivo:** SPA para clientes: catálogo, carrito y checkout con MercadoPago.

### 4.1 Crear proyecto Angular

```bash
npm install -g @angular/cli@latest
ng new tienda --standalone --routing --style=scss
cd tienda
ng serve   # http://localhost:4200
```

### 4.2 Estructura de módulos

```
src/app/
├── core/
│   ├── services/
│   │   ├── api.service.ts       # wrapper HttpClient con base URL
│   │   ├── carrito.service.ts   # estado del carrito (signal)
│   │   └── pedido.service.ts    # crear pedido, consultar estado
│   └── models/
│       ├── local.model.ts
│       ├── producto.model.ts
│       └── pedido.model.ts
├── tienda/
│   ├── catalogo/
│   │   ├── catalogo.component.ts
│   │   └── producto-card.component.ts
│   ├── carrito/
│   │   └── carrito.component.ts
│   ├── checkout/
│   │   └── checkout.component.ts
│   └── confirmacion/
│       └── confirmacion.component.ts
└── app.routes.ts
```

### 4.3 ApiService

```typescript
// core/services/api.service.ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly base = environment.apiUrl; // http://localhost:8000/api/v1

  constructor(private http: HttpClient) {}

  getLocales() {
    return this.http.get<Local[]>(`${this.base}/locales`);
  }

  getProductos(localId: number) {
    return this.http.get<Producto[]>(`${this.base}/locales/${localId}/productos`);
  }

  crearPedido(pedido: CrearPedidoDto) {
    return this.http.post<{ pedido_id: number; init_point: string }>(
      `${this.base}/pedidos`, pedido
    );
  }

  getPedido(id: number) {
    return this.http.get<Pedido>(`${this.base}/pedidos/${id}`);
  }
}
```

### 4.4 Carrito con Signals (Angular 21)

```typescript
// core/services/carrito.service.ts
@Injectable({ providedIn: 'root' })
export class CarritoService {
  items = signal<ItemCarrito[]>([]);
  total = computed(() =>
    this.items().reduce((sum, i) => sum + i.precio * i.cantidad, 0)
  );

  agregar(producto: Producto) {
    this.items.update(items => {
      const existente = items.find(i => i.productoId === producto.id);
      if (existente) {
        return items.map(i =>
          i.productoId === producto.id ? { ...i, cantidad: i.cantidad + 1 } : i
        );
      }
      return [...items, { productoId: producto.id, nombre: producto.nombre, precio: producto.precio, cantidad: 1 }];
    });
  }

  vaciar() { this.items.set([]); }
}
```

### 4.5 Flujo de Checkout

```typescript
// checkout.component.ts
checkout() {
  this.pedidoService.crear({
    local_id: this.localId,
    cliente_nombre: this.form.value.nombre,
    cliente_email: this.form.value.email,
    items: this.carrito.items().map(i => ({
      producto_id: i.productoId,
      cantidad: i.cantidad,
    }))
  }).subscribe(({ pedido_id, init_point }) => {
    // Guardar pedido_id en localStorage para mostrar confirmación al volver
    localStorage.setItem('ultimo_pedido_id', String(pedido_id));
    // Redirigir a MercadoPago
    window.location.href = init_point;
  });
}
```

### 4.6 Rutas

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', redirectTo: 'tienda', pathMatch: 'full' },
  { path: 'tienda', loadComponent: () => import('./tienda/catalogo/catalogo.component') },
  { path: 'tienda/carrito', loadComponent: () => import('./tienda/carrito/carrito.component') },
  { path: 'tienda/checkout', loadComponent: () => import('./tienda/checkout/checkout.component') },
  { path: 'tienda/pedido/exito', loadComponent: () => import('./tienda/confirmacion/confirmacion.component') },
  { path: 'tienda/pedido/error', loadComponent: () => import('./tienda/confirmacion/confirmacion.component') },
];
```

### ✅ Criterio de salida Phase 4

- [ ] Se lista el catálogo de productos del local
- [ ] Se puede agregar productos al carrito
- [ ] El formulario de checkout llama a la API y redirige a MercadoPago
- [ ] Al volver de MP (success), se muestra el estado del pedido

---

## Phase 5 — Deploy MVP

**Objetivo:** Sistema corriendo en producción, con backup automatizado.

### 5.1 Dockerizar

**`docker-compose.yml`:**
```yaml
services:
  db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: controlador
      POSTGRES_USER: controlador
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  app:
    build: .
    depends_on: [db]
    environment:
      - APP_ENV=production
    volumes:
      - .:/var/www/html
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    depends_on: [app]
    volumes:
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
    restart: unless-stopped

volumes:
  pgdata:
```

### 5.2 Backup automatizado

```bash
# Dentro del servidor (crontab -e):
0 3 * * * docker exec controlador-db-1 pg_dump -Fc controlador > \
  /backups/controlador_$(date +\%Y\%m\%d).dump

# Rotar: mantener últimos 30 días
0 4 * * * find /backups -name "*.dump" -mtime +30 -delete
```

### 5.3 Checklist de Deploy

- [ ] `.env` de producción configurado (sin `APP_DEBUG=true`)
- [ ] `php artisan config:cache && php artisan route:cache && php artisan view:cache`
- [ ] `php artisan migrate --force` (en producción)
- [ ] SSL configurado (Let's Encrypt con Certbot)
- [ ] Webhook MP apuntando a `https://tudominio.com/api/webhook/mp`
- [ ] Testar pago completo end-to-end con sandbox de MP

### ✅ Criterio de salida Phase 5 (MVP listo)

- [ ] `https://tudominio.com/admin` → Filament backoffice funcional
- [ ] `https://tudominio.com/tienda` → Angular tienda funcional
- [ ] Pago real con MercadoPago procesado correctamente
- [ ] Backup diario corriendo

---

## Handover JSON para OpenCode

```json
{
  "project": "controlador",
  "execute_with": "opencode",
  "stack": {
    "backend": "Laravel 12",
    "admin_panel": "Filament 3",
    "frontend": "Angular 21 standalone",
    "database": "PostgreSQL 18",
    "payments": "MercadoPago Checkout Pro",
    "php_version": "8.4",
    "node_version": "22"
  },
  "entrypoint": "phase1",
  "max_tokens_per_task": 4000,
  "vars": {
    "project_root": "/home/gabriel-mondino/Desktop/proyectos/controlador",
    "docs_dir": "./docs"
  },
  "constraints": [
    "Multi-tenant: todos los modelos owned tienen local_id, filtrar SIEMPRE",
    "stock_movimientos: append-only, nunca UPDATE ni DELETE",
    "Webhook MP: verificar firma HMAC antes de procesar cualquier evento",
    "Filament en /admin solamente, API en /api/v1/*",
    "precio_unitario se copia al pedido_item al momento de la compra",
    "Webhook handler debe ser idempotente: verificar estado antes de actualizar"
  ],
  "phase_order": ["phase1", "phase2", "phase3", "phase4", "phase5"]
}
```

---

*Ref: [README.md](../README.md) · [arquitectura.md](./arquitectura.md)*
