# So I Made Our Postmortems Run in CI

*"Experience is simply the name we give our mistakes."*
— Oscar Wilde

## The morning the test failed while passing

A few weeks ago, an automation suite I maintain had one of those weeks. Salesforce journeys, driven end to end with Playwright - create a customer, convert it, book the order, verify what a downstream system received. The unglamorous kind of testing.

It started with a run that failed at 96%. Not flaky infrastructure, not a broken selector. The test hit its five minute timeout **while it was still working**. The journey it automates takes about five and a half minutes when everything goes right. The cap expired during the final steps of a completely green run.

And here is the part that cost me an hour of cleanup: the runner retried.

The journey creates records in a shared staging org. Create, convert, book. Irreversible operations against a shared inventory. The retry started the whole journey again, against a system that had already accepted the first attempt. By the time anyone looked, there were duplicate orders sitting in the org, created by nobody, authored by a config file.

I raised the timeout and moved on. That is what you do. Fix, push, next fire.

What I did not do was ask the question that would have saved me the next three days:

**where else is this lesson currently not written down?**

## The week the suite confessed

Once I started looking, the lessons were everywhere. Each one paid for in a specific evening.

A success toast that appears for two seconds on fast machines and never on slow ones. The fix races the toast against something durable instead of trusting it alone.

A field reader that used `isVisible({ timeout: 5000 })` to wait for a label. Except `isVisible()` accepts no options and returns instantly, so labels still rendering came back as empty strings, silently, on every verification. The fix swaps in a real wait.

Picklist values typed as `web` where the platform demands exactly `Web`. Rejected on save, traced by hand, fixed with one character.

Smoke scenarios pinned to specific inventory items. Every green run consumed its own fixture, so each success quietly poisoned the next run.

Plaintext credentials sitting as fallback defaults in config, removed in a purge commit.

Eight fixes landed across five days in our repo. Each commit message is a tiny postmortem. Each one closed a wound in exactly one file.

Then I did something slightly uncomfortable. I checked whether the lessons had actually spread, or whether they were still standing where they fell.

## Fixes are local. Memory is not.

That `isVisible` fix? Correct, well-commented, committed with an explanation of the trap.

The same misuse was still sitting in **twelve other places across five page objects in our repo**. Some written *after* the fix existed.

The picklist purge? Thorough. It missed tokens in a different constants file entirely.

The credential cleanup? It cleaned three environments and left two others holding plaintext defaults.

And the `.env` file with credentials in it that someone added to `.gitignore`? Still tracked by git. Ignore rules never untrack anything. Every clone carried it.

Nothing here is unusual. This is just what organisations look like on the inside: fixes land where the pain was, lessons stay where the people are, and both drift apart the moment attention moves elsewhere.

We write postmortems as if reading them changes something. They don't. Reading is not enforcement.

**The problem was never fixing. It was remembering.**

## What if the retro could run?

So I tried a different move. Instead of writing down what we learned, I made the lessons executable.

The idea takes one diagram:

```
git history  →  extract once  →  rules file  →  review forever
```

Every incident becomes one JSON object: a one-line rule, the commit hash that proves it, how bad recurrence would be, and a pattern a machine can look for. Then a small offline engine reads that file and checks code against all of it, on every pull request if you want.

Here is a finding from our rule set, paths shortened, so you can see the shape:

```
[ HIGH] WAIT — Never pass options to isVisible(). It takes none
       and returns instantly.

  where : pages/orders-page.ts:186
  matched: if (await result.isVisible({ timeout: 5000 })...)

  origin : 9f2c41d
           isVisible() ignores options and returns immediately,
           so a label still rendering read as absent and the field
           value was silently reported as ''.

  do this: locator.waitFor({ state:'visible', timeout:N })
```

The shipped template calls this one `ST-004`; our rule set had its own names first.

Notice what the finding is not. It is not a lint opinion. It is not a model's vibe. It is a specific line, tied to a specific afternoon, with the fix that already worked once. A reviewer can argue with a style opinion. A commit is harder to argue with.

