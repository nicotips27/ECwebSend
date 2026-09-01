# Changelog

Todos los cambios notables de ECSend Pro se documentan aquí.

---

## v7.0 — 2026-09-01

### Publicidad (últimos cambios)
- `5d5eb61` **Publicidad en modal de transferencia**: banner GIF dentro del modal de transferencia (mientras se pasan archivos) que enlaza a Estalingrado Market.
- `890d7af` **Publicidad en vista de inicio**: banner GIF entre "Mi Código" y "Conectarse a dispositivo" que enlaza a Estalingrado Market.
- Ambos banners usan `target="_blank" rel="noopener nofollow"`.

### Modo LocalSend
- `45fbdd1` **Modo LocalSend: nombres aleatorios y descubrimiento en la misma red**:
  - Nombres aleatorios estilo LocalSend ("Zorro Cobalto") con emoji de avatar y botón dado para regenerar.
  - Panel "En tu misma red" que lista los dispositivos del mismo Wi-Fi y conecta con un toque.
  - Descubrimiento vía Trystero (relays nostr públicos) con sala derivada de la IP pública.
  - Toggle de privacidad "Descubrimiento en la red" en Configuración.
  - Chunks adaptativos 64–256 KB y high-water 8 MB para velocidad LAN.
  - `FileReader` reemplazado por `arrayBuffer()`.

---

## v6.5 — 2026-08-26

- `d888402` **Mejoras de seguridad, rendimiento y PWA instalable**:
  - Fix XSS: escape de nombres de archivo y mensajes de chat entrantes.
  - Transferencias más rápidas: chunks de 64 KB con control de flujo `bufferedamountlow`.
  - Botón cancelar transferencia en ambos dispositivos.
  - CDNs con versión fija (lucide, html5-qrcode) y carga diferida.
  - Logo e íconos locales (`assets/`) en lugar del hotlink externo.
  - PWA: manifest + service worker con caché offline.
  - Fixes: timer de código, botón eliminar datos, input bloqueado tras fallo.
  - `404.html` y `.nojekyll` para GitHub Pages.
  - README actualizado con arquitectura real (PeerJS, no Firebase).

---

## Historial anterior (antes de la reestructuración)

- `adce8ab` Update index.html
- `5f6b592` unos ajustes
- `2a22c0c` repare el enlace directo
- `df1f93e` / `1962fe6` Delete / Create CNAME
- `0ec3d25` / `4482dcc` Delete / Create CNAME
- `dc50f44` / `248279d` Delete / Create CNAME
- `edcae1a` Add link to EC News in index.html
- `8491a90` Fix HTML formatting and add doctype declaration