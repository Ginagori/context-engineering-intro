# Reporte: Agent Teams, Sub Agents y MCPs - Estrategia para Nivanta

**Fecha:** 2026-02-12
**Basado en:** Video de Cole Medin, docs oficiales de Anthropic, inventario de MCPs Nivanta

---

## 1. Resumen Ejecutivo

Claude Code Agent Teams es una feature **experimental** que permite coordinar multiples instancias de Claude Code con task list compartida y comunicacion peer-to-peer. Cole Medin propone un skill (`/build-with-agent-team`) que mejora la fiabilidad mediante **contract-first spawning** (spawn secuencial por dependencias).

**Hallazgo principal:** La combinacion optima para Nivanta es un flujo de 3 fases:
1. **Playbook** para research/planning (no consume tokens de contexto principal)
2. **Sub Agents** para tareas focalizadas de investigacion
3. **Agent Teams** para implementacion coordinada multi-capa

---

## 2. Analisis de la Propuesta de Cole Medin

### 2.1 Que Propone

| Concepto | Descripcion |
|----------|-------------|
| **Contract-First Spawning** | No spawn paralelo total. Primero DB, luego Backend (con contrato de DB), luego Frontend (con contrato de API) |
| **Lead como Relay** | El lider NO dice "hablen entre ustedes". Recibe contratos, los verifica, y los reenvía a downstream |
| **Cross-Cutting Concerns** | Asignar explicitamente responsabilidad de temas transversales (URLs, response shapes, SSE format) |
| **Validacion por Capas** | Cada agente valida su dominio + el lead hace E2E al final |
| **Skill Custom** | `/build-with-agent-team [plan-path] [num-agents]` automatiza todo el proceso |

### 2.2 Problemas que Identifica (y que Confirma la Doc Oficial)

| Problema | Severidad | Solucion Propuesta |
|----------|-----------|-------------------|
| Agentes divergen en interfaces cuando corren en paralelo | Alta | Contract-first spawning |
| Claude no siempre crea buenos teams sin instrucciones | Media | Skill con instrucciones detalladas |
| No hay visibilidad clara de la colaboracion | Media | Logs + preguntar al lead por status |
| 2-4x mas tokens que sub agents | Alta | Usar selectivamente (solo implementacion) |
| Requiere WSL en Windows | Media | Ya tenemos WSL configurado |
| No soporta split-pane en VS Code terminal | Baja | Usar tmux desde WSL directamente |

### 2.3 Lo que NO Cubre (y que Nosotros SI Tenemos)

Cole no menciona integracion con MCPs como Playbook, Context7, Obsidian, o n8n. Aqui es donde Nivanta puede ir mas alla.

---

## 3. Inventario de MCPs y Capacidades Relevantes

### 3.1 Playbook (Agente Autonomo)

| Herramienta | Fase Ideal | Uso en el Flujo |
|-------------|-----------|-----------------|
| `playbook_research(topic)` | Pre-planning | Investigar antes de crear el plan |
| `playbook_run_pipeline(feature)` | Planning completo | Research -> Plan -> Code -> Review -> Test |
| `playbook_supervised_task(task)` | Tareas complejas | Supervisor coordina agentes internos |
| `playbook_plan_feature(feature)` | Planning | Genera plan detallado que alimenta al Agent Team |
| `playbook_code_review(code)` | Post-implementacion | Review de calidad + seguridad en paralelo |
| `playbook_generate_tests(code)` | Post-implementacion | Genera tests para codigo producido |

**Rol en Agent Teams:** Playbook NO deberia ser un teammate (no puede ejecutar como instancia de Claude Code). En cambio, debe usarse **antes y despues** del Agent Team:
- **Antes:** `playbook_research` + `playbook_plan_feature` genera el plan
- **Despues:** `playbook_code_review` + `playbook_generate_tests` valida el output

### 3.2 Context7 (Library Docs)

| Herramienta | Uso |
|-------------|-----|
| `resolve-library-id(name)` | Encontrar ID de libreria |
| `get-library-docs(id, topic)` | Documentacion actualizada |

