# Plan de remediación — Vibe-Trading 0.1.15

Auditoría de capital-riesgo y seguridad. Este documento convierte los hallazgos en
trabajo ejecutable, ordenado por dependencia: cada fase deja el repo en un estado
consistente y verificable, y ninguna fase puede ejecutarse antes de que la anterior
 pase sus tests.

- **Base:** `0244ecea` (main, `HKUDS/Vibe-Trading`)
- **Alcance:** los 16 defectos de ejecución, 8 sesgos de backtest y 6 problemas de
  seguridad detectados. Absolutamente todo.
- **Ley:** ningún parche sin test de regresión que falle antes del cambio y pase después.

---

## Reglas de la casa

1. **Fail-closed es la ley.** Si no sabemos, denegamos. Un `[]` que no distingue
   "posición cero" de "lectura fallida" es un bug, no una optimización.
2. **Un commit, un defecto.** Mensajes en el estilo del repo (`fix(okx): ...`).
   Sin commits de "misc" ni de formato.
3. **Ninguna credencial real.** Solo testnets, fakes y fixtures con procedencia.
4. **Ningún cambio de comportamiento financiero sin su test.** Si un Sharpe baja,
   es información, no un motivo para revertir el fix.
5. **Windows es plataforma de primera clase.** El usuario corre Windows; cada fix de
   ledger/lock/fichero lleva su test en Windows.
6. **Ningún bypass por API.** Si una ruta programática (`service.place_order`) puede
   saltar algo que la herramienta LLM no puede, eso también se arregla.

---

## Fase 0 — Baseline y red de seguridad (antes de tocar nada)

Objetivo: saber qué se rompe hoy, para no culpar a un parche posterior.

- [ ] 0.1 Ejecutar la suite completa en Linux y Windows. Guardar línea base de
      fallos. `pytest agent/tests -q` con el `--ignore` actual **eliminado**: los dos
      paths ignorados (`e2e_backtest`, `test_e2e_harness_v2.py`) no existen, y un
      `--ignore` de algo inexistente esconde que no hay harness e2e.
- [ ] 0.2 `pytest --cov=agent/src --cov-report=term-missing` y guardar el reporte.
      No subir el umbral todavía (Fase 8).
- [ ] 0.3 Medir cobertura **específica de las rutas de dinero**, que es lo que
      importa: `agent/src/live/`, `agent/src/trading/connectors/`, `agent/backtest/engines/`.
      Objetivo de Fase 0: tener el número, no mejorarlo.
- [ ] 0.4 Instalar `python-okx` como dependencia declarable en `pyproject.toml` en
      modo opcional, para que CI pueda importar el SDK real (hoy no puede).
- [ ] 0.5 Declarar en el plan de trabajo que **no existe verificación end-to-end**
      contra ningún exchange. Esto es el hole raíz de la Fase 7.

**Criterio de salida:** línea base escrita y commiteada. Ningún cambio funcional aún.

---

## Fase 1 — Los 6 defectos que cuestan dinero

Esta fase es la que separa "el repo es interesante" de "el repo es utilizable".
Cada ítem indica el mecanismo exacto de pérdida, porque sin eso no se puede
revisar el parche.

### 1.1 · Idempotencia de órdenes en OKX y Binance  🔴 CRÍTICO

**Defecto.** `pending_order_journal.py` solo es funcional con Alpaca, porque solo
`alpaca/sdk.py:369` implementa `get_order_by_client_order_id`. OKX no envía
`clOrdId`; Binance genera uno aleatorio por orden, así que un reintento nunca puede
casar con la orden original. `FAILURE_BLOCK_THRESHOLD = 2` permite además un reintento
genérico de un envío que quizá sí llegó al exchange. Y el journal solo cubre el estado
pendiente: OKX devuelve ack síncrono, así que tras un crash no queda rastro.

**Pérdida.** Timeout de red → el exchange acepta la orden → el cliente cree que falló
→ reintenta → **dos órdenes reales**. Es el modo de fallo más probable en operación
normal.

