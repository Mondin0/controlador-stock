# Arquitectura Multi-Tenant — Controlador

> Estrategia: **Row-Level Multi-Tenancy** sobre un único schema PostgreSQL 18.

---

## ¿Por qué multi-tenant desde el día 1?

El negocio tiene hoy 2 locales (bebidas + market) y puede crecer o reducirse.
Diseñar multi-tenant desde el inicio evita migraciones dolorosas cuando aparezca el local 3.

---

## Estrategia elegida: Row-Level Tenancy

Existen tres estrategias clásicas de multi-tenancy en BD:

| Estrategia | Cómo funciona | Pros | Contras |
|---|---|---|---|
| **DB separada por tenant** | Una base de datos por local | Aislamiento total | Operaciones complejas, backup por separado |
| **Schema separado por tenant** | Un schema PostgreSQL por local | Buen aislamiento | Migraciones duplicadas, complejidad ops |
| **Row-Level (elegida ✅)** | Todas las tablas tienen `local_id` | Simple, un solo `pg_dump`, fácil dev | Requiere disciplina en queries |

### Por qué Row-Level es la correcta para este proyecto

1. **Operaciones triviales**: un solo `pg_dump controlador` hace backup de todos los locales.
2. **Migraciones únicas**: `php artisan migrate` aplica a todos los locales en un paso.
3. **Sin overhead**: no hay que conectar a distintas DBs ni schemas según el tenant.
4. **Escala suficiente**: para la escala de este negocio (decenas de locales, no miles), es más que suficiente.
5. **Laravel lo soporta nativamente**: Eloquent + Global Scopes = filtrado automático por tenant.

---

## Diagrama de Datos

```
┌──────────────┐
│    locales   │  ← El tenant root
│  id          │
│  nombre      │
│  tipo        │  (bebidas | market | mixto)
│  activo      │
└──────┬───────┘
       │ 1
       │ ←─────────────────────────────────────────┐
       │ N                                         │
┌──────▼────────┐   ┌──────────────┐   ┌──────────┴──────┐
│  categorias   │   │   usuarios   │   │    pedidos      │
│  id           │   │  id          │   │  id             │
│  local_id* ●──┤   │  local_id* ●─┤   │  local_id* ●───┤
│  nombre       │   │  nombre      │   │  estado         │
└──────┬────────┘   │  rol         │   │  total          │
       │            │  email       │   │  mp_payment_id  │
       │ 1          └──────────────┘   └──────┬──────────┘
       │ N                                    │ 1
┌──────▼────────┐                             │ N
│   productos   │◄────────────────────────────┤
│  id           │                     ┌───────▼──────────┐
│  local_id* ●  │                     │  pedido_items    │
│  categoria_id*│                     │  id              │
│  nombre       │◄────────────────────│  pedido_id*      │
│  precio       │                     │  producto_id*    │
│  atributos    │  (JSONB)            │  cantidad        │
│  activo       │                     │  precio_unitario │
└──────┬────────┘                     └──────────────────┘
       │ 1
       │ N
┌──────▼──────────────┐   ┌──────────────────────┐
│  stock_movimientos  │   │    stock_actual       │
│  id                 │   │  producto_id* (PK)    │
│  producto_id*       │   │  cantidad_actual      │
│  tipo               │   │  alerta_minima        │
│  cantidad           │   │  updated_at           │
│  stock_resultante   │   └──────────────────────┘
│  motivo             │   ← Tabla de caché, se recalcula
│  usuario_id*        │     con triggers o en cada movimiento
│  created_at         │
└─────────────────────┘
← Append-only. Nunca UPDATE ni DELETE.
```

> `*` = Foreign Key  |  `●` = campo tenant discriminator

---

## Reglas de Tenancy — Contrato del sistema

Estas reglas **nunca se rompen**. Son el contrato de integridad del sistema:

