Documentación de la solución: procesamiento de pedidos con API Gateway, EventBridge y Step Functions
Laboratorio BeTek · Región us-east-1 · Noviembre de 2026
1. Objetivo
Construir un flujo de procesamiento de pedidos completamente serverless y orientado a eventos. Un cliente envía un pedido por HTTP; la solución lo recibe, lo publica como evento y un flujo orquestado lo valida, revisa el inventario, procesa el pago, lo confirma y deja un registro final de auditoría.
El principio de diseño es el desacoplamiento: la Lambda de entrada responde rápido al cliente y no espera a que el pedido se procese. El trabajo ocurre de forma asíncrona, y cada pieza hace una sola cosa.
2. Arquitectura general
Flujo de extremo a extremo:
1.  El cliente (Postman) envía un POST a la ruta /pedidos de API Gateway.
2.  API Gateway invoca la Lambda receive-order.
3.  receive-order genera un orderId y publica un evento en el bus default de EventBridge, con Source demo.pedidos y DetailType NuevoPedido.
4.  Una regla de EventBridge detecta ese patrón y arranca la Step Function step-Diego.
5.  La Step Function ejecuta las Lambdas validate-order, check-inventory, process-payment, confirm-order y register-audit, con decisiones (Choice) entre ellas.
6.  Si algún Choice no se cumple, la ejecución termina en un estado de fallo; si todo sale bien, termina en Succeed.
El diagrama gráfico de esta arquitectura se entrega por separado en el archivo JPG.
3. Servicios utilizados
Cada servicio se explica en un solo párrafo; su configuración y uso en esta solución están en la sección 4.
3.1 Amazon API Gateway
Servicio que expone endpoints HTTP públicos y los conecta con un backend. Aquí es la puerta de entrada: recibe el POST /pedidos del cliente y lo entrega a la Lambda receive-order, sin servidores que administrar.
3.2 AWS Lambda
Ejecuta código bajo demanda, sin administrar servidores y cobrando solo por uso. Aquí contiene toda la lógica de negocio en seis funciones Python: recibir el pedido, validarlo, revisar inventario, procesar el pago, confirmarlo y registrar la auditoría.
3.3 Amazon EventBridge
Bus de eventos que recibe mensajes y los enruta a destinos mediante reglas. Aquí desacopla la recepción del procesamiento: receive-order publica un evento NuevoPedido y una regla inicia la Step Function, sin que la Lambda sepa quién lo consume.
3.4 AWS Step Functions
Orquestador que coordina pasos mediante una máquina de estados. Aquí define el orden de las Lambdas, decide según cada resultado, espera antes de confirmar y termina en Succeed o en un estado de fallo, con historial visual de cada ejecución.
3.5 AWS IAM
Servicio de identidades y permisos. Aquí cada pieza actúa con su propio rol y solo con lo necesario: publicar eventos, invocar Lambdas e iniciar ejecuciones de la Step Function.
3.6 Amazon CloudWatch Logs
Servicio de registros y monitoreo. Aquí guarda lo que cada Lambda escribe con print (inicio, evento recibido y resultado), y sirve para depurar cada paso del flujo.
3.7 Postman
Herramienta cliente, no un servicio de AWS, para probar APIs. Aquí envía el POST con el JSON del pedido a la URL de API Gateway y muestra la respuesta.
4. Documentación de la solución
Orden recomendado de creación: primero las Lambdas, luego la Step Function, después la regla de EventBridge y al final el API Gateway, porque cada pieza necesita el ARN o el nombre de la anterior.
4.1 Lambda receive-order
• Nombre receive-order-TUNOMBRE, runtime Python 3.11, arquitectura x86_64 y rol nuevo con permisos básicos de Lambda.
• Lee el cuerpo desde event["body"], genera el orderId con uuid4 y arma el detalle: orderId, cliente, producto, cantidad, total y timestamp.
• Publica el evento con put_events: Source demo.pedidos, DetailType NuevoPedido y bus default.
• Responde 200 con el orderId, o 500 con el mensaje de error.
• Permiso: en Configuration, Permissions, abrir el rol y crear una política en línea que permita events:PutEvents.
4.2 Lambdas del flujo
Las cinco se crean igual: nombre con el sufijo -TUNOMBRE, Python 3.11 y rol nuevo con permisos básicos.
Función
Qué hace
Campo que agrega
validate-order
Lee event.detail y exige cliente y producto no vacíos, cantidad > 0 y total > 0
validationStatus: VALID o INVALID
check-inventory
Si cantidad <= 5 hay stock; si es mayor, no
inventoryStatus: AVAILABLE u OUT_OF_STOCK
process-payment
Si total <= 1000 aprueba; si es mayor, rechaza
paymentStatus: APPROVED o REJECTED
confirm-order
Marca el pedido como confirmado
orderStatus: CONFIRMED
register-audit
Devuelve el resumen final del pedido
mensaje: Proceso finalizado
4.3 Step Function step-Diego
• Tipo Standard, creada con Author with code snippets y pegando el JSON de la definición.
• Reemplazar ARN_VALIDATE_ORDER, ARN_CHECK_INVENTORY, ARN_PROCESS_PAYMENT, ARN_CONFIRM_ORDER y ARN_REGISTER_AUDIT por los ARN reales de las Lambdas.
• Cada Task usa la integración arn:aws:states:::lambda:invoke con OutputPath $.Payload, así la salida de una Lambda es la entrada de la siguiente.
• Permiso: el rol de la state machine debe permitir lambda:InvokeFunction sobre las cinco Lambdas.
Estados, en orden:
1.  validate-order (Lambda).
2.  Choice de validación: si validationStatus es VALID continúa; si no, fallo en validación.
3.  check-inventory (Lambda).
4.  Choice de inventario: si inventoryStatus es AVAILABLE continúa; si no, fallo en el inventario.
5.  process-payment (Lambda).
6.  Choice de pago: si paymentStatus es APPROVED continúa; si no, fallo en el proceso.
7.  Wait antes de confirmar.
8.  confirm-order (Lambda).
9.  register-audit (Lambda).
10.  Succeed.
La guía escribe los Choice con JSONPath (Variable y StringEquals); la captura de la state machine los muestra como expresiones JSONata. Ambas formas son equivalentes.
4.4 Regla de EventBridge
• Nombre rule-nuevo-pedido-TUNOMBRE, bus default, tipo Rule with an event pattern, con Custom pattern (JSON editor).
• Patrón: source igual a demo.pedidos y detail-type igual a NuevoPedido.
• Destino: AWS service, Step Functions state machine, step-Diego.
• Permiso: el rol de la regla debe permitir states:StartExecution sobre step-Diego; la consola puede crearlo automáticamente.
4.5 API Gateway
• Crear una HTTP API con Build y Add integration, eligiendo Lambda y la función receive-order.
• Método POST y ruta /pedidos (la guía dice /pedido; debe coincidir con lo que se use en Postman).
• La URL de invocación tiene la forma https://ID.execute-api.us-east-1.amazonaws.com/pedidos.
5. Datos que viajan por la solución
Cuerpo enviado por el cliente:
{
  "cliente": "Johan",
  "producto": "Laptop",
  "cantidad": 1,
  "total": 900
}
Evento que publica receive-order: los mismos campos más orderId y timestamp, dentro de Detail, con Source demo.pedidos y DetailType NuevoPedido.
Entrada que recibe la Step Function desde EventBridge: el evento completo, con los datos del pedido dentro de detail. validate-order devuelve el contenido de detail aplanado más validationStatus, y de ahí en adelante el objeto es plano.
Entrada correcta para probar manualmente la Step Function desde la consola:
{
  "detail": {
    "orderId": "123",
    "cliente": "Johan",
    "producto": "Laptop",
    "cantidad": 1,
    "total": 900
  }
}
6. Cómo se usa: probar la solución
1.  Enviar desde Postman el pedido anterior: debe devolver una respuesta con el orderId.
2.  Abrir Step Functions, entrar a step-Diego y revisar la pestaña de ejecuciones: debe aparecer una nueva ejecución.
3.  Abrir la ejecución y revisar el grafo: los estados exitosos aparecen en verde.
4.  Revisar los logs de cada Lambda en CloudWatch.
Casos de prueba:
Caso
Datos
Resultado esperado
Pedido normal
cantidad 1, total 900
Succeed, con orderStatus CONFIRMED
Datos inválidos
cantidad 0
Fallo en validación
Sin inventario
cantidad mayor a 5
Fallo en el inventario
Pago rechazado
total mayor a 1000
Fallo en el proceso
7. Consideraciones de diseño
• Respuesta inmediata: el cliente no espera el procesamiento completo, solo la confirmación de recepción.
• Extensibilidad: se pueden agregar más reglas en EventBridge que reaccionen al mismo evento sin modificar receive-order.
• Trazabilidad: Step Functions conserva el historial visual de cada ejecución, con la entrada y salida de cada estado.
• Costo: todos los servicios cobran por uso, pero deben eliminarse al terminar el laboratorio.
8. Problemas encontrados y soluciones
1.  Error Rate exceeded al iniciar la ejecución. Es una limitación de cuota (throttling) de AWS. Se resuelve esperando uno o dos minutos y reintentando una sola vez, o solicitando un aumento de cuota.
2.  Error is not authorized to perform states:StartExecution. El usuario IAM no tenía permiso sobre la state machine. Se soluciona agregando una política que permita states:StartExecution, states:DescribeExecution y states:ListExecutions sobre el ARN de step-Diego, o asociando AWSStepFunctionsFullAccess en un entorno de laboratorio.
3.  Entrada manual con formato incorrecto. Al iniciar la ejecución a mano con los campos planos, validate-order no encuentra detail y marca INVALID. Hay que envolver los datos dentro de detail.
4.  Ruta del API distinta a la de la guía. La guía usa /pedido y la prueba se hizo en /pedidos. Debe coincidir con la ruta creada.
5.  Nombres de campos en los Choice. Los Choice distinguen mayúsculas y minúsculas. Las Lambdas devuelven validationStatus, inventoryStatus y paymentStatus. En la captura de la state machine el Choice de inventario parece leer InventoryStatus con I mayúscula; si es así, el valor nunca coincide y todos los pedidos terminan en fallo en el inventario. Debe escribirse exactamente inventoryStatus.
6.  Tiempo de espera. La guía define el Wait en 5 segundos, mientras que la captura lo muestra como 5 minutos. Con 5 minutos cada ejecución exitosa permanece abierta ese tiempo y se acumulan ejecuciones simultáneas.
9. Limpieza de recursos
Al terminar y tomar las evidencias, eliminar para evitar costos:
1.  La API de API Gateway.
2.  La regla de EventBridge.
3.  La state machine step-Diego.
4.  Las seis funciones Lambda.
5.  Los roles de IAM creados para el laboratorio y las políticas en línea.
6.  Los grupos de registros de CloudWatch de cada Lambda.