**Trabajo.**
- [ ] Definir un `client_order_id` determinista y estable por intento lógico
      (hash de mandate + símbolo + lado + qty + timestamp de la decisión), no por
      reintento. Debe ser el mismo en el reintento.
- [ ] Enviarlo en OKX vía `clOrdId` y en Binance vía `newClientOrderId`.
- [ ] Implementar `get_order_by_client_order_id` en ambos conectores, replicando el
      patrón Alpaca: **buscar por identidad exacta del broker, nunca reenviar**.
- [ ] Persistir el journal también en estado terminal durante una ventana de
      reconciliación, no solo en pendiente.
- [ ] Reconciliación post-crash al arrancar: si hay entradas pendientes, resolverlas
      antes de permitir nueva exposición.
- [ ] Tests: timeout tras aceptación → una sola orden; reintento → misma `clOrdId`;
      crash post-ack → reconciliado; `clOrdId` duplicado rechazado por el exchange.

### 1.2 · Allowlist de host en OKX  🔴 CRÍTICO

**Defecto.** `okx/sdk.py:135` declara `_OVERRIDE_KEYS = ("api_key", "api_secret",
"passphrase", "profile", "host", "expected_uid")`. `trading_connector_tool.py:215`
pasa `host` desde los argumentos de la herramienta, y `okx/sdk.py:601-615` lo usa
literalmente como `domain=` en `AccountAPI`/`TradeAPI`/`MarketAPI`. **Confirmado
agent-reachable hoy.** Sin restricción de dominio, un agente puede apuntar las
peticiones firmadas a un host arbitrario: filtra key ID + firma HMAC y puede aceptar
un order ID forjado.

**Trabajo.**
- [ ] Validar `host` contra una allowlist explícita por entorno
      (`https://www.okx.com`, `https://aws.okx.com`, y el host de demo), con match
      exacto de scheme+host+sin credenciales embebidas ni path.
- [ ] Rechazar cualquier otro host con error explícito, no degradar a default.
- [ ] Aplicar el mismo tratamiento a Binance (que usa `ccxt`, con su propio
      mecanismo de `urls`) y a cualquier otro conector con host configurable.
- [ ] Test: `https://evil.example` rechazado; `http://` (no TLS) rechazado; host con
      userinfo rechazado; los tres hosts válidos aceptados.

### 1.3 · `get_open_orders` de OKX traga errores de negocio  🔴 CRÍTICO

**Defecto.** Devuelve `{"status":"ok","open_orders":[]}` ante rate limit, respuesta
malformada y "no hay órdenes" por igual. En un rate limit, el sistema de riesgo
cree que no hay ninguna orden resting. Un rate limit deja de ser "no opero" y pasa a
ser "no tengo visibilidad de mi riesgo".

**Trabajo.**
- [ ] Separar las tres condiciones. Solo "código de negocio OK y `data` presente" es
      `status: ok`.
- [ ] Rate limit y error de negocio → estado de fallo explícito que el gate traduce a
      denegación.
- [ ] Test de cada caso por separado, incluyendo el que ya casi existe en
      `test_position_read_error_envelope.py:93-111` (que hoy solo cubre `code != "0"`
      y `data: []`).

### 1.4 · Posiciones spot invisibles para el límite de exposición  🔴 CRÍTICO

**Defecto.** `get_positions` de OKX usa un endpoint de derivados. En una cuenta spot
devuelve `[]`. `_extract_data` (`okx/sdk.py:684-697`) devuelve `[]` ante cualquier cosa
que no sea un `Mapping` con `code == "0"`, y `_coerce_position_rows` +
`_post_trade_gross_exposure` (`enforcement.py:318-343`) tratan la lista vacía como
exposición cero. Una lista vacía itera cero veces, así que `post_exposure` colapsa a
`abs(signed_order_notional)`: **solo la orden nueva**. El `max_total_exposure_usd`
(`enforcement.py:566`) compara una orden contra el tope de todo el portafolio con toda
la exposición real ignorada.

