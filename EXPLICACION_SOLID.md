# Diseño UML y refactorización con principios SOLID

## Sistema elegido: pagos con múltiples métodos

El sistema permite cobrar un pedido mediante tarjeta, PayPal o un bono interno, registrar el resultado, emitir una factura cuando se aprueba el pago y notificar al cliente. El bono es una decisión de este ejemplo: permite cobrar, pero no ofrece reembolsos.

## Requisitos que aparecen en las capturas

La actividad se titula «Actividad en clase principios SOLID» y el bloque central, «DISEÑO UML Y REFACTORIZACIÓN CON PRINCIPIOS SOLID». La plataforma muestra apertura el jueves 27 de agosto de 2026 a las 08:00 y cierre ese día a las 10:15. Son fechas del enunciado fotografiado, no fechas de realización de este trabajo.

La parte 1 pide diseñar UNO de estos sistemas: pagos con múltiples métodos, notificaciones multicanal, autenticación con Google/GitHub/etc., carrito de compras o generación/exportación de reportes. El diagrama debe tener al menos diez clases. El diagrama debe subirse a GitHub como primer commit, con evidencia de la hora.

La parte 2 permite usar IA generativa para mejorar el diseño, aplicando S, O, L, I y D. Pide una versión mejorada, indicar qué principios se aplicaron y mostrar visualmente dónde se aplica el patrón. La nueva versión debe subirse en un segundo commit y el enlace del repositorio debe entregarse en TEMA.

Las cuatro imágenes corresponden a fragmentos de esta misma actividad: la primera muestra el título y el inicio de la parte 1; la segunda termina la lista de opciones y empieza la parte 2; la tercera muestra la parte 2 completa; la cuarta repite el contenido de la tercera. Estas capturas no proporcionan por sí solas un segundo taller ni cinco ejercicios obligatorios: presentan cinco alternativas y piden elegir una.

## Diseño inicial de referencia

El archivo `01_diseno_inicial.mmd` contiene doce clases: Cliente, Pedido, LineaPedido, Producto, Pago, SolicitudPago, ResultadoPago, ProcesadorPagos, PasarelaTarjeta, PasarelaPayPal, Factura y BaseDatos.

`ProcesadorPagos` recibe una solicitud, consulta el pedido y selecciona la pasarela mediante el texto `tipoMetodo`. Después guarda el pago, crea la factura y envía un correo. Este diseño permite explicar una refactorización, pero concentra decisiones que cambian por motivos distintos.

Problemas del diseño inicial:

- **Responsabilidades mezcladas:** cambiar el formato de factura o el envío de correo obliga a tocar el procesador de cobro.
- **Extensión costosa:** cada medio de pago nuevo añade una rama de selección y una dependencia concreta.
- **Acoplamiento:** el procesador conoce las pasarelas y la base de datos concretas.
- **Ausencia de contrato común:** tarjeta y PayPal exponen nombres diferentes y no definen qué comportamiento deben compartir. Esto no demuestra por sí solo una violación de Liskov; muestra que todavía no hay una abstracción sustituible.
- **Capacidades sin separar:** al añadir reembolsos conviene evitar una interfaz única que obligue a todos los medios a reembolsar. El diseño inicial no contiene esa interfaz; no se le atribuye una violación inexistente de segregación.

## Diseño refactorizado

El archivo `02_diseno_refactorizado_SOLID.mmd` contiene 22 elementos: 16 clases concretas y 6 interfaces. Mantiene ocho clases de datos del modelo y separa la coordinación, el cobro, la persistencia, la facturación y las notificaciones.

`ServicioPago` recibe por constructor cinco colaboradores: `Cobrador`, `RepositorioPedidos`, `RepositorioPagos`, `Facturador` y `NotificadorPago`. La composición de la aplicación elige qué implementaciones entregar; el servicio no decide usando el nombre de una pasarela ni crea sus dependencias concretas.

### S — Responsabilidad única

`ServicioPago` coordina el caso de uso. `FacturadorSimple` emite facturas, `NotificadorEmail` envía avisos y los repositorios encapsulan almacenamiento y consulta. `Pedido` calcula el total a partir de sus líneas. Los adaptadores traducen el contrato común al medio de pago correspondiente.

