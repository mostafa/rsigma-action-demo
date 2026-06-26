# rsigma-action-demo

A minimal Sigma detection-as-code repository that runs [`timescale/rsigma-action`](https://github.com/timescale/rsigma-action) as a pull-request gate. It is the end-to-end demo consumer for the action: every push and pull request lints, validates, backtests, and ATT&CK-maps the rules in `rules/`.

## What the gate does

On each pull request, [`.github/workflows/detection-as-code.yml`](.github/workflows/detection-as-code.yml) runs the action, which installs a verified rsigma release and then:

- lints the rules and annotates findings on the diff,
- validates that every rule parses and compiles,
- diffs the rule field set against the merge-base,
- backtests the rules against the `tests/corpus/` events and checks `tests/expectations.yml`,
- maps the rules onto ATT&CK and reports coverage against `tests/targets.txt`,
- and keeps a single sticky summary comment up to date.

## The regression this would have caught

This demo is anchored on [SigmaHQ PR #5964](https://github.com/SigmaHQ/sigma/pull/5964): an Okta rule whose field name was silently lowercased, so it never matched real Okta events and quietly detected nothing for years. Linting and validation could not catch it, because the rule was still well-formed and still compiled.

A backtest can. `rules/okta_admin_role_assigned.yml` matches on `eventType` and `displayMessage`, the camelCase field names Okta actually emits (see `tests/corpus/okta.ndjson`), and `tests/expectations.yml` asserts it fires at least once.

To see the gate work, edit `rules/okta_admin_role_assigned.yml` and lowercase the field name (`eventType` to `eventtype`), then open a pull request. Lint and validate still pass, but the backtest fails: the rule no longer matches the camelCase event, its fire count drops to zero, and the `at_least: 1` expectation is violated. That is the exact class of bug #5964 was, and the gate #5964 lacked.

## Layout

```
rules/                 Sigma detection rules
tests/corpus/          NDJSON events the rules are backtested against
tests/expectations.yml Per-rule fire-count assertions
tests/targets.txt      ATT&CK techniques the ruleset should cover
.github/workflows/     The detection-as-code gate
```

## Local checks

With [rsigma](https://github.com/timescale/rsigma) installed you can run the same checks locally:

```bash
rsigma rule lint rules/ --fail-level warning
rsigma rule validate rules/ --resolve-sources
rsigma rule backtest --rules rules/ --corpus tests/corpus --expectations tests/expectations.yml
rsigma rule coverage --rules rules/ --targets tests/targets.txt --navigator coverage.json
```

## License

MIT. See [LICENSE](LICENSE).