**Esto es exactamente el fallo que `test_position_read_error_envelope.py` dice
querer evitar, y los tests lo normalizan**: `positions=[]` aparece ~10 veces en
`test_mandate_enforcement.py` como la fixture de "cuenta limpia", afirmando cada vez
que la orden se permite.

**Trabajo.**
- [ ] `_business_error` y `_extract_data` deben distinguir `None`/no-`Mapping` de
      `[]` legítimo. Añadir un centinela de lectura-no-verificada que
      `_coerce_position_rows` propague como `None` → fail-closed en
      `enforcement.py:559`.
- [ ] Detectar el tipo de cuenta y usar el endpoint correcto (spot vs derivados), o
      calcular exposición spot desde balances si no hay posiciones.
- [ ] **Reparar la fixture**: los tests de "cuenta limpia" deben usar una posición
      real, y añadir un caso explícito `lectura fallida → DENY` que hoy no existe.
- [ ] Test: OKX spot con 3 posiciones → la exposición post-trade las suma; lectura
      rota → DENY, no "solo la orden nueva".

### 1.5 · Binance ignora el veredicto del exchange  🟠 ALTO

**Defecto.** `binance/sdk.py:524+` no mira el estado real de la orden. Una `REJECTED`
o `EXPIRED` se reporta como `status: ok` y consume el daily count. En un exchange que
rechaza por filtro, silencio radio.

**Trabajo.**
- [ ] Leer el estado real de la respuesta y mapear `REJECTED`/`EXPIRED`/`CANCELED`
      (sin `FILLED`) a estado de fallo, con la razón del exchange en el mensaje.
- [ ] Solo `FILLED`/`PARTIALLY_FILLED` cuentan como aceptación.
- [ ] Test por cada estado de salida de ccxt.

### 1.6 · Fallback de ceilings a los números del modelo  🟠 ALTO

**Defecto.** Si la lectura del broker falla, los ceilings caen a los valores que
suministró el propio modelo. Un click humano lo activa. Peor:
`max_trades_per_day` y `leverage` **nunca** se limitan por el broker en ningún caso —
el circuit breaker principal es autodeclarado.

**Trabajo.**
- [ ] Lectura de estado fallida → denegar, sin excepción ni override.
- [ ] `max_trades_per_day` y `leverage` deben estar acotados por el broker siempre, o
      eliminarse del conjunto de ceilings delegables al modelo.
- [ ] Registrar explícitamente en el log de auditoría que el límite es
      "autodeclarado" cuando lo sea, en lugar de presentarlo como verificado.
- [ ] Test: broker caído + cualquier orden → DENY.

**Criterio de salida de la Fase 1:** los 6 con test de regresión, la suite verde en
Linux y Windows, y un test end-to-end del gate con OKX y Binance en estado fail-closed.

---

## Fase 2 — Integridad del gate y del mandato

### 2.1 · `max_trades_per_day` falla OPEN
- [ ] Cualquier error con el fichero de contador → denegar, no continuar con 0.
- [ ] Correlación con el ledger encadenado por hash (ver 2.5).
- [ ] Test: contador corrupto, ausente, no escribible, y reloj UTC cruzando medianoche.

### 2.2 · Límites no aplicados a órdenes limit
- [ ] `max_order_notional_usd`, `max_total_exposure_usd` y `max_leverage` deben
      aplicaмarse a limit orders usando **worst case** (precio favorable), no el
      precio de mercado actual.
- [ ] Una venta limit no worst-caseada puede ejecutarse a múltiplos del notional
      autorizado.

### 2.3 · Veredictos "advisory" calculados y no aplicados
- [ ] Los límites estrictamente-mayores se computan, se loguean en `checked_limits`
      como si comprobaran, y no se aplican.
- [ ] O se aplican, o se dejan de listar en `checked_limits`. Mentir en el log de
      auditoría es peor que no tener la regla.

