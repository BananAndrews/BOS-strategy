# BOS-strategy — context for coding agents

When working in D:\Dev, consult ../workspace-context/PROJECTS.md for workspace routing and ../workspace-context/WORKSPACE_CONTEXT.md for shared context if present. These workspace files are local context, not a dependency of the project.

- Purpose: TradingView Pine Script v6 indicator for break-of-structure buy/sell signals. This is distinct from the RSI retest and trend-confirmation indicators in sibling folders.
- Read `README.md`, `BOS-signals.pine`, and `docs/narration-transcript-ru.txt` for the recorded specification.
- A directional candle closes beyond the previous opposite-colour candle's extreme. Candle 1 arms a setup; candle 2 confirms a BOS break; entry is a retest of the BOS. Preserve long/short symmetry and concurrent setup tracking.
- Time deltas are bars; price delta supports percent/ATR/ticks/price. Candle-2 body clearance remains a percentage. A wick beyond candle 1 invalidates that setup.
- Validation is compilation and chart inspection in TradingView Pine Editor; use the existing rejection-debug labels to explain missing signals. No automated test suite is present.
- The README's calibration and synthetic-bar counts are historical experiments, not verified trading performance. Delta 3 remains a per-symbol calibration question.

Onboarding baseline, 2026-10-30: working tree was clean; latest local commit `0c29c95`. Recheck Git and relevant source before resuming.
