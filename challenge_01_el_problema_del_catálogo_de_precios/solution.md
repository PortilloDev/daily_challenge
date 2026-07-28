# Solución

---
link al código: https://github.com/PortilloDev/challenger-1-calculo-precios

**Rendimiento — Cerrado.**

Precarga + caché por TTL según naturaleza del dato. Tipos de cambio con TTL de horas, promos con invalidación por evento cuando marketing publica una nueva, descuentos por cliente con TTL más largo. No necesitas invalidación compleja si clasificas los datos por volatilidad. Aprobado.

---

**Modelo de dominio — La clave es separar reglas de código de reglas de datos.**

Tu instinto de  [[Patrón especificación]] y [[Rule Engine]] es correcto, pero necesitas dividir las reglas en dos categorías:

**Reglas estructurales** (viven en código): son las que definen *tipos* de descuento. "Descuento por volumen", "descuento por cliente", "promoción por categoría", "ajuste por tipo de cambio". Estas cambian rara vez, requieren un dev, y está bien que sea así porque representan la mecánica del dominio.

**Reglas configurables** (viven en base de datos): son las instancias concretas. "Del 1 al 15 de julio, 20% en la categoría Tornillería para pedidos de más de 50 unidades". Esto es lo que marketing necesita gestionar sin un dev.

El modelo quedaría así: tienes una entidad `PricingRule` que contiene los parámetros configurables — fecha inicio, fecha fin, tipo de descuento (porcentaje, fijo), valor, criterios de aplicación (categoría, producto, cliente, tramo de volumen). Marketing crea instancias de esta entidad desde un backoffice. Tu Rules Engine carga las reglas activas, evalúa cuáles aplican al contexto del pedido (aquí entra Specification como filtro), las ordena por prioridad, y las ejecuta.

En código simplificado, la idea sería algo así:

```php
interface PricingRule
{
    public function appliesTo(PriceContext $context): bool;
    public function apply(Money $currentPrice, PriceContext $context): Money;
    public function priority(): int;
}

// Regla estructural — vive en código
final class VolumeDiscountRule implements PricingRule
{
    public function appliesTo(PriceContext $context): bool
    {
        return $context->quantity() >= $this->minimumQuantity;
    }
    // ...
}

// Regla configurable — se hidrata desde BD
final class PromotionalRule implements PricingRule
{
    public function __construct(
        private DateRange $activePeriod,
        private Criteria $criteria,  // categoría, producto, cliente...
        private Discount $discount,  // porcentaje o fijo
        private int $priority,
    ) {}

    public function appliesTo(PriceContext $context): bool
    {
        return $this->activePeriod->containsNow()
            && $this->criteria->isSatisfiedBy($context);
    }
    // ...
}

// El motor simplemente orquesta
final class PricingEngine
{
    public function calculate(Money $basePrice, PriceContext $context): PriceBreakdown
    {
        $rules = $this->ruleRepository->findActive();
        
        usort($rules, fn($a, $b) => $a->priority() <=> $b->priority());

        $appliedRules = [];
        $price = $basePrice;

        foreach ($rules as $rule) {
            if ($rule->appliesTo($context)) {
                $price = $rule->apply($price, $context);
                $appliedRules[] = $rule; // para auditoría
            }
        }

        return new PriceBreakdown($basePrice, $price, $appliedRules);
    }
}
```

Fíjate que el `PriceBreakdown` te regala la auditoría: sabes qué reglas se aplicaron, en qué orden y con qué resultado. Conecta directamente con tu tabla de auditoría del punto 1.1.

La frontera es clara: si marketing necesita un *tipo nuevo* de descuento que no existe (por ejemplo, "descuento por fidelidad basado en histórico de compras"), eso sí requiere un dev que cree una nueva clase. Pero crear una promo de "15% en categoría X durante julio" es pura configuración. En la práctica, el 90% de lo que marketing pide son instancias de tipos que ya existen.

---

**Migración — El patrón que te faltaba: [[Parallel Run (o Shadow Testing)]].**

Concepto de Martin Fowler y muy usado en Strangler Fig Pattern. Funciona así:

1. **Mantienes el servicio legacy como fuente de verdad.** El precio que ve el cliente sigue saliendo del código viejo.
2. **En paralelo**, ejecutas el cálculo con tu motor nuevo.
3. **Comparas ambos resultados.** Si coinciden, bien. Si difieren, logueas la discrepancia con todo el contexto: cliente, producto, cantidad, reglas aplicadas, resultado legacy vs resultado nuevo.
4. **No afectas al usuario.** El nuevo motor es invisible para el cliente durante esta fase.

```
Request → [Legacy Engine] → precio que ve el cliente
       ↘ [New Engine]    → precio shadow → comparar → log discrepancia
```

La ejecución del motor nuevo puede ser síncrona (si es rápido, que debería serlo con la precarga) o asíncrona vía un evento que lanza el cálculo shadow en background.

Después de unas semanas, cuando el porcentaje de discrepancias baje a cero o casi cero, inviertes: el motor nuevo es la fuente de verdad y el legacy pasa a shadow. Otra semana sin problemas, eliminas el legacy.

Esto es lo que te protege de ese if en la línea 1.247 que nadie conoce. No tienes que entender el legacy; solo tienes que demostrar que tu sistema nuevo produce los mismos resultados con tráfico real.

---

**Evaluación final del reto:**

| Dimensión | Nota |
|---|---|
| Rendimiento | 8/10 |
| Modelo de dominio | 5/10 — Identificaste los patrones correctos pero te faltó la separación código/datos |
| Migración | 3/10 — Los tests unitarios son necesarios pero no cubrían el riesgo real |
| Auditoría | 7/10 — Buena propuesta base con correlation ID y eventos |
| **Global** | **5.5/10** |

No es una mala nota para calibración. Tienes buen instinto para elegir patrones y tu enfoque de monolito modular es pragmático. Los dos gaps principales son: pensar en el problema desde el negocio (marketing no puede depender de devs) y pensar en la operación (cómo verificas en producción, no solo en tests). Son exactamente los saltos que separan a un dev senior de un arquitecto.