### 2.4 · `readonly` muerto dentro de los conectores
- [ ] La enforcement vive solo en `service.py`. Un `module.place_order` programático
      la bypasea entero.
- [ ] Mover el check a una función compartida que ambos caminos excitationes no puedan
      saltar, y testear el bypass programático explícitamente.

### 2.5 · Primer ledger en Windows
- [ ] `test_windows_lock_no_sentinel.py:1-18` es un buen test comportamental que
      stubbea `msvcrt` para correr en todos los SO — pero protege la única plataforma
      que CI nunca ejercita. Mover el job principal a la matriz Windows.
- [ ] Confirmar que el fix del ledger (append atómico con sentinel `msvcrt`) es
      idempotente ante crash entre el sentinel y el write.

### 2.6 · `get_order_by_client_order_id` solo en Alpaca
- [ ] Tras la Fase 1.1 esto se resuelve, pero el test estructural debe permanecer:
      un conector nuevo que añada el método mal rompe la protección de duplicados
      en una cuenta real sin que nada lo note.
- [ ] Corregir el docstring de `sdk_order_gate.py:543`, que sigue diciendo "Resolve one
      Alpaca submission".

### 2.7 · Partial fills
- [ ] Un fill parcial nunca cancela el resto, pero sí consume el daily count.
- [ ] Definir política explícita: cancelar resto, o contabilizar la orden completa
      contra el límite. Hoy hace ambas cosas a medias.

### 2.8 · Bypass estructural de perfil
- [ ] `_OVERRIDE_KEYS` permite `profile`; un caller programático de
      `service.place_order` puede sobreescribir el perfil. La herramienta LLM no
      expone `profile`, y no hay ruta REST directa, pero la superficie pública lo
      permite.
- [ ] Decidir por diseño: o `profile` sale de `_OVERRIDE_KEYS`, o el override no puede
      cambiar la clase de riesgo (paper↔live). Es una decisión de política, no un
      parche mecánico; documentarla.

---

## Fase 3 — Verificación realista de los conectores

**El hole raíz.** El incidente Toss (commits `0b8c6bfc` y `03972104`) demuestra que
este patrón no es hipotético: tests en verde con fixtures fabricadas mientras **cada
llamada real devolvía nada**. Pasó en un conector read-only. El mismo patrón está hoy
sin cerrar en OKX. Nada en CI lo detectaría.

### 3.1 · Grabación de respuestas reales
- [ ] Capturar respuestas **reales** de OKX y Binance (testnet primero), guardadas con
      un comentario de procedencia, tal como hace ahora el fixture de Toss
      (`test_sdk_connectors_toss.py:83-131`, que además mantiene los numéricos como
      string a propósito).
- [ ] Assertion sobre esos fixtures en un test end-to-end `gate → place_order →
      parse`. Eso habría atrapado los tres bugs de Toss el primer día.

### 3.2 · Test end-to-end por conector live
- [ ] Hoy solo existen para Alpaca, MT5 y Trading212. Faltan `okx-live-trade`,
      `binance-live-trade`, `futu-live-trade`, `tiger-live-trade`.
- [ ] Ninguno de esos cuatro ha pasado nunca una respuesta realista por el gate.

### 3.3 · `python-okx` como dependencia
- [ ] Declararla. Hoy CI solo puede ejercitar fakes; el SDK real nunca se importa.

### 3.4 · Verificación dura de modo paper
- [ ] MT5 tiene un guard bidireccional real. OKX **no**: el propio código lo dice.
      Confías en el exchange. Cambiar de key, región o account mode a paper sigue
      operándote.
- [ ] Test unificado: para todo perfil con capacidad de orden, el modo efectivo se
      verifica en runtime contra la respuesta del broker, no contra la config.

### 3.5 · Testnet como smoke test
- [ ] Un test opt-in,标记 `VIBE_TRADING_TESTNET=1`, contra OKX demo y Binance testnet,
      que ejercite idempotencia, rate limit, partial fill y orden rechazada. Es lo
      único que cubre la clase de fallo que mata.