**Rol en Agent Teams:** Cada teammate carga los MCPs del proyecto automaticamente (confirmado en docs oficiales). Entonces cada agente del team puede consultar Context7 durante su implementacion para tener docs actualizadas de las librerias que use.

**Accion requerida:** Asegurar que Context7 este configurado en `.claude/settings.json` del proyecto (no solo global) para que los teammates lo hereden.

### 3.3 n8n (Automatizacion)

| Herramienta | Uso |
|-------------|-----|
| `create_workflow` | Crear workflows |
| `execute_workflow` | Ejecutar workflows |
| `list_workflows` / `get_workflow` | Consultar existentes |

**Rol en Agent Teams:** Util para proyectos que incluyen automatizacion. Un teammate podria especializarse en crear/configurar workflows n8n como parte de la implementacion.

### 3.4 Obsidian (Documentacion)

| Herramienta | Uso |
|-------------|-----|
| `obsidian_get_file_contents` | Leer docs del vault |
| `obsidian_append_content` | Agregar documentacion |
| `obsidian_simple_search` | Buscar en el vault |

**Rol en Agent Teams:** Phase de research - buscar documentacion previa del proyecto en Obsidian antes de planificar. Tambien para que el lead documente decisiones post-build.

### 3.5 Playwright/Browser (Docker)

| Herramienta | Uso |
|-------------|-----|
| `browser_navigate/click/fill` | Navegacion y testing |
| `browser_snapshot/screenshot` | Captura de estado |

**Rol en Agent Teams:** Validacion E2E por el lead agent despues de que todos los teammates terminen. Reemplaza al "agent-browser" que usa Cole en su ejemplo.

---

## 4. Matriz de Decision: Cuando Usar Que

### 4.1 Regla General

```
RESEARCH/PLANNING          IMPLEMENTACION              VALIDACION
===================        ================            ===========
Playbook Pipeline    --->  Agent Teams (2-5)    --->   Playbook Review
  o Sub Agents             con Contract-First          + Lead E2E
  o Obsidian search                                    + Playwright
```

### 4.2 Matriz Detallada por Escenario

| Escenario | Sub Agents | Agent Teams | Playbook | Razon |
|-----------|:----------:|:-----------:|:--------:|-------|
| Investigar codebase existente | **SI** | No | Opcional | Solo necesitas el resumen, no coordinacion |
| Research de librerias/APIs | **SI** | No | **SI** (research) | Context isolation es suficiente |
| Bug fix en 1-2 archivos | No | No | No | Single session es suficiente |
| Feature nueva full-stack (FE+BE+DB) | No | **SI** (3 agents) | **SI** (plan) | Necesita coordinacion de contratos |
| Refactor de modulo existente | **SI** | No | No | Generalmente secuencial, mismos archivos |
| Code review exhaustivo | No | **SI** (3 agents) | **SI** (review) | Seguridad + Calidad + Docs en paralelo |
| Proyecto nuevo desde cero | No | **SI** (3-4 agents) | **SI** (pipeline) | Maximo beneficio de paralelismo |
| Crear workflow n8n + integracion | No | **SI** (2 agents) | No | Backend agent + n8n agent coordinan |
| Documentacion tecnica | **SI** | No | No | Solo necesitas output, no coordinacion |
| Debugging con hipotesis multiples | No | **SI** (3-5 agents) | No | Cada agente prueba una teoria diferente |

### 4.3 Estimacion de Costo en Tokens

| Modo | Tokens Relativos | Cuando Vale la Pena |
|------|:----------------:|---------------------|
| Single Session | 1x | Tareas simples, fixes, refactors pequenos |
| Sub Agents (3) | ~1.3x | Research, exploration, tareas focalizadas |
| Agent Teams (3) | ~2-4x | Implementacion multi-capa, code review paralelo |
| Playbook Pipeline | ~1.5x (externo) | Planning completo antes de implementar |

---

## 5. Flujo Recomendado para Proyectos Nivanta

### 5.1 Proyecto Nuevo (Greenfield)

