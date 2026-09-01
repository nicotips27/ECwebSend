# Arquitectura técnica — ECSend Pro

Documentación técnica del funcionamiento interno de ECSend Pro. Es una **SPA de un solo archivo** (`index.html` + `js/app.js`) sin framework ni build, desplegada como sitio estático en GitHub Pages.

---

## 1. Visión general

```
[Dispositivo A]  <─── WebRTC DataChannel (P2P directo, cifrado E2E) ───>  [Dispositivo B]
     │                                                                            │
     │  señalización PeerJS Cloud (peer ID ecsend-XXXXXX)                        │
     └────────────────────────── + ──────────────────────────────────────────────┘
     │  descubrimiento Trystero (relays nostr) → sala por IP pública             │
     └───────────────────────────────────────────────────────────────────────────┘
```

- **Datos** (archivos/chat): WebRTC DataChannel, punto a punto, cifrado DTLS/SRTP.
- **Señalización** (cómo se encuentran dos navegadores): PeerJS Cloud mediante peer IDs derivados de un código de 6 dígitos.
- **Descubrimiento** (cómo aparecen los dispositivos cercanos): Trystero sobre relays públicos nostr, agrupados en una sala derivada de la IP pública.

---

## 2. Identidad y códigos

- Cada sesión genera un **código de 6 dígitos** `myRawCode` (aleatorio) y un **peer ID PeerJS** `ecsend-XXXXXX`.
- El código **rota cada 180 s** (`CODE_VALID_TIME`): al vencer, se regenera, se destruye el peer antiguo y se crea uno nuevo (`refreshMyCode`).
- El código se muestra formateado `XXX-XXX` y se publica en un **QR** (`QRious`) con el enlace `?connect=XXXXXX`.
- Al abrir la app con `?connect=XXXXXX`, el enlace llena automáticamente el campo destino y conecta (`checkURLParams`).

---

## 3. Señalización (PeerJS)

PeerJS abstrae el intercambio SDP/ICE. Cada dispositivo es un peer registrado en el **PeerJS Cloud broker** con su peer ID `ecsend-XXXXXX`.

- **Conectar** (`connectToPeer` / `connectToPeerId`): el emisor llama `peer.connect(targetId, { reliable: true })`. Usa `reliable: true` para DataChannel ordenado y sin pérdida (necesario para transferencias).
- **Recibir**: `peer.on('connection')` acepta la conexión entrante.
- **Reconexión**: `peer.on('disconnected')` y un listener de `visibilitychange` reintentan (`peer.reconnect()`) cuando la pestaña vuelve al frente (p. ej. al volver de WhatsApp).
- **ICE STUN**: en modo `global` se usan `stun.l.google.com:19302` y `global.stun.twilio.com:3478`. En modo `local` (`network: 'local'`) **no** hay STUN → la conexión se fuerza por la LAN.

### Manejo de errores / caídas
- `peer.on('error')`: `peer-unavailable` muestra aviso de dispositivo no encontrado.
- `iceconnectionstatechange` en `setupConnection`: da 30 s de gracia en `disconnected` (por si el navegador pasó a segundo plano) y cierra en `failed/closed`.
- `keepAliveInterval` envía `{type:'ping'}` cada 3 s; se ignora al recibirlo.

---

## 4. Descubrimiento en la misma red (Trystero)

`initDiscovery()` (async, no bloqueante) intenta:

1. Obtener la **IP pública** vía `fetch('https://api.ipify.org?format=json', { cache: 'no-store' })`.
2. Derivar la **sala** = `ecsend-net-` + primeros 3 octetos de la IP (`a.b.c`). Dos equipos detrás del mismo router → misma sala.
3. `import('https://cdn.jsdelivr.net/npm/trystero@0.25.3/+esm')` (carga diferida).
4. `joinRoom({ appId: 'ecsend-estalingrado' }, sala)` y `makeAction('presence')`.

### API Trystero (v0.25.x)
> La firma cambió respecto a versiones anteriores: `makeAction` devuelve un **objeto** (`{ send, onMessage }`) y `onPeerJoin`/`onPeerLeave` son **propiedades asignables**, no métodos.

```js
const presence = room.makeAction('presence');
presence.send(data, { target: peerId });   // dirigido
presence.send(data);                        // broadcast
presence.onMessage = (data, { peerId }) => {};
room.onPeerJoin = (peerId) => {};
room.onPeerLeave = (peerId) => {};
room.leave();
```

### Protocolo de presencia
- **Payload** publicado: `{ pid: <PeerJS id>, name, emoji }`.
- Broadcast cada `PRESENCE_INTERVAL` (30 s) y al recibir un peer nuevo (respuesta dirigida).
- TTL de `NEARBY_TTL` (90 s): `pruneNearby()` elimina dispositivos que no se anunciaron.
- Al tocar un dispositivo → `connectToPeerId(payload.pid)` conecta por el canal PeerJS habitual.
- Toggle "Descubrimiento en la red" (`appSettings.discovery`, por defecto `true`): al apagar se llama `stopDiscovery()` (leave de la sala).
- **Fallos tolerados**: si ipify o el import fallan, `discoveryRoom` queda `null` y el panel muestra "no disponible" sin romper el resto.