### 3.6 · Binance live spot no funcional
- [ ] `get_positions` lee `market_value`/`price`, que no existen → todo denegado.
      Fail-closed y seguro, pero inservible. Implementar o marcar el perfil como no
      soportado explícitamente.
- [ ] Test que afirme el comportamiento soporte-o-no-soportado, para que ningún
      usuario descubra por un DENY que el perfil no funciona.

### 3.7 · OKX solo spot
- [ ] `tdMode` hardcodeado a `"cash"` → perps y apalancamiento inalcanzables.
      Exponer el modo o declarar el perfil spot-only en la documentación del perfil.

---

## Fase 4 — Honestidad del backtest

El motor no miente sobre el futuro, lo cual ya es raro y bueno. Lo que falla es el
coste. Un Sharpe de este backtest es un techo optimista, no una estimación.

### 4.1 · Comisión US a 0.0
- [ ] `global_equity.py`: valor por defecto distinto de cero, o exigir configuración
      explícita y fallar si no está.

### 4.2 · Funding de cripto fabricado
- [ ] `agent/src/analysis/factor_costs.py:143-171` usa una constante de 1bp/8h
      siempre positiva. Los largos siempre pagan, los cortos siempre reciben. No hay
      régimen.
- [ ] Fuente de funding real con historical, o al menos un régimen que varíe con el
      tiempo y pueda ser negativo. Marcar como_LIMITACIÓN si no hay dato.

### 4.3 · Salidas cripto a maker rate
- [ ] 3bp por pierna, sistemáticamente optimista. Corregir a taker en la salida, o
      justificar cada caso.

### 4.4 · Sin borrow fee en acciones
- [ ] Los shorts son gratis. Toda estrategia long/short está sesgada al alza.
- [ ] Implementar borrow con borrow rate real y hard-to-borrow.

### 4.5 · `factor_costs.py` es código muerto
- [ ] 415 líneas que implementan costes por factor, nunca importadas. O se conectan,
      o se borran. Código muerto que "dice" modelar costes es peor que nada.
- [ ] Test que falle si el módulo sigue sin importarse.

### 4.6 · Sin margin call
- [ ] Un 10x de ES sobrevive a un wipeout. En real, liquidación.
- [ ] Implementar mantenimiento de margen y cierre forzado en futuros y apalancadas.

### 4.7 · Multiplicador 50 silencioso
- [ ] Futuros globales: contrato desconocido → asumes 50. Fallar en silencio es peor
      que fallar ruidoso. Exigir multiplicador explícito.

### 4.8 · Métricas medidas contra la cesta propia
- [ ] `information_ratio`, `tracking_error` y `excess_return` se comparan contra la
      propia estrategia salvo que se fije benchmark. Exigir benchmark explícito o
      etiquetar la métrica como no interpretable.

### 4.9 · Survivorship bias
- [ ] Sin manejar. Los nombres delisted se llevan al último close para siempre.
- [ ] Test con un delisted conocido que demonstrates el sesgo.

### 4.10 · Fills a apertura de barra
- [ ] Sin spread, sin partial fills. El precio de apertura es optimista en un ticker
      líquido y pesimista en uno ilíquido. Modelar spread mínimo y slippage.

### 4.11 · Bug clase `pct_change` en el camino de métricas
- [ ] El CHANGELOG afirma cierre; la auditoría lo encontró vivo en el camino de
      métricas. Verificar y cerrar de verdad, o corregir la afirmación del CHANGELOG.

### 4.12 · Declarar el modelo de slippage
- [ ] Documentar explícitamente qué NO se modela. Un backtest honesto declara sus
      supuestos, incluso los que no simula.

**Criterio de salida:** cada sesgo Documentado, corregido o etiquetado como
limitación conocida en la documentación del backtest. Ninguno silencioso.

---

## Fase 5 — Superficie de ataque

