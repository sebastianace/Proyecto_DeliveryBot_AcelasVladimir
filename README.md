# 🤖 DeliveryBot — Sistema de Pedidos por Telegram

> **Autor:** Vladimir Acelas  
> **Plataforma:** n8n Cloud + Google Sheets + Telegram Bot API  
> **Bot:** `@DeliveryBotCampers_bot`

---

## 📋 Descripción General

DeliveryBot es un sistema de pedidos automatizado para una cafetería, construido íntegramente en **n8n Cloud** sin código de backend. Los clientes interactúan con el bot de Telegram para explorar el menú, armar su carrito, confirmar pedidos y consultar el estado de sus órdenes — todo en tiempo real.

La arquitectura se divide en **3 workflows independientes**, cada uno con una responsabilidad clara:

| Workflow | Trigger | Responsabilidad |
|---|---|---|
| `WF_Principal_DeliveryBot` | Telegram Webhook | Bot conversacional completo |
| `WF_ManejoErrores` | Error Trigger | Notificación de fallos al admin |
| `WF_Reportes` | Schedule (23:55) | Reporte diario automático |

---

## 🗂️ Estructura del Repositorio

```
DeliveryBot/
├── WF_Principal_DeliveryBot.json   # Workflow principal (bot conversacional)
├── WF_ManejoErrores.json           # Workflow de manejo de errores global
├── WF_Reportes.json                # Workflow de reportes diarios automáticos
├── DeliveryBot_DB.xlsx             # Modelo de datos (Google Sheets base)
└── README.md                       # Este archivo
```

---

## 🗃️ Modelo de Datos (Google Sheets)

La base de datos vive en Google Sheets con 5 hojas operativas:

### `MENU`
| Columna | Tipo | Descripción |
|---|---|---|
| id_producto | TEXT | Código único (ej: `B01`, `C03`) |
| nombre | TEXT | Nombre del producto |
| categoria | TEXT | `Bebidas`, `Comidas` o `Snacks` |
| precio | NUMBER | Precio unitario en USD |
| stock | INTEGER | Unidades disponibles |

### `PEDIDOS`
| Columna | Tipo | Descripción |
|---|---|---|
| id_pedido | TEXT | ID único generado (ej: `ORD-20260604-174008`) |
| id_usuario | TEXT | Telegram user ID del cliente |
| detalles_pedido | JSON | Array de items `[{id, nombre, precio, cantidad}]` |
| total_pago | NUMBER | Total calculado incluyendo impuestos |
| estado | TEXT | `Recibido` → `Preparación` → `En camino` → `Entregado` |
| fecha | DATE | Fecha del pedido (YYYY-MM-DD) |
| hora | TIME | Hora del pedido (HH:MM) |

### `SESSIONS`
Almacena el estado conversacional de cada usuario. El campo `carrito_temporal` guarda el carrito activo como JSON serializado, permitiendo persistencia entre mensajes sin base de datos externa.

### `USUARIOS`
Registro de usuarios únicos para analítica y futuras funcionalidades de fidelización.

### `REPORTES`
Métricas diarias generadas automáticamente por `WF_Reportes`: total de pedidos, ingresos, ticket promedio y top 3 productos.

---

## 🔄 WF_Principal_DeliveryBot — Arquitectura Detallada

### Flujo de entrada y enrutamiento

```
Telegram Trigger (Webhook)
        ↓
Normalizar Entrada  [Code]
  • Unifica mensajes de texto y callback_query en {action, arg1, arg2}
  • Mapea comandos /start /menu /carrito → acciones internas
  • Detecta formato "B01 2" → ADDITEM
        ↓
Router  [Switch — 10 salidas]
  HOME | MENU | CAT | ADDITEM | CART | CLEAR | CONFIRM | MYORDERS | STATUS | UNKNOWN
```

### Módulo CATÁLOGO (CAT)
Lee la hoja `MENU` **filtrada por categoría** directamente en Google Sheets — no descarga toda la hoja, solo las filas de la categoría seleccionada. Construye la lista y genera botones inline dinámicos según el stock disponible (productos agotados aparecen marcados pero sin botón de compra).

### Módulo AGREGAR AL CARRITO (ADDITEM)
```
Leer MENU (item filtrado) ──→ Leer Sesión (usuario)
                                        ↓
                              Agregar al Carrito [Code]
                              • Valida existencia del producto
                              • Verifica stock suficiente
                              • Suma al carrito existente en sesión
                                        ↓
                              IF ¿Stock OK?
                              true  → Guardar Sesión → Confirmar Item
                              false → Aviso Sin Stock
```

### Módulo CONFIRMAR PEDIDO (núcleo)
Es el módulo más crítico del sistema. Ejecuta las siguientes operaciones en secuencia y en paralelo:

```
Leer Sesión → Leer MENU (stock completo)
                        ↓
               Procesar Pedido [Code]
               • Re-valida stock (otro usuario pudo agotarlo)
               • Calcula total
               • Genera ID único (ORD-YYYYMMDD-HHmmss)
                        ↓
               IF ¿Pedido OK?
               ┌── true ──────────────────────────────────────┐
               ↓                                              ↓
       Guardar Pedido (append)                   Preparar Stock [Code]
       Limpiar Sesión (carrito=[])               (expande a N items)
       Avisar Cliente                                         ↓
       Avisar Cocina                             Actualizar Stock (update batch)
               └── false ────────────────────────┐
                                        Pedido Rechazado
```

**Decisión de diseño clave:** la actualización de stock se hace en un **único nodo Google Sheets Update** que procesa N productos en paralelo (una sola llamada a la API), en lugar de N nodos secuenciales. Esto es la optimización de recursos más visible del sistema.

---

