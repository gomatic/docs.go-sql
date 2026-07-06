---
title: go-sql
---
**A thin, allocation-light Go layer over [pg_query](https://github.com/pganalyze/pg_query_go) for parsing PostgreSQL into its syntax tree, canonicalizing it, and deparsing it back to SQL — plus subpackages for formatting, AST-level diffing, and text normalization.** Every operation is a real PostgreSQL parse, so it understands the grammar rather than guessing at it, and every failure comes back as a matchable sentinel error.

- **Source:** [gomatic/go-sql](https://github.com/gomatic/go-sql)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-sql](https://pkg.go.dev/github.com/gomatic/go-sql)

## Install

```
go get github.com/gomatic/go-sql
```

## Usage

The root package works on `sql.SQL` (raw SQL text) and returns errors wrapped in sentinels you match with `errors.Is`.

```go
package main

import (
	"errors"
	"fmt"

	sql "github.com/gomatic/go-sql"
)

func main() {
	src := sql.SQL("SELECT B, a FROM t WHERE X = 1;")

	// Parse to PostgreSQL's AST and deparse back to canonical SQL.
	tree, err := sql.Parse(src)
	if err != nil {
		if errors.Is(err, sql.ErrParse) {
			// handle a parse failure
		}
		return
	}

	// Canonically reorder simple SELECT / INSERT … SELECT column lists in place.
	sql.SortColumnLists(tree)

	out, err := sql.Deparse(tree)
	if err != nil {
		return
	}
	fmt.Println(out) // SELECT a, b FROM t WHERE x = 1

	// Lowercase only keywords, leaving literals and identifiers untouched.
	lowered, _ := sql.LowerKeywords(src)
	fmt.Println(lowered) // select B, a from t where X = 1;

	// Structural fingerprint: same fingerprint ⇒ same statement modulo
	// formatting, literal values, and case.
	fp, _ := sql.Fingerprint(src)
	fmt.Println(fp)

	// Every comment, line and block, in source order.
	comments, _ := sql.Comments(sql.SQL("-- note\nSELECT 1 /* inline */;"))
	fmt.Println(comments) // [-- note /* inline */]
}
```

The root package's exported surface:

- `Parse` / `Deparse` — SQL text ⇄ `*pg_query.ParseResult` AST (`ErrParse`, `ErrDeparse`).
- `Scan` — tokenize without requiring a well-formed statement (`ErrScan`).
- `LowerKeywords` — lowercase keywords only, leaving literals, quoted identifiers, and dollar-quoted bodies verbatim.
- `Comments` — extract every line (`-- …`) and block (`/* … */`) comment in source order.
- `Fingerprint` — PostgreSQL's structural fingerprint (`ErrFingerprint`).
- `SortColumnLists` — reorder column lists in place for canonical comparison.
- `ToJSON` — one normalized JSON message per statement, with keys sorted and zero values dropped so equivalent statements render identically (`ErrMarshal`).

## Subpackages

- `formatter` — renders statements as canonically-styled SQL through a verification gate: a statement it can't prove it reformatted faithfully (same meaning, same comments) is emitted verbatim, so it never changes behavior or drops a comment. `formatter.New().Format(query)`.
- `compare` — reports what changed between two SQL scripts at the AST level. It pairs statements by a type-specific identity (a `CREATE TABLE` by its qualified name, a `GRANT` by object and grantee, and so on) and returns the added, removed, and changed statements. `compare.Compare(source, target)` yielding a `Result`.
- `normalize/sql` (package `sqlnorm`) — canonicalizes SQL text for meaning-based comparison via `SQL.Normalize`, `SQL.NormalizeRoutine` (also sorts column lists), and `SQL.NormalizeStrict`; falls back to whitespace collapsing when the input won't parse.
- `normalize/plpgsql` — canonicalizes PL/pgSQL: strips comments, keeps quoted and dollar-quoted literals verbatim, and normalizes whitespace and operator spacing. `Body.Normalize`, with no error path.

## Design

Errors are declared as [gomatic/go-error](https://github.com/gomatic/go-error) `errs.Const` sentinels (`ErrParse`, `ErrDeparse`, `ErrScan`, `ErrFingerprint`, `ErrMarshal`) and wrapped with `.With(cause)`, so callers match them with `errors.Is` rather than by string. `SortColumnLists` mutates its tree in place and is therefore not safe to run concurrently on the same `*pg_query.ParseResult` — serialize the calls or work on separate trees. `ToJSON` and the `formatter`/`compare` conversion seams take their marshal/deparse functions as injected parameters, keeping the failure paths reachable from tests without package-level globals.
