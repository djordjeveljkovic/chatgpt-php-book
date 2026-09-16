---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 41
title: Lexer
slug: lexer
status: complete
summary: ../../_ai/chapter-summaries/041-lexer-summary.md
---

# Chapter 41 — Lexer

## Why This Matters

The lexer is where a character stream first becomes meaningful to the PHP language machinery. It answers questions such as “is this word a keyword or an identifier?”, “does this quote begin a string?”, and “is this `<` PHP syntax or HTML text?” It does not decide whether a complete expression is legal. That is the parser’s job.

Knowing this boundary makes syntax diagnostics and source tooling less mysterious. A formatter, syntax highlighter, or static analyzer may consume tokens, while the engine’s parser consumes the scanner’s output using internal state and semantic values.

## Mental Model

```text
source characters
    ↓  scanner state and longest useful match
tokens + token text + source position
    ↓
parser grammar
```

A token is not necessarily a word. Punctuation such as `(`, `)`, `+`, and `;` can be returned as single-character tokens. Named tokens include `T_VARIABLE`, `T_STRING`, `T_FUNCTION`, and `T_OPEN_TAG`. A token can carry text and a line number; an operator can be represented simply by its character or token kind.

The lexer is usually described as recognizing regular patterns. PHP makes this less trivial than a toy language because PHP files can switch between literal output and PHP code, strings can interpolate variables, heredoc/nowdoc syntax has special delimiters, and the interpretation of a character can depend on scanner state.

## Core Concept

Tokenization preserves enough information for the parser to work, but it is not a complete parse. For example, the scanner can identify:

```text
T_VARIABLE "$total"
"="
T_LNUMBER "3"
"+"
T_LNUMBER "4"
";"
```

It does not decide that the `+` adds two operands, nor does it know whether the assignment is allowed in the current grammatical position. Those relationships require the parser.

