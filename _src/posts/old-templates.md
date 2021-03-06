    Title: New syntax template features
    Date: 2018-11-22T00:00:00
    Tags: racket, macros

In Racket 7, the `syntax` form supports two new template subforms:

- `~@`[racket] ("splice") splices the result of its subtemplate (which must
  produce a syntax list) into the enclosing list template, and

- `~?`[racket] ("try") chooses between alternative subtemplates depending on
  whether their pattern variables have "absent" values.

These features originated in `syntax/parse/experimental/template`: the
`template` form was like an extended version of `syntax`, and `?@` and
`??` were the splicing forms and the try/else template forms,
respectively. The names were changed to `~@` and `~?` to avoid name
collisions with other libraries. In Racket 7, the old
`syntax/parse/experimental/template` library just exports the new
standard forms under the old names (that is, it exports `syntax` under
the name `template`, and so on).

<!-- more -->

## Splicing

The `~@` template form splices its result into the enclosing list
template. It is the template analogue of `syntax-parse`'s `~seq` form,
but it can be used apart from `syntax-parse`. It is useful for
producing syntax with logical groups that don't correspond directly to
parenthesized structure. One example of an unparenthesized logical
group is paired key and value arguments in a call to the `hash`
function:

```racket
(define-syntax-rule (zeros-hash key ...)
  (hash (~@ key 0) ...))
(zeros-hash 'a 'b 'c) => (hash 'a 0 'b 0 'c 0)
```

Another example is keyword arguments (that is, keyword and expression)
in a function call.


## Try/Else

The `~?` template form produces its first subtemplate's result if that
subtemplate has no "absent" pattern variables; otherwise it produces
its second subtemplate (if present) or nothing at all. Absent pattern
variables arise from `syntax-parse`'s `~or` and `~optional` forms, for
example.

For example, here is a macro based very loosely on the result of
`define-ffi-definer` from `ffi/unsafe`:

```racket
(define-syntax (definer stx)
  (syntax-parse stx
    [(_ name key (~optional (~seq #:make-fail make-fail)))
     #'(define name (lookup key (~? (make-fail 'name) default-fail)))]))

(definer x 'x)
=> (define x (lookup 'x default-fail))
(definer y 'y #:make-fail make-not-available)
=> (define y (lookup 'y (make-not-available 'y)))
```


## Potential problems and incompatibilities

There are three ways that the new `syntax` features and implementation
may cause old code to break or misbehave. In the period before the
Racket 7 release, we fixed the cases we discovered, but there may be
more out there. Here are descriptions of the potential problems:


### Nonexistent nested attributes

If `x` is an attribute bound by `syntax-parse` but `x.y` is not, then
it is now illegal for `x.y` to occur in a syntax template. The
rationale is that `x.y` is probably a mistake, an attempt to reference
a nested attribute of `x` that doesn't actually exist.

```racket
(define-syntax-class binding-pair
  (pattern [name:id rhs:expr]))
(syntax-parse #'[a (+ 1 2)]
  [b:binding-pair
   (list #'b.var    ;; ERROR: b.var is not bound
         #'b.rhs)]) ;; OK: b.rhs is a nested attr of b
```

The restriction does not apply to syntax pattern variables bound by
`syntax-case`, etc.


### Using `syntax` and `template` together

Macros that use `syntax` to produce a `template` expression may break,
because the interpretation of the new template forms will happen at
the outer (`syntax`) level instead of the inner (`template`)
level. (The template forms are recognized by binding, so `syntax` will
treat a reference to `?@` from `syntax/parse/experimental/template`
exactly the same as `~@`.).

Here's an example loosely based on the `define-ffi-definer`:

```racket
(define-syntax (define-definer stx)
  (syntax-case stx ()
    [(_ definer #:default-make-fail default-make-fail)
     #'(begin
         (define dmf-var default-make-fail)
         (define-syntax (definer istx)
           (syntax-case istx ()
             [(_ name value (~optional (~seq #:fail fail)))
              (template
               (define name
                 (lookup value (?? fail (dmf-var 'name)))))])))]))
```

Previously, the `??` would get ignored (that is, treated as a syntax
constant) by `syntax`; it would get noticed and interpreted by the
`template` form in the generated macro. Now it gets interpreted by the
outer macro and since the first subtemplate, `fail`, is just a syntax
constant from the *outer* macro's perspective, it always produces the
first subtemplate's result. So the inner macro will fail if used
without the `#:fail` keyword.

One fix is to escape the `??` using ellipsis-escaping:

```racket
.... (lookup value ((... ??) fail (dmf-var 'name))) ....
```

Another fix is to use `quote-syntax` and an auxiliary `with-syntax` binding:

```racket
(define-syntax (define-definer stx)
  (syntax-case stx ()
    [(_ definer #:default-make-fail default-make-fail)
     (with-syntax ([inner-?? (quote-syntax ??)])
     #'(begin
         (define dmf-var default-make-fail)
         (define-syntax (definer istx)
           (syntax-case istx ()
             [(_ name value (~optional (~seq #:fail fail)))
              (template
               (define name
                 (lookup value (inner-?? fail (dmf-var 'name)))))]))))]))
```

A better fix is to avoid having the outer macro generate the inner
macro's entire implementation and use a compile-time helper function
instead:

```racket
(begin-for-syntax
  ;; make-definer-transformer : Identifier -> Syntax -> Syntax
  (define ((make-definer-transformer dmf-var) istx)
    (syntax-case istx ()
      [(_ name value (~optional (~seq #:fail fail)))
       (with-syntax ([dmf-var dmf-var])
         (template
           (define name
             (lookup value (?? fail (dmf-var 'name))))))])))

(define-syntax (define-definer stx)
  (syntax-case stx ()
    [(_ definer #:default-make-fail default-make-fail)
     #'(begin
         (define dmf-var default-make-fail)
         (define-syntax definer
           (make-definer-transformer (quote-syntax dmf-var)))))]))
```

### Strict argument checking in syntax/loc

The second issue is not directly related to the new subforms. The
`syntax/loc` form applies the source location of its first argument to
the syntax produced by the template---but only if the outermost syntax
wrapping layer comes from the template and not a pattern variable. For
example, in the expression

```racket
(with-syntax ([x #'(displayln "hello!")])
  (syntax/loc src-stx x))
```

the location of `src-stx` is *never* transferred to the resulting
syntax object. If the old implementation of `syntax/loc` determined
that the first argument was irrelevant, it discarded it without
checking that it was a syntax object. So, for example, the following
misuse ran without error:

```racket
(with-syntax ([x #'(displayln "hello!")])
  (syntax/loc 123 x))
```

In Racket 7, `syntax/loc` is rewritten to handle the new
subforms---consider, for example, the template `(~? x (+ 1 2))`; the
source location of the second template should be overridden, but not
the first. But the new implementation always checks the first
argument. So some programs that previously got away with bad source
arguments will raise errors in Racket 7.