The extraction is the only part that touches AI. First comes the deterministic pass: `sfqa-scar harvest` reads your git history, proposes a draft for every incident no existing rule claims yet, and hands the list back to a human - so the retro has something to run before it ever has to write. Then you point an assistant at the remaining history with a prompt that refuses to accept any rule without a citing commit. Everything after that is plain regex over local files. No network. No keys. Same input, same findings, every time, which is the only way a merge gate earns the right to fail your build.

## Proving it on our own scars

An agent like this is easy to demo and easy to fake, so I built a proof harness before publishing anything.

For each incident in the history, the script extracts the file as it existed **before** the fix commit, runs the reviewer against it, then extracts the file **at** the fix commit and runs it again.

Two criteria, because not every fix was complete. Most rules must fire on the past and go quiet on the present. For the credential rule, the honest criterion is weaker: the count must go down.

```
COMMIT    FILE                    RULE    PRE  POST  VERDICT
9f2c41d   pages/field-reader.ts   WAIT     1     0  PASS
cc1f7b2   config/env.ts           SECRETS 16→4  PASS*
79ac113   pages/orders-page.ts    POLL     3     0  PASS
58de613   playwright.config.ts    TIMEOUT  1     0  PASS
...8 incidents, 8 passes...
```

`PASS*` is a decrease-mode pass, not a clean one.

Look at the second row. Sixteen findings before the credential purge. Four after.

Not zero.

Four plaintext fallbacks survived the original cleanup, in environments everyone had stopped looking at. My agent found them on its first replay pass. It reviewed the work of the person who wrote it and handed back homework.

I have decided to find that encouraging rather than humiliating. Both, honestly.

## The catch that justified the whole build

The same first run also flagged this:

```
[HIGH] SECRETS — Never hardcode credentials or commit .env files.

  where : .env
  matched: (file name ".env")
  origin : cc1f7b2
           ...a .env with credentials in it was committed and later added to
           .gitignore - which never untracks existing files...
```

The file was sitting right there, ignored on paper and fully tracked in reality, in a repository maintained by someone who absolutely knew better. Git's most common footgun, caught not because anyone re-checked but because a rule written from an old mistake simply kept looking.

That is the entire pitch in one moment: **the check does not get tired of checking.**

## The second catch: a rule that could never fire

The replay was not the last gate I built.

Before publishing anything, I rebuilt the proof as a fixture harness: for every
rule with a machine detector, a positive fixture must make it fire, and a clean
fixture set must produce zero findings. Same idea as the replay, but it does not
depend on this repository's history, so anyone can run it.

The fixture harness failed on its first run.

Not because a fixture was wrong.

**One rule had a detector that could never fire.**

It checked that each environment block reads its own environment's variables -
the exact bug from the staging environment that quietly authenticated against the
wrong org. The pattern was written to span lines. The engine was testing every
line pattern against one line at a time. A cross-line regex cannot match a single
line, so the rule sat in the report looking thorough and could never flag
anything.

A rule that cannot fire is worse than a missing rule. It buys trust it has not
earned.

The fix scans multi-line patterns against the whole file and maps each match back
to its line. A regression fixture now proves it fires. The proof caught the tool
lying about itself before anyone else could.

That is the part I trust most: the gate does not care that I wrote it.

## The third catch: the detectors were the problem

Fixtures prove a rule fires. They do not prove it fires on the right thing, so I
ran the whole set against the repository it came from.

Thirty-seven findings came back at `high`. About eight were real.

That is the number nobody puts in a launch post, and it is the one that decides
whether a tool survives contact with a team. A reviewer that opens with
twenty-nine false alarms gets switched off in a week, and rightly.

Here is what most of them were:

```
[HIGH] POLL — never pass options to isVisible()

  test-pages/web/HomePage.ts:865
  const launcherVisible = await this.page
    .locator('.appLauncherMenu, one-app-launcher-modal')
    .first()
    .isVisible({ timeout: 3000 })
    .catch(() => false);
```

A deliberate probe. The rule counts `.isVisible({ timeout })` as a silent
assertion because of an incident where exactly that line quietly asserted the
wrong thing - but this one is guarded on the next line, so it cannot be silent.
The scanner was reading one line at a time.

Two more shapes: `'Email': 'email'` in a field map, read as a misspelt picklist
label, and `map.get('password')`, read as a hardcoded credential. Both are
values in code, not labels or secrets.