```
FASE 1: Discovery & Planning (Playbook + Sub Agents)
======================================================
1. playbook_start_project("descripcion del proyecto", mode="supervised")
   -> Discovery interactivo
2. playbook_research("tech stack options for [proyecto]")
   -> Investigacion de tecnologias
3. Context7: get-library-docs para cada libreria clave
   -> Docs actualizadas
4. playbook_plan_feature("complete project architecture")
   -> Plan detallado con contratos

OUTPUT: Plan markdown (como session-manager-plan.md de Cole)

FASE 2: Implementacion (Agent Teams)
======================================
5. /build-with-agent-team ./plan.md [num-agents]
   -> Contract-first spawning:
      a. DB Agent -> publica schema -> lead verifica
      b. Backend Agent (con contrato DB) -> publica API -> lead verifica
      c. Frontend Agent (con contrato API) -> implementa
   -> Cada teammate usa Context7 para consultar docs
   -> Cada teammate lee CLAUDE.md para seguir convenciones

FASE 3: Validacion (Playbook + Playwright)
============================================
6. playbook_code_review(codigo_generado)
   -> Quality + Security review en paralelo
7. playbook_generate_tests(codigo_generado)
   -> Tests automaticos
8. Lead agent: E2E con Playwright (browser MCP)
   -> Validacion de flujo completo

FASE 4: Documentacion
=======================
9. Obsidian: documentar decisiones arquitectonicas
10. Actualizar README, TASK.md
```

### 5.2 Feature Nueva en Proyecto Existente (Brownfield)

```
FASE 1: Analisis (Sub Agents)
===============================
1. Sub Agent: "Analiza la estructura actual del proyecto en [dirs]"
   -> Mapeo de codebase existente
2. Sub Agent: "Identifica puntos de integracion para [feature]"
   -> Dependencias y riesgos
3. Obsidian: buscar documentacion previa del proyecto

FASE 2: Planning (Playbook)
============================
4. playbook_plan_feature("[feature] integrando con [proyecto existente]")
   -> Plan que respeta la arquitectura existente

DECISION POINT: Evaluar el plan
- Si afecta 1-2 archivos -> Single session, no teams
- Si afecta 3+ capas (FE/BE/DB) -> Agent Teams
- Si es solo backend + tests -> Sub Agents

FASE 3: Implementacion (segun decision)
=========================================
Si Agent Teams:
  5. /build-with-agent-team ./plan.md
     -> Con ownership claro: "NO tocar archivos existentes X, Y, Z"

Si Single Session:
  5. Implementar directamente con Context7 para docs

FASE 4: Validacion
====================
6. playbook_code_review + playbook_generate_tests
7. pytest con --timeout=60 --tb=short -q
```

### 5.3 Code Review Exhaustivo (Proyecto en Ejecucion)

```
1. Agent Team (3 teammates):
   - Security Reviewer: OWASP top 10, SQL injection, XSS
   - Quality Reviewer: patterns, naming, complexity
   - Docs Reviewer: docstrings, README, API docs

2. Cada reviewer deposita findings en shared task list
3. Lead sintetiza y genera reporte
4. Opcional: playbook_code_review para segunda opinion
```

---

## 6. Configuracion Necesaria

### 6.1 Para Habilitar Agent Teams

**En `~/.claude/settings.json` (global):**
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "tmux"
}
```

**En WSL (ya instalado):**
```bash
sudo apt update && sudo apt install tmux
```

### 6.2 Para que Teammates Hereden MCPs

Asegurar que los MCPs esten en la config del **proyecto** (`.claude/settings.json`), no solo global. Los teammates cargan:
- CLAUDE.md del proyecto
- MCP servers configurados
- Skills del proyecto

**Esto significa que cada teammate tendra acceso a:**
- Context7 (docs de librerias)
- Playwright (browser testing)
- n8n (si el proyecto lo requiere)
- Obsidian (documentacion)

### 6.3 Instalar el Skill de Cole

```bash
# Copiar a skills global
cp -r use-cases/build-with-agent-team ~/.claude/skills/

