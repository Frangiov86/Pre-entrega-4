MÓDULO 4
Integraciones Avanzadas con n8n
Arquitectura multi-agente, memoria persistente y automatización empresarial
Proyecto	M4 - Manager Integraciones Avanzadas
Autor	Francisco Giovanaz
Plataforma	n8n + Google Gemini + integraciones externas
Fecha	Septiembre de 2026
Documento de arquitectura, implementación, guardrails y casos de prueba

Aclaracion: Se han subido 3 JSON. El Manager M4 general que es el Workflow Principal y los sub Workflows de Ventas y Soporte que acompañan al funcional general. 
 
1. Descripción general
Este proyecto implementa una arquitectura multi-agente en n8n orientada a procesar consultas recibidas por distintos canales, conservar contexto entre ejecuciones y coordinar acciones sobre servicios externos. La solución se organiza alrededor de un Manager principal que clasifica cada solicitud y deriva el trabajo hacia especialistas de Ventas o Soporte.
Objetivo central
Automatizar el análisis y enrutamiento de solicitudes sin perder control humano: los casos normales se procesan y preparan para revisión, mientras que las situaciones ambiguas o con errores se escalan a una intervención manual.

Objetivos funcionales
•	Recibir mensajes desde Gmail y Webhook.
•	Validar y normalizar los datos antes de procesarlos.
•	Mantener memoria persistente por Session_ID.
•	Clasificar cada mensaje como SALES_LEAD, SUPPORT o UNKNOWN.
•	Ejecutar un Worker especializado según la intención.
•	Registrar oportunidades comerciales y sincronizar contactos cuando corresponda.
•	Crear borradores en Gmail, sin envío automático.
•	Notificar eventos y escalaciones mediante Slack.
•	Aplicar guardrails para errores, datos inválidos y casos ambiguos.
2. Arquitectura general
Gmail / Webhook
      │
      ▼
Validación + normalización
      │
      ▼
Memoria persistente - Airtable
      │
      ▼
Manager - Router de intención
  ├── SALES_LEAD → Worker Ventas → Google Sheets / HubSpot
  ├── SUPPORT    → Worker Soporte
  └── UNKNOWN    → Escalamiento humano → Slack
      │
      ▼
Memoria corta / consolidación
      │
      ▼
Gmail Draft o escalamiento + observabilidad Slack

La taxonomía del Manager es cerrada. El modelo sólo puede devolver SALES_LEAD, SUPPORT o UNKNOWN. Esto reduce variaciones de salida y simplifica el enrutamiento posterior.
Intención	Ruta principal	Resultado esperado
SALES_LEAD	Worker Ventas	Lead estructurado, Google Sheets, HubSpot y borrador Gmail
SUPPORT	Worker Soporte	Análisis técnico, memoria y borrador Gmail
UNKNOWN	Escalamiento humano	Airtable ESCALADO + Slack; sin borrador
3. Workflows del sistema
3.1 M4 - Manager Integraciones Avanzadas
El Manager es el orquestador del sistema. Recibe la entrada, recupera memoria, clasifica la intención, llama al Worker correspondiente y coordina la persistencia, las integraciones externas y el tratamiento final por canal.
•	Recibe entradas desde Gmail Trigger o Webhook.
•	Detecta respuestas automáticas antes de continuar con el procesamiento.
•	Valida mensaje y Session_ID y normaliza los campos.
•	Consulta Airtable y verifica que no existan múltiples memorias para el mismo Session_ID.
•	Recupera contexto histórico y lo entrega al Manager como datos, no como instrucciones.
•	Clasifica en SALES_LEAD, SUPPORT o UNKNOWN.
•	Deriva SALES_LEAD y SUPPORT a sus respectivos subworkflows.
•	Controla la integración con HubSpot para oportunidades comerciales.
•	Actualiza memoria de corto y largo plazo.
•	Crea un Gmail Draft para respuestas válidas o escala el caso cuando corresponde.
3.2 M4 - Worker Ventas
El Worker de Ventas procesa únicamente solicitudes clasificadas como SALES_LEAD. Extrae información explícita del mensaje, califica la oportunidad y registra el lead mediante Google Sheets.
•	Datos de interés: nombre, empresa, email, necesidad, prioridad y resumen.
•	Prioridad permitida: ALTA, MEDIA o BAJA.
•	No inventa información faltante ni compromete precios o contratos.
•	No modifica ni elimina registros existentes.
•	El registro comercial se realiza mediante una operación Append sobre la hoja de Leads.
3.3 M4 - Worker Soporte
El Worker de Soporte procesa únicamente solicitudes SUPPORT y genera un análisis técnico estructurado. Su objetivo es identificar el problema reportado, la criticidad, la información adicional necesaria y una recomendación inicial.
•	No inventa errores, causas, códigos técnicos, credenciales ni sistemas afectados.
•	Si faltan datos para determinar la causa, indica qué información adicional se requiere.
•	No registra leads ni ejecuta acciones comerciales.
•	Puede indicar escalamiento humano cuando el problema sea de riesgo o requiera intervención especializada.
4. Integraciones externas
Servicio	Uso dentro del proyecto
Gmail	Canal de entrada, detección de auto-replies, creación de borradores y marcado como leído.
Airtable	Memoria persistente de sesiones, estado del caso, datos clave y acciones requeridas.
Google Sheets	Registro de nuevas oportunidades comerciales desde el Worker Ventas.
HubSpot	Búsqueda y sincronización de contactos para SALES_LEAD.
Slack	Observabilidad y notificación de escalamiento humano.
Google Gemini	Clasificación de intención, procesamiento especializado y resumen de memoria.
5. Memoria y persistencia
Airtable funciona como memoria persistente entre diferentes ejecuciones. El identificador principal es Session_ID, que permite recuperar contexto de una conversación aunque el flujo se ejecute en momentos distintos.
Campos persistidos
Session_ID
Nombre_Usuario
Resumen_Consolidado
Estado_del_Caso
Datos_Clave
Accion_Requerida
Mensajes_desde_resumen
Fecha_Actualizacion

