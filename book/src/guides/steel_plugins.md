## Adding Steel Plugins

Helix supports configuration and plugins via [steel], an embedded scheme interpreter in Rust. Be sure you have compiled `helix-term` crate with the [steel] feature enabled, or have ran `cargo xtask steel` on the Helix repository (See [STEEL.md] for more information about setup).

For a more complete reference, you can see the generated [steel-docs.md] in the repository to see all of the functions that are exposed to [steel].

## Hello World

Here, we will get to grips with `forge`, [steel]'s package manager. To start, we will create a new package with `forge` using `forge new`:


```shell
forge new hello-world && cd hello-world # creates a new steel package named `hello-world` and changes the cwd to the package
```

We should now have a directory with two files:
  - `cog.scm`
  - `main.scm`

Where `cog.scm` is our package manifest file, if we look inside, we see that `cog.scm` contains the following:

```scheme
(define package-name 'hello-world)

(define version "0.1.0")
(define dependencies '())

(define entrypoint '(#:name "hello-world" #:path "main.scm"))
```

Here, we see that it is just typical scheme, additionally, since we are making a library, it makes sense to remove the `entrypoint` definition like so:

```scheme
(define package-name 'hello-world)

(define version "0.1.0")
(define dependencies '())
```

Moving on to `main.scm`:
```scheme
(displayln "Hello world!")
```
Start by moving `main.scm` to `hello-world.scm` by using `:mv hello-world.scm` in Helix.

We will start by adding some includes of the steel libraries for helix:

```scheme
;; hello-world/hello-world.scm
(require "helix/editor.scm")
(require "helix/misc.scm")
```

Now we will define the our first function, this one will just display "Hello World!" below our status line!
```scheme
;; hello-world/hello-world.scm
(require "helix/editor.scm")
(require "helix/misc.scm")

;;@doc
;; Saying hello from Steel!
(define (say-hello)
  (set-status! "Hello World!"))
(provide say-hello) ; Export as a typeable command to helix
```

We can now exit out of helix and run `forge install .` to install our library.

Now opening our `init.scm` in the helix config dir: `hx <config-dir>/init.scm`

We will add the following line


```scheme
;; config/init.scm
(require "hello-world/hello-world.scm")
```

Now, relaunching helix, we can type `:say-hello`, and our editor will say "Hello World!" back!

## Using Helix commands and keybinds

In order to use Helix commands we can use the `"helix/static.scm"` and `"helix/commands.scm"`, `static.scm` includes static editor commands like insertion, deletion, and goto. Similarly, `commands.scm` includes typeable commands that you invoke from `:` in command mode.

Let's create a simple function that uses the commands from `static.scm`. Our function will insert Hello World! and then go to the beginning.

```scheme

;; hello-world/hello-world.scm
(require "helix/editor.scm")
(require "helix/misc.scm")
(require (prefix-in helix.static. "helix/static.scm")) ; Here, we use prefix-in to avoid collisions with Helix builtins

;;@doc
;; Saying hello from Steel!
(define (say-hello)
  (set-status! "Hello World!"))
(provide say-hello)

;;@doc
;; Insert "Hello" into the document and return to where the cursor was
(define (insert-hello)
  (helix.static.insert_string "Hello World!")
  (set-editor-count! (string-length "Hello World!")
  (helix.static.move_char_left)))
(provide insert-hello) ; Export as a typeable command to helix
```
As is apparent, everything we can do within Helix we can start scripting from steel, making it possible to create various shortcuts. You can see [commands](../commands.md) for more information about the commands available here.

Another module we use is `editor.scm`, the editor module provides ways of interacting with the internal editor functions, such as setting the command count (as above), getting the current document, document text, and registering hooks.

Now let's say we want to bind our `insert-hello` command to a key, we can define our keymap in our `helix.scm` like so

```scheme
;; config/helix.scm
(require "helix/keymaps.scm")

(keymap (global) (normal (space (H ":insert-hello")))) ;; Keymap config to run ":insert-hello" when `<space>+H` is pressed in normal mode
```

## GUI Components and Statusline

Another thing the Helix plugin system allows us to do is to add GUI components using the component system in the editor.
In our example we will be building a simple snippet list

To start, we will create the project:
```shell
forge new snippets-example && cd snippets-example
```
Make sure your `cog.scm` contains something like:
```scheme
;; cog.scm
(define package-name 'snippet-example)

(define version "0.1.0")
(define dependencies '())
```

Rename `main.scm` to be `snippets.scm` and we are set to start working.

Start by adding your requires at the top
```scheme
;; snippets.scm
(require "helix/editor.scm")
(require "helix/misc.scm")
```

