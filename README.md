# ICN292 — Laboratorio 3

**Automatización del triage de devoluciones de AndesHogar SpA en n8n**

| | |
|---|---|
| **Estudiante** | Martín Farfán Cornejo |
| **Curso** | ICN292 — Gestión y Rediseño de Procesos |
| **Semilla personal S** | 161 (tres últimos dígitos del RUT, sin dígito verificador) |
| **Umbral de monto U** | 30.000 + 1.000 × (161 mod 50) = **$41.000** |
| **Plazo máximo D** | 7 + 7 × (161 mod 4) = **14 días** |
| **Fecha** | 23 de septiembre de 2026 |

---

## Archivos del repositorio

| Archivo | Contenido |
|---|---|
| `ICN292-Lab3-Farfan-Martin.pdf` | Informe del laboratorio en PDF. |
| `ICN292-Lab3-Farfan-Martin.docx` | El mismo informe en Word. |
| `ICN292-Lab3-Farfan-Martin-triage.json` | Flujo de triage (Parte A): webhook, Switch Rules, Edit Fields, UF, notificación, registro y respuesta. |
| `ICN292-Lab3-Farfan-Martin-emisor.json` | Flujo emisor: envía las solicitudes en POST hacia el webhook de triage. |
| `ICN292-Lab3-Farfan-Martin-resumen.json` | Flujo programado de resumen diario (Parte B). |
| `ICN292-Lab3-Farfan-Martin-triage-code.json` | Bonus: variante del triage con un único nodo Code. |
| `capturas/` | Capturas de las ejecuciones exitosas y de la ejecución fallida. |

Ninguno de los `.json` contiene contraseñas, tokens ni URL con credenciales. Todos tienen `pinData` vacío: las salidas que se ven en las capturas fueron producidas por el flujo y no fijadas a mano.

---

## Cómo importar y reproducir los flujos

### 1. Importar en n8n

En n8n: **Workflows → Import from File...** y seleccionar cada `.json`. Importar en este orden:

1. `...-triage.json`
2. `...-resumen.json`
3. `...-emisor.json`

### 2. Configurar credenciales

Las credenciales no vienen en los archivos; se cargan en el gestor de credenciales de n8n.

- **SMTP** (nodos *Notificar Cliente* y *Enviar Resumen*): crear una credencial SMTP y seleccionarla en el nodo.
- **Google Sheets** (nodos *Registrar Solicitud* y *Leer Registro*): crear la credencial OAuth2 y reemplazar `1-u0oLgpSm4DzWYJWA2KtU5PETuyFWcWriGSPzfYAVEM` por el ID de la planilla propia.

La planilla necesita una hoja llamada `registro` con estos encabezados en la primera fila:

```
fecha_proceso | id_solicitud | sku | monto | monto_uf | dias_desde_compra | estado_producto | ruta | motivo | U_aplicado | D_aplicado | valor_uf
```

### 3. Configurar el Error Workflow

En **Settings** de los flujos de triage y de resumen, reemplazar `REEMPLAZAR_ID_ERROR_WORKFLOW` seleccionando el flujo de error propio (un flujo que parta con un nodo *Error Trigger*).

### 4. Conectar el emisor con el webhook

1. Abrir `...-triage.json` y copiar la URL del nodo **Webhook Devoluciones**.
   - **URL de prueba** (`https://farfans.app.n8n.cloud/webhook-test/devoluciones-triage-161`): funciona solo mientras el flujo está abierto en el editor y se presionó *Execute workflow*. Atiende una llamada y muestra los datos sobre el lienzo.
   - **URL de producción** (`https://farfans.app.n8n.cloud/webhook/devoluciones-triage-161`): funciona solo con el flujo activado (*Active*). Las corridas quedan en la lista de *Executions*.
2. Abrir `...-emisor.json`, nodo **Enviar a Triage**, y reemplazar `REEMPLAZAR_POR_LA_URL_DEL_WEBHOOK_DE_TRIAGE` por la URL elegida.

Pegar la URL en el navegador no sirve: la barra de direcciones siempre hace GET y el webhook está declarado en POST.

### 5. Ejecutar

En el flujo emisor, nodo **Lote de Solicitudes**, la constante `LOTE` elige qué se envía:

| Valor | Qué envía |
|---|---|
| `'base'` | Las 15 solicitudes del curso. |
| `'limites'` | Los dos casos de frontera: monto igual a U y días iguales a D. |
| `'invalidas'` | Las tres entradas inválidas (la tercera hace fallar un nodo). |
| `'irresolubles'` | Los dos casos que el flujo no puede resolver: `"monto": "veinte mil"` y `"sku": "ZZ-999"`. |

Cambiar el valor, guardar y presionar **Execute workflow**.

### 6. Flujo de resumen

Se dispara solo con un **Schedule Trigger** diario a las 19:00 (hora de Santiago). Para probarlo sin esperar, presionar *Execute workflow* manualmente.

---

## Resultado del lote de 15 solicitudes

| Ruta | Solicitudes | Monto total |
|---|---|---|
| APROBACION | 2 | $29.980 |
| REVISION | 9 | $1.020.890 |
| RECHAZO | 4 | $200.950 |
| **Total** | **15** | **$1.251.820** |

Tasa de aprobación automática: **13,3 %**.

---

## API externa utilizada

- **URL:** `https://mindicador.cl/api` (GET, sin autenticación)
- **Campo consumido:** `uf.valor` — valor de la Unidad de Fomento en pesos del día de la consulta.