Además de la persistencia en Airtable, el sistema utiliza una memoria de corto plazo con una ventana limitada. Al alcanzar el umbral configurado, se recupera el historial reciente, se genera una memoria consolidada y se limpia el buffer temporal. De esta forma se evita que el contexto crezca indefinidamente.
6. Estrategia de Gmail: Draft only
Regla de seguridad
El workflow no envía respuestas automáticamente. Cuando una respuesta es válida, Gmail crea un borrador asociado al hilo para que pueda ser revisado por una persona antes de enviarlo.

El diseño utiliza Gmail como canal de entrada y como superficie de revisión. Las acciones finales permitidas son crear un Draft y marcar mensajes como leídos. Los casos UNKNOWN, errores de Worker, destinatarios inválidos o fallos al crear el borrador se desvían hacia escalamiento humano.
Caso normal → Gmail - Crear Borrador → Marcar Gmail como leído
Caso escalado → Airtable ESCALADO → Slack → Marcar Gmail como leído
UNKNOWN → NO crear borrador

7. Guardrails implementados
•	Validación de mensaje y Session_ID antes de ingresar al flujo principal.
•	Detección de respuestas automáticas y cuentas no-reply para evitar ciclos.
•	Verificación de unicidad del Session_ID en la memoria persistente.
•	Validación de direcciones de correo antes de crear un Gmail Draft.
•	Taxonomía cerrada en el Manager: SALES_LEAD, SUPPORT o UNKNOWN.
•	Separación estricta de responsabilidades entre Manager y Workers.
•	Memoria histórica tratada como DATA y no como instrucciones ejecutables.
•	Escalamiento de UNKNOWN y de errores producidos por Workers.
•	Continuidad del flujo ante errores de HubSpot y Slack.
•	Escalamiento específico por email inválido o error de creación de Draft.
•	Ausencia de envío automático de Gmail.
8. Pruebas funcionales realizadas
Se realizaron pruebas controladas sobre las tres rutas principales. Las pruebas se ejecutaron con correos reales de entrada y se verificaron los efectos en las integraciones externas.
Caso	Validaciones observadas
SALES_LEAD	Manager SALES_LEAD; Worker Ventas success; registro comercial; HubSpot; memoria; Gmail Draft; Slack LEAD.
SUPPORT	Manager SUPPORT; Worker Soporte success; memoria; Gmail Draft; Slack SOPORTE; sin ruta comercial.
UNKNOWN	Manager UNKNOWN; Airtable ESCALADO; Slack ESCALAMIENTO HUMANO; correo leído; sin Gmail Draft.
Resultado de regresión
Las rutas SALES_LEAD, SUPPORT y UNKNOWN fueron verificadas. La prueba final de UNKNOWN confirmó que el caso se escala por Slack y no genera un borrador de respuesta.

9. Estructura sugerida del repositorio
/
├── checkpoint4_francisco_giovanaz.json
├── M4 - Worker Ventas.json
├── M4 - Worker Soporte.json
├── README.md
└── Documentacion_M4_Francisco_Giovanaz.docx

Los tres workflows deben importarse en n8n para reconstruir la arquitectura completa. El Manager referencia los subworkflows de Ventas y Soporte, por lo que cada uno debe existir en la instancia de destino.
10. Seguridad y credenciales
Las credenciales se administran desde el sistema de credenciales de n8n. El repositorio no debe incluir secretos de autenticación. Cada instalación debe configurar sus propias credenciales para los servicios utilizados.
No publicar
•	API Keys
•	Client Secrets
•	Access Tokens
•	Refresh Tokens
•	Contraseñas
•	Cookies o credenciales de sesión
Los exports de n8n pueden conservar nombres o identificadores de credenciales para reconstruir referencias internas, pero los valores secretos deben permanecer fuera del repositorio.
11. Conclusión
M4 amplía la arquitectura multi-agente con integraciones empresariales y controles operativos. El sistema separa clasificación, especialización, memoria e integración externa, y mantiene una intervención humana explícita en los puntos de riesgo. La estrategia Draft only evita respuestas automáticas no supervisadas, mientras que Slack y Airtable permiten registrar y escalar situaciones que requieren revisión.
La solución final queda preparada para ser compartida mediante GitHub junto con los workflows exportados y esta documentación.
Servicios utilizados
n8n • Google Gemini • Gmail • Google Sheets • Airtable • HubSpot • Slack