```
REGLA 1: Todo modelo "owned" tiene local_id.
         Excepciones: usuarios con rol ADMIN (ven todos los locales).

REGLA 2: Nunca se filtra sin local_id en producción.
         Usar Laravel Global Scopes para forzarlo automáticamente.

REGLA 3: stock_movimientos es append-only.
         Para corregir un error: crear un movimiento de ajuste, nunca modificar el original.

REGLA 4: El precio_unitario se copia al pedido_item al momento de la compra.
         Nunca se recalcula desde el producto (los precios pueden cambiar).

REGLA 5: Un usuario con local_id solo puede operar sobre ese local.
         Un usuario con local_id = NULL y rol ADMIN opera sobre todos.
```

---

## Implementación en Laravel: Global Scopes

Para no tener que recordar `->where('local_id', ...)` en cada query, usamos un **Global Scope** de Eloquent:

```php
// app/Traits/BelongsToLocal.php
trait BelongsToLocal
{
    protected static function bootBelongsToLocal(): void
    {
        // Si hay un local activo en sesión/request, filtra automáticamente
        static::addGlobalScope(new LocalScope());
    }
}

// Uso en modelos:
class Producto extends Model
{
    use BelongsToLocal;  // ← Filtra por local_id automáticamente en todos los queries
}
```

En el panel **Filament**, esto se controla por el usuario autenticado. En la **API para Angular**, el `local_id` viene en el token Sanctum o como parámetro de ruta.

---

## Flujo de Request (Backoffice Filament)

```
Admin abre /admin/productos
  → Filament carga ProductoResource::table()
  → Eloquent ejecuta: SELECT * FROM productos WHERE local_id = {sesión}
  → Si el usuario es ADMIN global: sin filtro (ve todos los locales)
  → Si el usuario es encargado: solo ve su local
```

## Flujo de Request (API Angular)

```
GET /api/v1/locales/2/productos
  → Middleware verifica Sanctum token
  → Controller: Producto::where('local_id', 2)->activos()->get()
  → PolicyCheck: ¿puede este token acceder al local 2?
  → Response JSON para Angular
```

---

## Schema PostgreSQL — Decisiones puntuales

### ¿Por qué JSONB en `productos.atributos`?

Los locales tienen productos con atributos heterogéneos:

```json
// Producto de bebidas
{ "abv": 5.0, "volumen_ml": 473, "estilo": "IPA", "origen": "Argentina" }

// Producto de almacén
{ "peso_kg": 0.5, "marca": "Marolio", "unidad_venta": "paquete" }
```

Con JSONB se guarda en la misma tabla sin columnas vacías. PostgreSQL 18 indexa JSONB con GIN indexes para búsquedas rápidas.

### ¿Por qué `stock_actual` separado de `stock_movimientos`?

`stock_movimientos` es el registro histórico (append-only). Calcular el stock actual con `SUM()` sobre miles de movimientos sería lento. `stock_actual` es una tabla de caché que se actualiza en cada movimiento. En Laravel:

```php
// Al registrar un movimiento:
DB::transaction(function () use ($movimiento) {
    $movimiento->save();
    StockActual::updateOrCreate(
        ['producto_id' => $movimiento->producto_id],
        ['cantidad_actual' => DB::raw("cantidad_actual + {$movimiento->cantidad}")]
    );
});
```

---

## Consideraciones de Backup

Con Row-Level tenancy, backup es trivial:

```bash
# Backup completo (todos los locales)
pg_dump -Fc controlador > backup_$(date +%Y%m%d_%H%M).dump

# Restaurar
pg_restore -d controlador_restore backup_20260224_1700.dump

# Backup solo de un local (si fuera necesario)
pg_dump -Fc --table=productos controlador | \
  psql -c "COPY (SELECT * FROM productos WHERE local_id=2) TO STDOUT" > local2_productos.csv
```

No hay que coordinar backups de múltiples DBs o schemas. Un comando, todo el sistema.

---

## Extensibilidad

Si en el futuro el sistema crece a cientos de locales y el aislamiento se vuelve crítico, Row-Level evoluciona a:

1. **PostgreSQL Row-Level Security (RLS)**: políticas de seguridad a nivel de motor, no solo aplicación.
2. **Schema por tenant**: solo si hay requisitos legales de aislamiento estricto.

Ambas migraciones son posibles desde Row-Level sin reescribir el negocio, solo cambia la capa de aislamiento.

---

*Ref: README.md · Siguiente: [fases-mvp.md](./fases-mvp.md)*