Every one became a detector fix with a regression test - never a severity
demotion in the shipped template. A rule that fires on clean code is a rule your
team turns off, and then it protects nothing at all.

Re-run on the same tree: 113 findings, 11 `high`, and most of those eleven are
things I genuinely owe an afternoon to. Two categories I am deliberately living
with - inventory rows in feature tables because there is no test-data API yet,
and an account-name assertion that is safe today because it reads a name the
test itself created - are demoted in *our* rule set, with the reason written
into the rule, not in the template everyone else gets.

That is the promise the tool makes, and the reason it is worth a pull request
slot: when it is wrong, it is wrong in a way you can read, fix, and prove.

## What this is not

It is not another AI code reviewer. The interesting part of the design is that the AI touches it only during extraction, under supervision, with receipts required. Enforcement afterwards is boring on purpose. Boring is what lets you put it in a pipeline and trust a red build.

Your rules file encodes your organisation's biography, warts included. It lives in your repo, reviewed like code, versioned like code. The scanner never leaves your machine; extraction is the one step where you decide what the model sees. If that sounds like your `.eslintrc`, except written by your worst weeks, that is exactly what it is.

## Where this fits

This is packaged now: a zero-dependency CLI, a rule-set format, an extraction
prompt, and a proof harness that proves the shipped rules fire and stay quiet
before you trust a word I said. MIT.

The replay above is mine, on our history. The fixture harness is the part you can
run.

It is one piece of a small family I am building around the same idea. There is a
portable core with no org vocabulary in it at all - a boundary test fails the
build if that changes. There is a probe that reads a Salesforce org's describe
metadata and writes the org adapter the main journey is built on: exact labels,
active picklist values, required flags, and a review sheet of what it could not
map. The probe writes the adapter; it does not write the journey. The journey is
authored against it, which is the point: grounded in what the org actually says,
not what I remembered about the last one.

Pointing the whole chain at an unfamiliar sandbox is the next milestone.

**The scar reviewer is still being tested.** Deliberately. It is v0.1.0, and the
shipped set is a template: 18 of the 19 rules have a detector and are proven both
ways; the nineteenth is listed as a gap, not counted as coverage. The public proof
runs on synthetic fixtures, not a field trial. The next round is running it against
repositories that have nothing to do with this one, measuring false positives
honestly, and publishing those numbers whether or not they flatter me.

## What it looks like when I run it

These are the commands, the inputs, and the outputs - the same loop I run in a
repo.

Install once with `npm install -D @sfqa/scar`; every command below runs through `npx`.

### 1. Scan: a known scar firing again

`scar-tissue.json` in the repo root is *my* rule set (auto-picked up). If a repo
has none, the scan falls back to the packaged template `rules/generic.json`; the
first `sfqa-scar new …` or a copy of that template gives you a file of your own
to tune.

```bash
$ npx sfqa-scar --markdown --fail-on high .
```

```text
## sfqa-scar

**113 finding(s)** across 1385 file(s) - 4 high, 46 med, 63 low.

| Rule | Severity | Where | Matched |
| --- | --- | --- | --- |
| ST-001 | high | `test-lib/setup/env.ts:472` | `Password: process.env.STAGING_ADMIN_PASSWORD \|\| '<redacted>'` |
| ST-001 | high | `test-lib/setup/env.ts:474` | `Password: process.env.STAGING_MANAGER_PASSWORD \|\| '<redacted>'` |
| ST-001 | high | `test-lib/setup/env.ts:498` | `password: process.env.STAGING_MANAGER_PASSWORD \|\| '<redacted>'` |
| ST-004 | high | `test-pages/web/LeadPage.ts:591` | `const spinnerExists = await … .isVisible({ timeout: 1000 });` |
| … | … | … | … |
```

That is the "after" number from the third catch on the repo that mined the rules:
37 highs down to 11 once the detectors stopped misreading code, and 4 once I
demoted, in *my* `scar-tissue.json` (never the template), the two categories I am
deliberately living with and wrote the reason into. To review only what changed:

```bash
npx sfqa-scar --base origin/main --fail-on high
npx sfqa-scar --staged --fail-on high
```

### 2. Harvest: the incidents nobody wrote a rule for

