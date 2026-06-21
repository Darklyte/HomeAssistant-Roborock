# Claude Code Session Notes

## Active Investigation: Split-Level Temperature Imbalance

### Context
User lives in a split-level house. Even when outdoor temps are in the 50–60°F range, the upstairs east-facing primary bedroom stays in the mid-70s at night and doesn't cool down without running AC continuously. A west-side bedroom cools off much more easily.

**Room details (from photo):**
- East-facing bedroom, upper level
- Large 3-panel window bank on east wall + smaller awning window on green accent wall
- Blackout blinds (user suspects these may restrict airflow)
- Hardwood floors, textured ceiling

### Hypotheses (ranked by likelihood)
1. **Morning solar gain / thermal mass** — East windows get direct sun ~6am–noon, heat soaks into walls/floor/ceiling and radiates back out all evening
2. **Roof/attic radiation** — Upper level has more ceiling exposure to attic heat absorbed during the day
3. **Stack effect** — Lower-level heat rises and pools upstairs; west rooms may have better cross-ventilation pathways
4. **Blackout blind heat trap** — Blinds trap hot air between glass and blind; opening at night releases stored heat; keeping them closed during the day actually helps if windows are also opened at night to flush

### Next Steps (need HA sensor data)
User has Home Assistant with temperature sensors. Need to pull up HA from desktop or via dispatch (not accessible in this web session).

**Key graphs to look at:**
- Upstairs vs. downstairs temperature differential over 24h cycle
- Time of peak upstairs temp vs. time outdoor temp drops (lag = thermal mass estimate)
- AC cycling behavior overnight — is it running constantly or just not keeping up?
- Ideally: east bedroom vs. west bedroom temp comparison over same period

### Session Note
User needs to restart this conversation from Claude Code desktop app or GitHub dispatch to get Home Assistant MCP/sensor access. Resume by referencing this file.
