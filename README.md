Micro Parser Combinators
========================

Version 0.8.8


About
-----

_mpc_ is a lightweight and powerful Parser Combinator library for C.

Using _mpc_ might be of interest to you if you are...

* Building a new programming language
* Building a new data format
* Parsing an existing programming language
* Parsing an existing data format
* Embedding a Domain Specific Language
* Implementing [Greenspun's Tenth Rule](http://en.wikipedia.org/wiki/Greenspun%27s_tenth_rule)


Features
--------

* Type-Generic
* Predictive, Recursive Descent
* Easy to Integrate (One Source File in ANSI C)
* Automatic Error Message Generation
* Regular Expression Parser Generator
* Language/Grammar Parser Generator


Alternatives
------------

The current main alternative for a C based parser combinator library is a branch of [Cesium3](https://github.com/wbhart/Cesium3/tree/combinators).

_mpc_ provides a number of features that this project does not offer, and also overcomes a number of potential downsides:

* _mpc_ Works for Generic Types
* _mpc_ Doesn't rely on Boehm-Demers-Weiser Garbage Collection
* _mpc_ Doesn't use `setjmp` and `longjmp` for errors
* _mpc_ Doesn't pollute the namespace


Quickstart
==========

Here is how one would use _mpc_ to create a parser for a basic mathematical expression language.

```c
mpc_parser_t *Expr  = mpc_new("expression");
mpc_parser_t *Prod  = mpc_new("product");
mpc_parser_t *Value = mpc_new("value");
mpc_parser_t *Maths = mpc_new("maths");

mpca_lang(MPCA_LANG_DEFAULT,
  " expression : <product> (('+' | '-') <product>)*; "
  " product    : <value>   (('*' | '/')   <value>)*; "
  " value      : /[0-9]+/ | '(' <expression> ')';    "
  " maths      : /^/ <expression> /$/;               ",
  Expr, Prod, Value, Maths, NULL);

mpc_result_t r;

if (mpc_parse("input", input, Maths, &r)) {
  mpc_ast_print(r.output);
  mpc_ast_delete(r.output);
} else {
  mpc_err_print(r.error);
  mpc_err_delete(r.error);
}

mpc_cleanup(4, Expr, Prod, Value, Maths);
```

If you were to set `input` to the string `(4 * 2 * 11 + 2) - 5`, the printed output would look like this.

```
>
  regex
  expression|>
    value|>
      char:1:1 '('
      expression|>
        product|>
          value|regex:1:2 '4'
          char:1:4 '*'
          value|regex:1:6 '2'
          char:1:8 '*'
          value|regex:1:10 '11'
        char:1:13 '+'
        product|value|regex:1:15 '2'
      char:1:16 ')'
    char:1:18 '-'
    product|value|regex:1:20 '5'
  regex
```

Getting Started
===============

Introduction
------------

Parser Combinators are structures that encode how to parse particular languages. They can be combined using intuitive operators to create new parsers of increasing complexity. Using these operators detailed grammars and languages can be parsed and processed in a quick, efficient, and easy way.

The trick behind Parser Combinators is the observation that by structuring the library in a particular way, one can make building parser combinators look like writing a grammar itself. Therefore instead of describing _how to parse a language_, a user must only specify _the language itself_, and the library will work out how to parse it ... as if by magic!

_mpc_ can be used in this mode, or, as shown in the above example, you can specify the grammar directly as a string or in a file.

Basic Parsers
-------------

### String Parsers

All the following functions construct new basic parsers of the type `mpc_parser_t *`. All of those parsers return a newly allocated `char *` with the character(s) they manage to match. If unsuccessful they will return an error. They have the following functionality.

* * * 

```c
mpc_parser_t *mpc_any(void);
```

Matches any individual character

* * * 

```c
mpc_parser_t *mpc_char(char c);
```

Matches a single given character `c`

* * *

```c
mpc_parser_t *mpc_range(char s, char e);
```

Matches any single given character in the range `s` to `e` (inclusive)

* * *

```c
mpc_parser_t *mpc_oneof(const char *s);
```

Matches any single given character in the string  `s`

* * *

```c
mpc_parser_t *mpc_noneof(const char *s);
```

Matches any single given character not in the string `s`

* * *

```c
mpc_parser_t *mpc_satisfy(int(*f)(char));
```

Matches any single given character satisfying function `f`

* * *

```c
mpc_parser_t *mpc_string(const char *s);
```

Matches exactly the string `s`


### Other Parsers

Several other functions exist that construct parsers with some other special functionality.

* * *

```c
mpc_parser_t *mpc_pass(void);
```

Consumes no input, always successful, returns `NULL`

* * *

```c
mpc_parser_t *mpc_fail(const char *m);
mpc_parser_t *mpc_failf(const char *fmt, ...);
```

Consumes no input, always fails with message `m` or formatted string `fmt`.

* * *

```c
mpc_parser_t *mpc_lift(mpc_ctor_t f);
```

Consumes no input, always successful, returns the result of function `f`

* * *

```c
mpc_parser_t *mpc_lift_val(mpc_val_t *x);
```

Consumes no input, always successful, returns `x`

* * *

```c
mpc_parser_t *mpc_state(void);
```

Consumes no input, always successful, returns a copy of the parser state as a `mpc_state_t *`. This state is newly allocated and so needs to be released with `free` when finished with.

* * *

```c
mpc_parser_t *mpc_anchor(int(*f)(char,char));
```

Consumes no input. Successful when function `f` returns true. Always returns `NULL`.

Function `f` is a _anchor_ function. It takes as input the last character parsed, and the next character in the input, and returns success or failure. This function can be set by the user to ensure some condition is met. For example to test that the input is at a boundary between words and non-words.

At the start of the input the first argument is set to `'\0'`. At the end of the input the second argument is set to `'\0'`.



Parsing
-------

Once you've build a parser, you can run it on some input using one of the following functions. These functions return `1` on success and `0` on failure. They output either the result, or an error to a `mpc_result_t` variable. This type is defined as follows.

```c
typedef union {
  mpc_err_t *error;
  mpc_val_t *output;
} mpc_result_t;
```

where `mpc_val_t *` is synonymous with `void *` and simply represents some pointer to data - the exact type of which is dependant on the parser.


* * *

```c
int mpc_parse(const char *filename, const char *string, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on some string.

* * *

```c
int mpc_parse_file(const char *filename, FILE *file, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on some file.

* * *

```c
int mpc_parse_pipe(const char *filename, FILE *pipe, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on some pipe (such as `stdin`).

* * *

```c
int mpc_parse_contents(const char *filename, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on the contents of some file.


Combinators
-----------

Combinators are functions that take one or more parsers and return a new parser of some given functionality. 

These combinators work independently of exactly what data type the parser(s) supplied as input return. In languages such as Haskell ensuring you don't input one type of data into a parser requiring a different type is done by the compiler. But in C we don't have that luxury. So it is at the discretion of the programmer to ensure that he or she deals correctly with the outputs of different parser types.

A second annoyance in C is that of manual memory management. Some parsers might get half-way and then fail. This means they need to clean up any partial result that has been collected in the parse. In Haskell this is handled by the Garbage Collector, but in C these combinators will need to take _destructor_ functions as input, which say how clean up any partial data that has been collected.

Here are the main combinators and how to use then.

* * *

```c
mpc_parser_t *mpc_expect(mpc_parser_t *a, const char *e);
mpc_parser_t *mpc_expectf(mpc_parser_t *a, const char *fmt, ...);
```

Returns a parser that runs `a`, and on success returns the result of `a`, while on failure reports that `e` was expected.

* * *

```c
mpc_parser_t *mpc_apply(mpc_parser_t *a, mpc_apply_t f);
mpc_parser_t *mpc_apply_to(mpc_parser_t *a, mpc_apply_to_t f, void *x);
```

Returns a parser that applies function `f` (optionality taking extra input `x`) to the result of parser `a`.

* * *

```c
mpc_parser_t *mpc_check(mpc_parser_t *a, mpc_check_t f, const char *e);
mpc_parser_t *mpc_check_with(mpc_parser_t *a, mpc_check_with_t f, void *x, const char *e);
mpc_parser_t *mpc_checkf(mpc_parser_t *a, mpc_check_t f, const char *fmt, ...);
mpc_parser_t *mpc_check_withf(mpc_parser_t *a, mpc_check_with_t f, void *x, const char *fmt, ...);
```

Returns a parser that applies function `f` (optionally taking extra input `x`) to the result of parser `a`. If `f` returns non-zero, then the parser succeeds and returns the value of `a` (possibly modified by `f`). If `f` returns zero, then the parser fails with message `e`.

* * *

```c
mpc_parser_t *mpc_not(mpc_parser_t *a, mpc_dtor_t da);
mpc_parser_t *mpc_not_lift(mpc_parser_t *a, mpc_dtor_t da, mpc_ctor_t lf);
```

Returns a parser with the following behaviour. If parser `a` succeeds, then it fails and consumes no input. If parser `a` fails, then it succeeds, consumes no input and returns `NULL` (or the result of lift function `lf`). Destructor `da` is used to destroy the result of `a` on success.

* * *

```c
mpc_parser_t *mpc_maybe(mpc_parser_t *a);
mpc_parser_t *mpc_maybe_lift(mpc_parser_t *a, mpc_ctor_t lf);
```

Returns a parser that runs `a`. If `a` is successful then it returns the result of `a`. If `a` is unsuccessful then it succeeds, but returns `NULL` (or the result of `lf`).

* * *

```c
mpc_parser_t *mpc_many(mpc_fold_t f, mpc_parser_t *a);
```

Runs `a` zero or more times until it fails. Results are combined using fold function `f`. See the _Function Types_ section for more details.

* * *

```c
mpc_parser_t *mpc_many1(mpc_fold_t f, mpc_parser_t *a);
```

Runs `a` one or more times until it fails. Results are combined with fold function `f`.

* * *

```c
mpc_parser_t *mpc_count(int n, mpc_fold_t f, mpc_parser_t *a, mpc_dtor_t da);
```

Runs `a` exactly `n` times. If this fails, any partial results are destructed with `da`. If successful results of `a` are combined using fold function `f`.

* * *

```c
mpc_parser_t *mpc_or(int n, ...);
```

Attempts to run `n` parsers in sequence, returning the first one that succeeds. If all fail, returns an error.

* * *

```c
mpc_parser_t *mpc_and(int n, mpc_fold_t f, ...);
```

Attempts to run `n` parsers in sequence, returning the fold of the results using fold function `f`. First parsers must be specified, followed by destructors for each parser, excluding the final parser. These are used in case of partial success. For example: `mpc_and(3, mpcf_strfold, mpc_char('a'), mpc_char('b'), mpc_char('c'), free, free);` would attempt to match `'a'` followed by `'b'` followed by `'c'`, and if successful would concatenate them using `mpcf_strfold`. Otherwise would use `free` on the partial results.

* * *

```c
mpc_parser_t *mpc_predictive(mpc_parser_t *a);
```

Returns a parser that runs `a` with backtracking disabled. This means if `a` consumes more than one character, it will not be reverted, even on failure. Turning backtracking off has good performance benefits for grammars which are `LL(1)`. These are grammars where the first character completely determines the parse result - such as the decision of parsing either a C identifier, number, or string literal. This option should not be used for non `LL(1)` grammars or it will produce incorrect results or crash the parser.

Another way to think of `mpc_predictive` is that it can be applied to a parser (for a performance improvement) if either successfully parsing the first character will result in a completely successful parse, or all of the referenced sub-parsers are also `LL(1)`.


Function Types
--------------

The combinator functions take a number of special function types as function pointers. Here is a short explanation of those types are how they are expected to behave. It is important that these behave correctly otherwise it is easy to introduce memory leaks or crashes into the system.

* * *

```c
typedef void(*mpc_dtor_t)(mpc_val_t*);
```

Given some pointer to a data value it will ensure the memory it points to is freed correctly.

* * *

```c
typedef mpc_val_t*(*mpc_ctor_t)(void);
```

Returns some data value when called. It can be used to create _empty_ versions of data types when certain combinators have no known default value to return. For example it may be used to return a newly allocated empty string.

* * *

```c
typedef mpc_val_t*(*mpc_apply_t)(mpc_val_t*);
typedef mpc_val_t*(*mpc_apply_to_t)(mpc_val_t*,void*);
```

This takes in some pointer to data and outputs some new or modified pointer to data, ensuring to free the input data if it is no longer used. The `apply_to` variation takes in an extra pointer to some data such as global state.

* * *

```c
typedef int(*mpc_check_t)(mpc_val_t**);
typedef int(*mpc_check_with_t)(mpc_val_t**,void*);
```

This takes in some pointer to data and outputs 0 if parsing should stop with an error. Additionally, this may change or free the input data. The `check_with` variation takes in an extra pointer to some data such as global state.

* * *

```c
typedef mpc_val_t*(*mpc_fold_t)(int,mpc_val_t**);
```

This takes a list of pointers to data values and must return some combined or folded version of these data values. It must ensure to free any input data that is no longer used once the combination has taken place.


Case Study - Identifier
=======================

Combinator Method
-----------------

Using the above combinators we can create a parser that matches a C identifier.

When using the combinators we need to supply a function that says how to combine two `char *`.

For this we build a fold function that will concatenate zero or more strings together. For this sake of this tutorial we will write it by hand, but this (as well as many other useful fold functions), are actually included in _mpc_ under the `mpcf_*` namespace, such as `mpcf_strfold`.

```c
mpc_val_t *strfold(int n, mpc_val_t **xs) {
  char *x = calloc(1, 1);
  int i;
  for (i = 0; i < n; i++) {
    x = realloc(x, strlen(x) + strlen(xs[i]) + 1);
    strcat(x, xs[i]);
    free(xs[i]);
  }
  return x;
}
```

We can use this to specify a C identifier, making use of some combinators to say how the basic parsers are combined.

```c
mpc_parser_t *alpha = mpc_or(2, mpc_range('a', 'z'), mpc_range('A', 'Z'));
mpc_parser_t *digit = mpc_range('0', '9');
mpc_parser_t *underscore = mpc_char('_');

mpc_parser_t *ident = mpc_and(2, strfold,
  mpc_or(2, alpha, underscore),
  mpc_many(strfold, mpc_or(3, alpha, digit, underscore)),
  free);

/* Do Some Parsing... */

mpc_delete(ident);
```

Notice that previous parsers are used as input to new parsers we construct from the combinators. Note that only the final parser `ident` must be deleted. When we input a parser into a combinator we should consider it to be part of the output of that combinator.

Because of this we shouldn't create a parser and input it into multiple places, or it will be doubly freed.


Regex Method
------------

There is an easier way to do this than the above method. _mpc_ comes with a handy regex function for constructing parsers using regex syntax. We can specify an identifier using a regex pattern as shown below.

```c
mpc_parser_t *ident = mpc_re("[a-zA-Z_][a-zA-Z_0-9]*");

/* Do Some Parsing... */

mpc_delete(ident);
```


Library Method
--------------

Although if we really wanted to create a parser for C identifiers, a function for creating this parser comes included in _mpc_ along with many other common parsers.

```c
mpc_parser_t *ident = mpc_ident();

/* Do Some Parsing... */

mpc_delete(ident);
```

Parser References
=================

Building parsers in the above way can have issues with self-reference or cyclic-reference. To overcome this we can separate the construction of parsers into two different steps. Construction and Definition.

* * *

```c
mpc_parser_t *mpc_new(const char *name);
```

This will construct a parser called `name` which can then be used as input to others, including itself, without fear of being deleted. Any parser created using `mpc_new` is said to be _retained_. This means it will behave differently to a normal parser when referenced. When deleting a parser that includes a _retained_ parser, the _retained_ parser will not be deleted along with it. To delete a retained parser `mpc_delete` must be used on it directly.

A _retained_ parser can then be _defined_ using...

* * *

```c
mpc_parser_t *mpc_define(mpc_parser_t *p, mpc_parser_t *a);
```

This assigns the contents of parser `a` to `p`, and deletes `a`. With this technique parsers can now reference each other, as well as themselves, without trouble.

* * *

```c
mpc_parser_t *mpc_undefine(mpc_parser_t *p);
```

A final step is required. Parsers that reference each other must all be undefined before they are deleted. It is important to do any undefining before deletion. The reason for this is that to delete a parser it must look at each sub-parser that is used by it. If any of these have already been deleted a segfault is unavoidable - even if they were retained beforehand.

* * *

```c
void mpc_cleanup(int n, ...);
```

To ease the task of undefining and then deleting parsers `mpc_cleanup` can be used. It takes `n` parsers as input, and undefines them all, before deleting them all.

* * *

```c
mpc_parser_t *mpc_copy(mpc_parser_t *a);
```

This function makes a copy of a parser `a`. This can be useful when you want to 
use a parser as input for some other parsers multiple times without retaining 
it. 

* * *

```c
mpc_parser_t *mpc_re(const char *re);
mpc_parser_t *mpc_re_mode(const char *re, int mode);
```

This function takes as input the regular expression `re` and builds a parser 
for it. With the `mpc_re_mode` function optional mode flags can also be given. 
Available flags are `MPC_RE_MULTILINE` / `MPC_RE_M` where the start of input 
character `^` also matches the beginning of new lines and the end of input `$` 
character also matches new lines, and `MPC_RE_DOTALL` / `MPC_RE_S` where the 
any character token `.` also matches newlines (by default it doesn't).


Library Reference
=================

Common Parsers
--------------


<table>

  <tr><td><code>mpc_soi</code></td><td>Matches only the start of input, returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_eoi</code></td><td>Matches only the end of input, returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_boundary</code></td><td>Matches only the boundary between words, returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_boundary_newline</code></td><td>Matches the start of a new line, returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_whitespace</code></td><td>Matches any whitespace character <code>" \f\n\r\t\v"</code></td></tr>
  <tr><td><code>mpc_whitespaces</code></td><td>Matches zero or more whitespace characters</td></tr>
  <tr><td><code>mpc_blank</code></td><td>Matches whitespaces and frees the result, returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_newline</code></td><td>Matches <code>'\n'</code></td></tr>
  <tr><td><code>mpc_tab</code></td><td>Matches <code>'\t'</code></td></tr>
  <tr><td><code>mpc_escape</code></td><td>Matches a backslash followed by any character</td></tr>
  <tr><td><code>mpc_digit</code></td><td>Matches any character in the range <code>'0'</code> - <code>'9'</code></td></tr>
  <tr><td><code>mpc_hexdigit</code></td><td>Matches any character in the range <code>'0</code> - <code>'9'</code> as well as <code>'A'</code> - <code>'F'</code> and <code>'a'</code> - <code>'f'</code></td></tr>
  <tr><td><code>mpc_octdigit</code></td><td>Matches any character in the range <code>'0'</code> - <code>'7'</code></td></tr>
  <tr><td><code>mpc_digits</code></td><td>Matches one or more digit</td></tr>
  <tr><td><code>mpc_hexdigits</code></td><td>Matches one or more hexdigit</td></tr>
  <tr><td><code>mpc_octdigits</code></td><td>Matches one or more octdigit</td></tr>
  <tr><td><code>mpc_lower</code></td><td>Matches any lower case character</td></tr>
  <tr><td><code>mpc_upper</code></td><td>Matches any upper case character</td></tr>
  <tr><td><code>mpc_alpha</code></td><td>Matches any alphabet character</td></tr>
  <tr><td><code>mpc_underscore</code></td><td>Matches <code>'_'</code></td></tr>
  <tr><td><code>mpc_alphanum</code></td><td>Matches any alphabet character, underscore or digit</td></tr>
  <tr><td><code>mpc_int</code></td><td>Matches digits and returns an <code>int*</code></td></tr>
  <tr><td><code>mpc_hex</code></td><td>Matches hexdigits and returns an <code>int*</code></td></tr>
  <tr><td><code>mpc_oct</code></td><td>Matches octdigits and returns an <code>int*</code></td></tr>
  <tr><td><code>mpc_number</code></td><td>Matches <code>mpc_int</code>, <code>mpc_hex</code> or <code>mpc_oct</code></td></tr>
  <tr><td><code>mpc_real</code></td><td>Matches some floating point number as a string</td></tr>
  <tr><td><code>mpc_float</code></td><td>Matches some floating point number and returns a <code>float*</code></td></tr>
  <tr><td><code>mpc_char_lit</code></td><td>Matches some character literal surrounded by <code>'</code></td></tr>
  <tr><td><code>mpc_string_lit</code></td><td>Matches some string literal surrounded by <code>"</code></td></tr>
  <tr><td><code>mpc_regex_lit</code></td><td>Matches some regex literal surrounded by <code>/</code></td></tr>
  <tr><td><code>mpc_ident</code></td><td>Matches a C style identifier</td></tr>

</table>


Useful Parsers
--------------

<table>

  <tr><td><code>mpc_startswith(mpc_parser_t *a);</code></td><td>Matches the start of input followed by <code>a</code></td></tr>
  <tr><td><code>mpc_endswith(mpc_parser_t *a, mpc_dtor_t da);</code></td><td>Matches <code>a</code> followed by the end of input</td></tr>
  <tr><td><code>mpc_whole(mpc_parser_t *a, mpc_dtor_t da);</code></td><td>Matches the start of input, <code>a</code>, and the end of input</td></tr>  
  <tr><td><code>mpc_stripl(mpc_parser_t *a);</code></td><td>Matches <code>a</code> first consuming any whitespace to the left</td></tr>
  <tr><td><code>mpc_stripr(mpc_parser_t *a);</code></td><td>Matches <code>a</code> then consumes any whitespace to the right</td></tr>
  <tr><td><code>mpc_strip(mpc_parser_t *a);</code></td><td>Matches <code>a</code> consuming any surrounding whitespace</td></tr>
  <tr><td><code>mpc_tok(mpc_parser_t *a);</code></td><td>Matches <code>a</code> and consumes any trailing whitespace</td></tr>
  <tr><td><code>mpc_sym(const char *s);</code></td><td>Matches string <code>s</code> and consumes any trailing whitespace</td></tr>
  <tr><td><code>mpc_total(mpc_parser_t *a, mpc_dtor_t da);</code></td><td>Matches the whitespace consumed <code>a</code>, enclosed in the start and end of input</td></tr>
  <tr><td><code>mpc_between(mpc_parser_t *a, mpc_dtor_t ad, <br /> const char *o, const char *c);</code></td><td> Matches <code>a</code> between strings <code>o</code> and <code>c</code></td></tr>
  <tr><td><code>mpc_parens(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between <code>"("</code> and <code>")"</code></td></tr>
  <tr><td><code>mpc_braces(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between <code>"<"</code> and <code>">"</code></td></tr>
  <tr><td><code>mpc_brackets(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between <code>"{"</code> and <code>"}"</code></td></tr>
  <tr><td><code>mpc_squares(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between <code>"["</code> and <code>"]"</code></td></tr>
  <tr><td><code>mpc_tok_between(mpc_parser_t *a, mpc_dtor_t ad, <br /> const char *o, const char *c);</code></td><td>Matches <code>a</code> between <code>o</code> and <code>c</code>, where <code>o</code> and <code>c</code> have their trailing whitespace striped.</td></tr>
  <tr><td><code>mpc_tok_parens(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between trailing whitespace consumed <code>"("</code> and <code>")"</code></td></tr>
  <tr><td><code>mpc_tok_braces(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between trailing whitespace consumed <code>"<"</code> and <code>">"</code></td></tr>
  <tr><td><code>mpc_tok_brackets(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between trailing whitespace consumed <code>"{"</code> and <code>"}"</code></td></tr>
  <tr><td><code>mpc_tok_squares(mpc_parser_t *a, mpc_dtor_t ad);</code></td><td>Matches <code>a</code> between trailing whitespace consumed <code>"["</code> and <code>"]"</code></td></tr>

</table>


Apply Functions
---------------

<table>

  <tr><td><code>void mpcf_dtor_null(mpc_val_t *x);</code></td><td>Empty destructor. Does nothing</td></tr>
  <tr><td><code>mpc_val_t *mpcf_ctor_null(void);</code></td><td>Returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_ctor_str(void);</code></td><td>Returns <code>""</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_free(mpc_val_t *x);</code></td><td>Frees <code>x</code> and returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_int(mpc_val_t *x);</code></td><td>Converts a decimal string <code>x</code> to an <code>int*</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_hex(mpc_val_t *x);</code></td><td>Converts a hex string <code>x</code> to an <code>int*</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_oct(mpc_val_t *x);</code></td><td>Converts a oct string <code>x</code> to an <code>int*</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_float(mpc_val_t *x);</code></td><td>Converts a string <code>x</code> to a <code>float*</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_escape(mpc_val_t *x);</code></td><td>Converts a string <code>x</code> to an escaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_escape_regex(mpc_val_t *x);</code></td><td>Converts a regex <code>x</code> to an escaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_escape_string_raw(mpc_val_t *x);</code></td><td>Converts a raw string <code>x</code> to an escaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_escape_char_raw(mpc_val_t *x);</code></td><td>Converts a raw character <code>x</code> to an escaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_unescape(mpc_val_t *x);</code></td><td>Converts a string <code>x</code> to an unescaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_unescape_regex(mpc_val_t *x);</code></td><td>Converts a regex <code>x</code> to an unescaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_unescape_string_raw(mpc_val_t *x);</code></td><td>Converts a raw string <code>x</code> to an unescaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_unescape_char_raw(mpc_val_t *x);</code></td><td>Converts a raw character <code>x</code> to an unescaped version</td></tr>
  <tr><td><code>mpc_val_t *mpcf_strtriml(mpc_val_t *x);</code></td><td>Trims whitespace from the left of string <code>x</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_strtrimr(mpc_val_t *x);</code></td><td>Trims whitespace from the right of string <code>x</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_strtrim(mpc_val_t *x);</code></td><td>Trims whitespace from either side of string <code>x</code></td></tr>
</table>


Fold Functions
--------------

<table>


  <tr><td><code>mpc_val_t *mpcf_null(int n, mpc_val_t** xs);</code></td><td>Returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_fst(int n, mpc_val_t** xs);</code></td><td>Returns first element of <code>xs</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_snd(int n, mpc_val_t** xs);</code></td><td>Returns second element of <code>xs</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_trd(int n, mpc_val_t** xs);</code></td><td>Returns third element of <code>xs</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_fst_free(int n, mpc_val_t** xs);</code></td><td>Returns first element of <code>xs</code> and calls <code>free</code> on others</td></tr>
  <tr><td><code>mpc_val_t *mpcf_snd_free(int n, mpc_val_t** xs);</code></td><td>Returns second element of <code>xs</code> and calls <code>free</code> on others</td></tr>
  <tr><td><code>mpc_val_t *mpcf_trd_free(int n, mpc_val_t** xs);</code></td><td>Returns third element of <code>xs</code> and calls <code>free</code> on others</td></tr>
  <tr><td><code>mpc_val_t *mpcf_freefold(int n, mpc_val_t** xs);</code></td><td>Calls <code>free</code> on all elements of <code>xs</code> and returns <code>NULL</code></td></tr>
  <tr><td><code>mpc_val_t *mpcf_strfold(int n, mpc_val_t** xs);</code></td><td>Concatenates all <code>xs</code> together as strings and returns result </td></tr>

</table>


Case Study - Maths Language
===========================

Combinator Approach
-------------------

Passing around all these function pointers might seem clumsy, but having parsers be type-generic is important as it lets users define their own output types for parsers. For example we could design our own syntax tree type to use. We can also use this method to do some specific house-keeping or data processing in the parsing phase.

As an example of this power, we can specify a simple maths grammar, that outputs `int *`, and computes the result of the expression as it goes along.

We start with a fold function that will fold two `int *` into a new `int *` based on some `char *` operator.

```c
mpc_val_t *fold_maths(int n, mpc_val_t **xs) {
  
  int **vs = (int**)xs;
    
  if (strcmp(xs[1], "*") == 0) { *vs[0] *= *vs[2]; }
  if (strcmp(xs[1], "/") == 0) { *vs[0] /= *vs[2]; }
  if (strcmp(xs[1], "%") == 0) { *vs[0] %= *vs[2]; }
  if (strcmp(xs[1], "+") == 0) { *vs[0] += *vs[2]; }
  if (strcmp(xs[1], "-") == 0) { *vs[0] -= *vs[2]; }
  
  free(xs[1]); free(xs[2]);
  
  return xs[0];
}
```

And then we use this to specify a basic grammar, which folds together any results.

```c
mpc_parser_t *Expr   = mpc_new("expr");
mpc_parser_t *Factor = mpc_new("factor");
mpc_parser_t *Term   = mpc_new("term");
mpc_parser_t *Maths  = mpc_new("maths");

mpc_define(Expr, mpc_or(2, 
  mpc_and(3, fold_maths,
    Factor, mpc_oneof("+-"), Factor,
    free, free),
  Factor
));

mpc_define(Factor, mpc_or(2, 
  mpc_and(3, fold_maths,
    Term, mpc_oneof("*/"), Term,
    free, free),
  Term
));

mpc_define(Term, mpc_or(2, mpc_int(), mpc_parens(Expr, free)));
mpc_define(Maths, mpc_whole(Expr, free));

/* Do Some Parsing... */

mpc_delete(Maths);
```

If we supply this function with something like `(4*2)+5`, we can expect it to output `13`.


Language Approach
-----------------

It is possible to avoid passing in and around all those function pointers, if you don't care what type is output by _mpc_. For this, a generic Abstract Syntax Tree type `mpc_ast_t` is included in _mpc_. The combinator functions which act on this don't need information on how to destruct or fold instances of the result as they know it will be a `mpc_ast_t`. So there are a number of combinator functions which work specifically (and only) on parsers that return this type. They reside under `mpca_*`.

Doing things via this method means that all the data processing must take place after the parsing. In many instances this is not an issue, or even preferable.

It also allows for one more trick. As all the fold and destructor functions are implicit, the user can simply specify the grammar of the language in some nice way and the system can try to build a parser for the AST type from this alone. For this there are a few functions supplied which take in a string, and output a parser. The format for these grammars is simple and familiar to those who have used parser generators before. It looks something like this.

```
number "number" : /[0-9]+/ ;
expression      : <product> (('+' | '-') <product>)* ;
product         : <value>   (('*' | '/')   <value>)* ;
value           : <number> | '(' <expression> ')' ;
maths           : /^/ <expression> /$/ ;
```

The syntax for this is defined as follows.

<table class='table'>
  <tr><td><code>"ab"</code></td><td>The string <code>ab</code> is required.</td></tr>
  <tr><td><code>'a'</code></td><td>The character <code>a</code> is required.</td></tr>
  <tr><td><code>'a' 'b'</code></td><td>First <code>'a'</code> is required, then <code>'b'</code> is required..</td></tr>
  <tr><td><code>'a' | 'b'</code></td><td>Either <code>'a'</code> is required, or <code>'b'</code> is required.</td></tr>
  <tr><td><code>'a'*</code></td><td>Zero or more <code>'a'</code> are required.</td></tr>
  <tr><td><code>'a'+</code></td><td>One or more <code>'a'</code> are required.</td></tr>
  <tr><td><code>&lt;abba&gt;</code></td><td>The rule called <code>abba</code> is required.</td></tr>
</table>

Rules are specified by rule name, optionally followed by an _expected_ string, followed by a colon `:`, followed by the definition, and ending in a semicolon `;`. Multiple rules can be specified. The _rule names_ must match the names given to any parsers created by `mpc_new`, otherwise the function will crash.

The flags variable is a set of flags `MPCA_LANG_DEFAULT`, `MPCA_LANG_PREDICTIVE`, or `MPCA_LANG_WHITESPACE_SENSITIVE`. For specifying if the language is predictive or whitespace sensitive.

Like with the regular expressions, this user input is parsed by existing parts of the _mpc_ library. It provides one of the more powerful features of the library.

* * *

```c
mpc_parser_t *mpca_grammar(int flags, const char *grammar, ...);
```

This takes in some single right hand side of a rule, as well as a list of any of the parsers referenced, and outputs a parser that does what is specified by the rule. The list of parsers referenced can be terminated with `NULL` to get an error instead of a crash when a parser required is not supplied.

* * *

```c
mpc_err_t *mpca_lang(int flags, const char *lang, ...);
```

This takes in a full language (zero or more rules) as well as any parsers referred to by either the right or left hand sides. Any parsers specified on the left hand side of any rule will be assigned a parser equivalent to what is specified on the right. On valid user input this returns `NULL`, while if there are any errors in the user input it will return an instance of `mpc_err_t` describing the issues. The list of parsers referenced can be terminated with `NULL` to get an error instead of a crash when a parser required is not supplied.

* * *

```c
mpc_err_t *mpca_lang_file(int flags, FILE* f, ...);
```

This reads in the contents of file `f` and inputs it into `mpca_lang`.

* * *

```c
mpc_err_t *mpca_lang_contents(int flags, const char *filename, ...);
```

This opens and reads in the contents of the file given by `filename` and passes it to `mpca_lang`.

Case Study - Tokenizer
======================

Another common task we might be interested in doing is tokenizing some block of 
text (splitting the text into individual elements) and performing some function
on each one of these elements as it is read. We can do this with `mpc` too.

First, we can build a regular expression which parses an individual token. For 
example if our tokens are identifiers, integers, commas, periods and colons we 
could build something like this `mpc_re("\\s*([a-zA-Z_]+|[0-9]+|,|\\.|:)")`. 
Next we can strip any whitespace, and add a callback function using `mpc_apply` 
which gets called every time this regex is parsed successfully 
`mpc_apply(mpc_strip(mpc_re("\\s*([a-zA-Z_]+|[0-9]+|,|\\.|:)")), print_token)`. 
Finally we can surround all of this in `mpc_many` to parse it zero or more 
times. The final code might look something like this:

```c
static mpc_val_t *print_token(mpc_val_t *x) {
  printf("Token: '%s'\n", (char*)x);
  return x;
}

int main(int argc, char **argv) {

  const char *input = "  hello 4352 ,  \n foo.bar   \n\n  test:ing   ";
  
  mpc_parser_t* Tokens = mpc_many(
    mpcf_all_free, 
    mpc_apply(mpc_strip(mpc_re("\\s*([a-zA-Z_]+|[0-9]+|,|\\.|:)")), print_token));
  
  mpc_result_t r;
  mpc_parse("input", input, Tokens, &r);
  
  mpc_delete(Tokens);
  
  return 0;
}
```

Running this program will produce an output something like this:

```
Token: 'hello'
Token: '4352'
Token: ','
Token: 'foo'
Token: '.'
Token: 'bar'
Token: 'test'
Token: ':'
Token: 'ing'
```

By extending the regex we can easily extend this to parse many more types of 
tokens and quickly and easily build a tokenizer for whatever language we are
interested in.


Error Reporting
===============

_mpc_ provides some automatic generation of error messages. These can be enhanced by the user, with use of `mpc_expect`, but many of the defaults should provide both useful and readable. An example of an error message might look something like this:

```
<test>:0:3: error: expected one or more of 'a' or 'd' at 'k'
```

Misc
====

Here are some other misc functions that mpc provides. These functions are susceptible to change between versions so use them with some care.

* * *

```c
void mpc_print(mpc_parser_t *p);
```

Prints out a parser in some weird format. This is generally used for debugging so don't expect to be able to understand the output right away without looking at the source code a little bit.

* * *

```c
void mpc_stats(mpc_parser_t *p);
```

Prints out some basic stats about a parser. Again used for debugging and optimisation.

* * *

```c
void mpc_optimise(mpc_parser_t *p);
```

Performs some basic optimisations on a parser to reduce it's size and increase its running speed.


Lexer
=====

the built in lexer was created (by mgood7123) to aid in tokenising input

the reason for this is to make parsing easier and more simple, and to avoid cases where parser AST grammer may conflict with rules or statements leading to the input being parsed incorectly sometimes in a way that is unavoidable regardless of how you re order the statements and rules

the lexer itself is actually just a wrapper for mpc_parse that parses the input against multiple user defined parsers, reducing some of the complexity of manually implementing there own lexer, in which they may or may not know how to do correctly, not to say that my method of implementation is not correct as there are always better implementations out there

the reason it uses mpc_parse internally and does not just return its own mpc_parser_t is due to how it needs to work, for it to be able to lex it needs to be able to parse, tokenise, and do something with its tokens, in which tokenisation is difficult if it only returns a mpc_parser_t as a user will likely not know that they would need to recursively call mpc_parse, nor may or may not know how to correctly advance the input so it does not infinitely parse the exact same input without advancing the input itself

plus the lexer would be an internal function and not a third party function such as a bison lexer, in which having it internal increases its portability as you dont need any extra third party lexing libraries

due to how grammer is specified it can be difficult to get what you want in specific cases, for example, you want to do one thing while also doing another, without sacrificing one or the other due to overlapping rules and definitions, but the grammar you have just doesnt want to cooperate with you

Here is how one would use the lexer to parse a line, delimited by '\n', and output what it has obtained:

```
mpc_lexer_action(outEOL) {
	printf("lexing End Of Line: '%s'\n", (char *) outEOL->output);
	return 0;
}

mpc_lexer_action(outLINE) {
	printf("lexing Line: '%s'\n", (char *) outLINE->output);
	return 0;
}

int main(int argc, char **argv) {
	char * st = input;
	
	mpc_parser_t * EOL = mpc_new("EOL");
	mpc_parser_t * Line = mpc_new("Line");

	mpc_define(EOL, mpc_or(2, mpc_newline(), mpc_eoi));
	mpc_define(Line, mpc_re("[^\n]*"));
	
	mpc_lexer_t * lexer = mpc_lexer_new("lexer");
	
	mpc_lexer_add(&lexer, EOL, outEOL);
	mpc_lexer_add(&lexer, Line, outLINE);
	
	mpc_result_t r;
	mpc_lexer(st, lexer, &r);
	mpc_lexer_free(lexer);
	return 0;
	
}
```

if you where to set `input` to the string `"abcHVwufvyuevuy3y436782\n\n\nrehre\nrew\n-ql.;qa\neg"`,  the printed output would look like this.

```
lexing Line: 'abcHVwufvyuevuy3y436782'
lexing End Of Line: '
'
lexing End Of Line: '
'
lexing End Of Line: '
'
lexing Line: 'rehre'
lexing End Of Line: '
'
lexing Line: 'rew'
lexing End Of Line: '
'
lexing Line: '-ql.;qa'
lexing End Of Line: '
'
lexing Line: 'eg'
```

Inner Workings of the lexer
---------------------------

The lexer is designed loosely around the AST and the Parser but differs in a few ways:

### No Abstract Syntax Tree

the lexer cannot accept an Abstract Syntax Tree for various reasons such as outputting and complexity, or rather lack of knowledge of how one would output a constructed AST in a way such that each match is outputted from the root of its matching tag recursively and thus would be very complex as input would be caotic and unclear, however this does not mean it will never support an AST, it is just discouraged due to its complexity

consider the following AST print:

```
parsing  a[b[0]]  
> 
  Command|> 
    Space|char:1:1 ' '
    Expression|Array|> 
      Alpha|regex:1:2 'a'
      Index|> 
        char:1:3 '['
        Array|> 
          Alpha|regex:1:4 'b'
          Index|> 
            char:1:5 '['
            Digit|regex:1:6 '0'
            char:1:7 ']'
        char:1:8 ']'
```

pretty confusing right?

how would you decide which node you are meant to start from, and which combinations you are ment to output, you could output the following: `Char`, `Digit`, `Index`, `Array`, `Expression`, or `Command`. it would eventually turn into and merge into parsing as opposed to lexing and just turn into a duplication of the parser itself

### Provides callbacks to lexer parsers

the lexer provides callbacks to lexer parsers, or parser actions, in which one can use to parse specific tokens based on the combination of parsers used and what order they are specified in. This way each token can theoretically be assigned a different parser, making more or less just as powerful as the parser itself, for example, you could have one parser for when the lexer tokenizes X, and another entirely different parser for when it tokenizes Z

or the callback can be used for something entirely different, it could for example, start lexing files based on what it has already lexed, or it could just abort when it lexes a specific token, or it could execute a system command and grab its output, the callback can do whatever you want it to do, providing a large deal of flexibility as it does not need to be limited to just parsing or lexing

### recursively adds upon itself

the lexer is designed so it can be created in a simple additive way, in that you stack up parsers similar to building an `if (...) {...} else if (...) {...}` loop, except it doesnt quite work like that

consider the following definition of a lexer with two parsers
```
	mpc_lexer_t * lexer = mpc_lexer_new("lexer");
	
	mpc_lexer_add(&lexer, EOL, outEOL);
	mpc_lexer_add(&lexer, Line, outLINE);
	mpc_lexer_print(lexer);
```


```
mpc_lexer_t * lexer = mpc_lexer_new("lexer");
```
a lexer is first created with `mpc_lexer_new` and assigned a name
then it is assigned some parsers:

```
mpc_lexer_add(&lexer, EOL, outEOL, NULL);
```
this causes `lexer` to have binded to it an End Of Line parser, and is registered the function `outEOL` as a parser action, in which the action to be performed when parsing is successful (this CAN be NULL to specify no function should be binded), and binds a lexer to `lexer` registering it as a mode change (this CAN be NULL to specify no mode should be binded), this is called via `mpc_lexer`, if it succeeds it executes its binded function if any, switches the lexer to the desired lexer if any, and consumes the input it has managed to match, then moves on to the next parser, if there are no other parsers then it checks itself again or returns if all parsers fail to match any input

```
mpc_lexer_add(&lexer, Line, outLINE, NULL);
```
then it binds a second parser, Line, and a function, outLINE, this will be checked AFTER the parser EOL has been checked, it follows the same samantics as described above

```
mpc_lexer_print(lexer);
```
finally it prints the specified lexer, and the parsers that have been binded to that lexer along with there assiciated action, `(nil)` means no action is associated

### able to redefine itself at runtime allowing for dynamic parsing

the lexer is able to redefine any of its elements during runtime

this allows the changing of:
* its own name
* its associated parser
* its associated action
* its associated mode
* its internal count
* modifying parts of other NAMED lexers via the global lexer pool, we will talks more about this later
* and any other members in `mpc_lexer_t`

this enables the ability to dynamically tokenise input by self modifying,

for example, you can parse multiple tokens with a single parser when the input would be otherwise unknown at runtime or in some cases unpredictable, by simply redefining its parser

or you could switch the action associated with a parser, normally dependant on a condition in the action itself, or at random

self modification can be done via the self pointer in the current action

for example, redefining a parser is as simple as putting `mpc_define(self->parser, new_parser);` in a parser's action

### supports mode changing

this enables the ability to change lexers during run time upon a desired token being parsed, effectively doing the following:
* the entire lexer grammer changes
* the input buffer is maintained


with these differences aside it is easy to create and use lexers, now back to the internals:

### the lexer structure

the structure of a lexer `mpc_lexer_t` is fairly simple.
```
struct mpc_lexer_t {
	char * name;
	mpc_parser_t * parser;
	MPC_LEX_ACTION(action);
	int count;
	struct mpc_lexer_t * self;
	struct mpc_lexer_pool_t * mpc_lexer_pool;
	mpc_lexer_t ** mode;
};

typedef struct mpc_lexer_t mpc_lexer_t;

```
here it can
* be named
* be assigned a parser
* be assigned an action
* keep a counter for maintaining internals
* keep a pointer to itself
* keep a pointer to the global lexer pool
* be assigned a mode

notice that action is a `macro`, these macros have hardcoded internal parameter types and thus cannot be left to the user to know what type is expected, the paramater is used internally by the lexer as a callback for the result of any matching parser expression, and does what mpc_apply cannot do, such as passing additional arguments other than just the output of the result

currently three macros are provided:
* * *
```
#define MPC_LEX_ACTION(name) int(* name)(mpc_lexer_t*, mpc_result_t*)
```
this is a macro for defining a function pointer used by the lexer, it takes a function pointer name, allowing it to be differenciated from duplicate function pointers, and expects a function consisting of two arguments, a lexer of type `mpc_lexer_t` and a result of type `mpc_result_t` and a function return value of `int`

* * * 

```
#define mpc_lexer_action(x) int x(mpc_lexer_t * self, mpc_result_t * x)
```
this is a macro for defining a function for use by the lexer, as with `MPC_LEX_ACTION`, it specifies a function that takes two arguments, a lexer of type `mpc_lexer_t` and a result of type `mpc_result_t` and a function return value of `int`, but the name of the function and the name of the parameter is defined to the same name, to avoid possible name conflicts when dealing with hardcoded names

the first argument will always point to the current lexer index

* * *

```
#define mpc_lexer_free(x) mpc_lexer_free__(x); x = NULL
```

this is a macro for `mpc_lexer_free__` and sets `x` to NULL to prevent memory bugs, since this is not always obvious to the user

the variable `count` is used internally to keep track of assigned parsers and should not be modified by the user under normal cases unless developing a new function

### Lexer Functions

```
mpc_lexer_t *mpc_lexer_undefined(void);
mpc_lexer_t *mpc_lexer_new(const char *name);
```
these functions, create a new lexer, this lexer is allocated and must be freed, `mpc_lexer_undefined` creates an unnamed lexer, and `mpc_lexer_new` creates a named lexer, creating a named lexer is recommended

```
void mpc_lexer_add(mpc_lexer_t ** l, mpc_parser_t * p, MPC_LEX_ACTION(action), mpc_lexer_t ** m);
```
as described in the section _recursively adds upon itself_, this binds a parser, then binds a function to it, if a mode is provided via `m` it will be binded

```
void mpc_lexer_print(mpc_lexer_t * l);
```
this prints a given lexer, and any parsers and modes associated with it

```
void mpc_lexer_free__(mpc_lexer_t * l);
```
this frees an allocated lexer

```
mpc_err_t *mpc_lexer(char * input, mpc_lexer_t * list, mpc_result_t * result);
```
this is where the magic happens, once we have built up a lexer we then run it against some input, how this is done is just above simple:
```
int mpc_lexer(char * input, mpc_lexer_t ** list, mpc_result_t * result) {
	int parsed_succesfully = 0;
	while(strcmp(input,"")!=0) {
		parsed_succesfully = 0;
		for (int i = 0; i < (*list)->count; i++) {
			if (mpc_parse("lexer", input, (*list)[i].parser, result)) {
				if (strcmp(result->output,"") != 0) {
					parsed_succesfully = 1;
					int len = strlen(result->output);
					input+=len;
					if ((*list)[i].action) (*list)[i].action(&(*list)[i], result);
					if ((*list)[i].mode) {
						puts("changing modes");
						list = (*list)[i].mode;
					}
					i = -1;
				}
			}
		}
		if (parsed_succesfully == 0) return -1; // return if the all of the parsers could not match anything to avoid an infinite while loop if the input has not been emptied
	}
	return parsed_succesfully;
}
```

here is a breakdown of the function and what it does:

```
	int parsed_succesfully = 0;
```
declare a variable to store the result of a lex run

```
	while(strcmp(input,"")!=0) {
```
first we run a while loop, checking if the input is empty

```
		parsed_succesfully = 0;
```
then we reset the variable each time the parsing loop starts

```
		for (int i = 0; i < list->count; i++) {
```
then while the input is not empty, we loop through our list of binded parsers

```
			if (mpc_parse("lexer", input, list[i].parser, result)) {
```
during the looping, parse the current parser

```
				if (strcmp(result->output,"") != 0) {
```
we check if our matched string is not empty

```
					parsed_succesfully = 1;
					int len = strlen(result->output);
					input+=len;
```
consuming input if it matches and setting the parse variable to 1 to indicate a successfull parse

```
					if ((*list)[i].action) (*list)[i].action(&(*list)[i], result);
```
and check if a function is associated with the current parser, and if so execute it with two arguments, the first argument is the address of the index of the current list itself, and the second is the result of the parser

```
					if ((*list)[i].mode) {
						puts("changing modes");
						list = (*list)[i].mode;
					}
```
then check if there is a mode change associated with the current parser and if so, reassign the current list

```
					i = -1;
				}
			}
		}
```
then we reset `i` to -1 to reset the parser list so it parses from list index 0, this is arguable as given `"abab"`, and the parsers `mpc_char('a')`, `mpc_any()`, `mpc_string("ab")`, in that order, if we dont reset, it will parse as `"a" > "b" > "ab"`, and if we do reset it will parse as `"a" > "b" > "a" > "b"`

```
		if (parsed_succesfully == 0) return -1;
```
next we check if `parsed_succesfully` has been set after the list loop, if it has been set this would mean that the any of the parsers have succeeded in matching a token at least one or more times, if it has failed we return `-1` to indicate an error if the all of the parsers could not match anything to avoid an infinite while loop occuring if the input has not been emptied

```
	}
	return parsed_succesfully;
}
```
we repeat the entire process again until all input has been consumed then we return `parsed_succesfully`

Basic Lexers
============

at the moment two pre-built lexers are provided:

```
mpc_lexer_t * mpcl_line(MPC_LEX_ACTION(EOL_ACTION), MPC_LEX_ACTION(LINE_ACTION));
```

this returns a lexer that will lex lines of input, ending in a `\n`, or end of input, normally `NULL`

```
mpc_lexer_t * mpcl_shline(MPC_LEX_ACTION(EOL_ACTION), MPC_LEX_ACTION(LINE_ACTION));
```

this returns a lexer that will lex lines of input, ending in a `;`, `\n`, or end of input, normally `NULL`, designed for lexing shell code, commands can also end in `;`, at the moment quoting is not supported so lexing `hello "world!;";` will lex `hello "world!` and `"`, whitespaces are supported


using this we can then modify our example to use the pre-built lexer:

```
mpc_lexer_action(outEOL) {
	printf("lexing End Of Line: '%s'\n", (char *) outEOL->output);
	return 0;
}

mpc_lexer_action(outLINE) {
	printf("lexing Line: '%s'\n", (char *) outLINE->output);
	return 0;
}

int main(int argc, char **argv) {
	char * st = "abcHVwufvyuevuy3y436782\n\n\nrehre\nrew\n-ql.;qa\neg";
		
	mpc_lexer_t * lexer = mpcl_line(outEOL, outLINE);
	
	mpc_result_t r;
	mpc_lexer(st, lexer, &r);
	mpc_lexer_free(lexer);
	return 0;
}
```

which outputs exactly the same as out manually built lexer:

```
lexing Line: 'abcHVwufvyuevuy3y436782'
lexing End Of Line: '
'
lexing End Of Line: '
'
lexing End Of Line: '
'
lexing Line: 'rehre'
lexing End Of Line: '
'
lexing Line: 'rew'
lexing End Of Line: '
'
lexing Line: '-ql.;qa'
lexing End Of Line: '
'
lexing Line: 'eg'
```

Self Modifying Lexer
===================

as stated earlier, the lexer allows an action to modify itself during run-time via two structures: the local `self` structure, and the global/local `mpc_lexer_pool` structure

lets use the `self` structure to dynamically parse "ab", starting with just lexing for 'a', and changing the parser to parse for 'b' instead of 'a' once 'a' has been parsed

```
char globalch = 0;

mpc_lexer_action(outCH) {
	printf("lexing Line: '%s'\n", (char *) outCH->output);
	if (globalch != 'b') {
		puts("changing globalch to 'b'");
		globalch = 'b';
		mpc_define(self->parser, mpc_char(globalch));
	}
	return 0;
}

int main(int argc, char **argv) {
	// lets attempt to lex something that will change
	
	globalch = 'a';
	
	mpc_result_t r;
	
	mpc_parser_t * Line = mpc_new("Line");
	mpc_define(Line, mpc_char(globalch));
		
	mpc_lexer_t * lexer = mpc_lexer_new("lexer");
	mpc_lexer_add(&lexer, Line, outCH);

	mpc_lexer("ab", lexer, &r);
	mpc_lexer_free(lexer);
	
	return 0;
	
}
```

if we where to run this, it would output

```
lexing Line: 'a'
changing globalch to 'b'
lexing Line: 'b'
```

the code is mostly the same as the previous example with the only difference being

```
mpc_define(self->parser, mpc_char(globalch));
```
what this does is redefines `lexer[0].parser` (since it only contains one parser we can evaluate `self` to `lexer[0]`) as the parser returned by `mpc_char(globalch)`, except this time, `globalch` has changed from `'a'` to `'b'` thus `lexer[0].parser` now contains a parser that parses `'b'` instead of `'a'`, and thus when `lexer[0].parser` gets parsed by `mpc_parse` again, it will parse `b` instead

global lexer pool
=================

when ever a named lexer is created it will get added to the `mpc_lexer_pool->lexer` array
when ever a named lexer is freed it is removed from the `mpc_lexer_pool->lexer` array

the structure `mpc_lexer_t` contains a pointer to this pool accessable via the exact same name, and is thus self recursive

this allows for lexers to access and modify other lexers including themselves

for example:
```
self->mpc_lexer_pool->lexer[0][0]->self->mpc_lexer_pool->lexer[0][0]->name
```
where `lexer[0][0][0]` specifies the first parser index ([0][0]`[0]`) of the first lexer (`[0]`[0][0]), and [0]`[0]`[0] (the middle `[0]`) specifies a constant address of the lexers, where reallocation does not change this address itself

for example, `mpc_lexer_pool->lexer[0][0][0].parser->name: EOL`, and `mpc_lexer_pool->lexer[0][0]->parser->name` specifies the name of the first parser of the first lexer

`mpc_lexer_pool` itself is a global variable in which the member `mpc_lexer_pool` from the struct `mpc_lexer_t` points to

### Pool Functions

```
void mpc_lexer_pool_undefined(void);
```
this assigns a new lexer pool, this is done internally via `mpc_lexer_add`

```
void mpc_lexer_pool_add(mpc_lexer_t *** l);
```
this adds a lexer to the lexer pool, internally done via `mpc_lexer_new`

```
void mpc_lexer_pool_remove(mpc_lexer_t ** l);
```
this removes a lexer from the lexer pool, internally done via `mpc_lexer_free__`

```
void mpc_lexer_pool_print(void);
```
this prints the current lexer pool

```
void mpc_lexer_pool_free(void);
```
this frees the current lexer pool, this is done internally via `mpc_lexer_free__` once all named lexers have been freed

```
mpc_lexer_find("lexer", "line");
```
this will search the lexer pool `mpc_lexer_pool` for a lexer named `lexer`, and all `lexer`'s indexes for a binded parser named `line` inside the lexer named `lexer`, returns NULL if not found otherwise returns the lexer's index that contains the parser as `mpc_lexer_t *` thus `mpc_lexer_find("lexer", "line")->parser->name` will result in lexer `lexer`'s index that contains a binded parser with the name `line`


Lexer Modes
===========

built into the lexer is support for lexer modes, this is equivilant to self modification but in a more compact form and is more user friendly

an example of this power is to modify the previous example of using self modification, and instead replace it with a lexer mode

```
#include "../mpc.h"
#include "../mpc.c"

mpc_lexer_action(lexerchange1) {
	printf("lexing test mode char a '%s'\n", (char *) lexerchange1->output);
	return 0;
}

mpc_lexer_action(lexerchange1a) {
	printf("lexing test mode char d '%s'\n", (char *) lexerchange1a->output);
	return 0;
}

mpc_lexer_action(lexerchange1b) {
	printf("lexing test mode char e '%s'\n", (char *) lexerchange1b->output);
	return 0;
}

mpc_lexer_action(lexerchange2) {
	printf("lexing test mode any char '%s'\n", (char *) lexerchange2->output);
	return 0;
}

mpc_lexer_action(lexerchange3) {
	printf("lexing test mode char c '%s'\n", (char *) lexerchange3->output);
	return 0;
}

int main(int argc, char **argv) {
	// lexer 1
	mpc_lexer_t ** lexer1 = mpc_lexer_new("lexer1");
	mpc_parser_t * L1 = mpc_new("lexer1 a");
	mpc_define(L1, mpc_char('a'));
	mpc_parser_t * L1a = mpc_new("lexer1 d");
	mpc_define(L1a, mpc_char('d'));
	mpc_parser_t * L1b = mpc_new("lexer1 e");
	mpc_define(L1b, mpc_char('e'));
	
	// lexer 2
	mpc_lexer_t ** lexer2 = mpc_lexer_new("lexer2");
	mpc_parser_t * L2 = mpc_new("lexer2 any");
	mpc_define(L2, mpc_any());
	mpc_parser_t * L3 = mpc_new("lexer2 c");
	mpc_define(L3, mpc_char('c'));
	
	// build the lexers
	mpc_lexer_add(&lexer1, L1, lexerchange1, &lexer2);
	mpc_lexer_add(&lexer1, L1a, lexerchange1a, NULL);
	mpc_lexer_add(&lexer1, L1b, lexerchange1b, NULL);
	
	mpc_lexer_add(&lexer2, L3, lexerchange3, &lexer1);
	mpc_lexer_add(&lexer2, L2, lexerchange2, NULL);
	
	// verify visually
	mpc_lexer_print(lexer1);
	mpc_lexer_print(lexer2);
	
	// run lexer1
	mpc_result_t r;
	mpc_lexer("abcde", lexer1, &r);
	
	// free the lexers
	mpc_lexer_free(lexer1);
	mpc_lexer_free(lexer2);
	
	
	return 0;
	
}
```
for this example we shall assign each parser there own function for extra clarity and ensure that the mode works and each function is called when it is supposed to be called, the expected outcome of this is as follows:

* first our lexer looks for character `a`, then changes from `lexer1` to `lexer2`
* in `lexer1` if `a` is not found it looks for char `d` then char `e`
* next `lexer2` looks for character `c`, then changes from `lexer2` back to `lexer1`
* in `lexer2`, if `c` is not found it finds any other character

the actual output if ran is this:
```
lexer1: parser: lexer1 a
lexer1: action: 0x55897938a76a
lexer1: mode: 0x55897b29b660

lexer1: parser: lexer1 d
lexer1: action: 0x55897938a79c
lexer1: mode: (nil)

lexer1: parser: lexer1 e
lexer1: action: 0x55897938a7ce
lexer1: mode: (nil)

lexer2: parser: lexer2 c
lexer2: action: 0x55897938a832
lexer2: mode: 0x55897b29b260

lexer2: parser: lexer2 any
lexer2: action: 0x55897938a800
lexer2: mode: (nil)

lexing test mode char a 'a'
changing modes
lexing test mode any char 'b'
lexing test mode char c 'c'
changing modes
lexing test mode char d 'd'
lexing test mode char e 'e'
```
which is correct for our expected output


Limitations & FAQ
=================

### Does _mpc_ support Unicode?

_mpc_ Only supports ASCII. Sorry! Writing a parser library that supports Unicode is pretty difficult. I welcome contributions!


### Is _mpc_ binary safe?

No. Sorry! Including NULL characters in a string or a file will probably break it. Avoid this if possible.


### The Parser is going into an infinite loop!

While it is certainly possible there is an issue with _mpc_, it is probably the case that your grammar contains _left recursion_. This is something _mpc_ cannot deal with. _Left recursion_ is when a rule directly or indirectly references itself on the left hand side of a derivation. For example consider this left recursive grammar intended to parse an expression.

```
expr : <expr> '+' (<expr> | <int> | <string>);
```

When the rule `expr` is called, it looks the first rule on the left. This happens to be the rule `expr` again. So again it looks for the first rule on the left. Which is `expr` again. And so on. To avoid left recursion this can be rewritten (for example) as the following. Note that rewriting as follows also changes the operator associativity.

```
value : <int> | <string> ;
expr  : <value> ('+' <expr>)* ;
```

Avoiding left recursion can be tricky, but is easy once you get a feel for it. For more information you can look on [wikipedia](http://en.wikipedia.org/wiki/Left_recursion) which covers some common techniques and more examples. Possibly in the future _mpc_ will support functionality to warn the user or re-write grammars which contain left recursion, but it wont for now.


### Backtracking isn't working!

_mpc_ supports backtracking, but it may not work as you expect. It isn't a silver bullet, and you still must structure your grammar to be unambiguous. To demonstrate this behaviour examine the following erroneous grammar, intended to parse either a C style identifier, or a C style function call.

```
factor : <ident>
       | <ident> '('  <expr>? (',' <expr>)* ')' ;
```

This grammar will never correctly parse a function call because it will always first succeed parsing the initial identifier and return a factor. At this point it will encounter the parenthesis of the function call, give up, and throw an error. Even if it were to try and parse a factor again on this failure it would never reach the correct function call option because it always tries the other options first, and always succeeds with the identifier.

The solution to this is to always structure grammars with the most specific clause first, and more general clauses afterwards. This is the natural technique used for avoiding left-recursive grammars and unambiguity, so is a good habit to get into anyway.

Now the parser will try to match a function first, and if this fails backtrack and try to match just an identifier.

```
factor : <ident> '('  <expr>? (',' <expr>)* ')'
       | <ident> ;
```

An alternative, and better option is to remove the ambiguity completely by factoring out the first identifier. This is better because it removes any need for backtracking at all! Now the grammar is predictive!

```
factor : <ident> ('('  <expr>? (',' <expr>)* ')')? ;
```


### How can I avoid the maximum string literal length?

Some compilers limit the maximum length of string literals. If you have a huge language string in the source file to be passed into `mpca_lang` you might encounter this. The ANSI standard says that 509 is the maximum length allowed for a string literal. Most compilers support greater than this. Visual Studio supports up to 2048 characters, while gcc allocates memory dynamically and so has no real limit.

There are a couple of ways to overcome this issue if it arises. You could instead use `mpca_lang_contents` and load the language from file or you could use a string literal for each line and let the preprocessor automatically concatenate them together, avoiding the limit. The final option is to upgrade your compiler. In C99 this limit has been increased to 4095.


### The automatic tags in the AST are annoying!

When parsing from a grammar, the abstract syntax tree is tagged with different tags for each primitive type it encounters. For example a regular expression will be automatically tagged as `regex`. Character literals as `char` and strings as `string`. This is to help people wondering exactly how they might need to convert the node contents.

If you have a rule in your grammar called `string`, `char` or `regex`, you may encounter some confusion. This is because nodes will be tagged with (for example) `string` _either_ if they are a string primitive, _or_ if they were parsed via your `string` rule. If you are detecting node type using something like `strstr`, in this situation it might break. One solution to this is to always check that `string` is the innermost tag to test for string primitives, or to rename your rule called `string` to something that doesn't conflict.

Yes it is annoying but its probably not going to change!