---

## 5. Protocolo de transferencia de archivos

### Secuencia

1. **Cabecera** `{ type:'header', name, size, mime, batchTotalSize, batchTotalFiles, batchCurrentIndex }`.
2. **Datos binarios**: se envían chunks del archivo (ArrayBuffer) como mensajes binarios.
3. **Fin** `{ type:'eof' }` → el receptor ensambla `Blob` y fuerza descarga.
4. **Confirmación** `{ type:'eof-ack' }` → el emisor pasa al siguiente archivo.

### Confirmación / seguridad del lote
- En el primer archivo (`batchCurrentIndex === 1`), el receptor muestra un `confirm()` con cantidad y peso total. Si rechaza, envía `{type:'transfer-rejected'}`.
- El emisor, al recibir `transfer-rejected`, detiene el envío (`transferCancelled = true`) y vacía la cola.

### Cancelación
- Botón **Cancelar** en el modal → `cancelTransfer()` envía `{type:'transfer-cancel'}`, detiene el pump y limpia la cola.
- El receptor al recibirlo aborta la recepción (limpia buffers y cierra el modal).

### Control de flujo y chunks adaptativos (`sendNextFile`)
- Chunk mínimo 64 KB, máximo 256 KB (`CHUNK_MIN`/`CHUNK_MAX`).
- **High water** 8 MB / **low water** 1 MB (`BUFFER_HIGH_WATER`/`BUFFER_LOW_WATER`).
- `pump()` lee un chunk con `file.slice(...).arrayBuffer()`, lo envía y, si `dc.bufferedAmount < low water` y el chunk no llegó al máximo, **duplica el chunk** (64→128→256 KB). Esto acelera en canales rápidos (LAN).
- Si `dc.bufferedAmount > high water`, espera el evento `bufferedamountlow` (con fallback de 250 ms) antes de seguir.

### Reconstrucción en el receptor (`processIncomingFile`)
- MIME validado con regex (`/^[\w.+-]+\/[\w.+-]+$/`), con fallback a `application/octet-stream`.
- Nombre saneado con `sanitizeFilename()` (quita caracteres de ruta, máx. 180 chars).
- Genera `Blob` → `URL.createObjectURL` → fuerza descarga y agrega ítem a la bandeja + historial.

---

## 6. Seguridad

- **XSS**: todos los nombres/mensajes de origen remoto se escapan con `escapeHTML()` o se insertan con `textContent` (chat). MIME y nombres se validan/sanean.
- **Chat**: burbujas construidas por DOM con `textContent`, nunca con `innerHTML` interpolado.
- **Cifrado**: transporte WebRTC cifrado (DTLS/SRTP); no se almacenan datos en servidores propios.
- **CSP/atributos**: enlaces publicitarios usan `rel="noopener nofollow"`.

---

## 7. Almacenamiento local

| Clave | Contenido |
|---|---|
| `ecsend_settings` | `username`, `network`, `sounds`, `dynamicBg`, `ecEncripP2P`, `discovery` |
| `ecsend_history` | Historial de transferencias (últimos 20) |

---

## 8. Dependencias (CDN, versiones fijas)

| Librería | Uso | Versión |
|---|---|---|
| Tailwind CSS | Estilos (Play CDN) | CDN latest (advertencia de producción) |
| Lucide Icons | Iconografía | 0.462.0 |
| QRious | Generar QR | 4.0.2 |
| html5-qrcode | Escanear QR con cámara | 2.3.8 |
| tsParticles | Fondo animado | 2.0.6 |
| PeerJS | WebRTC + señalización | 1.5.2 |
| Trystero | Descubrimiento en red | 0.25.3 |

---

## 9. PWA / Service worker (`sw.js`)

- **Caché**: `ecsend-v1`, cache-first para el shell (index, app.js, manifest, assets, CDNs).
- **Navegación**: network-first con fallback a `index.html` (offline).
- **Excepción**: `api.ipify.org` se consulta siempre en red (no se cachea la IP).
- Registro en `js/app.js` (solo en `https` o `localhost`).

---

## 10. Publicidad

Banners estáticos que enlazan a `https://estalingradocorp.github.io/EstalingradoCorp/market.html` (`target="_blank"`):

1. **Vista de inicio**: entre "Mi Código" y "Conectarse a dispositivo".
2. **Modal de transferencia**: visible mientras se transfieren archivos.

---

## 11. Límites conocidos

- Los **navegadores no soportan mDNS/multicast** (como LocalSend): el descubrimiento necesita conexión a internet. Los **datos** de archivos sí viajan por LAN cuando es posible.
- El "Modo Local" sin STUN solo funciona si ambos están en la misma red LAN.
- En iOS, múltiples descargas simultáneas pueden fallar por restricciones del sistema (la app avisa y recomienda enviar de a uno).