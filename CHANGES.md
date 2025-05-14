## Documentación extra sobre la implementación del desafío

Descripción breve acerca de los cambios necesarios para implementar los requerimientos del desafío.

## General

El objetivo fue crear un panel desplegable en cada mensaje de interacción del usuario con el chat que muestre
los pasos necesarios para conseguir la respuesta; haciendo más amigable y fluída la interacción.
A su vez, se incorporó un boton, visible mientras se obtiene de la respuesta del agente, que permite interrumpir y cancelar ese proceso.

Los cambios necesarios en el código, para estos dos objetivos se separan entre backend y frontend en la siguiente sección.

## Desarrollo

### Panel de Steps Intermedios - Backend

Se activan los pasos intermedios que fueron necesarios para el agente en el proceso de generar la respuesta. Es decir, se actualiza el `build_agent` de la clase del Agent para que retorne una instancia de AgentExecutor con el parámetro `return_intermediate_steps` en `True`. Esto permite que la función `ask` del Agente
reciba un objeto que además del texto de la respuesta contiene los pasos intermedios, si los hubo, que utilizó el LLM para generarla.

Además, en esa misma función `ask` se agregaron pasos predefinidos para todas las respuestas. Son dos
mensajes iniciales y uno final. Por ejemplo, si el mensaje del usuario es "time?", los pasos que muestra el chat son los siguientes:

- Analyzing user request (_predefinido_)
- Consulting agent (_predefinido_)
- Invoking: `clock` with `{}` (_real_)
- Composing final response (_predefinido_)

### Panel de Steps Intermedios - Frontend

Aquí se modificaron las funciones que atienden la recepción de un mensaje del usuario `onUserMessage` y el procesamiento de la respuesta obtenida de la llamada a la api `processUserMessage` de la clase `AgentSession`.

Específicamente, el processUserMessage itera sobre el response de la api para separar los relativo a mensajes de tipo string (como ya lo hacía antes) y lo relativo a los steps intermedios. En este último caso, se invoca al callback `stepHandler` encargado de asignar al mensaje actual, los steps recibidos.

Se incorporó un atributo `steps` a la clase ChatMessage que junto con la funcionalidad `reactive` de vue,
permite al componente Message mostrar cada step recibido. Cuando el step recibido es el último (verificado
mediante el atributo `action` de FlowStep) entonces se muestra el mensaje de respuesta del agente.

Para manejar el mensaje especial de bienvenida, ChatMessage incorpora además el atributo booleano isWelcome.

### Boton de cancelación - Backend

En el módulo api.py se incorpora `active_cancellations` que registra en False cada respuesta que empieza a procesarse
en la función agent_response_stream. En cada ciclo del async for, se verifica si la bandera fue cambiada a True.

Si se pidió la cancelación, sale del bucle y no guarda la respuesta. Al finalizar, borra la bandera del diccionario.

Se agrega el endpoint /sessions/{session*id}/cancel que mediante POST con permite cancelar la session \_session_id*.
Para ello simplmente coloca en True una entrada con clave session_id en active_cancellations.

### Boton de cancelación - Frontend

Se incorporó el botón de cancelación que está disponible mientra dura el proceso de obtención de respuesta del agente.
Cuando se cancela la respuesta, el panel muestra los pasos que llegó a ejecutar pero aborta el pedido a la api mediante
el endpoint de cancelación.

### Tests unitarios y de cobertura

_pendiente_