Ejemplo: cambiar la plantilla del correo afecta a `NotificadorEmail`, no a los adaptadores de cobro. El servicio puede coordinar varios pasos y seguir teniendo una responsabilidad: ejecutar el caso de uso de pagar un pedido.

### O — Abierto a extensión, cerrado a modificación

Para añadir otro medio se crea una implementación de `Cobrador` y se selecciona en la composición de la aplicación. No se añade una condición por tipo de pasarela dentro de `ServicioPago`.

Esto no significa que nunca cambie ningún archivo: la configuración de dependencias puede cambiar y los requisitos nuevos pueden exigir nuevos contratos. La extensión descrita mantiene estable el contrato de cobro.

### L — Sustitución de Liskov

Tarjeta, PayPal y bono deben poder sustituirse cuando el cliente depende de `Cobrador`. Implementar una interfaz no basta: todas las clases deben respetar su contrato.

Contrato previsto de `cobrar`:

- Entradas: importe positivo con precisión decimal, moneda admitida por el contrato de la aplicación, token opaco del medio y clave de idempotencia no vacía.
- El token debe corresponder al medio seleccionado; su estructura interna pertenece al adaptador, no al servicio.
- Un rechazo normal —saldo insuficiente, autorización denegada o token inválido— devuelve `ResultadoPago` con `aprobado=false` y motivo. No se considera una operación no soportada.
- Un resultado aprobado incluye una referencia del cobro. No basta con devolver un identificador local sin confirmación del proveedor.
- Repetir la misma clave con los mismos datos identifica el mismo intento lógico; el adaptador debe evitar un nuevo cargo. Reutilizar la clave con otros datos debe rechazarse.
- Los fallos técnicos tienen una política común: se propaga un error técnico y no se traduce una respuesta incierta del proveedor en un rechazo definitivo. La recuperación requiere conciliación antes de intentar un cargo nuevo.
- Ningún adaptador puede exigir requisitos adicionales que el contrato no anuncie ni afirmar éxito sin efectuar el cobro.

La clave y los repositorios expresan una intención de diseño. El UML no prueba idempotencia real bajo concurrencia ni resuelve por sí mismo las transacciones distribuidas. Una implementación debe garantizar unicidad de claves y recuperación ante fallos parciales.

### I — Segregación de interfaces

`Cobrador` solo exige cobrar y `Reembolsador` solo exige reembolsar. Tarjeta y PayPal implementan ambas; el bono solo implementa `Cobrador`. Por tanto, el bono no necesita un método vacío ni lanzar «operación no soportada» para un reembolso que nunca prometió ofrecer.

El reembolso es una capacidad separada incluida para mostrar la segregación; el caso de uso `ServicioPago.pagar` no lo invoca. Un futuro servicio de reembolsos dependería de `Reembolsador`.

### D — Inversión de dependencias

El servicio de alto nivel depende de interfaces. Las implementaciones SQL y los adaptadores concretos cumplen esas interfaces. Las dependencias se entregan por constructor, lo que también permite sustituir infraestructura por dobles de prueba.

La inyección de dependencias es el mecanismo de composición; la inversión consiste en que la política de pago no depende directamente de detalles concretos de almacenamiento o proveedores.

## Patrones y notación visual

El diagrama marca los cinco principios mediante notas junto a los elementos implicados. `ServicioPago` actúa como contexto de **Strategy** y `Cobrador` es su estrategia intercambiable. Los adaptadores uniforman el acceso a medios distintos; las integraciones externas reales no se detallan en este diagrama académico. Los repositorios abstraen el almacenamiento.

- `..|>`: realización de una interfaz; el triángulo apunta a la interfaz.
- `-->`: asociación navegable con un colaborador o entidad.
- `..>`: dependencia de uso, por ejemplo un argumento o resultado.
- `*--`: composición; un pedido contiene sus líneas.
- `1`, `0..*`, `1..*`, `0..1`: multiplicidades.
- `+`: operación pública; `-`: atributo privado.

Un cliente realiza cero o más pedidos; cada pedido tiene al menos una línea y cada línea refiere a un producto. Un pedido puede tener varios intentos de pago. Un pago puede no tener factura —si fue rechazado o la emisión sigue pendiente— o tener una. Este ejemplo no incluye notas crédito ni facturación parcial.

