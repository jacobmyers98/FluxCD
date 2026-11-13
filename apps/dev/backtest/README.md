# backtester (copy)

Copied from the `backtester` repo's `k8s/` folder. Edit it there and copy it back here with
`rsync -a --delete ../backtester/k8s/ apps/dev/backtest/` (keep this README).
Flux applies `overlays/home`. The files in `jobs/` are templates for `kubectl create -f` and are not applied.