### 5.1 · Separación datos/instrucciones
- [ ] Web, PDFs, noticias y resultados MCP entran al contexto sin separación
      estructural. Nada impide que el contenido de una página llame a
      `trading_place_order`. Lo único entre "el modelo leyó la web de un atacante" y
      "tu dinero" es el gate del mandato.
- [ ] Envoltura de contenido no confiable en un canal de datos, no de
      instrucciones, con presupuesto de tokens. Esto es un cambio de arquitectura, no
      un parche: planificar con cuidado.

### 5.2 · MCP sin escanear
- [ ] Los resultados MCP no pasan por el scanner de inyección. Integrarlos o
      declararlos fuera de alcance explícitamente.

### 5.3 · Credenciales en texto plano
- [ ] El keyring del SO solo está disponible para perfiles read-only → los perfiles
      con capacidad de orden quedan **forzosamente** en texto plano. `chmod 600` no
      protege nada en NTFS.
- [ ] Credential vault del SO disponible para perfiles con capacidad de orden. Es el
      punto más valioso de esta fase.

### 5.4 · `bash` con herencia total de secrets
- [ ] `bash_tool.py:129-140`: `subprocess.Popen(..., shell=True)`, sin allowlist, sin
      entorno saneado. Critical **solo si se activa** `VIBE_TRADING_ENABLE_SHELL_TOOLS=1`;
      actualmente está desactivado y API/MCP no pueden escalarlo por request. Esa
      decisión es correcta y no debe tocarse.
- [ ] Mantener el default off. Añadir allowlist de comandos y allowlist de env si se
      alguna vez habilita. Test que verifique que el default sigue off.

### 5.5 · Sandbox real de código generado (Windows)
- [ ] El código de backtest generado hereda el home real, la config real de mootdx y
      los tokens del proveedor. "Environment vars reducidas" no es un sandbox.
- [ ] Job Object en Windows, `rlimits` en Linux, cwd y home redirigidos a un temp
      limpio, red bloqueada.

### 5.6 · Supply chain
- [ ] 188 pins, sha256-locked, dependabot mensual, algunos pins deliberados. Es
      razonable. Falta: SBOM firmado y verificación de las descargas.

---

## Fase 6 — Fiabilidad del ledger y del estado

### 6.1 · Escrituras atómicas
- [ ] Hash-chained ledger sobre NTFS: validar el sentinel de `msvcrt` ante crash a
      mitad de append, y hacer el append idempotente.

### 6.2 · Reinicio y reanudación
- [ ] Ejecutar el backtest, matar el proceso, reanudar. ¿El estado es consistente?

### 6.3 · El benchmark es Alpa-fácil
- [ ] `DESKTOP_WINDOWS` existe. Windows nunca corrió el suite de ledger/live en CI
      (§2.5).

### 6.4 · Semántica de compartición de ficheros
- [ ] `factor_costs.py` es global. ¿Hay estado global compartido entre runs
      concurrentes? Si sí, es un bug de concurrencia esperando.

---

## Fase 7 — Endurecer la CI y los tests

### 7.1 · Quitar `fail_under = 0`
- [ ] `pyproject.toml` mide cobertura y la tira. Sin gate, cualquier regresión de
      cobertura aterriza en silencio. Subir a un umbral honesto en escalones.

### 7.2 · Job principal en matriz
- [ ] El job principal es Ubuntu-only. Mover live/ledger a la matriz completa.

### 7.3 · Limpiar los `--ignore` muertos
- [ ] `e2e_backtest` y `test_e2e_harness_v2.py` no existen. Un `--ignore` de algo
      inexistente esconde que no hay harness e2e. Y eso hay que resolverlo, no
      borrarlo: crear el harness (§3.5) o documentar que no existe.

### 7.4 · Property-based testing de verdad
- [ ] Los "hypothesis" del repo importan su propio módulo de dominio, no Hypothesis.
      Usar Hypothesis real sobre las funciones de parsing de respuestas de exchange —
      es donde viven los bugs de fixture.

