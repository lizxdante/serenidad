#Workflow Orchestation

### 1. Plan Node Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan inmediately - don't keep pushing
- Use plan mmode for veriication steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 2. Subagent Strategy
- Use subagents liberally to keep main context window clean
- Offload research, exploration, and paralel analysis to subagents
- For complex problems, throw more compute at it via subagents.
- One tack per subagent for focused execution

### 3. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` the pattern.
### 4. Verification Before Done
### 5. Demand Elegance (Balanced)
### 6. Autonomus Bug Fixing

## Task Management

1. **Plan First**: Write plan to `tasks/lessons.md` with checkable items.
2. **Verify Plan**: Check in before starting implementation.
3. **Track Progress**: Mark items complete as you go.
4. **Explain Changes** High-level summary at each step.
5. **Document Results**: Add review section to `tasks/todo.md`
6. **Capture Lessons**: Update `tasks/lessons.md` after corrections.

## Core Principles

- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Senior Developer standards.
- **Minimal Impact**: Changes should only touch what's neccesary. Avoid introducing bugs.
