# Vibe-Trading — Guía de instalación y API keys

Guía de puesta en marcha de [HKUDS/Vibe-Trading](https://github.com/HKUDS/Vibe-Trading), un agente de
trading personal (FastAPI + React + LangGraph) con 90 skills financieras, 462 alphas, backtesting multi-mercado
y 55 perfiles de bróker.

- **Repo original:** https://github.com/HKUDS/Vibe-Trading
- **Versión documentada:** v0.1.15
- **Commit de referencia:** `0244ecea` (2026-09-27)
- **Licencia:** MIT

---

## TL;DR

Solo necesitas **una** clave: la de un proveedor de LLM. **Los datos de mercado son gratuitos y no requieren
ninguna clave** (Yahoo/yfinance, OKX, mootdx, AKShare entran solos al chain de fallback). Las claves de bróker
solo hacen falta si quieres operar de verdad.

---

## 1. Clave de LLM (obligatoria)

Solo una, de un proveedor. El catálogo completo está en `agent/.env.example` (líneas 1-152) y los valores por
defecto en `agent/src/providers/llm_providers.json` (24 proveedores).

| Proveedor | Dónde se consigue | Variable | Modelo por defecto |
|-----------|-------------------|----------|---------------------|
| **OpenRouter** (recomendado) | [openrouter.ai/keys](https://openrouter.ai/keys) | `OPENROUTER_API_KEY` | `deepseek/deepseek-v4-pro` |
| DeepSeek | [platform.deepseek.com](https://platform.deepseek.com) → API keys | `DEEPSEEK_API_KEY` | `deepseek-v4-pro` |
| Anthropic | [console.anthropic.com](https://console.anthropic.com) → API Keys | `ANTHROPIC_API_KEY` | `claude-sonnet-4-6` |
| OpenAI | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) | `OPENAI_API_KEY` | `gpt-5.5` |
| Gemini | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) | `GEMINI_API_KEY` | `gemini-3.5-flash` |
| Groq | [console.groq.com/keys](https://console.groq.com/keys) | `GROQ_API_KEY` | `meta-llama/llama-4-maverick-17b-128e-instruct` |
| OpenCode (Go/Zen) | [opencode.ai/zen](https://opencode.ai/zen) | `OPENCODE_API_KEY` | `deepseek-v4.1-flash` |
| NVIDIA NIM | [build.nvidia.com](https://build.nvidia.com) | `NVIDIA_API_KEY` | `nvidia/nemotron-3-ultra-550b-a55b` |
| SiliconFlow CN | siliconflow.cn | `SILICONFLOW_API_KEY` | `deepseek-ai/DeepSeek-V3.1-Terminus` |
| SiliconFlow Global | siliconflow.com | `SILICONFLOW_GLOBAL_API_KEY` | `deepseek-ai/DeepSeek-V3.1-Terminus` |
| ModelScope | modelscope.cn | `MODELSCOPE_API_KEY` | `Qwen/Qwen3.5-27B` |
| Novita AI | [novita.ai/settings/key-management](https://novita.ai/settings/key-management) | `NOVITA_API_KEY` | `moonshotai/kimi-k3` |
| DashScope / Qwen | bailian.console.aliyuns.com | `DASHSCOPE_API_KEY` | `qwen-plus-latest` |
| Moonshot / Kimi | platform.moonshot.ai | `MOONSHOT_API_KEY` | `kimi-k2.6` |
| Kimi for Coding | [kimi.com/code/docs](https://kimi.com/code/docs) | `KIMI_CODING_API_KEY` | `kimi-for-coding` |
| Zhipu / GLM | open.bigmodel.cn | `ZHIPU_API_KEY` | `glm-5.1` |
| Z.ai (Coding Plan) | z.ai | `ZAI_API_KEY` | `glm-5.1` |
| MiniMax | minimaxi.com (CN) / minimax.io (global) | `MINIMAX_API_KEY` | `MiniMax-M3` |
| Xiaomi MIMO | api.xiaomimimo.com | `MIMO_API_KEY` | `MiMo-72B-A27B` |
| iFlytek Spark | [console.xfyun.cn](https://console.xfyun.cn) | `SPARK_API_KEY` | `4.0Ultra` |
| **Ollama** (local) | `ollama pull qwen2.5:32b` | — (sin clave) | `qwen2.5:32b` |
| **OpenAI Codex** (OAuth) | `vibe-trading provider login openai-codex` | — (sin clave) | `openai-codex/gpt-5.4` |
| **GitHub Copilot** | `gh auth login` | — (sin clave) | `claude-sonnet-5` |

**Con una sola clave basta**: si no defines `*_BASE_URL`, cada proveedor recurre automáticamente a su endpoint
canónico (`agent/src/providers/llm_providers.json`, campo `default_base_url`).

### Modelos recomendados

Vibe-Trading es un agente que depende intensamente de herramientas: skills, backtests, memoria y swarms fluyen
todos por llamadas a herramientas. La elección del modelo decide si el agente *usa* sus herramientas o fabrica
respuestas a partir de datos de entrenamiento.

| Nivel | Ejemplos | Cuándo usarlo |
|-------|----------|----------------|
| **Best** | `anthropic/claude-opus-4.7`, `anthropic/claude-sonnet-4.6`, `openai/gpt-5.5-pro`, `google/gemini-3.5-flash` | Swarms complejos (3+ agentes), sesiones largas, análisis de calidad de paper |
| **Sweet spot** (recomendado) | `deepseek/deepseek-v4-pro`, `x-ai/grok-4.20`, `z-ai/glm-5.1`, `moonshotai/kimi-k2.6`, `qwen/qwen3-max-thinking` | Uso diario — llamadas a herramientas fiables a ~1/10 del coste |
| **Evitar** | `*-nano`, `*-flash-lite`, `*-coder-next`, variantes pequeñas/destiladas | Las tool calls no son fiables: el agente "responde de memoria" en vez de cargar skills o ejecutar backtests |

> **Aviso de Anthropic:** para alcanzar Claude usa `LANGCHAIN_PROVIDER=anthropic` (nativo) o `ANTHROPIC_BASE_URL`
> apuntando a un proxy compatible. **No** es una ruta soportada poner `OPENAI_BASE_URL` en el endpoint
> OpenAI-compatible de Anthropic: eso toma la rama genérica de OpenAI, que no lleva la forma de request de Anthropic.

---

## 2. Claves de datos (opcionales)

Todas tienen fallback gratuito. Se activan **únicamente** si la variable está definida; si no, se omiten en
silencio (`agent/.env.example` líneas 213-226).

### Sin clave (ya funcionan por defecto)

| Mercado | Fuente |
|---------|--------|
| HK / US / Canadá / Reino Unido | Yahoo Finance / yfinance |
| Cripto | OKX (API pública), + 100 exchanges vía CCXT |
| Acciones A (China) | mootdx (TCP directo, sin límite de IP), AKShare, Eastmoney, Sina |
| Macro | AKShare y fuentes directas |

### Con clave (opcionales)

| Variable | Dónde se consigue | Para qué sirve |
|----------|-------------------|----------------|
| `TUSHARE_TOKEN` | [tushare.pro](https://tushare.pro) (2000 pts gratis) | Acciones A — se cuela en la cadena y da prioridad |
| `FINNHUB_API_KEY` | [finnhub.io](https://finnhub.io) | Fallback OHLCV US |
| `ALPHAVANTAGE_API_KEY` | [alphavantage.co](https://www.alphavantage.co) | Fallback OHLCV US |
| `TIINGO_API_KEY` | [tiingo.com](https://www.tiingo.com) | Fallback OHLCV US |
| `FMP_API_KEY` | [financialmodelingprep.com](https://financialmodelingprep.com) | Fallback OHLCV US |
| `GILDATA_TOKEN` | comercial vía datamap@gildata.com | Gildata (恒生聚源), A-share |
| `FRED_API_KEY` | [fred.stlouisfed.org/api](https://fred.stlouisfed.org/docs/api/fred/) | Series macro (herramienta `get_macro_series`) |
| `VIBE_TRADING_IWENCAI_KEY` | iwencai.com | Búsqueda en lenguaje natural sobre A-shares |
| `QVERIS_API_KEY` | [qveris.com](https://www.qveris.com) | Datos premium: Greeks de opciones, fundamentales, 63+ proveedores |
| `TICKERALL_API_KEY` + `TICKERALL_ACCOUNT_ID` | [tickerall.com](https://tickerall.com) | Feed forex/metales vía MetaTrader 5 alojado |
| `VIBE_TRADING_SEC_UA` | — | User-Agent de contacto para SEC EDGAR (hay un default integrado) |

**Otros conectores de datos con clave:** Futu OpenAPI (FutuOpenD local, `FUTU_HOST`/`FUTU_PORT`),
LongPort (`open.longbridge.com` → `LONGBRIDGE_APP_KEY` / `LONGBRIDGE_APP_SECRET` / `LONGBRIDGE_ACCESS_TOKEN`),
eToro (`ETORO_API_KEY` / `ETORO_USER_KEY`).

### Prioridad de fuentes

Se puede reordenar la cadena por mercado (el valor debe ser una **permutación** de la cadena por defecto, no
puede eliminar fuentes):

```bash
# Settings -> Data Source Priority en la Web UI, o en el .env:
MARKET_DATA_ORDER_A_SHARE=tushare,tencent,mootdx,eastmoney,baostock,akshare,gildata,local
```

> **Cuidado con el "caliber":** las fuentes difieren en base de ajuste (yahoo/yfinance sirven OHLC *raw*;
> tencent/eastmoney/tushare/baostock/gildata sirven precios *ajustados*). Reordenar la cadena **puede cambiar los
> números de un backtest**.

---

## 3. Claves de bróker (solo si quieres operar)

55 perfiles registrados. Lístalos con:

```bash
vibe-trading connector list
```

### Convención de nombres — el sufijo indica el riesgo

| Sufijo | Significado |
|--------|-------------|
| `*-paper-sdk`, `*-paper-trade` | Demo / testnet |
| `*-live-sdk-readonly` | Lectura sobre cuenta real |
| `*-live-trade` | **Dinero real** — coloca órdenes |
| `*-live-mcp*`, `*-local`, `*-local-readonly` | MCP remoto o socket local (TWS) |

### Brókers soportados

IBKR · Robinhood · Alpaca · OKX · Binance · Tiger (Futu) · Longbridge · Dhan · Shoonya · Trading 212 ·
MetaTrader 5 · eToro · Zerodha (Kite) · KIS (Corea) · Upbit · Toss Securities · Scalable Capital

### Ruta más rápida: IBKR local

Las credenciales se quedan dentro de la app de escritorio de IBKR; Vibe-Trading solo se conecta a `127.0.0.1`.

```bash
pip install "vibe-trading-ai[ibkr]"
```

Abre TWS en modo paper (o IB Gateway paper), habilita los clientes de socket de API, y luego:

```bash
vibe-trading connector list
vibe-trading connector use ibkr-paper-local
vibe-trading connector configure ibkr-paper-local --yes
vibe-trading connector check
vibe-trading connector account
vibe-trading connector positions
vibe-trading connector orders
vibe-trading connector quote AAPL
vibe-trading connector history AAPL --duration "30 D" --bar-size "1 day"
```

| App | Paper | Real (solo lectura) |
|-----|-------|----------------------|
| TWS | `7497` | `7496` |
| IB Gateway | `4002` | `4001` |

Herramientas que expone el agente: `trading_connections`, `trading_select_connection`, `trading_check`,
`trading_account`, `trading_positions`, `trading_orders`, `trading_quote`, `trading_history`.

### Modo TAP — aislamiento de credenciales (opcional, desactivado por defecto)

[TAP](https://tap.human.tech) es un proxy de credenciales: el agente **nunca** posee el secreto crudo del bróker,
y las escrituras quedan bloqueadas hasta **aprobación humana**.

1. En el panel de TAP crea una credencial multi-secreto llamada `alpaca` con tu par de claves en los campos
   `key_id` y `secret_key`, con hosts permitidos `paper-api.alpaca.markets` **y** `data.alpaca.markets`.
   Usa credenciales **separadas** para paper y real (`TAP_ALPACA_CREDENTIAL=alpaca-paper` / `alpaca-live`).
2. Añade al `.env`:

| Variable | Requerida | Descripción |
|----------|:---------:|-------------|
| `TAP_PROXY_URL` | Sí | URL base del proxy (`https://proxy.tap.human.tech`) |
| `TAP_AGENT_KEY` | Sí | Tu clave de API del agente TAP |
| `TAP_ALPACA_CREDENTIAL` | No | Nombre de la credencial (por defecto `alpaca`) |
| `TAP_APPROVAL_TIMEOUT` | No | Segundos esperando la decisión humana (300) |

- **Escrituras** (colocar/cancelar órdenes) se retienen hasta aprobación. Llevan un `client_order_id`
  determinístico, así que un reintento se deduplica en lugar de colocar la orden dos veces.
- **Lecturas** se aprueban automáticamente (son GETs) — es *aislamiento* de credenciales, no una compuerta.
- `allowed_hosts` fija a dónde puede enviarse la clave; un destino manipulado se rechaza con 403 antes de inyectar.
- **Alcance:** solo Alpaca. Los brókers con firma HMAC (Binance/OKX) quedan fuera: la firma del lado del cliente no
  encaja con la inyección de tráfico de salida.

---

## 4. Instalación

### Requisitos

- **Python 3.11+** (ruta local)
- **Node >= 22.22** (solo si compilas la Web UI)
- **Docker** (opcional, ruta A)
- Una clave de LLM, u **Ollama** local sin clave

### Ruta A — Docker (configuración cero)

```bash
git clone https://github.com/HKUDS/Vibe-Trading.git
cd Vibe-Trading
cp agent/.env.example agent/.env
# Edita agent/.env: descomenta tu proveedor de LLM y pon la API key
docker compose up --build
```

Abre `http://localhost:8899`. Backend + frontend en un mismo contenedor.

Los datos sobreviven a actualizaciones (volúmenes Docker con nombre): `git pull && docker compose up --build`.
Solo se borran con `docker compose down -v`.

> **Ollama con Docker:** el contenedor accede al host vía `host.docker.internal`, no `localhost`.
> `docker-compose.yml` ya lo define. Requiere Docker Engine >= 20.10 / Compose v2.

### Ruta B — Instalación local (Windows / PowerShell)

```powershell
cd C:\Proyectos\Vibe-Trading
python -m venv .venv

# Activar (si PowerShell se niega, ejecuta primero la línea siguiente)
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1

pip install -e .
vibe-trading init                  # asistente interactivo de configuración
vibe-trading provider doctor       # diagnóstico con todo redactado
vibe-trading                       # abre la TUI
```

> Si `pip install` falla, casi siempre es la versión de Python: 3.14 es muy reciente para el stack de
> dependencias. Crea el venv con **3.12** y reintenta.

### Ruta C — Plugin MCP

Conecta el agente a Claude Desktop, OpenClaw, Cursor, etc. Expone **74 herramientas MCP** por stdio.

```json
{
  "mcpServers": {
    "vibe-trading": {
      "command": "vibe-trading-mcp",
      "env": { "VIBE_TRADING_ALLOWED_RUN_ROOTS": "C:\\Users\\me\\research" }
    }
  }
}
```

Ubicaciones: `claude_desktop_config.json` (Claude Desktop), `~/.openclaw/config.yaml` (OpenClaw).

**Las variables de entorno hay que ponerlas en el bloque `env` del cliente**: el cliente lanza el servidor él
mismo, así que un `export` de shell nunca le llega.

### Ruta D — ClawHub (un comando, sin clonar)

```bash
npx clawhub@latest install vibe-trading --force
```

---

## 5. Configuración: dónde se escribe el `.env`

### Orden de búsqueda (importante)

```
~/.vibe-trading/.env   →   agent/.env   →   ./.env (cwd)
```
*(fuente: `agent/src/providers/llm.py:843-856`)*

> **Trampa habitual:** `vibe-trading init` escribe **solo** en `~/.vibe-trading/.env`, **nunca** en
> `agent/.env`. Si editas a mano `agent/.env` después de haber corrido el asistente, la configuración de la
> home tiene prioridad y tus cambios se ignoran en silencio.

### El asistente `vibe-trading init`

100% interactivo, sin flags. Pasos:

1. ¿Sobrescribir `~/.vibe-trading/.env` si existe?
2. Proveedor (número, 20 opciones)
3. API key (entrada enmascarada, con validación de prefijo)
4. Base URL (con default del proveedor)
5. Modelo (con default del proveedor)
6. Timeout y reintentos
7. Token de Tushare (opcional)

Escribe el archivo con permisos `0600`.

### Configuración manual

Copia el ejemplo y descomenta **un** bloque de proveedor:

```powershell
Copy-Item agent\.env.example agent\.env
```

```bash
LANGCHAIN_PROVIDER=openrouter
LANGCHAIN_MODEL_NAME=deepseek/deepseek-v4-pro
OPENROUTER_API_KEY=sk-or-...
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1

LANGCHAIN_TEMPERATURE=0.0
TIMEOUT_SECONDS=120
MAX_RETRIES=2
```

### Verificar la configuración

```bash
vibe-trading provider doctor
```

Imprime diagnósticos **con todos los secretos redactados**: proveedor, modelo, base URL, presencia de la key y
de dónde salió, cabeceras de autenticación, variables `OPENAI_*` ambientales que puedan hacer sombra, proxy
(`HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY`), versiones de paquetes instalados, timeout, reintentos, reasoning
effort, tipo de adaptador (nativo vs openai-compatible) y capacidades.

### Desde la Web UI

`http://localhost:8899` → **Settings**. Permite cambiar proveedor, modelo, base URL, parámetros de generación,
reasoning effort y credenciales de datos. Los ajustes se persisten en `agent/.env`. La misma página tiene un
panel de **Canales IM** con estado, sugerencias de recuperación y arranque/parada en caliente.

---

## 6. Variables de entorno relevantes

| Variable | Obligatoria | Descripción |
|----------|:-----------:|-------------|
| `LANGCHAIN_PROVIDER` | Sí | Nombre del proveedor (`openrouter`, `deepseek`, `groq`, `ollama`…) |
| `<PROVIDER>_API_KEY` | Sí* | Clave de API del proveedor |
| `<PROVIDER>_BASE_URL` | No | Si se omite, usa el endpoint canónico |
| `LANGCHAIN_MODEL_NAME` | Sí | Nombre del modelo |
| `API_AUTH_KEY` | Recomendado en red | Token Bearer para clientes no locales |
| `CORS_ORIGINS` | No | Reemplaza los defaults de loopback |
| `VIBE_TRADING_EXTRA_CORS_ORIGINS` | No | **Añade** orígenes a los defaults |
| `VIBE_TRADING_ENABLE_SCHEDULER` | No | `1` activa el planificador en segundo plano (off por defecto) |
| `VIBE_TRADING_ENABLE_SHELL_TOOLS` | No | `1` expone `bash` / `background_run` (off por defecto) |
| `VIBE_TRADING_DATA_CACHE` | No | `1` activa la caché local de barras históricas |
| `VIBE_TRADING_HOME` | No | Reubica el estado persistente (solo variable de **shell**, no del `.env`) |
| `LANGCHAIN_REASONING_EFFORT` | No | `none` / `low` / `medium` / `high` / `max` |
| `VIBE_TRADING_SSE_TIMEOUT` | No | Timeout de idle del stream (90 s) — súbelo con modelos locales lentos |
| `VT_MEMORY` | No | `off` (default) / `on` / `full` |

<sub>* Ollama no requiere clave. OpenAI Codex usa ChatGPT OAuth y guarda los tokens vía `oauth-cli-kit`, no en el `.env`.</sub>

> `VIBE_TRADING_HOME` **no** se pone en el `.env`: las rutas se resuelven al arrancar el proceso, antes de leer
> ese archivo, así que un override ahí aplicaría a unas rutas de código y no a otras.

---

## 7. Uso

### CLI

```bash
vibe-trading                                 # TUI interactiva
vibe-trading run -p "..."                    # una sola ejecución
vibe-trading serve --port 8899               # servidor web
vibe-trading alpha list                      # explora 462 alphas (show / bench / compare / export-manifest)
vibe-trading playbook list                   # 5 plantillas de investigación programada
vibe-trading channels status --local         # estado de canales IM
vibe-trading provider doctor                 # diagnóstico redactado
vibe-trading connector list                  # 55 perfiles de bróker
```

Comandos slash dentro de la TUI: `/help`, `/model`, `/connector`, `/playbook`, y `/playbook`.

### Web UI

```powershell
# producción: un solo servidor
cd frontend; npm install; npm run build; cd ..
vibe-trading serve --port 8899          # http://localhost:8899
```

```powershell
# modo dev (dos terminales)
vibe-trading serve --port 8899         # terminal 1
cd frontend; npm run dev               # terminal 2 → http://localhost:5899
```

En modo dev el frontend hace proxy de las llamadas a la API hacia `localhost:8899`.

### Primeros smoke tests

```bash
# Backtest sencillo
vibe-trading run -p "Backtest a BTC-USDT 20/50 moving-average strategy for 2024 and summarize return and drawdown"

# Datos sin ninguna clave de datos
vibe-trading run -p "Get daily OHLCV for AAPL.US over the last 6 months"
```

### Investigación programada

El ejecutor está **desactivado por defecto**:

```bash
VIBE_TRADING_ENABLE_SCHEDULER=1 vibe-trading serve --port 8899
```

```bash
# cada 6 horas
curl -X POST http://localhost:8899/scheduled-runs \
  -H "Content-Type: application/json" \
  -d '{"prompt":"Scan CSI300 for momentum breakouts and backtest the top 5","schedule":"0 */6 * * *"}'

curl http://localhost:8899/scheduled-runs          # listar
curl -X DELETE http://localhost:8899/scheduled-runs/<job_id>
```

Plantillas incluidas: `premarket-brief`, `earnings-season-tracker`, `portfolio-checkup`, `a-share-money-flow`,
`institutional-holdings-diff`.

---

## 8. Seguridad — lo que debes saber

### Acceso remoto

`vibe-trading serve` se enlaza a `0.0.0.0` pero **por defecto solo confía en loopback**:

- Abrir la UI en la **misma máquina** → funciona sin configurar nada.
- Abrirla desde **otra máquina, un host de VM o el móvil en tu LAN** → los endpoints sensibles devuelven `403`
  y el chat muestra *"Remote API access requires an API key"*.

Solución: pon un `API_AUTH_KEY` fuerte en el `.env`, reinicia, e introduce la misma clave una vez en **Settings**.
Los clientes JSON/upload envía `Authorization: Bearer <key>`.

### Herramientas de shell

`bash` / `background_run` / `cancel_background` están habilitadas **solo** en la CLI local interactiva. La API
HTTP/SSE y el servidor MCP (en todos los transportes, incluido stdio) las mantienen **desactivadas** salvo que
optes explícitamente con `VIBE_TRADING_ENABLE_SHELL_TOOLS=1`. El tipo de transporte nunca concede acceso a shell
implícitamente.

### El sandbox de backtest es deliberadamente limitado

El código de backtest generado se ejecuta como **subproceso local de Python** con un entorno recortado. Recibe
claves de **solo lectura de datos de mercado**: `TUSHARE_TOKEN`, `FMP_API_KEY`, `FRED_API_KEY` y
`VIBE_TRADING_IWENCAI_KEY`.

Por defecto **no** recibe: claves de proveedores de LLM, tokens de autenticación de la API, interruptores de
herramientas de shell, secretos de trading de bróker, ni toggles de live/advisory.

### Rutas de archivos

Los lectores de documentos y journals están limitados por defecto a las raíces de upload/import. Para ampliar:

```bash
VIBE_TRADING_ALLOWED_FILE_ROOTS=/ruta/extra
VIBE_TRADING_ALLOWED_RUN_ROOTS=/ruta/para/codigo/generado
```

### Dónde vive el estado

Bajo `~/.vibe-trading`: sesiones, ejecuciones, swarm runs, uploads, `sessions.db` (índice de búsqueda), memoria,
skills creadas, cuentas shadow, configuración de conectores y caches. Reubicable con la variable de shell
`VIBE_TRADING_HOME`.

---

## 9. Estructura del proyecto

```
Vibe-Trading/
├── agent/                  # Backend: FastAPI, CLI, agente LangGraph, trading, backtest
│   ├── .env.example        # <-- todas las variables documentadas
│   ├── api_server.py       # Servidor web
│   ├── mcp_server.py       # Servidor MCP (stdio)
│   ├── src/
│   │   ├── config/         # env_schema.py: todos los defaults
│   │   ├── providers/      # llm_providers.json: catálogo de 24 proveedores
│   │   ├── trading/        # 55 perfiles de conector
│   │   ├── factors/zoo/    # 462 alphas
│   │   └── skills/         # 90 skills financieras
│   └── cli/
├── frontend/               # React 19 + Vite (Node >= 22.22)
├── desktop/                # App de escritorio Electron
├── wiki/                   # Documentación
└── docker-compose.yml
```

---

## 10. Problemas frecuentes

| Síntoma | Causa y solución |
|---------|------------------|
| Los cambios en `agent/.env` no se aplican | `~/.vibe-trading/.env` tiene prioridad. Edítalo ahí, o bórralo |
| El agente responde "de memoria" sin llamar a herramientas | Modelo demasiado pequeño. Evita `*-flash-lite` / `*-nano` |
| `pip install -e .` falla con Python 3.14 | Crea el venv con Python 3.12 |
| Importaciones rotas tras actualizar | `pip install --force-reinstall vibe-trading-ai`, o recrea el venv |
| 403 al abrir la UI desde otro dispositivo | Falta `API_AUTH_KEY`. Búscalo en Settings |
| Los números del backtest cambian al reordenar fuentes | Fuentes distintas usan base de ajuste distinta (raw vs ajustado) |
| `vibe-trading init` no pregunta nada | Ya existe `~/.vibe-trading/.env`; responde `Y` al prompt de sobrescribir |

---

## 11. Enlaces

- **Documentación oficial:** https://vibetrading.wiki/docs/
- **Web del proyecto:** https://vibetrading.wiki/
- **PyPI:** https://pypi.org/project/vibe-trading-ai/
- **Discord:** https://discord.gg/6TdQnT5xcF
- **Issues:** https://github.com/HKUDS/Vibe-Trading/issues

---

## Aviso de seguridad

⚠️ La cuenta de X `VibeTrading_HKU`, el proyecto de Virtuals `101845` y el contrato de token
`0x640BDBF77b6447E8b7DB7894cED84BD1c40571f4` **no son canales oficiales** de Vibe-Trading. El proyecto nunca ha
lanzado ni respaldado ningún token o memecoin. No compres, no conectes una wallet, no firmes nada.
Ver [SECURITY.md](https://github.com/HKUDS/Vibe-Trading/blob/main/SECURITY.md).

Esta guía es material no oficial de consulta rápida, elaborado a partir del repositorio público. El software
tiene fines educativos y de investigación. **No es asesoramiento financiero.** Haz siempre *due diligence* y
considera consultar a un asesor financiero profesional antes de tomar decisiones de inversión.