The [PHP token list](https://www.php.net/manual/en/tokens.php) is useful for tooling, but token numeric values are not a stable cross-version protocol. Code should compare named constants or token names, not hard-code the integer value assigned to `T_STRING` in one PHP release.

## How It Works

The core scanner is implemented in the Zend source tree. In the current php-src layout, [Zend/zend_language_scanner.l](https://github.com/php/php-src/blob/master/Zend/zend_language_scanner.l) describes scanner rules that are processed into C code as part of the build. The scanner communicates with the parser through functions and semantic values declared around [Zend/zend_compile.h](https://github.com/php/php-src/blob/master/Zend/zend_compile.h). Exact generated files and internal state names are implementation details.

PHP’s mixed-mode behavior is central:

```php
This is literal output.
<?php
$name = 'Ada';
echo "Hello, $name";
?>
This is literal output too.
```

Conceptually, the scanner enters an HTML/literal mode, switches into PHP mode at the open tag, recognizes PHP tokens, and switches back after the closing tag. Literal text is eventually represented as output behavior by later compilation; the lexer does not send bytes to the browser merely because it saw them.

Strings show why “one character, one token” is too simple:

```php
<?php

$name = 'Ada';
$message = "Hello, $name";
$path = "{$name}/reports";
```

The scanner must recognize quote boundaries and interpolation syntax. It may return string-related tokens and text fragments that let the grammar construct the right expression. The precise token sequence is not a public compatibility contract; use the tokenizer API to inspect the version you are running.

## What PHP Does

The public tokenizer extension exposes a convenient view of lexical scanning. `token_get_all()` returns token identifiers; many entries are three-element arrays containing the token ID, original text, and line number, while single-character tokens are returned as strings. `PhpToken::tokenize()` offers an object representation ([Tokenizer introduction](https://www.php.net/manual/en/book.tokenizer.php)).

```php
<?php

declare(strict_types=1);

$source = <<<'PHP'
<?php
$total = 3 + 4;
PHP;

foreach (token_get_all($source) as $token) {
    if (is_string($token)) {
        printf("%-24s %s\n", "CHAR", json_encode($token));
        continue;
    }

    [$id, $text, $line] = $token;
    printf("%-24s line %d %s\n", token_name($id), $line, json_encode($text));
}
```

The tokenizer intentionally exposes whitespace and comments because source tools often need them. The parser can ignore those tokens where grammar says they are insignificant. A tool that removes comments must still preserve strings, heredocs, line endings, and token boundaries.

## What Zend Does

The Zend scanner is stateful and supplies the parser with token kinds plus semantic values such as strings and numeric values. It also tracks source positions used in diagnostics and later opcode metadata. Scanner internals can change, so do not write an extension or analyzer that assumes a private state enum or generated table layout unless it is pinned to a PHP build.

A lexer error is different from a parser error. If a byte sequence cannot be interpreted as part of a valid lexical construct, scanning may fail before the parser can report a grammatical expectation. In practice, users often see both categories as “syntax errors”; the distinction matters when building diagnostics or investigating encoding and delimiter issues.

## Minimal Example

Token IDs are version-specific, but token names and text are inspectable:

```php
<?php

$tokens = token_get_all('<?php $price = 120; // cents');

foreach ($tokens as $token) {
    echo is_array($token)
        ? token_name($token[0]) . ': ' . json_encode($token[1]) . PHP_EOL
        : 'CHAR: ' . json_encode($token) . PHP_EOL;
}
```

You should expect a variable token, an assignment character, a number token, a semicolon, and a comment token, among other entries. Do not assert a particular integer token ID in a portable test.

## Practical Example: A Safe Token Tool

To find variable names without pretending to understand the whole language:

```php
<?php

declare(strict_types=1);

function variableNames(string $source): array
{
    $names = [];

    foreach (token_get_all($source) as $token) {
        if (is_array($token) && $token[0] === T_VARIABLE) {
            $names[$token[1]] = true;
        }
    }

    return array_keys($names);
}
```

This is useful for a narrow report, but not for determining whether a variable is read, written, in scope, or shadowed. Those are structural or semantic questions and belong to an AST-based analyzer.

## Bad Example

```php
// Fragile: assumes T_STRING has the same integer value forever.
if ($token[0] === 262) {
    // Treat as an identifier.
}
```

The manual explicitly warns that generated token values may change between PHP versions. Compare `T_STRING` by name in code running under PHP, or normalize the token representation in your tool.

## Better Example

```php
if (is_array($token) && $token[0] === T_STRING) {
    $identifier = $token[1];
}
```

For a refactoring tool, also preserve the original text and line information, and test names in namespaces, attributes, interpolated strings, and heredocs. A lexical tool should state its limits instead of presenting a token scan as a semantic analysis.

## Edge Cases

- `<?php` is a mode switch, not merely an ordinary identifier sequence.
- A closing `?>` is optional and changes how following text is treated.
- `<?=` is a short echo tag with language-defined behavior in modern PHP; configuration history around short open tags should not be generalized to it.
- Comments and whitespace are observable to tokenizer clients even though they usually do not affect execution.
- Heredoc and nowdoc delimiters have strict placement and indentation rules that have changed across PHP versions; test the minimum supported version.
- A word can be a keyword in one grammatical context and usable as a name in another. Tokenization and grammar cooperate to handle such cases.
- An invalid byte sequence, unterminated string, or unterminated comment can prevent useful parser input from being produced.

## Performance

Scanning is normally linear in source size, but constants matter: large generated files, many includes, and repeated calls to `token_get_all()` allocate token data and retain source text. Do not tokenize every request merely to implement a runtime policy. Perform source analysis in CI, at build time, or in an explicitly offline tool.

## Security

Tokenization is not validation. A source scanner that looks for `eval`, `include`, or a function name can be bypassed by aliases, dynamic calls, encodings, or behavior it does not model. Do not execute untrusted PHP source to “see what tokens it produces.” If a tool rewrites source, treat its output as code and run syntax checks plus tests before deployment.

## Testing

Use fixture files rather than only inline strings. Cover:

1. PHP tags and literal text.
2. Namespaces, attributes, class declarations, and named arguments.
3. Single and double quoted strings, interpolation, heredoc, and nowdoc.
4. Comments, whitespace, Unicode identifiers where supported, and line endings.
5. Malformed input with a clearly documented expected failure.

Assert token names and text, not numeric IDs. Run the fixture suite on every supported PHP minor version because the token vocabulary and contextual distinctions can evolve.

## Common Mistakes

- Calling `token_get_all()` a parser.
- Dropping comments or whitespace when a formatter needs to preserve them.
- Splitting source with regular expressions around quotes or comments.
- Assuming all operators have named `T_*` tokens.
- Treating the tokenizer extension’s public output as identical to every private parser token.
- Forgetting that PHP alternates between literal and PHP modes.

## Senior Engineer Thinking

Choose the weakest representation that can answer the question safely. Tokenization is excellent for a narrow lexical report and often insufficient for refactoring or correctness analysis. When a requirement mentions nesting, scope, precedence, control flow, or symbol resolution, move up to an AST and semantic model rather than adding more regular expressions.

## Exercises

1. Write a token dumper that prints token name, line, and exact text while preserving single-character tokens.
2. Compare token output for a variable inside single quotes, double quotes, heredoc, and nowdoc.
3. Build a comment counter and test it against comments appearing inside strings.
4. Run your token tests on two PHP minor versions and identify which assumptions are version-sensitive.

## Review Questions

1. What information does a token add to a character stream?
2. Why can the tokenizer expose whitespace while the parser ignores it?
3. Why are token integer values not a stable API?
4. Which PHP features require scanner state rather than independent character matching?
5. When should a tool use an AST instead of tokens?

## Summary

The lexer turns PHP source characters into a stateful stream of tokens and source positions. PHP’s mixed HTML/PHP mode, strings, interpolation, and heredocs make scanning richer than a simple split operation. `token_get_all()` is valuable for source tooling, but it is not a parser and its numeric token IDs are version-sensitive. The parser uses this stream to construct grammatical structure.

## References

- [PHP Manual: Tokenizer](https://www.php.net/manual/en/book.tokenizer.php)
- [PHP Manual: `token_get_all()`](https://www.php.net/manual/en/function.token-get-all.php)
- [PHP Manual: List of Parser Tokens](https://www.php.net/manual/en/tokens.php)
- [php-src: `Zend/zend_language_scanner.l`](https://github.com/php/php-src/blob/master/Zend/zend_language_scanner.l)
- [php-src: `Zend/zend_language_scanner.h`](https://github.com/php/php-src/blob/master/Zend/zend_language_scanner.h)