### 7.5 · Property test de no-duplicado
- [ ] Invariante: para ninguna combinación de (timeout, retry, crash) puede existir
      más de una orden viva con el mismo `client_order_id` lógico.

### 7.6 · Snapshots
- [ ] Ya existen para factoring; extender a parsing de respuestas de exchange.

### 7.7 · Mutation testing en el gate
- [ ] El gate es la línea de defensa contra la pérdida de dinero. Mutar
      `check_mandate` y `_post_trade_gross_exposure` y verificar que los tests
      matan los mutantes. Si un mutante sobrevive, hay un test que no prueba nada.

### 7.8 · Network isolation
- [ ] Solo factors bloquean sockets. El resto de la suite puede salir a la red sin
      querer, y un test que depende de la red es un test que falla en CI un martes.

---

## Fase 8 — Documentación y honestidad del producto

### 8.1 · Corregir el CHANGELOG
- [ ] Afirma cierres que la auditoría encontró vivos (§4.11). Un CHANGELOG que miente
      es peor que no tener CHANGELOG: el usuario deja de fiarse del histórico.

### 8.2 · Declarar las limitaciones
- [ ] Sección explícita de "qué NO hace el backtest": funding real, borrow, margin
      call, impacto de mercado, partial fills, survivorship.

### 8.3 · Declarar qué conector soporta qué
- [ ] `okx-live-trade` solo spot y sin verificación dura de paper. `binance-live-trade`
      spot no funcional. Eso va escrito junto al perfil, no en un issue.

### 8.4 · Pauta de capital
- [ ] Documentar el tamaño de cuenta para el que esto es razonable, y por qué.

### 8.5 · El aviso de que paper no es garantía
- [ ] La conclusión central de esta auditoría: los bugs críticos son de idempotencia,
      propagación de errores y contabilidad de exposición, y **ninguno se manifiesta en
      paper**. Documentarlo donde el usuario lo leerá antes de encender el dinero.

---

## Orden de ejecución

| Fase | Contenido | Depende de | Riesgo que elimina |
|---|---|---|---|
| 0 | Baseline, cobertura, `python-okx` | — | (medición) |
| 1 | **6 defectos de dinero** | 0 | Pérdida directa de capital |
| 2 | Gate + mandato | 1 | Bypass del circuit breaker |
| 3 | Verificación realista | 1 | Detección de bugs de fixture |
| 4 | Honestidad del backtest | 0 | Decisiones sobre números falsos |
| 5 | Seguridad | 1 | Exfiltración, RCE, prompt injection |
| 6 | Ledger / estado | 0 | Corrupción de estado |
| 7 | CI y tests | todas | Regresión silenciosa |
| 8 | Documentación | todas | Decisiones sobreDocumentation falsa |

**Fases 4 y 5 no dependen de la 1** y pueden ir en paralelo. La 7 y la 8 cierran.

---

## Definición de "arreglado"

Vibe-Trading está listo para operar con capital real cuando **todas** estas son
verdades y hay un test que lo demuestra:

- [ ] Ninguna ruta de fallo de lectura puede producir un estado que el gate interprete
      como "cero riesgo".
- [ ] Ningún reintento puede producir más de una orden viva.
- [ ] Ninguna herramienta puede redirigir una petición firmada fuera del exchange.
- [ ] Ninguna ruta programática bypasea el gate.
- [ ] El backtest declara todo lo que no simula.
- [ ] Existe un test que corre contra un testnet real y ejercita la clase de fallo que
      mata.
- [ ] La CI corre el suite de dinero en Windows.

Sin el último punto, el resto es idioma.

---

## Estado

- [ ] Fase 0 — Baseline
- [ ] Fase 1 — Los 6 de dinero
- [ ] Fase 2 — Gate y mandato
- [ ] Fase 3 — Verificación de conectores
- [ ] Fase 4 — Backtest
- [ ] Fase 5 — Seguridad
- [ ] Fase 6 — Ledger
- [ ] Fase 7 — CI
- [ ] Fase 8 — Documentación