# O a nivel proyecto
cp -r use-cases/build-with-agent-team .claude/skills/
```

---

## 7. Limitaciones Criticas a Tener en Cuenta

| Limitacion | Impacto | Mitigacion |
|------------|---------|------------|
| **No resume de teammates** | Si la sesion se pierde, los teammates no se restauran | Guardar progreso frecuentemente en archivos |
| **Task status puede desincronizar** | Teammates a veces no marcan tasks como completadas | El lead debe monitorear y nudge |
| **No nested teams** | Teammates no pueden crear sus propios teams | Disenar equipos flat |
| **1 team por sesion** | No puedes tener 2 teams simultaneos | Limpiar team antes de crear otro |
| **Split-pane NO funciona en VS Code terminal** | No hay visualizacion en panes desde VS Code | Usar tmux desde WSL directamente |
| **Windows requiere WSL** | No funciona en PowerShell/CMD nativo | Ya tenemos WSL |
| **2-4x tokens** | Costo significativo | Reservar para implementacion real, no para tareas simples |

---

## 8. Integracion Playbook + Agent Teams: El Flujo Hibrido

La clave es que Playbook y Agent Teams operan en **capas diferentes**:

```
+--------------------------------------------------+
|  CAPA ESTRATEGICA (Playbook)                     |
|  - Research del dominio                          |
|  - Planning de features                          |
|  - Code review post-implementacion               |
|  - Generacion de tests                           |
+--------------------------------------------------+
         |  Plan markdown  |
         v                 v
+--------------------------------------------------+
|  CAPA TACTICA (Agent Teams)                      |
|  - Lead coordina team                            |
|  - Contract-first spawning                       |
|  - Teammates implementan en paralelo             |
|  - Cada teammate usa Context7, Playwright, etc.  |
+--------------------------------------------------+
         |  Codigo         |
         v                 v
+--------------------------------------------------+
|  CAPA VALIDACION (Playbook + Lead)               |
|  - playbook_code_review                          |
|  - playbook_generate_tests                       |
|  - Lead: E2E con Playwright                      |
|  - Obsidian: documentar decisiones               |
+--------------------------------------------------+
```

**Playbook NO es un teammate** porque:
1. Es un MCP server, no una instancia de Claude Code
2. Sus agentes internos (researcher, planner, coder, reviewer, tester) ya coordinan entre si
3. Su valor es **antes** (planning) y **despues** (review) del Agent Team

**Los teammates SI usan MCPs** porque heredan la configuracion del proyecto:
- Un backend teammate puede consultar Context7 para docs de FastAPI
- Un frontend teammate puede consultar Context7 para docs de React
- El lead puede usar Playwright para E2E

---

## 9. Recomendaciones Finales

### Hacer Ahora
1. **Instalar tmux en WSL** (si no esta)
2. **Habilitar Agent Teams** en settings.json global
3. **Copiar el skill** de Cole a `~/.claude/skills/`
4. **Asegurar MCPs en config de proyecto** (no solo global)

### Adopcion Gradual
1. **Semana 1:** Probar Agent Teams con code review (bajo riesgo)
2. **Semana 2:** Probar con feature nueva simple (2 agents: FE+BE)
3. **Semana 3:** Flujo completo: Playbook plan -> Agent Team build -> Playbook review

### No Hacer
- NO usar Agent Teams para tareas que un solo agente resuelve bien
- NO usar Agent Teams para research (sub agents son mas eficientes)
- NO dejar teams corriendo sin monitoreo (consumen tokens)
- NO confiar en que los teammates se comuniquen solos (usar contract-first)

---

## 10. Tabla Resumen: Que Usar y Cuando

| Necesito... | Uso | Tokens | MCPs Involucrados |
|-------------|-----|:------:|-------------------|
| Investigar un tema | `playbook_research` | Bajo | Playbook |
| Planificar feature | `playbook_plan_feature` | Bajo | Playbook |
| Buscar docs de libreria | Sub Agent + Context7 | Bajo | Context7 |
| Implementar feature full-stack | Agent Teams (3) | Alto | Context7, Playwright |
| Fix rapido | Single session | Minimo | Ninguno extra |
| Code review exhaustivo | Agent Teams (3) | Medio-Alto | Playbook (post) |
| Crear workflow n8n | Agent Teams (2) o Single | Medio | n8n |
| Documentar decisiones | Single + Obsidian | Bajo | Obsidian |
| Proyecto nuevo completo | Playbook -> Teams -> Playbook | Alto | Todos |
