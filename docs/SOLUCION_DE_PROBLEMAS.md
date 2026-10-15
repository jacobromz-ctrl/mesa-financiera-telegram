# Solución de problemas · Mesa Financiera en Telegram

Rutas de resolución que ya funcionaron. Léelas **antes** de diagnosticar un error nuevo.
Cada caso: síntoma → causa → solución → cómo comprobarlo.

---

## 0. Antes de empezar: información que debes tener lista

| Dato | Dónde se obtiene | Dónde se guarda |
|---|---|---|
| Acceso de Claude a GitHub | https://claude.ai/connect-github → instalar la Claude GitHub App con **All repositories** | Se comprueba en https://github.com/settings/installations |
| `TELEGRAM_BOT_TOKEN` | @BotFather en Telegram | Secret de GitHub / `.env` |
| `TELEGRAM_CHAT_ID` | `python obtener_chat_id.py` (antes escribe "hola" al bot) | Secret de GitHub / `.env` |
| `ANTHROPIC_API_KEY` | Claude Console (o la docente) | Secret de GitHub / `.env` |
| `ANTHROPIC_WORKSPACE_ID` | Claude Console → Settings → Workspaces (empieza con `wrkspc_`). **Solo si la llave no pertenece a un workspace** (ver caso 3) | Secret de GitHub / `.env` |
| `ALPHAVANTAGE_API_KEY` | https://www.alphavantage.co/support/#api-key | Secret de GitHub / `.env` |

Orden recomendado: instalar → `pytest -q` → `python analizar.py AAPL --simulacion` →
`python verificar_configuracion.py` → modo REAL solo cuando la persona lo pida.

---

## 1. Claude no puede abrir el repositorio

- **Síntoma:** "repository not found… or this session's GitHub credential doesn't have access".
- **Causa:** el repositorio es privado y Claude no está autorizado. Tener la sesión abierta en GitHub
  desde el navegador **no** basta; tampoco basta con dar permiso por chat.
- **Solución:** https://claude.ai/connect-github → *Authorize* → instalar la **Claude GitHub App** en la
  cuenta con **All repositories** (así los repos nuevos quedan incluidos).
- **Comprobar:** Claude aparece en https://github.com/settings/installations.

## 2. Aviso "Branch … has unpushed commit(s) and no remote branch" aunque ya se subió

- **Causa:** el repo se clonó de forma reducida (`--depth 1`, solo `main`) y git no registra las ramas remotas.
- **Solución:**
  ```bash
  git config --add remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
  git fetch origin
  git branch -u origin/<rama>
  ```
- **Comprobar:** `git status -sb` muestra `<rama>...origin/<rama>` sin "ahead".

## 3. Error 400 "This API key is not scoped to a workspace"

- **Síntoma:** en Actions, Euler falla con
  `anthropic.BadRequestError: Error code: 400 … must include the anthropic-workspace-id header`.
  Antes, Iris muestra `DATOS DEMO … (MCP no disponible: BadRequestError)`: **no es culpa del MCP**,
  es la misma llave.
- **Causa:** la llave pertenece a la organización, no a un workspace. Cambiar por otra llave creada igual **no** lo arregla.
- **Solución (ya implementada en el código):** crear el Secret `ANTHROPIC_WORKSPACE_ID` con el ID `wrkspc_…`.
  `mesa/agentes.py` lo envía como encabezado `anthropic-workspace-id` si existe.
  Alternativa sin Secret extra: usar una llave creada **dentro** de un workspace.
- **Comprobar:** la ficha de Iris dice `"fuente": "Alpha Vantage vía MCP"`.
- **Pendiente:** el notebook de Colab (`notebook/`) aún no envía este encabezado.

## 3b. La nota dice "NO VERIFICADO, revisa las cifras (…)"

- **Causa:** el guardrail de Iris (`revisar_ficha` en `mesa/agentes.py`) detectó que Claude respondió sin
  consultar Alpha Vantage, o que el precio es incoherente (≤ 0 o fuera de su rango de 52 semanas).
- **Qué hacer:** no usar esas cifras. Reintentar más tarde; si se repite, revisar la cuota de Alpha Vantage
  y `python verificar_configuracion.py --mcp` (gasta 1 consulta).

## 4. "Re-run" repite el mismo error después de corregir algo

- **Causa:** *Re-run* vuelve a ejecutar el **mismo commit** de la ejecución original. Si el arreglo está en
  un commit nuevo, no lo incluye.
- **Solución:** usar **Actions → mesa-financiera → Run workflow**, elegir la rama correcta en
  *Use workflow from* y el ticker. Marcar *Modo simulación* para probar sin gastar.

## 5. Cambié el cron y no se nota

- **Causa:** GitHub solo ejecuta `schedule` desde la **rama principal** (`main`).
- **Solución:** unir la rama a `main` (Pull request → Merge).
- **Nota de costo:** en repos **privados** GitHub regala 2,000 min/mes y cobra mínimo 1 min por ejecución.
  `*/10` ≈ 4,320 ejecuciones/mes (excede); `*/30` ≈ 1,440 (cabe). El cron está en hora **UTC**
  (Ciudad de México = UTC−6).

## 6. El workflow termina en rojo cuando falla la llave de Claude

- **Causa:** solo Iris tiene "plan B" (cambia a datos demo). Euler y Vera no capturan el error y el
  proceso sale con `exit code 1`.
- **Mejora pendiente:** agregar el mismo plan B a Euler y Vera y avisar por Telegram con un mensaje claro.

## 7. Claude no puede leer los registros de GitHub Actions

- **Causa:** desde la nube de Claude, la descarga de logs de Actions está bloqueada (Forbidden).
- **Solución:** pedir a la persona que abra el paso que falló y copie **la última línea** del error
  (empieza con `anthropic.` o `Error:`). Esa línea no contiene llaves.

## 8. Personalizar un agente sin romper nada

- La personalidad vive en el prompt (`PROMPT_RECOLECTOR`, `PROMPT_ANALISTA`, `PROMPT_COMUNICADOR`) y,
  para el modo SIMULADO, en `ficha_demo`, `lectura_simulada` y `nota_simulada` de `mesa/agentes.py`.
- Las pruebas exigen que la nota de Vera **empiece** con `FIRMA_VERA`, **incluya** el ticker y
  **termine** con el descargo. No mover cálculos al prompt (regla de oro).
- Después de editar: `pytest -q` (20 passed) y `python analizar.py AAPL --simulacion`.

---

## Reglas que no cambian
- Nunca imprimir ni pegar el valor de una llave; usar `python verificar_configuracion.py`.
- Empezar en SIMULADO; pasar a REAL solo cuando la persona lo pida.
- Explicar la finalidad de cada edición antes de hacerla.
- Pendientes pedidos por la persona: **aprobación humana** antes del modo real y **registro de tokens** por corrida
  (aún no existen en el código).
