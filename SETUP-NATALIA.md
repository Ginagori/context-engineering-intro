# Instrucciones de Configuracion - Agent Teams

Hola Natalia, estas son las instrucciones para habilitar Agent Teams en tu maquina.

## Que es Agent Teams

Es una feature experimental de Claude Code que permite crear equipos de multiples agentes
que trabajan en paralelo, se comunican entre si, y comparten una lista de tareas.
Muy util para implementar features full-stack (frontend + backend + database) en paralelo.

Para mas contexto, revisa estos archivos en el repo:
- `REPORT-agent-teams-analysis.md` - Analisis completo con matrices de decision
- `use-cases/build-with-agent-team/TRANSCRIPT-agent-teams-video.md` - Transcript del video explicativo

## Paso 1: Clonar el Fork (si no lo tienes)

```bash
git clone https://github.com/Ginagori/context-engineering-intro.git
```

## Paso 2: Habilitar Agent Teams

Abre (o crea) tu archivo `~/.claude/settings.json` y agrega la seccion `env`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Si ya tienes contenido en ese archivo (como MCPs configurados), solo agrega la seccion
`env` al inicio, antes de `mcpServers`. Ejemplo:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "mcpServers": {
    ...tus MCPs existentes...
  }
}
```

## Paso 3: Instalar el Skill (opcional pero recomendado)

Este skill da instrucciones a Claude para crear agent teams de forma mas confiable
usando el patron "contract-first" (spawn secuencial por dependencias).

Crea la carpeta de skills si no existe y copia el archivo:

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path "$HOME\.claude\skills\build-with-agent-team" -Force
Copy-Item "use-cases\build-with-agent-team\SKILL.md" "$HOME\.claude\skills\build-with-agent-team\SKILL.md"
```

**Mac/Linux:**
```bash
mkdir -p ~/.claude/skills/build-with-agent-team
cp use-cases/build-with-agent-team/SKILL.md ~/.claude/skills/build-with-agent-team/
```

## Paso 4: Verificar

Abre Claude Code en cualquier proyecto y escribe algo como:

```
Crea un agent team de 2 para revisar este proyecto.
Uno enfocado en seguridad y otro en calidad de codigo.
```

Si Claude crea el team y ves los teammates, todo funciona.

## Como Usar

### Modo basico (sin skill)
Simplemente dile a Claude que cree un agent team:
```
"Crea un agent team de 3 para implementar [feature].
 Un agente para backend, otro para frontend, otro para base de datos."
```

### Modo avanzado (con skill)
Primero crea un plan en markdown, luego:
```
/build-with-agent-team ./mi-plan.md 3
```

### Cuando usar Agent Teams vs Sub Agents

| Tarea | Que usar |
|-------|----------|
| Investigar un tema | Sub Agents o Playbook |
| Fix rapido | Single session |
| Feature full-stack | **Agent Teams** |
| Code review exhaustivo | **Agent Teams** |
| Research de librerias | Sub Agents + Context7 |

## Obtener Actualizaciones de Cole

El repo original de Cole esta configurado como `upstream`. Para traer sus actualizaciones:

```bash
git fetch upstream
git merge upstream/main
git push origin main
```

## Notas Importantes

- Agent Teams es **experimental** - puede tener bugs
- Usa mas tokens que una sesion normal (~2-4x)
- Funciona en VS Code en modo in-process (sin split-panes visuales)
- Usa `Shift+Up/Down` para navegar entre teammates en VS Code
- `Ctrl+T` para ver la task list compartida
- Los teammates heredan todos tus MCPs (Context7, Playwright, etc.)
