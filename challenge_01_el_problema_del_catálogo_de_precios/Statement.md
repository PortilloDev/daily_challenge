Primera sesión. Vamos a calibrar nivel. Empiezo con un 🟢 **Medio** y si lo resuelves bien, subo.

---

## Reto: El problema del catálogo de precios

**Contexto de negocio:**
Eres el lead técnico de una plataforma B2B de distribución de material industrial. El catálogo tiene ~50.000 productos. Cada producto tiene un precio base, pero el precio final depende de:

- El **cliente** (hay ~2.000 clientes, cada uno puede tener descuentos negociados por categoría o producto específico).
- El **volumen** del pedido (tramos: 1-10 uds, 11-50, 51-200, 200+).
- **Promociones temporales** que marketing lanza sin previo aviso (2-3 por semana, pueden afectar a categorías enteras o productos individuales).
- Un **tipo de cambio** diario (algunos proveedores facturan en USD/EUR y el precio se recalcula).

**Situación actual (el marrón):**
El monolito legacy en PHP 7.4 calcula el precio en un servicio de 1.800 líneas con 47 `if/else` anidados. Nadie se atreve a tocarlo. Cuando marketing mete una promo nueva, el dev tarda 2 días en implementarla y rezar para que no 
rompa otra regla. El cálculo tarda ~200ms por producto, y en un pedido de 80 líneas la página de checkout tarda 16 segundos.

**Requisitos:**

- **Funcional:** El precio debe calcularse correctamente aplicando todas las reglas en el orden de prioridad adecuado. Marketing quiere poder crear promos sin intervención de desarrollo.
- **No funcional:** El checkout de un pedido de 100 líneas debe responder en < 2 segundos. El sistema debe ser auditable (saber por qué un cliente pagó X por un producto en una fecha dada).
- **Restricciones:** Equipo de 3 devs backend (PHP/Symfony). No hay presupuesto para reescribir todo de golpe. El monolito sigue vivo y otros módulos dependen de él.

**Entregable esperado:**

1. **Modelo de dominio** para el motor de precios (entidades, value objects, reglas).
2. **Estrategia de migración** desde el monolito (¿cómo convives con el código legacy mientras migras?).
3. **Decisión sobre rendimiento:** ¿cómo bajas de 16s a <2s?

No me des una respuesta genérica. Quiero decisiones concretas con justificación de trade-offs. Si dibujas un diagrama, mejor. Si escribes código del modelo, mejor aún.