`harvest` reads fix-shaped commits and local report artifacts, proposes a draft
for each unclaimed incident, and hands the list back - it writes nothing unless
you ask it to.

```bash
$ npx sfqa-scar harvest --since 2026-01-01
```

```text
CANDIDATE 1/1
  commit : 402ef2d  2026-11-09
  subject: fix: replace fixed sleep with an element wait
  files  : tests/orders.ts
  removed: await page.waitForTimeout(2000);
  added  : await locator.waitFor({ state: 'visible', timeout: 5000 });
  detect : page\.waitForTimeout\(  [confidence high]
  scope  : tests/**
  id     : commit:402ef2d9c1...

No rule claims this incident yet. Review the draft, prove it, then add it.
```

`--json` or `--write scar-candidates.json` gives you the machine-readable form;
`--strict` in CI turns "we forgot to record last month's hotfix" into a red build.

### 3. Turn an incident into a rule (`new` → fill → `prove`)

One command scaffolds the rule and puts both fixtures where the rule will
actually look:

```bash
$ npx sfqa-scar new --rule "Never wait on a fixed sleep for a UI element" --scope tests/
created ST-020 (new rule file)

  rule      scar-tissue.json
  positive  scar-fixtures/positive/tests/st-020.ts
  clean     scar-fixtures/negative/tests/st-020.ts

what to do next
  1. open scar-tissue.json, replace the TODOs, and put the pattern in match.linePatterns
  2. not sure of the pattern? prompts/turn-incident-into-rule.md drafts it from the incident
  3. paste the bad line into scar-fixtures/positive/tests/st-020.ts
  4. paste the same code with the fix into scar-fixtures/negative/tests/st-020.ts
  5. prove just this rule: sfqa-scar prove --id ST-020
```

The rule it wrote is the input - data, with an empty detector (a gap) until you
fill it. Positive fixture gets the bad line, clean fixture gets the fixed line:

```json
{
  "id": "ST-020",
  "rule": "Never wait on a fixed sleep for a UI element",
  "severity": "med",
  "match": { "pathInclude": ["tests/**"], "linePatterns": [] },
  "origin": { "commit": null, "summary": "TODO: what broke, and what it cost." },
  "recommendation": "TODO: the fix that worked."
}
```

Not sure of the pattern? Hand the incident to `prompts/turn-incident-into-rule.md`
(in the package) and an assistant drafts the detector and both fixtures. Then
prove it - `prove` is the part I trust, not the draft:

```bash
$ npx sfqa-scar prove --id ST-020
rule    severity  detector   positive   clean
ST-020  med       yes        fires      clean

PROOF: 1/1 executable rules fire on a positive fixture and stay silent on the clean set.
```

`fires` = the bad line was caught. `clean` = the fixed line was not. A rule with
no detector, or a fixture that never fires, is reported as a gap/miss and the
command exits 1. The shipped set sits at **18/18 executable rules proven, 1
documented gap**.

If you want to start today: run `harvest`, turn one candidate into a rule with
`new` + the prompt + `prove`, and let the resulting file start reviewing your pull
requests. Then watch what your history confesses when it finally gets asked.

## Links

- The package: [`@sfqa/scar` on npm](https://www.npmjs.com/package/@sfqa/scar)
- The repository: [github.com/desardavaishali/sfqa-scar](https://github.com/desardavaishali/sfqa-scar)
- The proof: [`scar-validation/REPORT.md`](https://github.com/desardavaishali/sfqa-scar/blob/main/scar-validation/REPORT.md)
- The extraction prompt: [`prompts/extract-rules.md`](https://github.com/desardavaishali/sfqa-scar/blob/main/packages/scar/prompts/extract-rules.md)
- The rule-set template: [`rules/generic.json`](https://github.com/desardavaishali/sfqa-scar/blob/main/packages/scar/rules/generic.json)

---

*If you run the extraction on your own repository and it finds something that made you wince, I would genuinely like to hear what it was. Those stories are how the starter rules get better.*

*This sits alongside [Built on Scar Tissue](https://www.vaishalidesarda.com/blog/built-on-scar-tissue-part-3/): Part 3 asked what happens when a request arrives twice. This month I watched my own test suite answer that question against a shared staging org, and decided the retro should be the thing that runs first next time.*