## Flujo del caso de uso

1. El punto de entrada compone `ServicioPago` con el cobrador seleccionado y los demás colaboradores.
2. El servicio consulta el pedido, verifica que se pueda pagar y calcula su importe desde sus líneas, sin confiar en un total enviado por el cliente.
3. Consulta la clave del intento para reconocer una operación previamente registrada. Los estados en progreso o inciertos requieren recuperación, no un segundo cargo automático.
4. Delega el cobro en `Cobrador` y registra el resultado en `RepositorioPagos`.
5. Si el pago fue aprobado, delega la emisión de factura y la notificación. Si fue rechazado, devuelve el motivo y no genera factura.
6. Un fallo de facturación o correo no deshace ni repite silenciosamente un cargo aprobado. Se conserva el pago y se gestiona el paso fallido aparte.

Estas reglas describen el comportamiento esperado de una futura implementación. Esta actividad entrega diseño UML; no se ha implementado ni ejecutado una integración financiera.

## Comparación

| Aspecto | Referencia inicial | Refactorización |
| --- | --- | --- |
| Cobro | Selección por texto y pasarelas concretas | Estrategia `Cobrador` inyectada |
| Facturación y correo | Métodos del procesador | Colaboradores especializados |
| Persistencia | Clase `BaseDatos` concreta | Interfaces de repositorio |
| Nuevos medios | Modificar el procesador | Añadir implementación y configurar |
| Reembolso | No modelado | Capacidad independiente |
| Sustitución | Sin contrato compartido | Contrato conductual documentado |

## Validación conceptual prevista

La revisión del diseño debe comprobar que se pueden componer tarjeta, PayPal o bono sin cambiar el servicio; que un rechazo no produce factura; que un bono no está obligado a reembolsar; que los detalles SQL no entran en el servicio; y que cambiar el correo no modifica el cobro. Son criterios de comprobación del diseño, no pruebas de ejecución aprobadas.

## Alcance de la entrega

Los archivos incluyen el diseño inicial de referencia, la refactorización y su justificación. La entrega en TEMA sigue pendiente.


## Precisiones de la revisión técnica

- Las operaciones declaran el tipo de cada parámetro. Los métodos de consulta devuelven `Optional<Pedido>` y `Optional<Pago>` para representar explícitamente la ausencia de registros. El servicio rechaza un pedido inexistente antes de intentar cobrar.
- La moneda pertenece al pedido; la solicitud identifica el pedido y aporta el token y la clave. El cliente no puede cambiar el importe ni la moneda de una compra mediante la solicitud.
- `Pedido` navega hacia su `Cliente`, de modo que el servicio puede obtener el destinatario. Los accesores triviales se omiten para mantener legible el diagrama. Las asociaciones representan referencias aunque no se dupliquen como atributos.
- `ServicioPago` muestra sus cinco dependencias con tipos de interfaz. Todas son obligatorias y se reciben por constructor; su firma se omite del dibujo para evitar una caja excesivamente ancha.
- Cada cantidad de línea es positiva y cada precio es no negativo. El total debe ser positivo para efectuar el cobro. Todos los importes del pedido usan su moneda y una regla única de redondeo al sumar.
- Los estados de pedido utilizados en este alcance son PENDIENTE, PAGADO y CANCELADO. Un pedido PAGADO o CANCELADO no admite un nuevo intento lógico de cobro; repetir una clave previamente finalizada devuelve su resultado conocido.
- Los estados de pago contemplados son EN_PROCESO, APROBADO, RECHAZADO e INCIERTO. Ante aprobación, el servicio llama a `Pedido.marcarPagado()` y persiste el pedido mediante `RepositorioPedidos.guardar()`. La escritura del pedido y el registro del resultado necesitan coordinación transaccional en la implementación; un error local después del cobro requiere recuperación, no repetir el cargo.
- `Reembolsador` opera sobre la referencia de una transacción previa y su moneda original. El importe debe ser positivo y no exceder el saldo reembolsable. Una negativa de negocio se expresa en ResultadoPago; los fallos técnicos usan la misma política común del cobro.
- Se verifica la visualización de ambos diagramas en GitHub. La revisión documental no equivale a pruebas de un programa Java ni a validación de proveedores reales.
