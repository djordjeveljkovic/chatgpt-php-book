# AI Summary — Chapter 41 — Lexer

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 41 explains the lexer as the stateful conversion from source characters to tokens and source positions. It covers mixed HTML/PHP mode, strings and interpolation, heredoc/nowdoc, comments and whitespace, single-character versus named tokens, lexer/parser boundaries, tokenizer APIs, version-sensitive token IDs, performance, security, testing, exercises, and review questions.

## Concepts already explained

Lexer/scanner, token, token text, source position, scanner state, PHP/literal mode, parser boundary, tokenizer, `T_*` token IDs.

## Terminology established

`token_get_all()` token dumper; variable-name token tool; mixed-mode source; string interpolation and heredoc examples; token-tool testing matrix.

## Examples used

None.

## Cross-references

Cross-references Chapters 40–43 for the complete pipeline, parser/AST boundaries, and later compiler behavior. References include the PHP Tokenizer manual, token list, and php-src scanner files.

## Open threads

No open chapter-writing thread remains; parser grammar is developed in Chapter 42.

## Exact next section

None — chapter complete.

## Technical verification notes

Tokenizer behavior is based on the PHP Manual for `token_get_all()`, Tokenizer, and parser tokens; scanner implementation references point to current php-src `Zend/zend_language_scanner.l/.h`. Token numeric values are explicitly treated as version-sensitive.

## Writing notes

Keep this summary short and preserve the distinction between public tokenizer output and private parser/scanner internals.
