# Bot Administrativo — Cobros, Ventas y CxP (n8n)

Bot de Telegram para gestión administrativa sobre datos de ERP: consulta de cartera, reporte de pagos con comprobante, pedidos tipo carrito y pagos a proveedores, con aprobación humana por correo. Construido en **n8n + Supabase + Claude (AI Agent)**, con entrada por **texto, botones o nota de voz**.

## Arquitectura

```
Telegram ──► Nucleo Cobros ──┬─► Módulo Cobros - Consultas   (AI Agent por rol: cliente / vendedor / analista CxP)
  (texto o voz)              ├─► Módulo Cobros - Reportar Pago
                             ├─► Módulo Ventas - Pedido
                             └─► Módulo CxP - Reportar Pago

Correo (Aprobar / Rechazar) ──► Webhook Aprobacion Cobros
                            └─► Webhook Aprobacion Pago Proveedor
```

| Archivo | Rol |
|---|---|
| `nucleo-cobros.json` | Entrada única: voz→texto, licencia por módulo, login por chat, enrutamiento por flujo activo |
| `modulo-cobros-consultas.json` | Tres AI Agents (cliente, vendedor, CxP) con tools sobre Supabase y sub‑flujos |
| `modulo-cobros-reportar-pago.json` | Máquina de estados: factura → monto → método → referencia → comprobante |
| `modulo-cobros-aviso-deuda-cliente.json` | Tool del vendedor: envía estado de cuenta por correo al cliente |
| `modulo-ventas-pedido.json` | Carrito multi‑producto con validación de stock y aprobación del vendedor |
| `modulo-cxp-reportar-pago.json` | Reporte de pago a proveedor con aprobación del gerente CxP |
| `webhook-aprobacion-cobros.json` | Aprueba/rechaza cobros y actualiza saldos |
| `webhook-aprobacion-pago-proveedor.json` | Aprueba/rechaza pagos a proveedores |

### Entrada por voz

En el Núcleo, `01a - IF Es Voz` detecta notas de voz o audios. Si es voz: descarga el archivo desde Telegram, lo transcribe con Gemini y reinyecta el texto como si el usuario lo hubiera escrito. El resto del bot no distingue entre voz y texto.

## Importar

1. En n8n: **Workflows → Import from File**, uno por uno. Importa primero los módulos y webhooks y al final el Núcleo.
2. Si los IDs de flujo cambian al importar, re‑selecciona el sub‑flujo en los nodos `Call '…'` y `Tool_…`.
3. Crea y asigna las credenciales (se importan solo por nombre):

| Credencial | Tipo en n8n |
|---|---|
| Telegram account 4 | Telegram API |
| Supabase account 2 | Supabase API |
| Gmail account | Gmail OAuth2 |
| Anthropic account 2 | Anthropic |
| Postgres account | Postgres (memoria de chat) |
| Google Gemini(PaLM) Api account | Google Gemini (transcripción de voz) |

4. Reemplaza los placeholders:

| Placeholder | Valor |
|---|---|
| `__TELEGRAM_BOT_TOKEN__` | Token del bot (nodos HTTP Request a `api.telegram.org`) |
| `__SUPABASE_PROJECT_REF__` | Ref del proyecto Supabase (URLs de Storage) |
| `__SUPABASE_SERVICE_ROLE_KEY__` | Key de Supabase para Storage |
| `__N8N_HOST__` | Dominio de tu instancia n8n (links de aprobación) |
| `__CLIENTE_ID__` | ID del cliente en `licencias_modulos` |

> Recomendado: en vez de pegar token/keys en los HTTP Request, usa *Authentication → Predefined/Generic Credential* para que nunca queden en el JSON.

## Base de datos

Schema `cobros_demo` en Supabase con tablas prefijadas estilo Profit Plus (`scxc_`, `scxp_`, `sinv_`, `sven_`), sesiones del bot (`bot_sesiones`, `vista_sesion_activa`), `licencias_modulos`, `configuracion` y bucket de Storage `comprobantes-pago`.
