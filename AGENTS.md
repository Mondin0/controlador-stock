# 🤖 INSTRUCCIONES PARA AGENTES IA (Claude Code / OpenCode / Kimi)

> **ESTE ARCHIVO ES LA FUENTE DE VERDAD ABSOLUTA PARA CUALQUIER ASISTENTE DE IA OPERANDO EN ESTE REPOSITORIO.**  
> Lee este documento antes de realizar cualquier cambio en el código, planificar nuevas funcionalidades o refactorizar.

---

## 1. Contexto Core
Este es el proyecto **Controlador**. Un sistema de gestión multi-local (Backoffice) + Tienda Online.
Gabriel Mondino (SRE/DevOps) es el owner. Las decisiones arquitectónicas acá priorizan **estabilidad, mantenibilidad SRE y aislamiento estricto de datos** sobre soluciones "rápidas pero frágiles".

### 🛠️ Stack Tecnológico Restringido
No introduzcas librerías, dependencias o frameworks no listados aquí sin extrema justificación.
- **Backend / API**: Laravel 12 (PHP 8.4)
- **Panel Administrativo (Backoffice)**: Filament 3 (Integrado directamente en Laravel)
- **Frontend Público (Tienda)**: Angular 21 (Standalone Components)
- **Base de Datos**: PostgreSQL 18
- **Pagos**: MercadoPago Checkout Pro (Webhooks)
- **Entorno DEV**: Docker / Docker Compose

---

## 2. Mapa de Conocimiento (Source of Truth)
Debes consultar estos archivos para entender el negocio antes de codificar:

1. 📄 `docs/arquitectura.md`: Decisiones sobre el Multi-Tenancy (Row-Level mediante Local_id) y el por qué del paradigma elegido.
2. 📄 `docs/estructura_datos_postgres.md`: El diseño del Schema SQL. Las migraciones de Laravel **deben coincidir exactamente** con las restricciones (CHECKs, FKs, Tipos) descritos acá.
3. 📄 `docs/fases-mvp.md`: El roadmap paso a paso. No saltes tareas. Termina una fase antes de plantear la siguiente.

---

## 3. 🚨 Reglas SRE y Restricciones Intratables (HARD CONSTRAINTS) 🚨
Si violas estas reglas, la PR será rechazada.

### A. Aislamiento Multi-Tenant
- Absolutamente todos los Eloquent Models de negocio (ej: Producto, Pedido, Categoría) **deben** estar atados a un `local_id`.
- Es mandatorio aplicar **Global Scopes** en Laravel para el filtrado transparente. Ninguna query API debe poder "olvidarse" de filtrar por `local_id`.

### B. Inmutabilidad de Auditoría
- La tabla de movimientos de stock (`stock_movimientos`) es **APPEND-ONLY**.
- **PROHIBIDO** hacer `UPDATE` o `DELETE` sobre registros de movimientos de stock en el código o en los endpoints. Las reversiones se hacen insertando un registro negativo (tipo "ajuste").

### C. Consistencia Transaccional
- Toda operatoria que rebaje o calcule stock (ej: al concretar un pedido) **debe** ir envuelta en transacciones de base de datos (`DB::transaction()`).
- Usa bloqueos pesimistas (`lockForUpdate()`) en el check-out si es necesario para evitar condiciones de carrera (race conditions).

### D. Seguridad de Pagos
- El endpoint del Webhook de MercadoPago no debe confiar en el payload a ciegas. **Debe** verificar la firma criptográfica (HMAC `x-signature`) usando el secret de la app.
- El handler del webhook debe ser **Idempotente** (si MercadoPago envía 4 veces el evento "Aprobado" para el pedido 100, la app solo debe procesarlo la primera vez y las demás devolver HTTP 200 silenciosamente).

---

## 4. Workflows Iniciales (Prompt Starter)

Cuando vayas a iniciar el desarrollo o asignarle una tarea al agente de código, envíale el siguiente bloque JSON de contexto para arrancar:

```json
{
  "role": "Senior Backend & Infrastructure Engineer",
  "project": "Controlador Stock",
  "phase_to_execute": "Phase 1: Laravel Models & Migrations",
  "context_files": [
    "AGENTS.md",
    "docs/estructura_datos_postgres.md",
    "docs/fases-mvp.md"
  ],
  "instructions": "Lee los context_files. Inicia la implementación de las migraciones SQL exactas definidas en la estructura de datos para Laravel 12 ubicadas en la carpeta 'back/'. Configura los modelos de Eloquent con las relaciones. NO generes vistas HTML, usa exclusivamente los recursos de Filament o APIs JSON."
}
```
