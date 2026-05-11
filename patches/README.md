# Patches against the upstream reference repos

`ev-open-can-tools/` is cloned fresh from
`github.com/ev-open-can-tools/ev-open-can-tools.git` per the AGENTS.md
pre-flight and is gitignored from this guide repo. We do not own a fork
and cannot push branches back. To keep work-in-progress survivable
across sessions and machines, each substantial change to that clone is
captured as a `git apply`-compatible patch and committed here.

## Index

| File | Targets | Baseline | Notes |
|---|---|---|---|
| `2026-05-11-feather-hypery11-port.patch` | `ev-open-can-tools` | `3b0bd91` | Section 1 (CAN mode API) + Sections 2 + 3 (FeatherHypery11Handler + tests + Feather build env). See `FEATHER_HYPERY11_PORT_PLAN.md` "Resume Instructions". |

## Applying

```bash
cd <upstream-clone>
git apply --check ../patches/<file>.patch && \
git apply ../patches/<file>.patch
```

If `--check` fails the upstream has advanced past the recorded baseline.
Treat the patch as a reference and re-port by hand.