## 🛡️ Gestión de Errores y Resiliencia

### Nivel 1 — Retry automático por nodo
Todos los nodos de Google Sheets y Telegram tienen configurado:
- **Retry on Fail:** activado
- **Max retries:** 3 intentos
- **Wait between retries:** 2000 ms

Esto absorbe fallos transitorios de red o de la API sin interrumpir la conversación.

### Nivel 2 — Error Workflow global
`WF_ManejoErrores` está configurado como Error Workflow de `WF_Principal`. Cualquier excepción no capturada por el retry (falla permanente, token inválido, Sheets caído) dispara automáticamente este workflow, que:
1. Formatea el error con nodo afectado, mensaje y timestamp (hora Colombia).
2. Notifica al administrador por Telegram con el ID de ejecución para trazabilidad.

### Nivel 3 — Validaciones en lógica de negocio
El nodo `Procesar Pedido` re-valida el stock contra `MENU` en el momento de confirmar, no en el de agregar al carrito. Esto previene condiciones de carrera (race conditions) donde dos usuarios podrían pedir el último ítem simultáneamente.

---

## ⚡ Optimización de Recursos

| Técnica | Implementación | Impacto |
|---|---|---|
| Lectura filtrada | `filtersUI` en nodos Sheets de CAT, ADDITEM, MYORDERS | Reduce filas transferidas |
| Actualización por lote | 1 nodo Sheets Update para N productos | 1 llamada API vs. N llamadas |
| Estado en sesión | Carrito serializado como JSON en SESSIONS | Sin llamadas extra para reconstruir carrito |
| Workflows separados | Reportes y errores no corren en el bot conversacional | No añaden latencia al flujo del usuario |

---

## 🌿 Lógica de Decisiones y Ramificación

El sistema tiene **4 puntos de ramificación explícitos**:

1. **Router (Switch):** enruta 10 acciones distintas desde un único punto de entrada.
2. **IF ¿Stock OK?:** bifurca entre confirmar adición al carrito o avisar agotamiento.
3. **IF ¿Pedido OK?:** bifurca entre confirmar el pedido completo o rechazarlo.
4. **Condicional en Construir Lista:** oculta el botón de compra para productos con stock = 0.

---

## 🔗 Integración de Webhooks y Salidas

- **Entrada:** `Telegram Trigger` opera como webhook registrado automáticamente por n8n al publicar el workflow. Recibe tanto mensajes de texto (`message`) como clics de botones inline (`callback_query`).
- **Salidas múltiples:** cada confirmación de pedido genera **2 mensajes Telegram simultáneos** — uno al cliente y uno al canal de cocina.
- **Salida persistente:** cada pedido se escribe en Google Sheets (`PEDIDOS`) para historial y reportes.
- **Salida programada:** `WF_Reportes` escribe en `REPORTES` y envía resumen al admin cada noche a las 23:55.

---

## 🚀 Instalación y Configuración

### Prerrequisitos
- Cuenta en [n8n Cloud](https://n8n.io)
- Bot de Telegram creado con [@BotFather](https://t.me/botfather)
- Google Sheet basado en `DeliveryBot_DB.xlsx` (publicado, con acceso de edición para la cuenta OAuth)

### Pasos
1. **Crear credenciales en n8n:**
   - `Telegram API` → pegar el token del bot.
   - `Google Sheets OAuth2` → autenticar con la cuenta de Google del Sheet.

2. **Importar workflows** en este orden:
   ```
   WF_ManejoErrores.json  →  activar/publicar
   WF_Principal_DeliveryBot.json  →  asignar credenciales  →  publicar
   WF_Reportes.json  →  asignar credenciales  →  activar
   ```

3. **Enlazar error workflow:**  
   `WF_Principal` → ⚙️ Settings → Error Workflow → `WF_ManejoErrores`

4. **Configurar chat IDs:**  
   Reemplazar `PEGA_TU_CHAT_ID_ADMIN` en `WF_ManejoErrores` y `WF_Reportes` con el ID numérico del administrador (obtener con `@userinfobot`).

5. Escribir `/start` al bot — el sistema está operativo.

---

## 🧪 Pruebas Realizadas

| Escenario | Resultado |
|---|---|
| Explorar menú por categoría | ✅ Lista filtrada con stock en tiempo real |
| Agregar producto con stock | ✅ Carrito actualizado en SESSIONS |
| Agregar producto sin stock | ✅ Mensaje de aviso, carrito sin cambios |
| Confirmar pedido válido | ✅ Pedido en PEDIDOS, stock decrementado, cliente y cocina notificados |
| Ver carrito acumulado | ✅ Resumen con items y total |
| Vaciar carrito | ✅ SESSIONS reseteado a `[]` |
| Consultar historial | ✅ Últimos 5 pedidos con estado |
| Fallo simulado (nodo) | ✅ WF_ManejoErrores notifica al admin |

---

## 📐 Decisiones de Diseño

**¿Por qué Google Sheets como base de datos?**  
Para este contexto académico y de cafetería, Sheets ofrece visibilidad directa de los datos, facilidad de modificación del menú por personal no técnico, y eliminación de costos de infraestructura de base de datos.

**¿Por qué un solo workflow principal en lugar de microworkflows?**  
Mantener el flujo conversacional en un único workflow reduce la latencia (no hay overhead de llamadas entre workflows) y facilita la trazabilidad en las ejecuciones. Los concerns genuinamente independientes (errores, reportes) sí están separados.

**¿Por qué re-validar el stock en CONFIRM?**  
El carrito se arma a lo largo del tiempo. Entre que el usuario agrega un producto y confirma, otra persona puede agotarlo. La re-validación en el momento de confirmar es la única forma de garantizar consistencia.
