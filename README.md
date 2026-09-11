# kotoba-lang/org-ietf-jcs

**[RFC 8785](https://www.rfc-editor.org/rfc/rfc8785.html) JSON Canonicalization
Scheme, portable `.cljc`.** Turns a Clojure value into the one byte sequence
RFC 8785 defines for it, so two parties holding the same logical document produce
the same bytes — and therefore the same signature.

This is the transformation step of the `eddsa-jcs-2022` cryptosuite
([W3C vc-di-eddsa](https://www.w3.org/TR/vc-di-eddsa/)). Nothing here signs;
see `kotoba-lang/org-w3-vc-data-integrity` for that.

```clojure
(require '[jcs.core :as jcs])

(jcs/canonicalize {:b 1 :a 2})              ;=> "{\"a\":2,\"b\":1}"
(jcs/canonicalize {"n" 1e21})               ;=> "{\"n\":1e+21}"
(jcs/canonicalize-bytes {"k" "é"})          ;=> UTF-8 bytes, ready to hash
```

## The four rules

1. **Object members sort by property name, compared as UTF-16 code units**
   treated as unsigned integers (§3.2.3). Both hosts' native string compare is
   exactly this, so `sort-by` on the name is the rule rather than an
   approximation of it.
2. **No whitespace** (§3.2.1).
3. **Strings escape only `"` and `\`, plus C0 controls** — U+0008/09/0A/0C/0D as
   `\b \t \n \f \r`, every other control as *lowercase* `\uhhhh` (§3.2.2.2).
   Everything else is literal UTF-8, including `/`, U+007F, and all non-ASCII.
4. **Numbers serialize as ECMAScript `Number::toString`** (§3.2.2.3 → ECMA-262
   7.1.12.1). NaN and Infinity terminate with an error.

## Why the JVM branch does not use `Double/toString`

`Double/toString` is specified to produce *a* representation that round-trips,
not the *shortest* one. For `Double/MIN_VALUE` it yields `4.9E-324` where
ECMAScript — and RFC 8785's own Appendix B sample — require `5e-324`. Relying on
it would make the canonical form host-dependent, which defeats the point.

So `:clj` derives the digits directly: `BigDecimal.` of a double is its *exact*
binary value, and rounding that to `p` significant digits and checking the result
still parses back to the same double finds the true shortest `p`. `HALF_UP` on a
positive magnitude implements ECMA-262's tie rule ("choose the larger `s`").

`:cljs` delegates to `Number.prototype.toString`, because in JavaScript that
**is** the algorithm the RFC cites.

Both hosts are pinned to the same vectors (`test/jcs/core_test.cljk` and
`test/nbb_smoke.cljk`) and produce byte-identical output for all of them.

## Fail-closed inputs

A canonicalizer that silently coerces is worse than one that refuses, because
the caller is about to sign the result. Each of these throws `ex-info` carrying a
`:jcs/error` key:

| `:jcs/error` | Input |
|---|---|
| `:jcs/non-finite` | NaN, +∞, −∞ (§3.2.2.3 requires termination) |
| `:jcs/integer-out-of-range` | exact integer beyond ±(2^53−1) — App. D says carry it as a JSON string |
| `:jcs/unsupported-number` | `BigDecimal`, `Ratio` — no IEEE-754 double identity |
| `:jcs/invalid-key` | property name that is neither string nor keyword |
| `:jcs/duplicate-key` | distinct map keys collapsing to one JSON name (`:a` and `"a"`) |
| `:jcs/unsupported-type` | sets (no JSON form, no defined order) and anything else |

### One deliberate host asymmetry

`:clj` refuses exact integers beyond ±(2^53−1); `:cljs` does not, because there
is nothing to refuse — JavaScript has a single `number` type, so such a value is
*already* a double by the time it arrives and the precision has been lost
upstream. `Number.isInteger` is not the analogue of Clojure's `integer?`: it asks
"is this double integral", which `1e20` and `1e300` both are, and both are
exactly representable and must serialize.

So a `:clj` caller holding the exact integer 10^20 gets an error while a `:cljs`
caller holding `1e20` gets `100000000000000000000`. `:clj` is the stricter host,
which is the correct direction — App. D says such a value does not belong in a
JSON number at all, and a host able to detect that should say so rather than sign
a silently-rounded value.

## Test

```bash
kbb -M:test                              # JVM
kbb -M:lint
kbb --backend sci --classpath src test/nbb_smoke.cljk      # :cljs branch
```

## License

MIT. See `LICENSE`.
