# 📖 GTS APP - BIBLIA (Fuente de Verdad)

> ⚠️ **LA VERDAD SUPREMA:** Este documento define las reglas de negocio y la arquitectura de alto nivel. Si un Agente tiene dudas funcionales, debe consultar este archivo primero.

---

## 1. 📚 Glosario y Entidades Principales

Estas son las definiciones funcionales de los datos que manejamos.

- **Shop (`almacenes`):**
  - Entidad física. Puede ser **Tienda** o **Almacén Central** (`ID_CENTRAL_WAREHOUSE`).
  - Siempre está vinculada a una `Region`.
  - _Relación:_ Tiene mucho Stock (`almacenes_articulos`) y Tickets.

- **Ticket (`tickets`):**
  - Representa una venta o transacción en caja.
  - **Estados:** `OPEN` (en curso), `PENDING` (aparcado), `CLOSED` (pagado/finalizado).
  - _Regla:_ Un ticket cerrado es inmutable (salvo para Admin).

- **Stock (`almacenes_articulos`):**
  - Es el inventario físico real.
  - **Fórmula Maestra:** `Stock Actual = Entradas - Salidas`.
  - **Validación:** Nunca puede ser negativo (< 0).

- **MaterialReserve (`app_material_reserves`):**
  - Gestión compleja de pedidos.
  - Controla la diferencia entre `qtt_ordered` (pedido) y `qtt_separate` (recibido/apartado).
  - Se relaciona con `ps_orders` (Pedidos Web Legacy) y `ps_customer`.

---

## 2. 🚦 Reglas de Negocio (Inviolables)

Si el código viola alguna de estas reglas, es un **BUG**.

### A. Gestión de Stock

1.  **Insuficiencia:** Si un usuario intenta sacar más stock del disponible -> Lanzar excepción `StockInsufficientException`.
2.  **Origen de Descuento:**
    - Si es **Almacén Central**: Descontar de `ps_stock_available`.
    - Si es **Tienda Física**: Descontar de `almacenes_articulos`. Si el artículo no existe en la tabla intermedia, crearlo con stock 0 (y fallar transacción).
3.  **Estados de Pago:**
    - Si `paid === false`: Se resta el stock en el momento de la reserva.
    - Si `paid === true`: Se asume que el stock ya fue restado por la caja (Legacy) y NO se vuelve a restar.

### B. Seguridad y Roles

1.  **Borrado:** Solo el rol `SUPER_ADMIN` puede ejecutar `DELETE` físico.
2.  **Archivado:** Los demás roles solo pueden "archivar" (`isDeleted = true` o `status = archived`).
3.  **Visibilidad:** Un `SHOP_MANAGER` solo puede ver datos de su propia `id_shop`.

### C. Frontend "Redux First"

1.  **Optimización:** Nunca llamar a la API (`axios`) si el dato ya existe en el estado global.
2.  **Consulta:** Usar siempre `useSelector` antes de lanzar un `useEffect`.
3.  **Cálculos:** No realizar cálculos monetarios complejos en el cliente (JS/TS). Mostrar siempre lo que devuelve el Back.

---

## 3. 🏗️ Arquitectura Técnica (Resumen)

El sistema es híbrido. Es vital respetar la separación de arquitecturas.

### Backend (`gts_app_back`)

- **Legado (Mantenimiento):** MVC Tradicional.
  - _Uso:_ Funcionalidades antiguas.
  - _Ubicación:_ `src/controllers/`, `src/routes/`.
- **Moderno (Nuevas Features):** Arquitectura Hexagonal.
  - _Uso:_ Todo lo nuevo que implementemos.
  - _Ubicación:_ `src/context/[Feature]/`.
  - _Dependencias:_ Inyección estricta con `awilix`. Enriquecido con patrones **Builder** y **Object Mother** en los tests.
- **Sistema Verifacti (Facturación):**
  - _Regla:_ Numeración secuencial por CIF, Serie y Año.
  - _Atomicidad:_ Usa `SELECT FOR UPDATE` para evitar duplicados en concurrencia.
  - _QR AEAT:_ Generado automáticamente al crear el ticket (si `verifactu.active = 1`).
- **Trazabilidad:** Proceso de pago documentado en secuencia de 10 queries críticas.
- ⚠️ **Detalles de implementación:** Ver `.cursor/rules/backend.mdc` y `back/BACKEND_STYLE_GUIDE.md`.

### Frontend (`gts_app_front`)

- **Stack:** Next.js 13 (Pages Router) + TypeScript.
- **UI:** Material UI (MUI) v5.
- **Estado:** Redux Toolkit (Global) + React Hook Form (Local).
- ⚠️ **Detalles de implementación:** Ver `.cursor/rules/frontend.mdc`.

---

## 4. 🔄 Protocolos de Trabajo (Workflow)

Para mantener la integridad entre los agentes y el código.

1.  **Mirror Testing (TDD):**
    - Los tests NUNCA se mezclan con el código fuente.
    - Deben estar en una estructura espejo en la carpeta `tests/`.
    - _Flujo:_ Test en Rojo -> Código -> Refactor.

2.  **Comunicación Asíncrona:**
    - Antes de codificar, consultar `.cursor/shared/changes-log.md`.
    - Si Backend cambia un endpoint, debe actualizar `.cursor/shared/api-contracts.md` OBLIGATORIAMENTE.
