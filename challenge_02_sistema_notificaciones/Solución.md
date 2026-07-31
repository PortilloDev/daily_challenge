# Solución

---

link al código: https://github.com/PortilloDev/challenger-2-notificaciones-reserva

Primero que patrón podemos seguir para cuando se incluya un nuevo canal la implementación sea rápida. ?
Como aligerar la carga de los cron ?
Cómo desacoplamos las notificaciones de las reservas?? esto debería ser un evento
Es posible escalar a 200.000 reservas/mes sin degradarse.? sí, porque no deben pasar todas por el mismo job

De momento todo los envios sincronos se van a convertir en asincronos. Y para cada tipo de notificación se creará un cola + consumidor exclusivo. También se puede analizar el tiempo de consumo de un mensaje por si hicera falta añadir más consumidores por cola.

Para evitar perder mensajes, las notificaciones le daremos un estado, y cuando hayan sido enviadas, se marcarán como "sent", en caso contrario podemos añadir un cron que pase cada 5 minutos revisando que notificaciones están pendientes o fallidas.

Para no enviar notificaciones duplicadas, cada notificación+canal se le asignará una idempotencía que controlaremos update sql en una tabla de la base de datos, si existe es que ya ha sido procesado, en caso contrario se procesado

---

## Primer feedback:

1. Dices "cada tipo de notificación tendrá una cola + consumidor exclusivo". ¿Qué defines como "tipo"? ¿Es por evento de dominio (reserva confirmada, reserva rechazada)? ¿Por canal (email, SMS, push)? ¿Por combinación de ambos? Esta decisión tiene implicaciones directas en cuántas colas gestionas y en qué pasa cuando un canal se cae. Piénsalo antes de seguir.

````Las colas no son por evento de dominio, sino por canal. **¿Porque por canal y no por evento u otro criterio?** .El sistema de colas tendra configurado una politica de reintentos, hasta 5, con un tiempo de intentos exponecial entre cada uno. Además de una configuración cola de fallidos para analizar y reintentar. Pero de esta forma no perdemos mensajes.
````


2. Mencionas un cron de fallback para notificaciones pendientes o fallidas, pero no dices nada sobre las notificaciones programadas (48h antes de check-in, recordatorio de pago a las 24h). Ese es uno de los problemas gordos del enunciado — los crons actuales tardan 4 minutos escaneando toda la tabla. Tu análisis todavía no toca cómo resolverías eso de forma diferente. ¿Sigues con cron o hay otra forma de modelar "algo que tiene que pasar en el futuro"?.**Deben existir dos cron, pero no tienen que escanear todas las tablas. Cada día y teniendo en cuenta la franja horaria de dónde tengamos nuestros servidores. y a una hora temprana (00:00 y las 05:00, habría que definir) el cron del check, buscaría por la fecha que de reservas que deben hacer el check en 48 horas. Para ello cogemos la fecha de hoy y sumamos 48h, y todo los que en la tabla de reserva coincida, se le notificará. Para el job de 24horas sería igual, pero añadiendo un estado pendiente de pago**

3. No mencionas nada sobre cómo desacoplas el dominio del módulo de notificaciones. Dices "esto debería ser un evento" y es correcto, pero ¿qué eventos emite el dominio? ¿Quién los define? ¿El dominio de reservas sabe que existen las notificaciones o no? Esto es importante porque afecta a la dirección de la dependencia.
**Crearía un segundo dominio de Notificaciones descoplando este flujo de las reservas. Las reservas al confirmarse por parte del cliente, lanzaría unos eventos del dominio de reservas, que llamaría al evento de notificaciones para notificar según la configuracion establecida por cada propietario. Con este desacoplamiento de reserva y notificación, eliminamos errores 500 porque la notificación no se enviará. Además de ahora tener el control con la gestión de colas de si alguna no se envía no perder esa notificación y poder reintentar su envío**

Veredicto: El instinto analítico es bueno. Te faltan las notificaciones programadas y la definición concreta de los eventos de dominio. Esos son los dos agujeros principales. Ciérralos y pásame la solución completa.

## Solución:

### Arquitectura del sistema de notificaciones desacoplado del dominio de reservas. ¿Qué eventos emite el dominio? ¿Qué consume el módulo de notificaciones?

