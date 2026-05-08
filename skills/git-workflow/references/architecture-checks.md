# Architecture Constraint Checks

Every commit touches code that lives within a layered architecture. These checks catch violations early — before they become entrenched and expensive to fix.

Run these checks mentally (or via grep) before each commit. If a violation is found, warn the user but don't block the commit — let them decide.

## Constraints

| Constraint | What to check | Why it matters |
|------------|---------------|----------------|
| core independence | `core/` has no imports from external AI frameworks/SDKs | core/ defines stable data structures and events; coupling it to a vendor SDK means every SDK upgrade ripples through the entire system |
| No reverse dependencies | `runtime/` only imports from `ports/`, never from domain directories | runtime/ orchestrates domains through their interfaces; importing implementations directly defeats the plug-and-play architecture |
| Domain isolation | Domain directories (`voice/`, `agent/`, `rag/`, etc.) never import each other | Domains communicate through ports/; direct cross-domain imports create hidden coupling that makes it impossible to swap implementations |
| Registry registration | New implementations are registered in their domain's `registry.py` | The factory system discovers implementations via registry; an unregistered implementation is invisible to the runtime |

## How to check

```bash
# core independence — look for AI SDK imports in core/
grep -r "import openai\|import anthropic\|import langchain\|from openai\|from anthropic\|from langchain" src/core/

# No reverse dependencies — runtime importing domain dirs
grep -r "from src\.\(voice\|agent\|rag\|memory\|tools\|trace\|context\)" src/runtime/

# Domain isolation — domains importing each other
# (check each domain dir for imports from sibling domains)
grep -r "from src\.voice\|from src\.agent\|from src\.rag\|from src\.memory\|from src\.tools\|from src\.trace\|from src\.context" src/agent/
# repeat for other domains...

# Registry — new class not in registry.py
# manual: if you added a new implementation class, check its domain's registry.py
```

## When a violation is found

Present it clearly:

```
⚠️ Architecture note: `src/runtime/pipeline.py` imports `src/voice/whisper.py` directly.
   runtime/ should only import from ports/ — this keeps implementations swappable.
   
   Want to proceed anyway, or should I refactor to use the port interface?
```
