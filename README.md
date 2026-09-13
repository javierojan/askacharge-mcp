# askacharge.com MCP server

[![Listed on mcpservers.org](https://mcpservers.org/badge.svg)](https://mcpservers.org/servers/javierojan/askacharge-mcp)
[![Glama score](https://glama.ai/mcp/connectors/com.askacharge/askacharge/badges/score.svg)](https://glama.ai/mcp/connectors/com.askacharge/askacharge)

**Let an AI agent operate a network of EV charge points.**

[askacharge.com](https://askacharge.com/askacharge/) is a CSMS — a Charging Station Management
System — for electric vehicle charging: OCPP 1.6J and 2.0.1 to the hardware, tariffs and billing,
QR and RFID driver payments, OCPI 2.2 roaming between operators, and fiscal invoicing. This
repository documents its **remote MCP server**, which exposes that platform to any
[Model Context Protocol](https://modelcontextprotocol.io) client.

Nothing to install. It is a hosted endpoint:

```
https://askacharge.com/askacharge/api/mcp
```

Transport is JSON-RPC 2.0 over HTTP POST (`streamable-http`). Protocol versions accepted:
`2025-06-18`, `2025-03-26`, `2024-11-05`. `GET` returns 405 on purpose — there is no SSE stream.

---

## Why this exists

Charge point operators spend their day on questions an agent can answer directly: *which chargers
are offline right now, what did site X earn last month, is it cheap enough to charge at this hour,
reboot the one that stopped responding.* The REST API has always been able to do that. MCP is what
makes it usable by an agent without anyone writing an integration first — the value is not the
protocol, it is the description of when to use each tool and what happens if you get it wrong.

## Authentication

One header, the brand's API key, created from the panel under *API keys*:

```
Authorization: Bearer ask_live_...
```

`X-Api-Key: ask_live_...` works too.

Once a key is present, its **scopes decide which tools you can see**. A read-only key never gets shown
`askacharge_parar_carga`. A key restricted to one location (`commands:write@hotel-madrid`) sees the
command tools and receives a 403 when it aims them at a charger somewhere else. A key belongs to one
brand and reaches only that brand's data.

## The 21 tools

**Read**

| Tool | What it answers |
|---|---|
| `askacharge_listar_cargadores` | Every charge point with OCPP status, location and last contact |
| `askacharge_ver_cargador` | One charger in detail: connectors, power, access mode, status |
| `askacharge_estado_en_vivo` | Right now: chargers connected, sessions running, power being drawn |
| `askacharge_resumen` | Period KPIs — energy, revenue, net margin, sessions, customers |
| `askacharge_listar_sesiones` | Charging session history with energy, revenue and cost |
| `askacharge_listar_incidencias` | Open faults per charger |
| `askacharge_listar_tarifas` | Tariff groups: per kWh, per hour, per session, PVPC or free |
| `askacharge_precio_pvpc_hoy` | Today's hour-by-hour Spanish PVPC electricity price |
| `askacharge_listar_clientes` | Fleet customers with their tariff and monthly consumption |
| `askacharge_limites_de_potencia` | kW ceiling per location and load-balancing strategy |
| `askacharge_traza_de_actividad` | What the brand's API keys did: writes and rejected attempts |

**OCPP commands** — these act on physical hardware

| Tool | What it does |
|---|---|
| `askacharge_arrancar_carga` | Remote start (OCPP `RemoteStartTransaction`) |
| `askacharge_parar_carga` | Remote stop (OCPP `RemoteStopTransaction`) — cuts a live session |
| `askacharge_reiniciar_cargador` | Reboot, soft or hard (OCPP `Reset`) |
| `askacharge_cambiar_disponibilidad` | Put a charger in or out of service (OCPP `ChangeAvailability`) |
| `askacharge_diagnosticos` | Ask the charger to upload its diagnostics file (OCPP `GetDiagnostics`) |

**Write**

| Tool | What it does |
|---|---|
| `askacharge_crear_cargador` | Register a charge point |
| `askacharge_crear_tarifa` | Create a tariff group (`per_kwh`, `per_hour`, `per_session`, `pvpc_margin`, `free`…) |
| `askacharge_crear_cliente` | Register a fleet customer |
| `askacharge_crear_tag_rfid` | Authorise an RFID tag on the brand |

**Escape hatch**

| Tool | What it does |
|---|---|
| `askacharge_llamar_api` | Call any of the platform's 438 API operations, under the same permission checks |

## It does not reimplement the product

Every tool runs against askacharge.com's own REST API with the same key, so it passes through the
same scopes, the same audit trail, the same rate limit and the same idempotency handling. If a tool
could ever do something the API forbids, that is a bug in the server, not a feature. The envelope
itself is not what gets logged — the concrete operation it triggered is.

## Using it

### Claude Code

```bash
claude mcp add --transport http askacharge https://askacharge.com/askacharge/api/mcp \
  --header "Authorization: Bearer ask_live_..."
```

### Clients that take a JSON config

```json
{
  "mcpServers": {
    "askacharge": {
      "type": "http",
      "url": "https://askacharge.com/askacharge/api/mcp",
      "headers": { "Authorization": "Bearer ask_live_..." }
    }
  }
}
```

### Raw, to check it is alive

```bash
curl -s -X POST https://askacharge.com/askacharge/api/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

No key needed for that: `initialize`, `ping` and `tools/list` answer anonymously, and the
anonymous `tools/list` returns the whole catalogue of 21 tools (it is public anyway — it is the
table above). Everything else — `tools/call` — needs the key, and without one you get a JSON-RPC
error (`-32001`) saying so. A key that is present but invalid is an error too, not a fallback to
anonymous.

## Getting a key

Create an account at [askacharge.com](https://askacharge.com/askacharge/register) — 30-day trial,
no card — then *API keys* in the panel. Pricing is a flat €20/month with unlimited chargers and no
software commission per session; the whole list, including what does carry a percentage, is at
[askacharge.com/askacharge/en/pricing.html](https://askacharge.com/askacharge/en/pricing.html).

## Honest limits

- **Tool names and descriptions are in Spanish.** The platform's first market is Spain and the
  Basque Country. Models handle it without trouble, but you should know before you wire it in.
- **Scoping a key by location restricts what it can act on, not what it can read.** A listing still
  returns the whole brand. This is documented behaviour, not an oversight.
- **There is no separate sandbox.** To try things without consequences, create a brand in trial mode.
- **No SSE stream.** Request/response only.

A fuller statement of what the platform is and is not — including current architectural limits —
lives at [askacharge.com/llms.txt](https://askacharge.com/llms.txt).

## Related

- [Technical documentation](https://askacharge.com/askacharge/docs.html) · [in English](https://askacharge.com/askacharge/en/docs.html)
- [OpenAPI 3.1 contract](https://askacharge.com/askacharge/openapi.json)
- [`askacharge` Python SDK](https://pypi.org/project/askacharge/) · [source](https://github.com/javierojan/askacharge-python)

---

## En español

Servidor MCP remoto de **askacharge.com**, la plataforma de gestión de cargadores de vehículo
eléctrico (CSMS) con OCPP 1.6J/2.0.1, roaming OCPI 2.2, pago por QR y facturación fiscal
(Verifactu y TicketBAI/Batuz).

Endpoint: `https://askacharge.com/askacharge/api/mcp`. No hay nada que instalar. Se autentica con la
API key de la marca en `Authorization: Bearer`, y **los scopes de esa key deciden qué herramientas
se ven**: una key de solo lectura nunca ve las de parar una carga, y una acotada por ubicación
(`commands:write@hotel-madrid`) recibe un 403 si apunta a un cargador de otra sede.

Son 21 herramientas —cargadores, estado en vivo, sesiones, tarifas, PVPC, clientes, límites de
potencia, comandos OCPP y altas— más `askacharge_llamar_api`, que abre las 438 operaciones de la
API con el mismo control de permisos. Ninguna reimplementa lógica de negocio: todas se ejecutan
contra la propia API REST, con los mismos scopes, la misma traza y el mismo límite de uso.

Cuenta y key: [askacharge.com](https://askacharge.com/askacharge/register), 30 días de prueba sin
tarjeta. Precios en [askacharge.com/askacharge/precios.html](https://askacharge.com/askacharge/precios.html).

## License

MIT — see [LICENSE](LICENSE).

## Registry

Published in the official [MCP Registry](https://registry.modelcontextprotocol.io) as
`com.askacharge/askacharge`, authenticated against the askacharge.com domain.

```bash
curl -s "https://registry.modelcontextprotocol.io/v0/servers?search=askacharge"
```

## See also

- [awesome-ocpp](https://github.com/javierojan/awesome-ocpp) — curated list of OCPP servers, libraries, simulators and roaming resources.
- [askacharge-python](https://github.com/javierojan/askacharge-python) — Python SDK for the same API.
- [CSMS comparison 2026](https://askacharge.com/askacharge/en/blog/mejores-software-gestion-cargadores.html) — where askacharge.com fits among eleven platforms, with public prices.
