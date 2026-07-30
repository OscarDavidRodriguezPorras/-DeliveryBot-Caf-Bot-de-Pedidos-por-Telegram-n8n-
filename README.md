# ☕ DeliveryBot Café — Bot de Pedidos por Telegram (n8n)

Bot conversacional para Telegram que permite a los clientes de una cafetería consultar el menú, armar un pedido, elegir método de pago, registrar dirección de entrega y consultar su historial de compras. Todo el flujo está construido en **n8n** y usa **Google Sheets** como base de datos.

---

## 📌 Tabla de contenido

- [Descripción general](#-descripción-general)
- [Arquitectura](#-arquitectura)
- [Requisitos previos](#-requisitos-previos)
- [Estructura de la base de datos (Google Sheets)](#-estructura-de-la-base-de-datos-google-sheets)
- [Instalación y configuración](#-instalación-y-configuración)
- [Flujo conversacional (máquina de estados)](#-flujo-conversacional-máquina-de-estados)
- [Detalle de los flujos internos](#-detalle-de-los-flujos-internos)
- [Automatización de reportes diarios](#-automatización-de-reportes-diarios)
- [Sistema de puntos de lealtad](#-sistema-de-puntos-de-lealtad)
- [Manejo de errores y validaciones](#-manejo-de-errores-y-validaciones)
- [Limitaciones conocidas](#-limitaciones-conocidas)
- [Posibles mejoras futuras](#-posibles-mejoras-futuras)

---

## 🧾 Descripción general

**DeliveryBot Café** es un bot de Telegram que simula el proceso completo de una tienda de delivery:

1. El cliente escribe al bot y ve un menú principal (Pedir / Historial de pedidos).
2. Elige una categoría (Bebidas, Comidas, Snacks).
3. Elige un producto disponible (con validación de stock).
4. Indica la cantidad deseada.
5. Puede seguir agregando productos o continuar al pago.
6. Elige método de pago (Transferencia o Efectivo).
7. Registra la dirección de entrega.
8. El pedido se envía automáticamente a un grupo interno de cocina en Telegram.
9. El cliente puede consultar su historial de pedidos ya entregados.

Además, el sistema:

- Lleva el **estado de la conversación** de cada usuario (para saber en qué "pantalla" se encuentra).
- Descuenta automáticamente el **stock** del producto en el momento de la reserva.
- Acumula **puntos de lealtad** por cada pedido.
- Genera **reportes de ventas diarios** de forma automática (cron).

---

## 🏗️ Arquitectura

```
┌────────────────────┐
│  Telegram Trigger   │  (mensaje del usuario)
└─────────┬───────────┘
          │
          ▼
┌───────────────────────────┐
│ Guardar/actualizar sesión │  (Google Sheets → SESSIONS)
└─────────┬──────────────────┘
          │
          ▼
┌───────────────────────────┐
│   Leer sesión actual       │
└─────────┬──────────────────┘
          │
          ▼
┌───────────────────────────┐
│  Switch (pantalla_actual)  │ → enruta a 11 ramas distintas
└─────────┬──────────────────┘
          │
   ┌──────┴───────────────────────────────────────────────┐
   │  INTERFAZ · DECISION_INTERFAZ · CATEGORIAS · MENU ·   │
   │  CANTIDADES · DECISION · PEDIDOS · PAGO · DIRECCION · │
   │  HISTORIAL · (vacío = usuario nuevo)                  │
   └──────┬───────────────────────────────────────────────┘
          │
          ▼
   Google Sheets (MENU, PEDIDOS, USUARIOS, SESSIONS, REPORTES)
          │
          ▼
   Respuesta al usuario vía Telegram
```

El motor de estados vive en la hoja **SESSIONS**: cada usuario de Telegram tiene una fila con el campo `pantalla_actual`, que le dice al bot qué rama del `Switch` debe ejecutar en el siguiente mensaje que envíe.

---

## ✅ Requisitos previos

| Recurso | Uso |
|---|---|
| **n8n** (self-hosted o cloud) | Motor de automatización que ejecuta el workflow |
| **Bot de Telegram** (via [@BotFather](https://t.me/BotFather)) | Canal de conversación con el cliente |
| **Grupo de Telegram para cocina** | Recibe notificación de cada pedido nuevo |
| **Cuenta de Google** con **Google Sheets** | Base de datos del proyecto |
| Credenciales configuradas en n8n | `telegramApi` y `googleSheetsOAuth2Api` |

---

## 🗄️ Estructura de la base de datos (Google Sheets)

El workflow usa un único libro de Google Sheets llamado **`DeliveryBot_DB`**, con las siguientes pestañas:

### 1. `SESSIONS`
Guarda el estado de conversación de cada usuario.

| Columna | Descripción |
|---|---|
| `telegram_id` | ID del chat de Telegram (clave única) |
| `pantalla_actual` | Estado actual del usuario (INTERFAZ, CATEGORIAS, MENU, etc.) |
| `carrito_temporal` | Nombre del producto que el usuario está por confirmar |
| `cantidad` | Cantidad ingresada temporalmente |
| `ultimo_cambio` | Timestamp del último mensaje procesado |

### 2. `USUARIOS`
Registro de clientes.

| Columna | Descripción |
|---|---|
| `telegram_id` | ID del chat (clave única) |
| `nombre_completo` | Nombre de Telegram del usuario |
| `departamento_oficina` | Dirección de entrega registrada |
| `puntos_lealtad` | Puntos acumulados por pedidos |
| `fecha_registro` | Fecha/hora del primer contacto |

### 3. `MENU`
Catálogo de productos.

| Columna | Descripción |
|---|---|
| `id_producto` | Identificador del producto |
| `nombre` | Nombre del producto (usado para búsquedas) |
| `descripcion` | Descripción breve |
| `precio` | Precio unitario |
| `categoria` | Bebidas / Comidas / Snacks |
| `stock` | Unidades disponibles (se descuenta automáticamente) |

### 4. `PEDIDOS`
Historial transaccional de pedidos.

| Columna | Descripción |
|---|---|
| `id_usuario` | `telegram_id` del cliente |
| `detalles_pedido` | Nombre del producto pedido |
| `cantidad` | Unidades pedidas |
| `total_pago` | Total pagado por ese ítem |
| `Pago_en_total` | Total consolidado de todo el pedido (todos los ítems) |
| `estado` | `En camino` → `Entregado` |
| `fecha` / `hora` | Marca temporal del pedido |

### 5. `REPORTES`
Reporte diario generado automáticamente.

| Columna | Descripción |
|---|---|
| `fecha` | Fecha del reporte |
| `total_vendido` | Suma de ventas del día |
| `cantidad_pedidos` | Número de pedidos entregados |
| `producto_estrella` | Producto más vendido del día |
| `hora_pico` | Hora con más pedidos |

> ⚠️ Todas las hojas deben existir **con esos nombres exactos** antes de activar el workflow, ya que los nodos de Google Sheets referencian tanto el `documentId` como los `gid` internos.

---

## ⚙️ Instalación y configuración

1. **Importar el workflow** en n8n:
   - `Workflows → Import from File` y selecciona el `.json` de este proyecto.

2. **Crear el bot de Telegram**:
   - Habla con [@BotFather](https://t.me/BotFather), crea el bot y copia el token.
   - En n8n, crea una credencial `Telegram API` con ese token.
   - Asigna esa credencial a **todos** los nodos `Telegram Trigger` y `Telegram` (envío de mensajes) del workflow.

3. **Crear el grupo de cocina**:
   - Crea un grupo de Telegram para el equipo de preparación.
   - Agrega el bot a ese grupo y obtén su `chat_id` (negativo, ej. `-1003992409372`).
   - Reemplaza el `chatId` fijo del nodo **"Send a text message15"** (notificación a cocina) con el ID de tu grupo.

4. **Preparar la hoja de cálculo**:
   - Duplica o crea un Google Sheet con las 5 pestañas descritas arriba (`SESSIONS`, `USUARIOS`, `MENU`, `PEDIDOS`, `REPORTES`) y sus columnas exactas.
   - Carga el catálogo inicial en `MENU` (nombre, descripción, precio, categoría, stock).

5. **Conectar credenciales de Google Sheets**:
   - Crea una credencial `Google Sheets OAuth2` en n8n.
   - En **cada nodo de Google Sheets** del workflow, actualiza el `documentId` para que apunte a tu propio spreadsheet.

6. **Revisar datos sensibles hardcodeados**:
   - Número de Nequi para pagos por transferencia (nodo *"Send a text message11"*).
   - `chatId` del grupo de cocina (nodo *"Send a text message15"*).
   - Nombre de las categorías del menú (Bebidas / Comidas / Snacks) — deben coincidir exactamente con la columna `categoria` de la hoja `MENU`.

7. **Activar el workflow** (toggle `Active` en la esquina superior de n8n).

---

## 🔁 Flujo conversacional (máquina de estados)

El campo `pantalla_actual` en `SESSIONS` determina la rama del `Switch` que se ejecuta. Estos son los estados posibles:

| Estado (`pantalla_actual`) | Qué hace |
|---|---|
| *(vacío)* | Usuario nuevo o reinició conversación → muestra menú principal (Pedir / Historial) |
| `DECISION_INTERFAZ` | Usuario respondió al menú principal (1=Pedir, 2=Historial) |
| `INTERFAZ` | Muestra categorías de productos (Bebidas/Comidas/Snacks) |
| `CATEGORIAS` | Usuario eligió categoría → muestra productos disponibles con stock |
| `MENU` | Usuario eligió producto → pide cantidad |
| `CANTIDADES` | Valida cantidad, verifica stock, reserva producto y pregunta si desea agregar algo más |
| `DECISION` | Usuario responde Y/N a "¿deseas ordenar algo más?" |
| `PEDIDOS` | Muestra resumen del pedido y pide método de pago (1=Transferencia, 2=Efectivo) |
| `PAGO` | Usuario eligió método de pago → pide dirección |
| `DIRECCION` | Usuario envía dirección → se registra el pedido, se notifica a cocina y se suman puntos de lealtad |
| `HISTORIAL` | Muestra pedidos anteriores con estado "Entregado" |

### Diagrama simplificado del recorrido feliz (happy path)

```
Usuario escribe /start o cualquier mensaje
        │
        ▼
  Menú principal (Pedir / Historial)
        │  elige "Pedir"
        ▼
  Categorías (Bebidas/Comidas/Snacks)
        │  elige categoría
        ▼
  Lista de productos con stock
        │  escribe nombre del producto
        ▼
  "¿Cuántas unidades?"
        │  ingresa número
        ▼
  Valida stock → reserva producto → resumen del ítem
        │  "¿Deseas ordenar algo más? (Y/N)"
        │
   Y ───┴─── N
   │         │
   ▼         ▼
 vuelve a  Resumen total + método de pago
 categorías      │  elige 1 (Transferencia) o 2 (Efectivo)
                 ▼
         Instrucciones de pago + pide dirección
                 │  ingresa dirección
                 ▼
   Pedido registrado → notificación a cocina
   → puntos de lealtad → confirmación al cliente
```

---

## 🧩 Detalle de los flujos internos

### 1. Registro y actualización de sesión
Cada mensaje entrante dispara:
- **Append or update row in sheet1**: crea o actualiza la fila del usuario en `SESSIONS` con su `telegram_id` y `ultimo_cambio`.
- **Get row(s) in sheet1**: lee el estado actual (`pantalla_actual`) para decidir el enrutamiento.

### 2. Menú principal / bienvenida (usuario nuevo)
- Se registra al usuario en `USUARIOS` (nombre, fecha de registro).
- Se envía el mensaje de bienvenida con las categorías.
- Se actualiza `pantalla_actual = DECISION_INTERFAZ`.

### 3. Selección de categoría (`CATEGORIAS`)
- **Get row(s) in sheet**: busca productos de `MENU` cuya `categoria` coincida con el texto enviado.
- **If**: valida que la categoría exista (si no, mensaje de error).
- **Code in JavaScript / JavaScript14**: filtra productos con `stock > 0` y construye el mensaje de menú numerado.
- **Code in JavaScript6 / If7**: valida que la categoría tenga al menos un producto disponible en total; si no, informa que está agotada.

### 4. Selección de producto (`MENU`)
- **Get row(s) in sheet9**: busca el producto exacto por nombre.
- **If6**: valida que el producto exista.
- Se pide la cantidad y se actualiza `pantalla_actual = CANTIDADES`.

### 5. Validación de cantidad y reserva de stock (`CANTIDADES`)
- **Code in JavaScript15 / If8**: valida que la cantidad ingresada sea un número entero positivo.
- **Code in JavaScript1**: busca el producto en el menú completo, compara `cantidad_solicitada` vs `stock`, y:
  - Si no alcanza el stock → `resultado: 0` → mensaje de error.
  - Si alcanza → `resultado: 1` → calcula `total_pagar`, `nuevo_stock`, y arma el resumen del ítem.
- **If1**: enruta según el resultado.

### 6. ¿Agregar más productos? (`DECISION`)
- **Code in JavaScript2 / If2**: valida respuesta Y/N.
- **Code in JavaScript4 / If3**: según la respuesta, vuelve a categorías (Y) o pasa al resumen de pago (N).

### 7. Resumen y método de pago (`PEDIDOS` → `PAGO`)
- **Code in JavaScript5**: arma el resumen de todos los productos y el total.
- **Code in JavaScript7 / If4**: valida que la opción de pago sea `1` o `2`.
- **Code in JavaScript8 / If5**: según la elección, muestra instrucciones de transferencia (Nequi) o de pago en efectivo contraentrega.

### 8. Registro de dirección y cierre del pedido (`DIRECCION`)
- El texto enviado se guarda como dirección del usuario (`departamento_oficina` en `USUARIOS`).
- **Append row in sheet**: crea la fila del pedido en `PEDIDOS` con estado `En camino`.
- **Update row in sheet13**: descuenta el stock definitivo del producto en `MENU`.
- **Send a text message15**: notifica el pedido al grupo de cocina en Telegram.
- **Code in JavaScript9**: suma 1 punto de lealtad al usuario en `USUARIOS`.
- Se limpia `pantalla_actual` (vuelve a estado vacío) para permitir un nuevo ciclo.

### 9. Historial de pedidos (`HISTORIAL`)
- **Get row(s) in sheet12**: filtra pedidos del usuario con estado `Entregado`.
- **Code in JavaScript18 / If11**: valida si existen pedidos entregados.
- **Code in JavaScript19**: agrupa los pedidos y construye un mensaje legible con fecha, producto, cantidad, valor unitario y total pagado.

---

## 📊 Automatización de reportes diarios

Un **Schedule Trigger** se ejecuta todos los días a las **23:00** y:

1. Lee todos los pedidos con estado `Entregado` (`Get row(s) in sheet11`).
2. Calcula (`Code in JavaScript12`):
   - Total vendido en el día.
   - Cantidad de pedidos.
   - Producto más vendido ("producto estrella").
   - Hora con más pedidos ("hora pico").
3. Guarda una nueva fila en la hoja `REPORTES`.

> 💡 Este cron es independiente del flujo conversacional; no requiere interacción del usuario.

---

## 🏆 Sistema de puntos de lealtad

Cada vez que un cliente completa un pedido (llega hasta el paso de `DIRECCION`):

- Se consulta su registro actual en `USUARIOS`.
- Se suma **+1 punto** a `puntos_lealtad`.
- Se actualiza la fila junto con la dirección de entrega ingresada.

Actualmente el sistema **no** canjea ni notifica los puntos acumulados; solo los almacena para uso futuro (por ejemplo, promociones o descuentos).

---

## 🛡️ Manejo de errores y validaciones

El workflow incluye validaciones explícitas en varios puntos, devolviendo mensajes claros al usuario en caso de error:

- ❌ Categoría inexistente o sin productos disponibles.
- ❌ Producto inexistente en el menú.
- ❌ Cantidad no numérica.
- ❌ Cantidad solicitada mayor al stock disponible.
- ❌ Respuesta inválida en la confirmación Y/N ("¿deseas ordenar algo más?").
- ❌ Opción de pago distinta de `1` o `2`.
- ❌ Opción inválida en el menú principal (distinta de `1`, `2`, "pedir" o "historial de pedidos").

En todos los casos, el bot **no avanza de pantalla** hasta que el usuario responda correctamente.

---

## ⚠️ Limitaciones conocidas

- El bot **no tiene manejo de carrito multi-producto persistente**: cada producto se procesa y reserva de forma individual antes de decidir si se agrega otro.
- Las opciones se detectan por **texto exacto** (no hay botones inline de Telegram), por lo que el usuario debe escribir el nombre tal cual aparece en el menú.
- El número de Nequi y el chat de cocina están **hardcodeados** en los nodos de mensaje; deben actualizarse manualmente si cambian.
- No hay confirmación de pago real (no hay integración con pasarela de pagos ni OCR de comprobantes): el "pago por transferencia" se basa en la confianza del cliente en enviar el comprobante.
- El estado de pedido pasa de `En camino` a `Entregado` de forma manual/externa (no se ve en este workflow el nodo que hace ese cambio), probablemente actualizado directamente en la hoja `PEDIDOS`.

---

## 🚀 Posibles mejoras futuras

- Agregar **botones inline** de Telegram (`reply_markup`) en lugar de depender de texto exacto.
- Soportar un **carrito real** que permita elegir varios productos antes de calcular el total.
- Integrar una **pasarela de pagos** (Wompi, PayU, MercadoPago) para transferencias.
- Añadir un **panel de administración** para marcar pedidos como "Entregado" directamente desde Telegram o una app.
- Implementar **notificaciones de pedido cancelado** o **tiempo estimado de entrega**.
- Migrar la base de datos de Google Sheets a una base de datos real (PostgreSQL/Airtable) para mayor escalabilidad y concurrencia.

---

# Update: Examen [Número]

## Descripción

En esta actualización se implementó la lógica necesaria para el examen, incorporando nuevos nodos en el flujo de n8n que permiten automatizar el proceso solicitado. La solución incluye la recepción y procesamiento de mensajes desde Telegram, la ejecución de la lógica correspondiente y, cuando aplica, el registro automático de la información en Google Sheets.

## Cambios implementados

- Se añadieron nuevos nodos al workflow de n8n para soportar la funcionalidad del examen.
- Se configuró la comunicación con Telegram para validar el funcionamiento del flujo.
- Se registran los datos en Google Sheets cuando el proceso requiere almacenar información.
- Se verificó el funcionamiento completo mediante pruebas exitosas.

## Evidencias

### Nuevos nodos en n8n

![Nodos nuevos](/node%20edit%20fields.png)

### Prueba exitosa en Telegram

![Prueba en Telegram](/puntos%20telegram.png)

### Evidencia en Google Sheets

![Google Sheets](/puntos%20sheets.png)

## 📂 Créditos técnicos

- **Plataforma de automatización:** [n8n](https://n8n.io)
- **Canal de mensajería:** [Telegram Bot API](https://core.telegram.org/bots/api)
- **Base de datos:** Google Sheets (vía Google Sheets API)
