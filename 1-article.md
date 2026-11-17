PHPStan 2.3 landed on October 1 with a big promise: up to 7.5x faster than PHPStan 2.1 from last December. The numbers in the release post come from WordPress and Drupal. Nice projects, but not the kind of code I work on every day.

So I upgraded a production Laravel app the same month and timed it. The cold run got two times faster. The run I make fifty times a day saved about a second. And the new unused-variable detector found nothing, because Rector had already cleaned up.

Here's what I measured, what broke, and why I left bleeding edge off for now.

## The Setup

The app is a mid-size Laravel 11 project I work on:

- `app/` with 1145 PHP files
- Larastan, level 5, plus a `phpstan-baseline.neon`
- PHP 8.3 in a Docker container, 4 CPUs, on WSL2, code on the Linux filesystem
- PHPStan 2.2.8 + Larastan 3.10 before, PHPStan 2.3.1 + Larastan 3.13 after

Larastan follows PHPStan releases closely, so I bumped both together. That means every number below is "the PHPStan and Larastan upgrade", not PHPStan alone.

A small shell script ran `phpstan analyse -vv` 10 times per scenario and wrote time, peak memory and the number of reanalysed files into a CSV. For cold runs it deleted the cache first. I report the median, and the min-max range where the runs jumped around.

Heads up: this is one machine, inside Docker, on WSL2, at level 5. Measure your own project before you trust these numbers. Higher levels do more work and can behave differently.

## Cold Runs: The Headline Holds

A cold run means no result cache. That's your first local run after `git clone`, or every CI job that doesn't restore the cache.

| Scenario | 2.2.8 | 2.3.1 |
|---|---|---|
| Cold run, median | 34.0 s | 15.7 s |
| Cold run, min-max | 27-55 s | 12-19 s |

Two times faster, and much more stable. With 2.2.8 one run took 27 seconds and another one 55. With 2.3.1 everything stayed between 12 and 19.

The release post compares against PHPStan from last December, which is much older than 2.2.8. So my 2x doesn't argue with their 7.5x. A lot of that speed already shipped in the 2.2 releases I had before.

Memory went up a bit: my script measured about 1.4 GB on 2.2.8 and 1.55 GB on 2.3.1. Take that with salt. The script adds up memory over all PHPStan processes, and 2.3 forks its workers, so some shared memory is probably counted twice. If you run CI on a small runner, check it yourself before the upgrade.

For CI, the release post is right. If your pipeline runs PHPStan without a cached result, this upgrade alone cuts that step in half.

## Warm Runs: Where You Actually Live

But here's the thing: locally I almost never run PHPStan cold. I edit a file, run it, edit again. That's the warm path, with a result cache from the previous run.

| Scenario | 2.2.8 | 2.3.1 |
|---|---|---|
| Nothing changed | 1.2 s | 1.6 s |
| One file edited | 3.2 s | 2.0 s |
| `guzzlehttp/guzzle` bumped | 2.6 s, 1 file | 5.7 s, 198 files |
| `laravel/framework` bumped | 26.7 s, all files | 10.5 s, all files |

With nothing changed, 2.3.1 is 0.4 seconds slower. With one file edited, it's a second faster. That's the real day-to-day difference, and you will barely feel either.

For the package rows, I didn't run a real `composer update`. I changed the package `reference` in `vendor/composer/installed.php` and one byte in `composer.lock`, which is enough for PHPStan to see a changed package. Close to a real update, but not the same thing.

A `laravel/framework` bump makes both versions throw the whole result cache away. With `-vv`, PHPStan tells you why:

```plaintext
Composer packages changed (laravel/framework) and register a class in the container; re-analysing everything
```

The "container" here is PHPStan's own DI container, not Laravel's. Larastan registers Laravel classes like `Illuminate\Foundation\Application` as PHPStan services, so a framework bump touches a package that PHPStan itself depends on. Full reanalysis is the safe answer. It's still faster than cold, because the reflection cache survives, and 2.3.1 needs 10.5 seconds instead of 26.7.

The guzzle row surprised me. The release post says:

> Previously, the project was fully reanalysed when any Composer dependency was updated, or when the Symfony DI container changed in any way.

On my app, with my fake bump, 2.2.8 reanalysed only 1 file. 2.3.1 reanalysed 198 and took twice as long. I haven't checked this with a real `composer update`, and I don't know which files the 198 are. So I'm not drawing a conclusion. Just don't be surprised if a small package bump takes a few seconds longer than before.

## What the Upgrade Cost

Without bleeding edge, the upgrade gave me 4 new errors. All of them had the same shape: an empty `Collection::make()` that PHPStan now reads as a collection of `never`.

