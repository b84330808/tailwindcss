# tailwindcss: custom property at the end of the input loses its last character

## Branch

- Fork: `b84330808/tailwindcss`
- Branch: `claude/exciting-allen-k462k1`
- SHA: `64a9bd6bde4c4602c4ab1e34f49d42c3cf74cf6c`
- Base: `upstream/main` at `8cc0551` ("Add changelog entry for #20591 (#20594)")
- One commit, by `b84330808 <b84330808@gmail.com>` with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`
- Files: `packages/tailwindcss/src/css-parser.ts` (+16 / -4), `packages/tailwindcss/src/css-parser.test.ts` (+25)
- `CHANGELOG.md` was not changed. The suggested line is below.

## PR title

```
Parse a custom property at the end of the input without a trailing `;`
```

## PR body

````markdown
## Summary

When a custom property is the last thing in the CSS input and has no `;` or `}` after it, the CSS parser loses part of it:

- `--foo: bar` is parsed as `--foo: ba`. When the scan reaches the last character, it takes `input.slice(start, j)`, which leaves out the character at `j`.
- `--foo: bar !important` is parsed as the non-important `--foo: bar !importan`.
- If the last character is consumed by another branch instead (a closing `)`, a string, a comment or an escape), the loop ends without filling the buffer and the parser emits an empty `: ;` declaration. This happens for `--foo: var(--bar)` or `--foo: 'bar'`, for example.

The existing `should parse a custom property with a missing ending ;` test only passes because its input ends with a newline and indentation.

This can reach the build output through `@import`, when a file of custom properties without a trailing newline is imported inside a rule:

```css
/* input.css */
:root {
  @import './vars.css';
}
```

```css
/* vars.css, no trailing newline */
--brand: red;
--accent: blue
```

This builds to `--accent: blu;`. With `--accent: var(--brand)` it builds to `: ;`.

This PR drops the `input.length - 1 === j` check from the loop. When the loop reaches the end of the input without a `;` or `}` ending the custom property, the rest of the input is used as its value. This matches how PostCSS parses the same input:

| Input | Before | After (same as PostCSS) |
| --- | --- | --- |
| `--foo: bar` | `--foo: ba` | `--foo: bar` |
| `--foo: bar !important` | `--foo: bar !importan` | `--foo: bar !important` |
| `--foo: var(--x)` | `: ;` | `--foo: var(--x)` |
| `--foo: 'bar'` | `: ;` | `--foo: 'bar'` |
| `--a: 1; --b: 2` | `--a: 1; --b: ;` | `--a: 1; --b: 2` |

## Test plan

- Added a test to `css-parser.test.ts` for a custom property at the end of the input with a plain value, `!important`, `var(…)`, a string, a block, a comment, and after another declaration. Like the other tests in that file, it runs with both Unix and Windows line endings.
- With the fix reverted, the new test fails (`"value": "ba"` instead of `"bar"`). With the fix, it passes.
- `pnpm exec vitest run` in `packages/tailwindcss`: 44 files, 5164 tests passed, 1 skipped.
- `pnpm exec vitest run --project @tailwindcss/node` (it uses this parser to rewrite `url(…)`s in imported files): 8 passed.
- Compared the parsed declarations with `postcss.parse` for the inputs in the table above. They now match.
- `pnpm exec prettier --check` on the changed files passes.

I used Claude Code to write this change.
````

## Suggested CHANGELOG line

Add it at the end of `## [Unreleased]` → `### Fixed` once the PR number is known:

```markdown
- Don't drop the end of a custom property at the end of a CSS file without a trailing `;` (e.g. `--foo: bar` no longer becomes `--foo: ba`) ([#XXXXX](https://github.com/tailwindlabs/tailwindcss/pull/XXXXX))
```

## Fail before (fix reverted, new test kept)

`packages/tailwindcss/src/css-parser.ts` reset to `upstream/main`, then
`npx vitest run src/css-parser.test.ts -t "at the end of the input"` in `packages/tailwindcss`:

```
 RUN  v4.1.10 /home/user/tailwindcss/packages/tailwindcss

 ❯ src/css-parser.test.ts (168 tests | 2 failed | 166 skipped) 28ms
         × should parse a custom property with a missing ending `;` at the end of the input 21ms
         × should parse a custom property with a missing ending `;` at the end of the input 2ms

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 2 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/css-parser.test.ts > Line endings: Unix > declarations > Custom properties > should parse a custom property with a missing ending `;` at the end of the input
 FAIL  src/css-parser.test.ts > Line endings: Windows > declarations > Custom properties > should parse a custom property with a missing ending `;` at the end of the input
AssertionError: expected [ { kind: 'declaration', …(3) } ] to deeply equal [ { kind: 'declaration', …(3) } ]

- Expected
+ Received

  [
    {
      "important": false,
      "kind": "declaration",
      "property": "--foo",
-     "value": "bar",
+     "value": "ba",
    },
  ]

 ❯ src/css-parser.test.ts:330:37
    328|
    329|       it('should parse a custom property with a missing ending `;` at …
    330|         expect(parse('--foo: bar')).toEqual([
       |                                     ^
    331|           { kind: 'declaration', property: '--foo', value: 'bar', impo…
    332|         ])

 Test Files  1 failed (1)
      Tests  2 failed | 166 skipped (168)
```

The test stops at its first assertion. Here is every case, parsed with the old parser, the fixed parser and `postcss.parse`:

