## Reto #3 — Sistema de Gestión de Inventario Multi-canal

🟡 **Avanzado** | Categorías: Modelado de dominio + Datos y consistencia + Integración

---

**Contexto de negocio:**

Una empresa de e-commerce de productos deportivos vende a través de **4 canales simultáneos**: tienda web propia, app móvil, marketplace (Amazon/Miravia) y 2 tiendas físicas con TPV. Facturan ~€3M/año con un catálogo de **~8.000 SKUs**. Tienen picos en Black Friday y rebajas de enero donde el tráfico se multiplica x10.

**El problema actual:**

El inventario se gestiona con un monolito PHP (Symfony 5.4, MySQL) que expone un endpoint REST síncrono `/api/stock/{sku}` que todos los canales consultan antes de confirmar una venta. En Black Friday pasado:

- Se produjeron **~120 oversells** (ventas de producto sin stock real) porque dos canales vendían la última unidad "al mismo tiempo".
- El endpoint de stock respondía con latencias de **2-4 segundos** bajo carga, causando timeouts en el checkout del marketplace (SLA de Amazon: respuesta < 500ms).
- El equipo de almacén reporta que el stock del sistema "nunca cuadra" con el stock físico. Hay discrepancias de hasta un 8% en algunos SKUs.
- Cuando llega una devolución, el stock tarda **hasta 30 minutos** en reflejarse porque pasa por un proceso manual de "reentrada".

**El equipo:**

3 backend devs PHP/Symfony, 1 dev Python que mantiene las integraciones con marketplaces, 1 frontend. No hay SRE dedicado. Infra actual en AWS (ECS, RDS MySQL, SQS ya configurado).

**Requisitos:**

1. Eliminar oversells — la consistencia del stock es innegociable.
2. Responder a consultas de stock en < 200ms bajo carga (p99).
3. Soportar el modelo multi-canal sin que un canal bloquee a otro.
4. Trazabilidad: saber en todo momento por qué el stock de un SKU tiene el valor que tiene (qué movimientos lo cambiaron).
5. Las devoluciones deben reflejar stock disponible en < 5 minutos.
6. El equipo no puede parar el negocio para migrar. Hay que hacerlo de forma incremental.

**Restricciones:**

- Presupuesto limitado: no se puede contratar más gente ni licenciar software externo caro (tipo ERP enterprise).
- MySQL como base de datos principal (no hay presupuesto ni expertise para migrar a PostgreSQL o NewSQL).
- El marketplace tiene su propio sistema de stock y se sincroniza vía API cada X minutos (tú decides la estrategia).

---

**Entregable esperado:**

1. **Modelo de dominio** del bounded context de inventario: aggregates, entidades, value objects, eventos de dominio.
2. **Estrategia de consistencia** para la reserva de stock multi-canal (¿cómo evitas oversells sin matar el rendimiento?).
3. **Diseño del flujo de lectura** para cumplir el SLA de < 200ms.
4. **Estrategia de sincronización** con el marketplace.
5. **Plan de migración incremental** desde el monolito actual.

Cuando quieras, manda tu análisis — parcial o completo.