Vamos a quitar todas las acciones relacionadas con reservas y llevarlas a un dominio independiente que gestione las notificaciones y los canales y vías que se usen para enviarlas.
De ese modo aplicamos un principio solid como el de responsabilidad única y abierto y cerrado al no tener que modificar las acciones de reserva ante un nuevo canal de notificación.

Una vez separado las responsabilidades, todo los envios sincronos se van a convertir en asincronos. Y para cada tipo de notificación se creará un cola + consumidor exclusivo. También se puede analizar el tiempo de consumo de un mensaje por si hicera falta añadir más consumidores por cola.

Las colas no son por evento de dominio, sino por canal. El sistema de colas tendra configurado una politica de reintentos, hasta 5, con un tiempo de intentos exponecial entre cada uno. Además de una configuración cola de fallidos para analizar y reintentar. Pero de esta forma no perdemos mensajes.

Para el seguimiento de los mensajes fallidos, se puede automatizar un servicio que nos informe de los mensajes fallidos para poder analizarlos y poder incluirlo reencolarlos en el flujo. De primeras este proceso sería "manual", hasta que tengamos datos suficiente de como poder actuar y de si es posible automatizar el reencolado.

El dominio de Reservas, al guardar la información de reservas emitira un evento, y cualquier dominio que este escuchando ese evento podrá tomar el mensaje y procesarlo. Seguiremos el patrón aggreate root.

── Notifier
│   ├── Application
│   │   ├── Events
│   │   │   └── ReserveCreatedEventHandler.php
│   │   └── NotifierService.php
│   ├── Domain
│   ├── Notification.php
│   └── NotificationType.php
└── Reserve
    ├── Application
    │   └── CreateReserveAction.php
    ├── Domain
    │   ├── Events
    │   │   └── ReserveCreatedEvent.php
    │   ├── Exception
    │   │   └── ReserveAlreadyExistsException.php
    │   └── Reserve.php
    └── Infraestructure
        └── Repository
            └── ReserveRepository.php

### Modelo para las notificaciones programadas (48h antes de check-in, recordatorio de pago). ¿Cómo eliminas los crons pesados?

Aplicaría un patrón [[Scheduled Task - Delayed Job]].

Crearía y modelaría una entidad llamada scheduled_actions por ejemplo. Guardaría el tipo, fecha de ejecución, estado, payload por ejemplo. Y lo único que tendría que hacer es implementar un cron que ejecutara cada minuto una consulta para obtener las consultas de ese momento y procesarlas.

En el caso de que tuviera una reserva que ya tenía su notificación guardada en la tabla y se modificará. Eliminaría ese registro de la tabla scheduled_actions y la crearía de nuevo.

Pero ya que me lo has comentado otra opción sería usar DelayStamp de messenger. Al crear el mensaje le decimos cuando debe ejecutarlo. En esta parte la responsabilidad de ejecutarlo pasaría a ser de la parte de infraestructura, y de la otra opción el responsable es aplicaciones, es un enfoque distinto.

## Estrategia contra duplicados. ¿Por qué ocurren y cómo los previenes?

Para evitar perder mensajes, las notificaciones le daremos un estado, y cuando hayan sido enviadas, se marcarán como "sent", en caso contrario podemos añadir un cron que pase cada 5 minutos revisando que notificaciones están pendientes o fallidas.

Para no enviar notificaciones duplicadas, cada notificación+canal se le asignará una idempotencía que controlaremos update sql en una tabla de la base de datos, si existe es que ya ha sido procesado, en caso contrario sera procesado

### Cómo incorporas un canal nuevo (WhatsApp) sin modificar el código existente de los otros canales.

Migraría a un patrón strategy, dónde l comportamiento es el mismo, cada canal debe enviar una notificación, y cada canal se encarga de implementar el como se envía. Ejemplo:

```php

interface NotificationChannel
{
    public function send(Message $message): void;
}

class EmailChannel implements NotificationChannel
{
    public function send(Message $message): void
    {
        // enviar email
    }
}

class SlackChannel implements NotificationChannel
{
    public function send(Message $message): void
    {
        // enviar Slack
    }
}

// el servicio principal

class NotificationService
{
    public function __construct(
        private NotificationChannel $channel
    ) {
    }

    public function send(Message $message): void
    {
        $this->channel->send($message);
    }
}

```

Ahora añadir WhatsApp significa únicamente crear WhatsAppChannel, sin tocar el resto de sistema.
