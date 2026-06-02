1. Conexión de Datos: Protocolo WebRTC
El núcleo de la transferencia de archivos se basa en WebRTC (Web Real-Time Communication), específicamente utilizando el canal RTCDataChannel.

Transferencia Directa: Una vez establecida la conexión, los archivos viajan directamente del navegador del emisor al navegador del receptor. Esto garantiza máxima velocidad (limitada solo por el ancho de banda de los usuarios) y privacidad.

Cifrado de Extremo a Extremo (E2EE): WebRTC implementa de forma nativa cifrado mediante protocolos DTLS (Datagram Transport Layer Security) y SRTP, asegurando que los datos no puedan ser interceptados en tránsito.

Superación de NATs (STUN Servers): Para que dos dispositivos se encuentren en Internet a través de routers y cortafuegos (NAT), el proyecto utiliza servidores STUN públicos de Google y Cloudflare (stun.l.google.com, stun.cloudflare.com). Estos servidores ayudan a los dispositivos a descubrir sus direcciones IP públicas y puertos reales.

2. Negociación de la Conexión: Señalización con Firebase
WebRTC requiere un canal externo para que los dos dispositivos intercambien sus metadatos de configuración (ofertas, respuestas y candidatos ICE) antes de conectarse directamente. Este proceso se llama Señalización (Signaling).

Identificación (Código de 6 dígitos): La aplicación genera un código numérico aleatorio único para el dispositivo receptor (myRawCode).

Base de Datos en Tiempo Real: Utiliza Firebase Firestore para publicar y escuchar las señales. Cuando el emisor introduce el código del receptor, se añade un documento a la colección signals.

Intercambio SDP (Session Description Protocol): 1. El emisor crea una Oferta (Offer) con sus capacidades WebRTC y la sube a Firestore dirigida al código del receptor.
2. El receptor, que está escuchando en tiempo real mediante onSnapshot, detecta la oferta, la procesa y sube una Respuesta (Answer).
3. Una vez intercambiadas las respuestas y los candidatos ICE (rutas de red), la conexión P2P se abre y el documento de señalización en Firestore se elimina automáticamente (deleteDoc) para no dejar rastro.

3. Procesamiento y Fragmentación de Archivos (Chunking)
Dado que los canales de datos WebRTC tienen límites de capacidad por mensaje, la aplicación fragmenta los archivos para su envío seguro en "pedazos" o chunks.

Lectura Binaria (FileReader): El archivo seleccionado por el usuario se procesa nativamente en el navegador utilizando la API de JavaScript FileReader, leyéndolo como un flujo binario de datos (ArrayBuffer).

Fragmentación a 64KB: Para evitar saturar el canal de datos de WebRTC, el archivo se divide secuencialmente en fragmentos óptimos de 64 KB. Un temporizador controlado (setTimeout) envía cada fragmento de forma progresiva.

Protocolo de Mensajería Interno: * Paso 1 (Cabecera): Envía un objeto JSON inicial (type: 'header') con el nombre, tamaño y tipo MIME del archivo para preparar al receptor.

Paso 2 (Datos): Envía los fragmentos binarios uno tras otro de forma síncrona mientras se actualiza la barra de progreso en la interfaz.

Paso 3 (Fin de Archivo): Envía un mensaje de cierre (type: 'eof'). El receptor toma todos los fragmentos acumulados en un array, reconstruye el archivo mediante un objeto Blob, genera una URL local temporal (URL.createObjectURL) y fuerza la descarga automática en el dispositivo receptor.

Arquitectura de la Interfaz (Frontend)
Toda la aplicación está construida en un único archivo (index.html), estructurada bajo un diseño moderno de tipo aplicación móvil:

Estilos: Utiliza Tailwind CSS con configuración nativa para modo oscuro (dark), paneles con efecto esmerilado (glassmorphism) y animaciones personalizadas en CSS para el radar de búsqueda.

Vinculación QR: Integra la librería QRious para codificar la URL del dispositivo receptor en un código QR. Si otro usuario escanea este código con su cámara, la URL procesa los parámetros de búsqueda (?connect=XXXXXX) permitiendo un enlace inmediato sin escribir el código a mano.

Iconografía: Gestionada mediante Lucide Icons para renderizar componentes gráficos vectoriales limpios y responsivos.
