# ⚠️ DeliveryBot — Alerta de Stock Crítico

**Estudiante:** Vladimir Acelas  
**Examen Final:** Lógica de inventario y notificaciones administrativas automáticas  
**Bot:** `@DeliveryBotCampers_bot`

---

## 📋 Descripción de la Funcionalidad

Se implementó un sistema de **alertas automáticas de stock crítico** integrado al flujo de confirmación de pedidos. Cada vez que un cliente confirma un pedido exitoso, el sistema verifica automáticamente si alguno de los productos comprados bajó de 3 unidades y notifica al administrador en tiempo real por Telegram.

---

## 🔍 Paso 1 — Identificación del Nodo

El punto exacto donde se descuenta el stock es el nodo **`Actualizar Stock`**, ubicado en el módulo de confirmación de pedidos (`CONFIRM`). Este nodo ejecuta una operación `Update Row` en la hoja `MENU` de Google Sheets, reduciendo el stock de cada producto comprado.

```
Preparar Stock [Code]
      ↓
Actualizar Stock [Google Sheets Update]   ← PUNTO DE ENGANCHE
      ↓
Verificar Stock Crítico [Code]            ← NUEVO
```

---

## ✅ Paso 2 — Validación de Umbral

Se agregó el nodo **`Verificar Stock Crítico`** (Code) inmediatamente después de `Actualizar Stock`. Este nodo:

1. Toma los productos actualizados como entrada.
2. Cruza con los datos de `Leer MENU (stock)` ya disponibles en memoria — **sin llamadas adicionales a la API** (optimización).
3. Filtra los productos con `stock ≤ 3`.
4. Retorna un item por cada producto crítico con toda la información necesaria.

```javascript
// Umbral de stock crítico
if (nuevoStock <= 3) {
  criticos.push({
    id_producto: item.id_producto,
    nombre,
    stock: nuevoStock,
    agotado: nuevoStock <= 0,
    alerta: '⚠️ ALERTA DE STOCK: El producto ' + nombre +
            ' solo tiene ' + nuevoStock + ' unidades. Favor reabastecer.'
  });
}
```

El nodo **`IF ¿Stock Crítico?`** evalúa `hayCriticos == true` y bifurca el flujo:
- **TRUE** → Envía alerta al administrador.
- **FALSE** → El flujo termina sin acción adicional.

---

## 📲 Paso 3 — Notificación Proactiva

El nodo **`Alerta Stock Admin`** (Telegram) envía el mensaje al Chat ID del administrador con el formato exacto requerido:

```
⚠️ ALERTA DE STOCK: El producto Café Americano 
solo tiene 2 unidades. Favor reabastecer.
```

**Configuración del nodo:**

| Campo | Valor |
|---|---|
| Chat ID | ID numérico del administrador |
| Text | `={{ $json.alerta }}` |
| Parse Mode | HTML |
| Retry on Fail | 3 intentos / 2000ms |

---

## 🗺️ Diagrama del Flujo Completo

```


```

---

## ⚡ Decisiones de Diseño

**¿Por qué enganchar después de `Actualizar Stock` y no antes?**  
Porque necesitamos validar el stock **real después del descuento**, no el anterior. Si validáramos antes, un producto con stock 3 que se compra en 3 unidades pasaría la validación sin alertar, pero quedaría en 0.

**¿Por qué reutilizar `Leer MENU (stock)` en vez de hacer una nueva lectura?**  
El nodo `Leer MENU (stock)` ya ejecutó su lectura en la misma ejecución y los datos están disponibles en memoria mediante `$('Leer MENU (stock)').all()`. Hacer una nueva lectura de Sheets sería redundante y añadiría latencia innecesaria — esto es una **optimización directa de recursos**.

**¿Por qué un IF separado para stock = 0?**  
La alerta de stock crítico (≤ 3) y marcar como agotado (= 0) son acciones con diferentes consecuencias. Separarlas en dos nodos IF mantiene la lógica clara y modular: el admin siempre recibe la alerta, pero la modificación del nombre solo ocurre en el caso extremo.

---

## 🧪 Cómo Probar

1. En Google Sheets → pestaña `MENU` → ajusta el stock de cualquier producto a **4**.
2. Desde Telegram, realiza un pedido de ese producto (2 unidades).
3. El stock baja a **2** (≤ 3) → se dispara la alerta automáticamente.
4. El administrador recibe en su Telegram:
   ```
   ⚠️ ALERTA DE STOCK: El producto [Nombre] 
   solo tiene 2 unidades. Favor reabastecer.
   ```
5. Para probar el caso de agotado: ajusta stock a **1** y pide 1 unidad → stock = 0 → nombre se actualiza a `(AGOTADO) [Nombre]`.

---

## 📁 Archivos del Proyecto

| Archivo | Descripción |
|---|---|
| `WF_Principal_DeliveryBot_EXAMEN.json` | Workflow principal con alerta de stock incluida |
| `WF_ManejoErrores.json` | Workflow de manejo de errores global |
| `WF_Reportes.json` | Workflow de reporte diario automático |
| `DeliveryBot_DB.xlsx` | Modelo de datos (Google Sheets) |
| `README_EXAMEN.md` | Este archivo |