Three of them were readonly properties filled in the constructor. Here's one, from a service that loads price types for a user (names shortened):

```php
// BEFORE: the property @var no longer helps after the upgrade
final readonly class PriceTypeService
{
    /**
     * @var Collection<int, PriceTypeDto>
     */
    private Collection $priceTypes;

    public function __construct()
    {
        $this->priceTypes = Collection::make();
    }
}
```

The property has a correct `@var`, but after the upgrade the type comes from the assignment in the constructor instead. An empty collection has no keys and no values, so the type becomes `Collection<never, never>`, and the code that reads from it starts to fail.

The fix is a `@var` on a local variable before the assignment:

```php
// AFTER
public function __construct()
{
    // PHPStan 2.3 infers a readonly property from its constructor assignment over @var
    /** @var Collection<int, PriceTypeDto> $priceTypes */
    $priceTypes = Collection::make();
    $this->priceTypes = $priceTypes;
}
```

Ugly, but the comment tells the next developer why the extra line is there.

I can't tell you whether this change comes from PHPStan 2.3 or Larastan 3.13. They went in together.

Seven entries in `phpstan-baseline.neon` stopped matching, and the baseline got 42 lines shorter. Two of them were about a cart repository that I fixed in the same commit, so they don't count. The other five went away on their own, because 2.3 infers collection types better. For example:

```plaintext
Method SupplierRepository::getSupplierDeliveryCurrencyData() should return
Collection<int, CurrencyEnum|null> but returns Collection<(int|string), CurrencyEnum|null>.
```

That one was a false positive. 2.3 understands it now.

## Bleeding Edge, Counted Before Enabling

Bleeding edge is an opt-in config that turns on rules planned for the next major version. In 2.3 the best new features sit behind it: the unused-variable detector and the new generics inference. I wanted to know what they cost before I turned them on.

The trick is to **count before you regenerate the baseline.** If you enable bleeding edge and run `--generate-baseline` straight away, every new error disappears into the baseline, and you never learn what it found.

So I enabled it, ran the analysis, and grouped the errors by identifier:

| Identifier | Count |
|---|---|
| `property.internal` | 8 |
| `ignore.unmatched` | 7 |
| `return.type` | 6 |
| `argument.type` | 2 |
| All unused-variable identifiers | 0 |

23 errors, and none of them was a bug:

- `property.internal`: our Artisan commands use `$this->components` for the nice console output, and Laravel marks it `@internal`. One ignore rule silences all 8.
- `ignore.unmatched`: baseline entries that no longer match. This is cleanup. I didn't check if they are the same 7 entries from the plain upgrade.
- `return.type` and `argument.type`: the new generics are more precise than our `@return` tags. That's 8 PHPDoc edits.

And the unused-variable detector found zero. At first I thought it wasn't running at level 5, so I checked it on a test file with a dead assignment. It fired. Our code was simply clean.

Turns out, Rector got there first. The project runs Rector with `SetList::DEAD_CODE` in `composer test`, so a dead assignment fails the check before it is merged. That set covers unused variable assignments. It doesn't cover everything PHPStan now finds (the `*Flow` rules, for example, have no Rector twin), but on this codebase there was nothing left for them either.

With bleeding edge on, the 4 `Collection<never, never>` errors from the previous section disappear. The new inference looks at how the collection is used later, so it finds the type without my `@var` lines.

So it costs 8 PHPDoc edits and one ignore rule. In return I'd delete 4 workarounds and find no bugs. Not worth it yet, so on this project I left it off. I'll look again if Rector's dead-code set leaves the project, or when PHPStan 3 makes these rules the default anyway.

## My Rule of Thumb

- Upgrade to 2.3 for the cold runs. In CI without a restored cache, PHPStan gets about two times faster.
- Don't expect your local loop to change much. Warm runs were already fast.
- Watch for `Collection<never, never>` on readonly properties. A `@var` on a local variable fixes it.
- Before enabling bleeding edge, count the errors by identifier. Don't regenerate the baseline first.
- If Rector's `DEAD_CODE` set already runs in your project, the unused-variable detector may have little left to find. Count first and see.

💡 Want to know what bleeding edge would cost you? Add this to a copy of your config:

```neon
includes:
    - vendor/phpstan/phpstan/conf/bleedingEdge.neon
```

Then run it with `--error-format=json` and group the errors by their `identifier` value.

---

- [PHPStan 2.3: Leap in Performance, Generics Improvements, Detecting Unused Variables, and More!](https://phpstan.org/blog/phpstan-2-3-leap-in-performance)
- [PHPStan releases](https://github.com/phpstan/phpstan/releases)
- [Larastan releases](https://github.com/larastan/larastan/releases)

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Laravel, after the happy path.**
