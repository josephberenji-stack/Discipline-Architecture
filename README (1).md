# discipline

**A referee for computational claims made with a language model.**

`discipline` is a small, dependency-free Python suite that separates the party that
computes a result from the party that announces whether the result came out. It consists
of a runtime module that a computation file imports (`referee.py`), twelve static checks
that run over the file and the project around it, a results index that makes "what does
the project already contain?" a search rather than a memory, a set of plain-text registers,
and a protocol (`PROTOCOL.md`) that states, for every rule, the failure it exists to
prevent.

The suite was built for a setting in which a person without formal training supplies ideas
in prose, a language model turns them into computation, and nothing in that arrangement
stops the model from writing the conclusion it was steered toward. Every check here is a
mechanical answer to a specific way that goes wrong: a verdict typed rather than computed;
a count printed with no rows behind it; a prediction written after the number existed; a
citation that nothing reads; a word that quietly acquires a second meaning; a working
document that grows until it can no longer be carried. The checks are file-level and
language-level. They need nothing from the model, and the same checks apply to every model.

---

## Contents

1. [Design](#1-design)
2. [What is in the release](#2-what-is-in-the-release)
3. [Requirements and installation](#3-requirements-and-installation)
4. [Quick start](#4-quick-start)
5. [Writing a computation file](#5-writing-a-computation-file)
6. [The runtime referee](#6-the-runtime-referee)
7. [The file-level checks](#7-the-file-level-checks)
8. [The corpus-level checks](#8-the-corpus-level-checks)
9. [The baseline](#9-the-baseline)
10. [The results index and the citation address](#10-the-results-index-and-the-citation-address)
11. [The registers](#11-the-registers)
12. [The project log](#12-the-project-log)
13. [Storage: one canonical space](#13-storage-one-canonical-space)
14. [Configuration](#14-configuration)
15. [Exit codes](#15-exit-codes)
16. [The controls: testing the checks](#16-the-controls-testing-the-checks)
17. [Limits and non-goals](#17-limits-and-non-goals)
18. [Adoption: hooks, CI, agents](#18-adoption-hooks-ci-agents)
19. [Licence and citation](#19-licence-and-citation)

---

## 1. Design

Four principles run through every module.

**Rows, not counts.** A count is believable in a way that the rows behind it are not. Every
check prints the violating instances — file, line, kind, and why — and exits on the count
of rows it cannot hold. A count with no rows beside it is itself a row (`rowsplit`).

**Predictions before runs.** A number can be reinterpreted after the fact, and a fluent
reinterpretation is indistinguishable on the page from a prediction that came true. The
only mechanical defence is to have the failure condition on disk before the number exists.
A file declares what it predicts and what would kill it, in its docstring, above any code;
the referee records the declaration before the verdict and refuses a verdict with none.

**The verdict is computed.** The printed word — HOLDS or FAILS — is chosen by a value, never
typed. The referee parses its own call site and refuses a literal argument; the static
check `forced` finds the subtler cases, where the argument is computed by the letter but
traces back only to constants written into the file.

**The record is the transcript.** Corrections are numbered and logged in the docstring of
the file where the error happened, naming who found it; the consolidated ledger is
regenerated from the files, never kept by hand. The human's own words enter the log
verbatim. Working documents are frozen at a size cap with their line counts intact, because
everything cites them by line.

The suite has two halves. The **runtime** half is `referee.py`, imported by the
computation file itself; it exits non-zero from `close()` when the run does not count. The
**static** half is the twelve checks, run by `sh check.sh FILE.py`; they read the file and
the project and print rows. What neither half can catch — an identity that is computed
rather than typed — is what the negative-control requirement is for: a file must name a
case that must fail, and if no such case can be named, the check is not a check.

## 2. What is in the release

```
discipline/                 the suite (standard library only)
  config.py                 where the suite looks and what it skips -- one place
  referee.py                the runtime referee: declare / verdict / rows / control / withdraw / close
  run.py                    runs every check with each one scoped correctly; writes discipline-report.md
  predeclare.py             a file that prints a verdict carries a prediction above it
  rowsplit.py               a kill-check reports the rows that violate, and a witness
  guard.py                  a function whose domain is narrower than its signature asserts it
  withdraw.py               a withdrawal is written above the result, and propagates
  noliteral.py              no printed column and no printed verdict may be a constant
  names.py                  one word, one definition; the census of senses is generated
  forced.py                 was the outcome ever at risk?  the static half of the referee
  cited.py                  a citation that nothing reads is decoration; the name is the address
  ground.py                 a claim-bearing file builds its own mathematics from the ground up
  primitive.py              one verified instance, many carried copies, drift made checkable
  ledger.py                 the correction ledger, regenerated from the files, with gaps listed
  rollover.py               a working document is frozen at the cap with its line count intact
  baseline.py               standing rows recorded as identities, so a NEW row is visible
  index.py                  the results index: every claim, verdict, heading and declaration, by file:line
  log.py                    the project log: the human's words, verbatim, connected to the record
  storage.py                one canonical space, a mirror kept exact, a quick check between reindexes
check.sh                    sh check.sh FILE.py -- the whole of the author's obligation
reindex.sh                  regenerate RESULTS-INDEX.md and prove every citation resolves
quickcheck.sh               has anything the index reads changed?  is the mirror exact?
mirror.sh                   copy canonical final files into the mirror
registers/                  plain-text registers, shipped empty with their formats in the header
  NAMES.md                  the word registry
  BASELINE.md               standing rows (generated by baseline.py --write)
  FROZEN.md                 documents closed at the rollover cap
  PRIMITIVES-REGISTER.md    the log space (generated by primitive.py --rebuild)
  INDEPENDENT.md            reimplementations that must not be reconciled (hand-written)
  PROJECT-LOG.md            the human's entries (appended by log.py)
  storage.cfg               parts, canonical globs, mirror, exclusions
example/                    a two-file project and every output the suite produced on it
PROTOCOL.md                 the method and the standing protocols
LOG-PROTOCOL.md             the [[LOG]] flag, template, and what the model does with it
CITATION.cff                how to cite this software
```

## 3. Requirements and installation

- Python 3.8 or later. Nothing outside the standard library is imported anywhere in the
  suite. (On Python 3.10+ `ground.py` uses `sys.stdlib_module_names` to recognise the
  standard library; on older interpreters it falls back to a built-in list.)
- A POSIX shell for the four `.sh` scripts (`sh`, `dash`, `bash`, `zsh`). The scripts are
  thin wrappers; every function is reachable with `python3 -m discipline.<module>` alone.

There is nothing to install. Copy `discipline/`, the four scripts and `registers/` into the
root of the project they will check, and create the marker file that tells the suite where
the project begins:

```sh
cp -r discipline check.sh reindex.sh quickcheck.sh mirror.sh registers  /path/to/project/
cd /path/to/project
touch .discipline
```

The directory holding `.discipline` is **the space**: the directory whose documents
`rollover` watches, whose computation files `baseline` records, and against which every
citation is written. Section 14 describes the environment variables that override this.

## 4. Quick start

The `example/` directory is a complete project: `example.py` states a claim that holds,
`broken.py` is a file the referee must refuse, and `walkthrough.out` is the transcript of
every command below, exactly as the suite printed it. To reproduce it:

```sh
cd example
cp -r ../discipline ../*.sh ../registers . && touch .discipline

python3 example.py            # the referee: "1 of 1 declared claims hold ... The run counts."   exit 0
python3 broken.py             # "B1 REFUSED.  literal verdict."  "no negative control ran"        exit 1
sh check.sh example.py        # twelve checks, 0 rows                                             exit 0
sh check.sh broken.py         # forced 1 row, ground 1 row; rows in discipline-report.md          exit 1
sh reindex.sh                 # RESULTS-INDEX.md rebuilt; every citation resolves
python3 -m discipline.baseline --write     # record the two standing rows as identities
python3 -m discipline.run broken.py        # "2 file-level rows (2 held by baseline)"            exit 0
python3 -m discipline.log add message.txt  # L-0001, both connections resolved against the index
sh quickcheck.sh              # "FULL REINDEX DUE" -- the log changed                            exit 2
sh reindex.sh && sh quickcheck.sh          # "index current; mirror exact."                     exit 0
```

Each of these outputs is in `example/walkthrough.out`; the referee's own output on the two
files is in `example/example.out` and `example/broken.out`.

## 5. Writing a computation file

A computation file is an ordinary Python script with three properties: its docstring
carries a declaration, it imports `referee`, and it lets the referee print every verdict.
`example/example.py` is the model:

```python
"""
example.py -- is the sum of the first n odd numbers a perfect square?

Human: "I think adding up odd numbers always gives you a square."

# MIRRORS.  The identity 1 + 3 + ... + (2n-1) = n^2 is a classical result; it is
# rebuilt here from the definition and nothing is imported for it.

DECLARED IN ADVANCE.  For every n from 1 to 200 the sum of the first n odd numbers is
the square of some integer.  KILL: an n in that range whose sum is not a square.
CONTROL: the sum of the first n EVEN numbers is a square -- which must FAIL, since
2 + 4 = 6 is not one.

THE OBJECTS.  An ODD NUMBER is 2k - 1 for k >= 1.  The SUM S(n) is the sum of the
first n of them.  A PERFECT SQUARE is an integer m*m for some integer m >= 0.

WHAT IS FORCED.  I looked for a theorem forcing the answer whatever the objects are:
the identity above forces it, and it is the thing being checked here, so the run is
a calibration of the code against a known result and is declared as such.
"""
import sys
sys.path.insert(0, "discipline")
import referee as R

R.declare("SQ1",
          predicts="for every n from 1 to 200 the sum of the first n odd numbers is a perfect square",
          kills="an n in 1..200 whose sum of the first n odd numbers is not a perfect square",
          requires=[],
          control="the sum of the first n even numbers is a perfect square, which must fail",
          budget="1 of 3")
...
R.rows("SQ1", bad)
R.verdict("SQ1", len(bad) == 0, f"{N - len(bad)} of {N} sums are squares")
...
R.control("even-ctrl", len(even_bad) == 0,
          f"{len(even_bad)} of {N} even sums are not squares", vacuous=0, rows=N)
R.close()
```

The docstring blocks are what the static checks read. `DECLARED IN ADVANCE` (or `PREDICT`,
`before the run`, `would kill`, and similar markers) is what `predeclare` looks for above
the first loop or function. `# MIRRORS` within the first forty lines, or a `MIRRORS`
section in the docstring, is what `ground` requires of any file that calls `declare()`. A
number written in the declaration is a declared prediction, and `forced` will not row a
comparison against it; a number that appears only in a comparison is an expectation the
file never committed to, and will be rowed. `WHAT IS FORCED` is a protocol requirement
(`PROTOCOL.md` §3.8) rather than a mechanical one: before declaring, the author names the
theorem that would force the answer whatever the objects are, or states that none applies.

A file's `requires=[...]` list names the results it leans on, each as `path/FILE:LINE ITEM`
taken from `RESULTS-INDEX.md`. `cited` checks that each one resolves, that the item is
anchored in the cited file, that the line is the item's own material, and whether the
citing file actually reads what it cites.

## 6. The runtime referee

`discipline/referee.py` is imported by the computation file and owns every line that
announces an outcome.

| call | what it does |
|---|---|
| `declare(name, predicts, kills, requires=(), control=None, budget="1 of 3")` | records the declaration and prints it. Must precede `verdict(name, …)`. A declaration after its own verdict is an error. |
| `verdict(name, ok, detail="")` | prints `name HOLDS.` or `name FAILS.` according to `bool(ok)`. The call site is parsed: a literal `ok` (`True`, `1`, `"yes"`) is **refused** and the run will not count. A verdict with no declaration is refused. On FAILS the prediction and kill condition are reprinted. |
| `rows(name, violating, show=8)` | prints the violating rows behind a count (the first `show`). `close()` lists every verdict that had no `rows()` call. |
| `control(name, ok, detail="", vacuous=None, rows=None)` | a negative control: a case that **must fail**. `ok == True` is recorded as a failure of the file. `vacuous` states how many of the control's rows agree with the claim by construction and are therefore silent rather than failing; omitting it is printed as `NOT MEASURED`. A literal `ok` is refused. |
| `withdraw(name, text)` | prints a withdrawal. Calling it after `verdict(name, …)` has printed is an error: withdrawals go above results. |
| `close()` | prints the ledger of declarations against outcomes. Exits with status 1 when any verdict lacked a declaration or was a literal, when no control ran, when a control passed, or when a withdrawal followed its result. Otherwise prints `The run counts.` and returns 0. |

Both the bare form `verdict(...)` and the attribute form `R.verdict(...)` are parsed for
the literal check, including multi-line calls.

## 7. The file-level checks

`sh check.sh FILE.py` runs these ten on the file; each is also runnable alone as
`python3 -m discipline.<name> FILE.py`. All of them print rows in the form
`  kind  file:line  detail` and return the rows.

| check | rule | row kinds |
|---|---|---|
| `predeclare` | a file that prints a verdict word (HOLDS, FAILS, CONFIRMED, REFUTED, SURVIVES, RESOLVES …) carries a declaration marker in its docstring or in a `print` before the first loop or function | `undeclared-verdict` |
| `rowsplit` | a `print` whose text reads `N of M`, `0 of`, `violations`, `failures` must have a printed row or witness within a window of six lines. Also provides `killcheck(name, rows, violates, key=…)`, an API that prints the violating rows and refuses to report "0 violations" on 0 examined rows (it prints VACUOUS) | `count-without-rows` |
| `guard` | every slice of a name-bound constant list or tuple by a variable (`XS[:k]`) and every constant sequence indexed by a loop variable is a row: a domain narrower than the signature. Also provides `@domain(pred, msg)` and `guard(cond, msg)`, an assertion that survives `python -O` | `unguarded-slice`, `unguarded-index` |
| `withdraw` | the first WITHDRAWN / RETRACTED / WITHDRAWAL must occur before the first result-section header; every file named within three lines of a withdrawal must itself carry the notice | `withdrawal-below-result`, `unpropagated` |
| `noliteral` | every f-string passed to `print()` is walked; a formatted field whose expression is a constant is a row, and a printed sentence carrying a strong verdict word with no formatted field at all is a `constant-verdict`. Column headers (a plain alignment spec) and constants labelled `(const)` are skipped | `literal-field`, `constant-verdict` |
| `names` | given `registers/NAMES.md`, prints every definitional line for each registered word and groups them by sense; a word with more than one sense-group is a row. Without a registry it says so and returns no rows | `multi-sense` |
| `forced` | the static half of the referee. **A** a verdict/control with a constant second argument; **B** a constant container of two or more entries on one side of a comparison, whose value is not named in the file's own `declare()` text; **C** a verdict argument whose free names all trace back through assignments to constants; **D** a comparison against an undeclared numeric literal in a test that decides a claim (0, 1 and 2 are structural and exempt; a declared fraction declares the float it equals); **E** a `control()` with no `vacuous=` | `forced-A` … `forced-E` |
| `cited` | every entry of a `requires=[...]` list is resolved (§10). A cited `.py` that the citing file neither imports, reads at runtime, nor copies a construction from is `not-load-bearing`; a cited `.md` or `.out` can never be load-bearing by that test and is rowed as `prose-citation` for a human to judge | `no-such-file`, `bare-file`, `item-not-anchored`, `line-stale` (with its repair target), `label-not-at-line`, `label-not-in-file`, `not-load-bearing`, `prose-citation` |
| `ground` | in every file that calls `declare()`: any import that is neither the standard library nor `referee` is a row (a numerical library is allowed only where a registered primitive the file carries declares it); a runtime read of a project file, or a read or `exec` of another file's source, is a row; and a missing `# MIRRORS` line or docstring section is a row | `corpus-import`, `third-party-import`, `runtime-read`, `no-mirrors-block` |
| `primitive` | a function whose name is registered in `registers/PRIMITIVES-REGISTER.md` and whose normalised source hash (AST with docstrings blanked) differs from the canonical one is `drift`; two or more files carrying an identical unregistered function is `unregistered`; a registered primitive no file carries is `dead-primitive`; a list of carrying files that does not match the disk is `stale-register`. Names listed with a reason in `registers/INDEPENDENT.md` are exempt from drift | `drift`, `unregistered`, `dead-primitive`, `stale-register` |

`rowsplit` and `guard` are heuristics: they produce rows for a human, not a verdict. The
others are exact on what they check and say in their docstrings what they do not.

## 8. The corpus-level checks

Two checks need the roots where the numbering and the working documents live, not a
single file. `run.py` passes them the `--corpus` roots that `check.sh` supplies (the space
plus `DISCIPLINE_ROOTS`); invoked by hand they take the roots as arguments.

| check | rule | options |
|---|---|---|
| `ledger` | corrections are numbered `#N` in the docstring where the error happened, naming the finder. The consolidated ledger is regenerated by walking the files; a number below the highest that no file carries is a **gap**. Over a single file it would call every number below that file's highest a gap, so it is never handed a bare file | `--floor=N` ignore gaps below N; `--baseline=2,3` known-absent numbers, held so a new gap is visible (`LEDGER_BASELINE` in `check.sh`); `--max=N` |
| `rollover` | every `.md` in the space and its immediate subdirectories that is not named in `registers/FROZEN.md` must be under the cap (120 KB); every document that is named there must still have the line count it was frozen at | none; the cap lives in the module docstring |

## 9. The baseline

A check whose backlog is not zero cannot signal a new violation: a row returned tomorrow
cannot be told from the rows returned today. `baseline.py` records the standing rows as
identities — `check | kind | file basename | line` — in `registers/BASELINE.md`, and
`run.py` splits every file-level check's output into HELD and NEW. The new rows are printed
by name, and **the exit code is the NEW count**. Held rows are still printed under their
check and still appear in `discipline-report.md`; nothing is cleared.

```sh
python3 -m discipline.baseline            # how many identities are on record
python3 -m discipline.baseline --write    # regenerate, over every file matching DISCIPLINE_GLOB in the space
```

Regeneration is deliberate and by hand, at a reassessment point, after any backlog work
and never before (`PROTOCOL.md` §9). The identity is tight on purpose: a row that moves to
a different line is NEW, because a file whose lines moved is a file somebody just touched.

## 10. The results index and the citation address

`sh reindex.sh` walks the space (and `DISCIPLINE_ROOTS`) and writes `RESULTS-INDEX.md`, one
row per item: every theorem, axiom or claim heading in a `.md` (a bolded item name at the
start of a line or table row), every `verdict(...)` and printed section head in a `.py`,
each file's docstring first line, every numbered correction `#N`, every project-log entry,
and every `theorem` / `lemma` / `def` / `axiom` declaration in a `.lean` or `.v` file — each
with `file:line`. Paths are written relative to the part that holds them, under the part's
basename, so a citation reads the same wherever the index is generated. The same command
then checks that every row resolves to a real file and a line inside it, prints the rows
that do not, and exits 1 if any exist. Finally it records a snapshot of every indexed file
(size, mtime, MD5) in `registers/INDEX-MANIFEST.tsv` for `quickcheck.sh`.

A citation is written `path/FILE:LINE ITEM`. The rule `cited` enforces (`PROTOCOL.md`
§10): **the name is the address; the line is advisory.** The cited file must anchor the
item (a `declare("X"`, `verdict("X"` or `print("X.` in a `.py`; a `**X — ` header or a
`**X**` table cell in a `.md`; a `[X]` marker or `X.` head in an `.out`), and the line must
fall inside that item's own material — the smallest enclosing statement that names it in a
`.py`, the territory from its anchor to the next anchor in prose. A stale line is reported
with the line it should be, so the citation is repairable without opening the cited file.

## 11. The registers

Every register is plain text under `registers/` (or `DISCIPLINE_REGISTERS`), shipped empty
with its format stated in its header.

| register | written by | read by | format |
|---|---|---|---|
| `NAMES.md` | hand, the moment a load-bearing word is first used | `names`, `log` | `word \| canonical definition \| file` |
| `BASELINE.md` | `baseline.py --write` | `run.py` | `check \| kind \| file \| line` |
| `FROZEN.md` | hand, at each rollover | `rollover` | `\| document \| lines when frozen \| bytes when frozen \| successor \|` |
| `PRIMITIVES-REGISTER.md` | `primitive.py --rebuild` | `primitive`, `ground` | one section per primitive: hash, library, the files carrying it |
| `INDEPENDENT.md` | hand | `primitive` | `name \| reason, cited file:line` — a blank reason is not an exemption |
| `PROJECT-LOG.md` | `log.py add` | `index.py`, `log.py` | append-only entries `## L-0001 · timestamp · disposition` |
| `INDEX-MANIFEST.tsv` | `reindex.sh` (via `storage.py snapshot`) | `quickcheck.sh` | tab-separated: part, path, bytes, mtime_ns, md5 |
| `MIRROR-MANIFEST.tsv` | `mirror.sh` | — | tab-separated: canonical, mirror, md5, bytes |
| `storage.cfg` | hand | `storage.py` | `key = value`; see §13 |

The canonical text of each primitive lives in `primitives/<name>.py` under the space;
`primitive.py --rebuild` reads the copies on disk, takes the majority hash as canonical,
and writes the register. Keying on the majority rather than on `(name, hash)` is what makes
two files carrying different versions of one function into a drift row rather than two
silent primitives.

## 12. The project log

`LOG-PROTOCOL.md` describes the flag. In brief: a block opened by `[[LOG]]` (or
`[[LOG connect]]`, `[[LOG draft: <target>]]`) and closed by `[[/LOG]]`, with `what:` and
`connects:` fields, is appended verbatim to `registers/PROJECT-LOG.md` with an id and a UTC
timestamp, and each item in `connects` is resolved against `RESULTS-INDEX.md` and
`NAMES.md` — never against memory. An item that resolves nowhere is written UNRESOLVED,
which is a finding about the registry or the index, not a correction of the author.

```sh
python3 -m discipline.log add MESSAGE.txt        # or:  add -   to read the message from stdin
python3 -m discipline.log pending                # held entries, grouped by target
python3 -m discipline.log triggers --threshold 3 # targets with >= N held entries
python3 -m discipline.log resolve L-0007 --status "connected -> DEFINITIONS.md D3"
```

Entries are indexed by `index.py` (`L-0001 | log (connect) | …`), so a search of the index
returns the author's words next to the theorems.

## 13. Storage: one canonical space

`storage.py` records what the last full reindex saw and keeps one mirror exact. The rule
(`PROTOCOL.md` §12) is that one space is canonical and cited, every other copy is a mirror,
and a mirror is never cited or indexed.

`registers/storage.cfg`, `key = value`, one per line:

```
part      = NAME=PATH     a root the index reads, under NAME (repeatable; default: the space itself)
canonical = GLOB          final files to mirror, relative to the space; a .out beside each goes with it (repeatable)
mirror    = PATH          the mirror directory, relative to the space; no mirror if unset
exclude   = PREFIX        a path prefix, relative to its part, the index does not read (repeatable)
```

```sh
sh reindex.sh                          # full reindex, resolution check, then a snapshot of every indexed file
sh quickcheck.sh                       # exit 0: index current and mirror exact; 2: full reindex due; 3: mirror drifted
sh mirror.sh                           # copy canonical files whose mirror is missing or differs; never deletes
python3 -m discipline.storage snapshot # the snapshot alone
```

`quick` compares size and mtime first and reads an MD5 only where the stat differs, so it
is fast enough to run after every exchange.

## 14. Configuration

Nothing in the suite hard-codes where a project is. `discipline/config.py` answers every
"where?" question from three sources, in order:

1. **Environment.**

   | variable | meaning | default |
   |---|---|---|
   | `DISCIPLINE_SPACE` | the working space: the directory `rollover` watches, `baseline --write` records, and `reindex.sh` indexes | nearest ancestor holding `.discipline`, else the file's directory |
   | `DISCIPLINE_ROOTS` | further project roots, `os.pathsep`-separated, that a citation may resolve into and that `reindex.sh` indexes as parts | none |
   | `DISCIPLINE_REGISTERS` | the directory holding the registers | `$SPACE/registers` |
   | `DISCIPLINE_GLOB` | the pattern naming a computation file | `*.py` |
   | `DISCIPLINE_AUTHOR` | the label the project log writes above a verbatim entry | `Author` |
   | `DISCIPLINE_CITE_PREFIX` | a prefix citation paths are written relative to | none |
   | `LEDGER_BASELINE` | passed through by `check.sh` to `ledger`, e.g. `--baseline=2,3` | none |

2. **The marker.** A file named `.discipline` in the working space; the nearest one above
   the file being checked wins, and its directory is the space.

3. **The file's own directory**, when neither of the above applies.

`config.SKIP` lists the directory names no check walks (`.git`, `__pycache__`, `.lake`,
`build`, `node_modules`, `.venv`, `venv`, `primitives`, `discipline`, …); add to it there,
in one place. `config.INDEXED` lists the suffixes the index and the citation check read
(`.md`, `.py`, `.out`, `.lean`, `.v`).

The shell scripts export `DISCIPLINE_SPACE` (the directory holding the script unless
already set) and translate `DISCIPLINE_ROOTS` into `--corpus` arguments, so
`sh check.sh FILE.py` from anywhere behaves as the protocol expects.

## 15. Exit codes

| command | exit status |
|---|---|
| `python3 FILE.py` (with `referee.close()`) | 0 when the run counts; 1 when any verdict lacked a declaration or was a literal, no control ran, a control passed, or a withdrawal followed its result |
| `sh check.sh FILE.py` / `python3 -m discipline.run` | 0 when there are no NEW file-level rows (or no rows at all, with no baseline on record); 1 otherwise. The full output of every check is in `discipline-report.md` |
| `python3 -m discipline.<check>` | 0 with no rows; 1 with rows (`names` without a registry: 0) |
| `sh reindex.sh` | 0 when every index row resolves; 1 with the unresolved rows printed |
| `sh quickcheck.sh` | 0 current; 2 a full reindex is due; 3 the mirror has drifted |
| `sh mirror.sh` | 0 |
| `python3 -m discipline.<check> --self` | 0 when the control behaves; an assertion otherwise |

## 16. The controls: testing the checks

A check that cannot fail is not a check, and that applies to the checks themselves. Four
modules carry a built-in negative control, run with `--self`:

| module | what the control plants | what must happen |
|---|---|---|
| `forced --self` | a declared threshold (`ratio = 1/2`, compared as `0.5`) and an undeclared one (`0.37`) | the declared one does not row; the undeclared one does |
| `ground --self` | a self-contained file with a MIRRORS block, and an importing file without one | the first does not row; the second rows twice |
| `primitive --self` | two identical copies of a function, then one changed to `x*2`, then an identical pair the register does not hold | both register and pass; the change rows as drift; the pair rows as unregistered |
| `cited --self` | a citation the file imports, and a citation of a file it never touches | the imported one does not row; the untouched one does |

`example/broken.py` is the control for the runtime referee and for `check.sh` as a whole:
a typed verdict with no negative control. Its refusal is `example/broken.out`, and the two
static rows it earns (`forced-A`, `no-mirrors-block`) are in `example/walkthrough.out`. If
a change to the suite lets `broken.py` count, the suite is broken.

## 17. Limits and non-goals

- **The suite checks Python computation files.** `.lean` and `.v` files are indexed (their
  declarations become citable items) but not checked; the protocol's proof-engine stage
  (`PROTOCOL.md` §19–20) is enforced by the proof assistant, not by this suite.
- **An identity computed rather than typed is invisible to every static check.** A file
  that computes `x == x` has a computed verdict argument. The defence is the negative
  control: a case that must fail, whose vacuity is stated. The referee enforces that one
  exists and fails; it cannot judge whether it was the right one. That judgement is the
  human's and the outside check's.
- **`rowsplit` and `guard` are heuristics.** They print rows for a human to read, and the
  baseline exists so that their standing rows do not drown a new one.
- **A prose citation is never load-bearing by the mechanical test.** A cited `.md` or `.out`
  cannot be imported or copied from, so `cited` rows it as `prose-citation` and a human
  applies the deletion test.
- **The checks read source, not behaviour.** A verdict argument that is computed from data
  the file fetched at runtime from outside the project passes `forced`; `ground` rows the
  fetch.
- **The suite does not evaluate ideas**, rank claims, or decide what is worth testing. It
  decides whether a run counts.
- **Nothing here needs the model, and nothing here runs the model.** The checks are the same
  for every model and for a human author.

## 18. Adoption: hooks, CI, agents

The checks are file-level and language-level, so they scale by being placed where files are
already gated.

**A pre-commit hook.**

```sh
#!/bin/sh
# .git/hooks/pre-commit
for f in $(git diff --cached --name-only --diff-filter=ACM | grep '\.py$'); do
  sh check.sh "$f" || exit 1
done
sh reindex.sh >/dev/null || exit 1
```

**A CI step.** Run `sh check.sh` over every computation file, `sh reindex.sh`, and
`python3 -m discipline.ledger --floor=N .`; fail the job on any non-zero status. Commit
`registers/BASELINE.md` so that the job exits on NEW rows only, and regenerate it in a
reviewed commit at each reassessment point.

**An agent loop.** After every exchange in which a model writes a computation file, the
harness runs `python3 FILE.py > FILE.out` and `sh check.sh FILE.py` and shows the model the
rows; `sh quickcheck.sh` decides whether the index needs rebuilding before the next
requires-list. The model's obligation to the machinery is exactly `sh check.sh FILE.py`;
the protocol forbids it from reasoning about the checks in place of satisfying them.

A team that adopts the suite adopts one `NAMES.md` per project, one ledger, one canonical
space, and one rule: a computation with no transcript has no result.

## 19. Licence and citation

The licence under which this software is released is to be chosen by the author before
publication; no licence file is included in this snapshot, and the software is not
licensed for redistribution until one is. `CITATION.cff` carries the citation metadata
with the fields the author has not yet fixed marked as placeholders.
