# PHPStan 2.3 on a Real Laravel App: What Got Faster, What Broke

![PHPStan 2.3 on a Real Laravel App: What Got Faster, What Broke](assets/poster.jpg)

PHPStan 2.3 promised up to 7.5x faster analysis, measured on WordPress and Drupal. I upgraded a 1145-file production Laravel app the same month and timed it.

The cold run got two times faster. The run I make fifty times a day saved about a second. And the new unused-variable detector found nothing, because Rector had already cleaned up.

## What's Inside

- Cold and warm run timings, 2.2.8 vs 2.3.1, under Larastan at level 5
- Why a `laravel/framework` bump still reanalyses everything, and a guzzle bump went from 1 file to 198
- The one upgrade error: readonly properties and an empty `Collection::make()`, and the `@var` fix
- Counting bleeding-edge errors by identifier before touching the baseline
- Why Rector's `DEAD_CODE` set left the unused-variable detector nothing to find

## 📎 Read Full

[PHPStan 2.3 on a Real Laravel App: What Got Faster, What Broke](https://dev.to/tegos/PLACEHOLDER)
