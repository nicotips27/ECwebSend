# ECSend Pro

Transferencia de archivos P2P (peer-to-peer) entre dispositivos directamente desde el navegador, sin subir nada a ningún servidor. Interfaz móvil oscura con chat privado, escaneo de QR e historial local.

**Demo:** https://nicotips27.github.io/ECwebSend/

## Cómo funciona

1. **Conectarse:** cada dispositivo genera un código de 6 dígitos (rotativo, se renueva cada 180 s). El otro usuario lo ingresa a mano, escanea el QR o abre el enlace directo (`?connect=XXXXXX`).
2. **Conexión rápida:** si ambos están en el mismo Wi-Fi, el panel **"En tu misma red"** lista los dispositivos cercanos con su nombre y avatar; un toque conecta sin códigos ni QR.
3. **Enviar:** seleccioná archivos desde la galería o el explorador (también drag & drop en PC).
4. **Recibir:** los archivos llegan a la bandeja de entrada y se descargan automáticamente. Incluye confirmación previa mostrando cantidad y peso total del lote.

Cada dispositivo recibe un **nombre aleatorio estilo LocalSend** ("Zorro Cobalto", "Tigre Turquesa"...) con emoji de avatar; podés regenerarlo con el dado en Configuración.

Incluye además **chat privado en tiempo real** por el mismo túnel P2P.

## Arquitectura

### 1. Conexión de datos — WebRTC DataChannel
- Los archivos viajan directo entre navegadores vía `RTCDataChannel`. Nada pasa por servidores intermedios.
- Cifrado nativo E2E (DTLS/SRTP).
- Servidores STUN públicos de Google y Twilio para atravesar NATs. El "Modo Local" desactiva STUN y fuerza transferencia solo por LAN.
- Señalización (intercambio de ofertas/respuestas SDP y candidatos ICE) mediante **PeerJS Cloud**, identificada por el código de 6 dígitos como peer ID (`ecsend-XXXXXX`). No se usa Firebase.

### 2. Descubrimiento de dispositivos en la misma red
- Clave de sala: los primeros 3 octetos de la **IP pública** (consultada a `api.ipify.org`). Dos equipos detrás del mismo router comparten sala.
- Presencia e intercambio de `{peerId, nombre, emoji}` mediante **Trystero** (WebRTC sobre relays públicos nostr, sin servidores propios ni cuentas).
- Al tocar un dispositivo cercano, la conexión de transferencia se establece por el canal PeerJS habitual.
- Se puede desactivar en Configuración → "Descubrimiento en la red".
- Límite conocido: los navegadores no soportan mDNS/multicast como LocalSend, por lo que el descubrimiento necesita conexión a internet (los **datos** de los archivos igualmente viajan por tu LAN cuando es posible).

### 3. Protocolo de transferencia
- Cabecera JSON (`header`) con nombre, tamaño, MIME e info del lote → fragmentos binarios → marcador `eof`.
- **Chunks adaptativos de 64 a 256 KB** con control de flujo por `bufferedAmount` + evento `bufferedamountlow` (high water 8 MB / low water 1 MB): en LAN el tamaño de chunk crece automáticamente para acercarse a la velocidad de red local.
- Confirmación por archivo (`eof-ack`) antes de enviar el siguiente; el receptor puede rechazar (`transfer-rejected`) o cancelar en curso (`transfer-cancel`).

### 3. Frontend
- Un único `index.html` + `js/app.js` (sin framework ni build).
- Tailwind CSS (Play CDN), Lucide Icons, QRious (generar QR), html5-qrcode (escanear QR), tsParticles (fondo animado), PeerJS (WebRTC), Trystero (descubrimiento en red).
- Configuraciones e historial guardados solo en `localStorage`.

## PWA instalable

La app es instalable (manifest + service worker con caché offline del shell) y funciona como app independiente en Android/iOS/Chrome/Edge.

## Estructura

```
├── index.html            Markup de la app
├── js/app.js             Toda la lógica (UI, WebRTC, chat, QR)
├── sw.js                 Service worker (caché offline)
├── manifest.webmanifest  Manifiesto PWA
├── assets/               Logo e íconos locales
├── 404.html              Redirect para rutas sueltas en Pages
└── .nojekyll             Sirve los archivos tal cual en Pages
```

## Desarrollo local

```bash
python -m http.server 8080
# o
npx serve .
```

Abrir `http://localhost:8080` en dos pestañas/dispositivos de la misma red para probar una transferencia real.

## Deploy en GitHub Pages

El sitio se publica desde la rama `main` (raíz del repo): **Settings → Pages → Branch: main / root**. Cada push a `main` actualiza el sitio automáticamente.

## Privacidad

- Cero almacenamiento: no hay backend propio; las transferencias no tocan ningún servidor de la app.
- Historial y ajustes viven únicamente en el dispositivo (`localStorage`).
- El software se provee "tal cual"; el usuario es responsable de lo que comparte.
