# Tamarin Formal Verification - Personal Lab

This repository contains my personal notes, models, and exercises for formal verification using the Tamarin Prover.

This is a personal/academic project.

## Repository Goals

- Keep structured notes while studying Tamarin.
- Build protocol models incrementally.
- Collect solved exercises and proof/attack reports.
- Track progress over time in a clean, reproducible way.

## Suggested Structure

```text
.
├── README.md
├── .gitignore
├── TamarinRecap.md
├── notes/
│   ├── concepts/
│   └── lecture-notes/
├── models/
│   ├── protocols/
│   └── sandbox/
├── exercises/
│   ├── basics/
│   ├── intermediate/
│   └── advanced/
├── reports/
│   ├── proofs/
│   └── attacks/
├── resources/
└── templates/
```

## Quick Start

1. Create a new model from `templates/protocol-template.spthy`.
2. Run:

```bash
tamarin-prover <file>.spthy --prove
```

3. If needed, use GUI mode:

```bash
tamarin-prover interactive <file>.spthy
```

## Naming Conventions

- Models: `models/protocols/<protocol-name>.spthy`
- Exercises: `exercises/<level>/exXX_<topic>.spthy`
- Reports: `reports/proofs/<model-name>.md`, `reports/attacks/<model-name>.md`

## Notes

- `TamarinRecap.md` currently contains the main summary notes.
- Additional notes can be split into `notes/concepts` and `notes/lecture-notes`.
