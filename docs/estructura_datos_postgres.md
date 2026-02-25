# Estructura de Datos (PostgreSQL 18) - Controlador Stock

## Decisiones de Arquitectura a Nivel Base de Datos (MVP)

1. **Multi-tenancy Lógica (Row-Level)**: Todas las tablas relacionadas al negocio llevarán la columna `local_id`. A nivel aplicativo usaremos `Global Scopes` en Laravel para filtrar automáticamente. Es el enfoque más simple y rápido para empezar sin contaminar la BD con perfiles complejos (RLS) en etapas tempranas.
2. **Inmutabilidad en Auditoría**: `stock_movimientos` es una tabla *append-only*. Se rige por el principio de no hacer `UPDATE` ni `DELETE` sobre registros de stock, para mantener un log real. Si hay un error, se hace un movimiento de tipo "ajuste".
3. **Escalabilidad Horizontal**: Los atributos variables de los productos residirán en un tipo nativo `JSONB`. Evita unir docenas de tablas Entity-Attribute-Value (EAV).
4. **Control de Concurrencia Básico**: `stock_actual` se gestionará con validaciones estrictas tipo `CHECK (cantidad_actual >= 0)`. PostgreSQL rechazará cualquier COMMIT que deje el stock negativo, protegiéndonos de *race-conditions*.

---

## Esquema Propuesto

### 1. locales
```sql
CREATE TABLE locales (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    tipo VARCHAR(50) NOT NULL CHECK (tipo IN ('bebidas', 'market', 'mixto')),
    activo BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 2. usuarios
```sql
CREATE TABLE usuarios (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    rol VARCHAR(50) NOT NULL CHECK (rol IN ('admin', 'encargado')),
    local_id BIGINT REFERENCES locales(id) ON DELETE RESTRICT, -- Nullable para superadmins que ven todo
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_usuarios_local_id ON usuarios(local_id);
```

### 3. categorias
```sql
CREATE TABLE categorias (
    id BIGSERIAL PRIMARY KEY,
    local_id BIGINT NOT NULL REFERENCES locales(id) ON DELETE CASCADE,
    nombre VARCHAR(100) NOT NULL,
    descripcion TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(local_id, nombre) -- Mismo nombre no duplicado en el mismo local
);
```

### 4. productos
```sql
CREATE TABLE productos (
    id BIGSERIAL PRIMARY KEY,
    local_id BIGINT NOT NULL REFERENCES locales(id) ON DELETE CASCADE,
    categoria_id BIGINT NOT NULL REFERENCES categorias(id) ON DELETE RESTRICT,
    nombre VARCHAR(255) NOT NULL,
    descripcion TEXT,
    precio NUMERIC(12, 2) NOT NULL CHECK (precio >= 0),
    precio_costo NUMERIC(12, 2) CHECK (precio_costo >= 0),
    unidad VARCHAR(50) DEFAULT 'un',
    atributos JSONB,
    activo BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_productos_local_id ON productos(local_id);
```

### 5. stock_movimientos
```sql
CREATE TABLE stock_movimientos (
    id BIGSERIAL PRIMARY KEY,
    producto_id BIGINT NOT NULL REFERENCES productos(id) ON DELETE RESTRICT,
    tipo VARCHAR(20) NOT NULL CHECK (tipo IN ('entrada', 'salida', 'ajuste')),
    cantidad NUMERIC(10, 3) NOT NULL, -- Permite decimales por pesos
    stock_resultante NUMERIC(10, 3) NOT NULL,
    motivo VARCHAR(255),
    usuario_id BIGINT REFERENCES usuarios(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_stock_mov_producto ON stock_movimientos(producto_id);
```

### 6. stock_actual 
```sql
CREATE TABLE stock_actual (
    producto_id BIGINT PRIMARY KEY REFERENCES productos(id) ON DELETE CASCADE,
    cantidad_actual NUMERIC(10, 3) NOT NULL DEFAULT 0 CHECK (cantidad_actual >= 0),
    alerta_minima NUMERIC(10, 3) DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 7. pedidos
```sql
CREATE TABLE pedidos (
    id BIGSERIAL PRIMARY KEY,
    local_id BIGINT NOT NULL REFERENCES locales(id) ON DELETE CASCADE,
    cliente_nombre VARCHAR(150),
    cliente_email VARCHAR(255),
    estado VARCHAR(30) NOT NULL DEFAULT 'pendiente' 
        CHECK (estado IN ('pendiente', 'pagado', 'preparando', 'listo', 'entregado', 'cancelado')),
    total NUMERIC(12, 2) NOT NULL CHECK (total >= 0),
    mp_preference_id VARCHAR(255),
    mp_payment_id VARCHAR(255) UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_pedidos_local_id ON pedidos(local_id);
CREATE INDEX idx_pedidos_estado ON pedidos(estado);
```

### 8. pedido_items
```sql
CREATE TABLE pedido_items (
    id BIGSERIAL PRIMARY KEY,
    pedido_id BIGINT NOT NULL REFERENCES pedidos(id) ON DELETE CASCADE,
    producto_id BIGINT NOT NULL REFERENCES productos(id) ON DELETE RESTRICT,
    cantidad NUMERIC(10, 3) NOT NULL CHECK (cantidad > 0),
    precio_unitario NUMERIC(12, 2) NOT NULL CHECK (precio_unitario >= 0),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```
