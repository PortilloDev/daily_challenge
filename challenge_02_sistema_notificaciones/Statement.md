# Enunciado

## Reto: El sistema de notificaciones que creció sin control

**Contexto de negocio:**

Plataforma SaaS de gestión de alquileres vacacionales. Conecta a propietarios de apartamentos con huéspedes. Tiene ~3.000 propietarios y ~40.000 reservas/mes.

**El problema:**

El sistema necesita enviar notificaciones en múltiples momentos del ciclo de vida de una reserva:

- Cuando un huésped **solicita** una reserva → email al propietario + push al propietario.
- Cuando el propietario **confirma** → email al huésped + SMS al huésped + push al huésped.
- Cuando el propietario **rechaza** → email al huésped.
- **48h antes del check-in** → email con instrucciones al huésped + SMS con código de acceso + email recordatorio al propietario.
- Al hacer **check-out** → email al huésped pidiendo valoración + email al propietario con resumen.
- Si el huésped **no paga** en 24h tras confirmar → email de recordatorio, y si no paga en 48h → cancelación automática + email a ambos.

Cada propietario puede **configurar sus preferencias**: qué canales quiere recibir (email, SMS, push, WhatsApp) y qué notificaciones quiere desactivar. Algunos propietarios tienen channel managers externos (Booking, Airbnb) y las notificaciones de esas reservas deben ir también al channel manager vía webhook.

**Situación actual (el marrón):**

Todo está acoplado en los servicios de dominio. Cuando se confirma una reserva, el `ReservationService::confirm()` tiene 200 líneas que envían emails, SMS, push y webhooks de forma síncrona. Si el gateway de SMS falla, la confirmación de la reserva falla entera y el huésped ve un error 500. Esto pasa 2-3 veces por semana.

Las notificaciones programadas (48h antes del check-in, recordatorio de pago) están implementadas con cron jobs que hacen queries pesadas cada 5 minutos sobre la tabla de reservas. Con el crecimiento actual, esos crons ya tardan 4 minutos en ejecutar.

El equipo acaba de descubrir que algunos huéspedes reciben emails duplicados. No saben por qué ni con qué frecuencia ocurre.

**Requisitos:**

- **Funcional:** Todas las notificaciones descritas deben funcionar. Propietarios configuran sus preferencias. Soporte para nuevos canales en el futuro (WhatsApp ya está pedido). Debe haber un log consultable de toda notificación enviada.
- **No funcional:** Un fallo en el envío de una notificación nunca debe afectar a la operación de dominio (confirmar, rechazar, etc.). Las notificaciones deben enviarse en < 5 minutos tras el evento. Los crons de notificaciones programadas deben escalar a 200.000 reservas/mes sin degradarse.
- **Restricciones:** Mismo equipo de 3 devs PHP/Symfony. Infraestructura actual: Symfony Messenger con transporte SQS. MySQL como base de datos principal. El dominio de reservas ya existe y funciona (excepto por el acoplamiento con notificaciones).

**Entregable esperado:**

1. **Arquitectura del sistema de notificaciones** desacoplado del dominio de reservas. ¿Qué eventos emite el dominio? ¿Qué consume el módulo de notificaciones?
2. **Modelo para las notificaciones programadas** (48h antes de check-in, recordatorio de pago). ¿Cómo eliminas los crons pesados?
3. **Estrategia contra duplicados.** ¿Por qué ocurren y cómo los previenes?
4. **Cómo incorporas un canal nuevo** (WhatsApp) sin modificar el código existente de los otros canales.

Misma regla: decisiones concretas, trade-offs explícitos, nada genérico.