| Input | Before | After | PostCSS |
| --- | --- | --- | --- |
| `"--foo: bar"` | `--foo="ba"` | `--foo="bar"` | `--foo="bar"` |
| `"--foo: bar !important"` | `--foo="bar !importan"` | `--foo="bar" !important` | `--foo="bar" !important` |
| `"--foo: var(--x)"` | `=""` | `--foo="var(--x)"` | `--foo="var(--x)"` |
| `"--foo: \"bar\""` | `=""` | `--foo="\"bar\""` | `--foo="\"bar\""` |
| `"--foo: {a:b}"` | `="" a="b"` | `--foo="{a:b}"` | `--foo="{a:b}"` |
| `"--a: 1; --b: 2"` | `--a="1" --b=""` | `--a="1" --b="2"` | `--a="1" --b="2"` |
| `"--a: 1;\r\n--b: rgb(0 0 0)"` | `--a="1" =""` | `--a="1" --b="rgb(0 0 0)"` | `--a="1" --b="rgb(0 0 0)"` |

End to end, through `compile()` with `:root { @import './vars.css'; }` and a `vars.css` without a trailing newline:

| `vars.css` | Before | After |
| --- | --- | --- |
| `--brand: red;\n--accent: blue` | `:root { --brand: red; --accent: blu; }` | `:root { --brand: red; --accent: blue; }` |
| `--brand: red;\n--accent: var(--brand)` | `:root { --brand: red; : ; }` | `:root { --brand: red; --accent: var(--brand); }` |

## Pass after

Same command, with the fix:

```
 RUN  v4.1.10 /home/user/tailwindcss/packages/tailwindcss

 Test Files  1 passed (1)
      Tests  168 passed (168)
```

The whole `packages/tailwindcss` suite at `64a9bd6` (`pnpm exec vitest run`):

```
 Test Files  44 passed (44)
      Tests  5164 passed | 1 skipped (5165)
```

Other checks:

- `pnpm exec vitest run --project @tailwindcss/node`: 3 files, 8 tests passed.
- `pnpm exec prettier --check packages/tailwindcss/src/css-parser.ts packages/tailwindcss/src/css-parser.test.ts`: "All matched files use Prettier code style!"
- `tsc --noEmit -p packages/tailwindcss` reports 39 errors both before and after the change. All of them are pre-existing, mostly `Cannot find module '@tailwindcss/oxide'`, because the Rust package isn't built in this container. None are in `css-parser.ts`.

## Duplicate search (tailwindlabs/tailwindcss, all states)

PR searches (open, closed and merged):

| Query | Results |
| --- | --- |
| `custom property semicolon end of input` | [#17775](https://github.com/tailwindlabs/tailwindcss/pull/17775) "Add support for source maps" (closed), unrelated |
| `css-parser custom property last character` | [#20630](https://github.com/tailwindlabs/tailwindcss/pull/20630) (ours, lone CR/FF), [#20508](https://github.com/tailwindlabs/tailwindcss/pull/20508) (`\` in comments), [#20089](https://github.com/tailwindlabs/tailwindcss/pull/20089), [#19282](https://github.com/tailwindlabs/tailwindcss/pull/19282) (parse error locations), all unrelated |
| `missing semicolon custom property` | Two dependency bumps, unrelated |
| `parser "end of the input"` | [#19282](https://github.com/tailwindlabs/tailwindcss/pull/19282), [#16306](https://github.com/tailwindlabs/tailwindcss/pull/16306), unrelated |
| `trailing semicolon parse declaration` | No results |
| `in:title custom property parse` | [#18251](https://github.com/tailwindlabs/tailwindcss/pull/18251) "CSS Parser: Handle string with semi-colon in custom properties" (closed). That one is about `;` inside strings, a different bug. |

Issue searches (all states):

| Query | Results |
| --- | --- |
| custom property without trailing semicolon at end of file loses last character | [#19200](https://github.com/tailwindlabs/tailwindcss/issues/19200), [#18523](https://github.com/tailwindlabs/tailwindcss/issues/18523), [#16983](https://github.com/tailwindlabs/tailwindcss/issues/16983), [#16188](https://github.com/tailwindlabs/tailwindcss/issues/16188) and older ones, all closed and unrelated |
| imported css file variables truncated last value missing semicolon custom property cut off | 87 hits. The top 15 are all closed and unrelated (theme prefixing, `var()` fallbacks, minification). |
| CSS parser drops last character of custom property when file has no trailing newline | [#19531](https://github.com/tailwindlabs/tailwindcss/issues/19531), [#16660](https://github.com/tailwindlabs/tailwindcss/issues/16660) and older ones, all closed and unrelated |

No existing issue or PR covers this bug.

## Other checks

- **AI policy / CLA:** checked again on `upstream/main` at `8cc0551`. `.github/CONTRIBUTING.md`, the PR template, the workflows (`ci.yml`, `integration-tests.yml`, `prepare-release.yml`, `release.yml`) and the root files have no AI policy and no CLA requirement. The PR template asks for `## Summary` and `## Test plan`, and CONTRIBUTING requires tests and a test plan.
- **Overlap with our open PRs:** [#20627](https://github.com/tailwindlabs/tailwindcss/pull/20627) (`fix-important-in-strings`) only changes `parseDeclaration()` (upstream lines 635–640), and [#20630](https://github.com/tailwindlabs/tailwindcss/pull/20630) (`fix-lone-cr-line-endings`) adds a constant after line 23 and a block after line 96. This change touches upstream lines 273–276 and adds a block after line 296. `git merge-tree` of this branch with each of them is clean, including `css-parser.test.ts`.
- **Possible follow-up, not included:** a regular (non-custom) declaration at the top level at the end of the input without `;` is dropped silently: `parse('color: red')` returns `[]`, because the leftover buffer after the loop is only handled when it starts with `@`. It isn't included here because a leftover top-level buffer can also be a selector without a block. Turning it into a declaration or an error changes behavior more than this fix does.