Then, draw the rest of the owl...
```scheme
;; snippets.scm
(require "helix/editor.scm")
(require "helix/misc.scm")
(require "helix/components.scm")
(require (prefix-in hx. "helix/static.scm"))

(struct SnippetsState (selected) #:mutable)

(define TITLE-TEXT "Snippets: [0-9] paste a snippet, [c]lear snippets, [a]dd a snippet")
(define *snippets-list* '())

(define (calculate-popup-area rect snippets)
  (let* ([rect-width (area-width rect)]
         [rect-height (area-height rect)]
         [width (max (+ (if (not (empty? snippets))
                                (max (apply max (map (lambda (x) (string-length x)) snippets)) 8)
                                (string-length TITLE-TEXT))
                            6)
                     (- rect-width 4))]
         [height (min (+ (max (length snippets) 1) 3)
                      (- rect-height 4))]
         [x (ceiling (max 0 (- (ceiling (/ rect-width 2)) (floor (/ width 2)))))]
         [y (ceiling (max 0 (- (ceiling (/ rect-height 2)) (floor (/ height 2)))))])
    (area (- x 1) (- y 1) width height)))


(define (calculate-text-area popup-area)
  (let ([padding-x 2]
        [padding-y 1])
    (area (+ (area-x popup-area) padding-x)
          (+ (area-y popup-area) padding-y)
          (- (area-width popup-area) (* padding-x 2))
          (- (area-height popup-area) (* padding-y 2)))))

(define (render-snippets state area buf)
  (let* ([popup-style (theme-scope-ref "ui.popup")]
         [linenr-selected (theme-scope-ref "ui.linenr.selected")]
         [linenr (theme-scope-ref "ui.linenr")]
         [popup-area (calculate-popup-area area *snippets-list*)]
         [text-area (calculate-text-area popup-area)])
    (buffer/clear buf popup-area)
    (block/render buf popup-area (make-block (style) popup-style "all" "plain"))
    (frame-set-string! buf (area-x text-area) (area-y text-area) TITLE-TEXT popup-style)
    (for-each (lambda (i)
                (begin
                  (frame-set-string!
                    buf
                    (+ (area-x text-area) 2) (+ (area-y text-area) i 1)
                    (list-ref *snippets-list* i)
                    popup-style)
                  (frame-set-string!
                    buf
                    (+ (area-x text-area)) (+ (area-y text-area) i 1)
                    (number->string (modulo (add1 i) 10))
                    (if (= i (SnippetsState-selected state)) linenr-selected linenr))))
              (range (length *snippets-list*)))))

(define (handle-event state event)
  (cond
    [(key-event-escape? event) event-result/close]
    [(equal? (key-event-char event) #\q) event-result/close]
    [(number? (char->number (or (key-event-char event) #\null)))
     (let* ([i (modulo (sub1 (char->number (key-event-char event))) 10)]
            [snip (try-list-ref *snippets-list* i)])
        (if (string? snip)
            (begin
              (set-SnippetsState-selected! state i)
              (set-status! (string-append "Inserted: " snip))
              (hx.insert_string snip)
              event-result/consume)
            (set-status! "Unable to insert!")))]
    [(equal? (key-event-char event) #\a) (begin
       (push-component! (prompt "Add snippet:"
                                (lambda (snip)
                                  (if (> 10 (length *snippets-list* ))
                                    (begin
                                      (set-status! (string-append "Snippet added"))
                                      (set! *snippets-list*
                                        (cons snip *snippets-list*)))
                                    (set-status! (string-append "Too many snippets! Try clearing with [c]"))))))
       event-result/consume)]

    [(equal? (key-event-char event) #\c) (begin (set! *snippets-list* '()) (set-status! "Snippets cleared!") event-result/consume)]
    [else event-result/ignore]))


;;@doc Open Snippets list
(define (open-snippets)
  (push-component!
   (new-component! "snippets" (SnippetsState 0) render-snippets (hash "handle_event" handle-event))))
```

## [Tree-sitter]

Helix also provides a [Tree-sitter] module `"helix/treesitter.scm"`. There are two ways of using the [Tree-sitter] module, you can either use an existing [Tree-sitter] parse tree from a document, or construct your own syntax tree

Here's a couple of common examples of things you may want to do with treesitter.

```scheme

  (document->tree-byte-range doc-id pos pos)
  (document->tree doc-id)
  (document->layers-byte-range)
  (query-document)
  (query-document-byte-range)

  (TSTree?)
  (tstree->language tree)
  (tstree->root)
  (tsquery-loader)

  (query-tssyntax)
  (query-tssyntax-byte-range)

  (TSQuery?)
  (TSSyntax?)
  (TSMatch?)
```


## Text API

[steel]: https://github.com/mattwparas/steel
[tree-sitter-queries]: https://tree-sitter.github.io/tree-sitter/using-parsers/queries/1-syntax.html
[tree-sitter-captures]: https://tree-sitter.github.io/tree-sitter/using-parsers/queries/2-operators.html#capturing-nodes
[textobject-examples]: https://github.com/search?q=repo%3Ahelix-editor%2Fhelix+path%3A%2A%2A/textobjects.scm&type=Code&ref=advsearch&l=&l=
[Tree-sitter]: https://tree-sitter.github.io/tree-sitter/
