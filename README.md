# ECSend Pro

Transferencia de archivos P2P (peer-to-peer) entre dispositivos directamente desde el navegador, sin subir nada a ningún servidor. Interfaz móvil oscura con chat privado cifrado, escaneo de QR, descubrimiento automático de dispositivos en la misma red e historial local.

**Demo:** https://nicotips27.github.io/ECwebSend/

---

## Características

- **Transferencia P2P directa** vía WebRTC: los archivos viajan de navegador a navegador, sin servidores intermedios.
- **Cifrado de extremo a extremo** (DTLS/SRTP nativo de WebRTC).
- **Conexión por código de 6 dígitos** (rotativo, se renueva cada 180 s), **QR** o **enlace directo** (`?connect=XXXXXX`).
- **Descubrimiento automático**: el panel "En tu misma red" lista los dispositivos conectados al mismo Wi-Fi y conecta con un toque.
- **Nombres aleatorios estilo LocalSend** ("Zorro Cobalto", "Tigre Turquesa"...) con emoji de avatar y botón de dado para regenerar.
- **Chat privado** en tiempo real por el mismo túnel P2P.
- **Envío en lote** con confirmación previa (cantidad y peso total), rechazo y cancelación en curso.
- **Velocidad adaptativa**: chunks de 64 a 256 KB según el rendimiento del canal.
- **PWA instalable** con caché offline del shell de la app.
- **Publicidad integrada** que enlaza a Estalingrado Market.

---

## Cómo funciona

1. **Conectarse:** cada dispositivo genera un código de 6 dígitos. El otro usuario lo ingresa a mano, escanea el QR o abre el enlace directo (`?connect=XXXXXX`).
2. **Conexión rápida:** si ambos están en el mismo Wi-Fi, el panel **"En tu misma red"** lista los dispositivos cercanos; un toque conecta sin códigos ni QR.
3. **Enviar:** seleccioná archivos desde la galería o el explorador (también drag & drop en PC).
4. **Recibir:** los archivos llegan a la bandeja de entrada y se descargan automáticamente, con confirmación previa mostrando cantidad y peso total del lote.

---

## Publicidad

La app muestra banners publicitarios de **Estalingrado Market** que al hacer click abren `https://estalingradocorp.github.io/EstalingradoCorp/market.html` en una pestaña nueva:

| Ubicación | GIF |
|---|---|
| Vista de inicio (entre "Mi Código" y "Conectarse a dispositivo") | `9dbdc2_d7f4662631574a52ada08bdf9d4cfcb8~mv2.gif` |
| Modal de transferencia (mientras se pasan archivos) | `9dbdc2_23a9d1da1ade40809dc9afbaee74d186~mv2.gif` |

Ambos usan `target="_blank" rel="noopener nofollow"`.

---

## Configuración

Todo se guarda en `localStorage` (clave `ecsend_settings`):

| Ajuste | Descripción | Clave |
|---|---|---|
| Mi Nombre en la Red | Nombre visible + botón dado para nombre aleatorio | `username` |
| Modo de Conexión | `global` (STUN activo, Internet y Wi-Fi) / `local` (sin STUN, solo LAN) | `network` |
| Descubrimiento en la red | Mostrar dispositivos del mismo Wi-Fi | `discovery` |
| Alertas de Sonido | Beeps al conectar/recibir/chat | `sounds` |
| Fondo Dinámico | Animación de partículas de red | `dynamicBg` |
| ECEncripP2P | Marca visual del túnel cifrado | `ecEncripP2P` |
| Borrar Datos Locales | Limpia ajustes, historial y reinicia | — |

Historial de transferencias: clave `ecsend_history` (máx. 20 ítems).

---

## Arquitectura

La documentación técnica completa (protocolo de transferencia, señalización, descubrimiento, mensajes internos, flujos) está en **[ARCHITECTURE.md](./ARCHITECTURE.md)**.

### Resumen

- **WebRTC DataChannel** para el transporte de datos (P2P directo, cifrado E2E).
- **PeerJS Cloud** para la señalización (peer ID `ecsend-XXXXXX`).
- **Trystero** (relays públicos nostr) para el descubrimiento de dispositivos en la misma red.
- **STUN** público (Google/Twilio) para atravesar NATs; desactivable en modo Local.
- Dependencias de CDN con versión fija: Tailwind, Lucide, QRious, html5-qrcode, tsParticles, PeerJS, Trystero.

---

## PWA instalable

Instalable gracias al manifest + service worker con caché offline del shell. Funciona como app independiente en Android/iOS/Chrome/Edge.

---

## Estructura

```
├── index.html            Markup de la app (vistas, modales, publicidad)
├── js/app.js             Toda la lógica (UI, WebRTC, chat, QR, descubrimiento)
├── sw.js                 Service worker (caché offline)
├── manifest.webmanifest  Manifiesto PWA
├── assets/               Logo e íconos locales
├── 404.html              Redirect para rutas sueltas en Pages
└── .nojekyll             Sirve los archivos tal cual en Pages
```

---

## Desarrollo local

```bash
python -m http.server 8080
# o
npx serve .
```

Abrir `http://localhost:8080` en dos pestañas/dispositivos de la misma red para probar una transferencia real.

---

## Deploy en GitHub Pages

El sitio se publica desde la rama `main` (raíz del repo): **Settings → Pages → Branch: main / root**. Cada push a `main` actualiza el sitio automáticamente.

```bash
git add -A
git commit -m "mensaje"
git push origin main
```

---

## Privacidad

- Cero almacenamiento: no hay backend propio; las transferencias no tocan ningún servidor de la app.
- Historial y ajustes viven únicamente en el dispositivo (`localStorage`).
- El descubrimiento publica tu nombre aleatorio en la sala de tu red (desactivable).
- El software se provee "tal cual"; el usuario es responsable de lo que comparte.

---

## Cambios

Historial de versiones en **[CHANGELOG.md](./CHANGELOG.md)**.