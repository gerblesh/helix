## Adding [Steel] Plugins

Helix supports configuration and plugins via [Steel], an embedded [scheme] interpreter in Rust. Be sure you have compiled `helix-term` crate with the `"steel"` feature enabled, or have ran `cargo xtask steel` on the Helix repository (See [STEEL.md] for more information about setup).

For a more complete reference, you can see the generated [steel-docs.md] in the repository to see all of the functions that are exposed to [Steel].

## Setting up configurations for helix

There are 2 important files you'll want, which should be auto generated during the installation process if they don't already exist:

* `~/.config/helix/helix.scm`
* `~/.config/helix/init.scm`

> [!NOTE]
> Both of these live inside the same directory that helix sets up for runtime configurations.

### `helix.scm`

The `helix.scm` file is a [Steel] module, that will be loaded first before
anything else. The runtime will `require` this module, which means, that any
functions, that are exported via `provide` will now be available to be used as
typed commands. For example:

```scheme
; helix.scm

(require "helix/editor.scm")
(require (prefix-in helix. "helix/commands.scm"))
(require (prefix-in helix.static. "helix/static.scm"))

(provide shell git-add open-helix-scm open-init-scm)

(define (current-path)
  (let* ([focus (editor-focus)]
         [focus-doc-id (editor->doc-id focus)])
    (editor-document->path focus-doc-id)))

;;@doc
;; Specialized shell implementation, where % is a wildcard for the current file
(define (shell . args)
  (helix.run-shell-command
    (string-join
      ;; Replace the % with the current file
      (map (lambda (x) (if (equal? x "%") (current-path) x)) args)
      " ")))

;;@doc
;; Adds the current file to git	
(define (git-add)
  (shell "git" "add" "%"))

;;@doc
;; Open the helix.scm file
(define (open-helix-scm)
  (helix.open (helix.static.get-helix-scm-path)))

;;@doc
;; Opens the init.scm file
(define (open-init-scm)
  (helix.open (helix.static.get-init-scm-path)))
```

Now, if you'd like to add the current file you're editing to git, simply
type `:git-add` - you'll see the doc pop up with it since we've annotated the
function with a doc comment, which is `@doc` symbol. Hitting enter will execute
the command, like with the builtin.

You can now also conveniently open the `helix.scm` file by using the typed
command `:open-helix-scm`.

### `init.scm`

The `init.scm` file is run at the top level, immediately after the `helix.scm`
module is loaded via `require`. **This means, that all of the top-level `define`s
in this file will be available as a typed command.**

Additionally, since it is run _after_ the `helix.scm` file is `require`d you can
use all of the exported functions from there.

The helix context is available here, bound to the top level variable
`*helix.cx*`, so you can interact with the editor.

For example, if we wanted to select a random theme at startup:

```scheme
; init.scm

(require-builtin steel/random as rand::)
(require (prefix-in helix. "helix/commands.scm"))
(require (prefix-in helix.static. "helix/static.scm"))

;; Picking one from the possible themes
(define possible-themes '("ayu_mirage" "tokyonight_storm" "catppuccin_macchiato"))

(define (select-random lst)
  (let ([index (rand::rng->gen-range 0 (length lst))]) (list-ref lst index)))

(define (randomly-pick-theme options)
  ;; Randomly select the theme from the possible themes list
  (helix.theme (select-random options)))

(randomly-pick-theme possible-themes)
```

### Libraries for helix

There are a handful of extra libraries in development for extending helix, and can be found here <https://github.com/mattwparas/helix-config>.

If you'd like to use them, create a directory called `cogs` in your `.config/helix` directory, and copy the files in there.

### options.scm

If you'd like to override configurations from your toml config:

```scheme
; init.scm

(require "helix/configuration.scm")

(file-picker (fp-hidden #f))
(cursorline #t)
(soft-wrap (sw-enable #t))

```

### keymaps.scm

You can use the `keymap` macro provided in the `"helix/keymaps.scm"` module to
modify your keymap, like in the example below:

```scheme
; init.scm

(require (only-in "helix/keymaps.scm" keymap))

(keymap (global)
        (normal (C-s ":w") (C-o ":open ~/.config/helix/config.toml") (g (a "code_action")))
        (insert (A-x "normal_mode") (j (k "normal_mode"))))
```

You can also use this macro to add [Steel] functions to your keymap.
This example uses the [`mattwparas/recentf.hx`](https://github.com/mattwparas/recentf.hx) cog.

> [!NOTE]
> To add Steel functions to the keymap, they must be available as typable commands.  
> See the [`helix.scm`](#helixscm) and [`init.scm`](#initscm) sections for more info.

```scheme
; init.scm

(require (only-in "helix/keymaps.scm" keymap))
(require (only-in "cogs/recentf.scm" recentf-open-files))

(keymap (global) (normal (C-r (f ":recentf-open-files"))))
```

If you want your command to show a description in the keymap menu popups,
you will have to use an `@doc` doc comment, as explained in the [helix.scm
section](#helixscm). The first line of the doc comment will be used.

The `(global)` specifier specifies the scope in which the keymap is active, in
this case always.

In addition to the `(global)` specifier, there are two more: `(extension
"...")` and `(buffer "...")`, which make the keybindings only apply to specific
extensions or buffers, respectively.

```scheme
; init.scm

(require (only-in "helix/keymaps.scm" keymap))

(keymap (extension ".scm") (insert (ret ":scheme-indent") (C-l ":insert-lambda")))
```

This will override the `ret` keybinding in insert mode in `.scm` files to
instead use a custom scheme indent function.  
In this case the `scheme-indent` and the `insert-lambda` functions are defined
and exported in the `helix.scm` module.

## Hello World

Before you start, you should make sure that your configuration for the [Steel] lsp is wired up correctly. This will give you
access to the documentation that will help you as you write your plugin. To configure the LSP, you can add this to your
`init.scm`:

```scheme
; init.scm

(require "helix/configuration.scm")
(define-lsp "steel-language-server" (command "steel-language-server") (args '()))
(define-language "scheme"
                 (language-servers '("steel-language-server")))
```

This will give you an interactive setup that can help you run plugins as you go. I also like to evaluate commands
via the buffer, by either typing them in to the command prompt or by loading the current buffer. To load the current
buffer, you can type `:eval-buffer`, or to evaluate an individual command, you can run `:evalp` - note, in your init.scm, you
may need to add:

```scheme
; init.scm

(require (only-in "helix/ext.scm" evalp eval-buffer))
```

This brings those functions to the top level scope so that you can interact with them. You may also be keen to peruse all of the [Steel]
functions and modules available. Those can be found in `steel-docs.md`.

Here, we will get to grips with `forge`, [Steel]'s package manager. To start, we will create a new package with `forge` using `forge new`:


```shell
forge new hello-world && cd hello-world # creates a new Steel package named `hello-world` and changes the cwd to the package
```

We should now have a directory with two files:
  - `cog.scm`
  - `main.scm`

Where `cog.scm` is our package manifest file, if we look inside, we see that `cog.scm` contains the following:

```scheme
; cog.scm
(define package-name 'hello-world)

(define version "0.1.0")
(define dependencies '())

(define entrypoint '(#:name "hello-world" #:path "main.scm"))
```

Here, we see that our package manifest is also [Steel], additionally, since we are making a library, it makes sense to remove the `entrypoint` definition like so

```scheme
; cog.scm
(define package-name 'hello-world)

(define version "0.1.0")
(define dependencies '())
```

Moving on to `main.scm`:
```scheme
; main.scm
(displayln "Hello world!")
```
Start by moving `main.scm` to `hello-world.scm` by using `:mv hello-world.scm` in Helix.

We will start by adding some includes of the [Steel] libraries for helix, and removing the call to `displayln`

```scheme
; hello-world.scm
(require "helix/editor.scm")
(require "helix/misc.scm")
```

Now we will define the our first function, this one will just display "Hello World!" below our status line!
```scheme
; hello-world.scm
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
; init.scm
(require "hello-world/hello-world.scm")
```

Now, relaunching helix, we can type `:say-hello`, and our editor will say "Hello World!" back!

## Using Helix commands and keybinds

There are two levels of the functionality exposed to plugins. The first is simply based around
chaining builtin commands, as if you're a lightning fast human typing commands very quickly. The other level
is a bit lower, and deals directly with the component API that helix uses to draw the text editor and various
popups, like the file picker or buffer selection.

To understand the first level, which is accessing typed commands and static commands, i.e. commands that you
typically type via `:`, or static commands, commands which are bound to keybindings, you can look at the modules:

* helix/commands.scm
* helix/static.scm

Every function here implicitly has access to a context, the helix context. This assumes that you're focused onto
some buffer, and any actions are assumed to be done within that context. For example, calling `vsplit` will
split the currently focused into a second, and move your focus to that window. Keeping track of that is important
to understand where your focus is.

In general, these functions do not return anything, given that they're purely for side effects. There are some functions
that do, and they should be documented as such. The API will need to be improved to return useful things where relevant.

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
As is apparent, everything we can do within Helix we can start scripting from [Steel], making it possible to create various shortcuts. You can see [commands](../commands.md) for more information about the commands available here.

Another module we use is `editor.scm`, the editor module provides ways of interacting with the internal editor functions, such as setting the command count (as above), getting the current document, document text, and registering hooks.

Now let's say we want to bind our `insert-hello` command to a key, we can define our keymap in our `helix.scm` like so

```scheme
;; config/helix.scm
(require "helix/keymaps.scm")

(keymap (global) (normal (space (H ":insert-hello")))) ;; Keymap config to run ":insert-hello" when `<space>+H` is pressed in normal mode
```

## GUI Components

Another thing the Helix plugin system allows us to do is to add GUI components using the component system in the editor.
In our example we will be building a simple snippet list

To start, we will create the project:
```shell
forge new snippets-example && cd snippets-example
```
Make sure your `cog.scm` looks like this, notice how there's no `entrypoint` defined and the name is properly set.
```scheme
; cog.scm
(define package-name 'snippet-example)

(define version "0.1.0")
(define dependencies '())
```

Rename `main.scm` to be `snippets.scm` and we are set to start working.

Start by adding your requires at the top
```scheme
; snippets.scm
(require "helix/editor.scm")
(require "helix/misc.scm")
(require "helix/components.scm")
(require (prefix-in hx. "helix/static.scm"))
```
We use `prefix-in` to avoid namespacing issues when including large `scm` files like `static.scm`. Now whenever we use something from `static.scm`, we prefix it with `hx.`.

With components we need to store UI state, in Helix, we store the state directly in a mutable struct. It is also helpful to define some help text and a global snippets list, as our snippets should persist even when our component popped off the stack.
```scheme
(struct SnippetsState (selected) #:mutable) ; here, `selected` is the index of the currently selected snippet
(define TITLE-TEXT "Snippets: [0-9] paste a snippet, [c]lear snippets, [a]dd a snippet")
(define *snippets-list* '())
```

Next, we will start creating the UI component for our snippet list. Define a new function `open-snippets` like so
```scheme
;;@doc Open Snippets list
(define (open-snippets)
  (push-component!
   (new-component! "snippets" (SnippetsState 0) render-snippets (hash "handle_event" handle-event))))
```
Here, `new-component!` is a function taking in the name of the component (`"snippets"`), the state struct `(SnippetsState 0)`, makes a new `SnippetsState` with `0` as the selected index, the render function `render-snippets` and finally we have a hashmap containing the `handle-event` function.

We need to start defining these new functions, so lets run through them together. First, we have `render-snippets`

```scheme
  (define (render-snippets state area buf)
    (begin))
```
In the arguments, we take in the current state of the component `state`, which is of the type `SnippetsState`, our `area` is a `Rect`, containing positioning and sizing information of the screen, and `buf` a `Buffer` to write our UI into.

We can test this out by adding `frame-set-string!` which given a `Buffer`, x and y position, `String` to write, and a `Style` writes to the given buffer at the x and y position given, with the specified text and style.

```scheme
(define (render-snippets state area buf)
    (frame-set-string! buf (area-x area) (area-y area) TITLE-TEXT (style)))
```

Next, we should start defining our event handler function
```scheme
(define (handle-event state event)
  (cond
    [(key-event-escape? event) event-result/close]
    [else event-result/ignore]))
```
This will just close our component when pressing escape. Now to test our code, evaluate the buffer with `:eval-buffer`, and run `:open-snippets`. You should see `"Snippets: [0-9] paste a snippet, [c]lear snippets, [a]dd a snippet"` in the top left corner of the Helix window.

Next, we will implement the actual keybinds in the event handling, adding a case for each option.

```scheme
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
```

We first add another keybind to quit, using `key-event-char` in order to get the character from the key event. Notice how in our next `cond` block we can set our snippet state with `set-SnippetState-selected!` to set which snippet we have selected to paste. Our snippet plugin is now functionally complete, we just need to add a UI.

To start, we will make sure that our colors and styles are fetched from the theme properly using `theme-scope-ref`, and our popup window is centered.

```scheme
(define (render-snippets state area buf)
  (let* ([popup-style (theme-scope-ref "ui.popup")]
         [linenr-selected (theme-scope-ref "ui.linenr.selected")]
         [linenr (theme-scope-ref "ui.linenr")]
         [popup-area (calculate-popup-area area *snippets-list*)]
         [text-area (calculate-text-area popup-area)]))
    (buffer/clear buf popup-area)
    (block/render buf popup-area (make-block (style) popup-style "all" "plain"))
    (frame-set-string! buf (area-x text-area) (area-y text-area) TITLE-TEXT popup-style))
```
Here, we use 2 functions we have not qutie defined yet (`calculate-popup-area`, `calculate-text-area`). These two functions mainly serve to simplify our layout process. I defined them as such
```scheme
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
```

Then, we must finish off our `render-snippets` function by iterating through our global `*snippets-list*` and rendering them to our window.

```scheme
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
```

You can find the full code for this example on codeberg: https://codeberg.org/gwid/snippet-example.hx

## Statusline
With [Steel] bindings in Helix, we can also create custom statusline elements. For example, we can make a simple statusline element that displays the date.

```scheme
(require-builtin steel/time)
(require "helix/components.scm")
  
(define date-status-element
    (status-element (lambda (view-id focused)
                      (list (span (local-time/now! "%Y-%m-%d %H:%M:%S") (style))))))

(define (date-enable side)
  (push-status-element! side date-status-element))
```
The functions to pay attention to are `status-element`, and `span`, these functions, from `"helix/components.scm"` allow creating custom status elements. `status-element` constructs a custom status element from a given `render` function, in this case, our `lambda` which takes in the `view-id` and whether or not the statusline is focused. This `lambda` then returns a list of `Span` which are constructed from strings and a `Style`. We can test this by running `:eval-buffer` and then running `date-enable center`, to push our element to the center of the statusline.

## [Tree-sitter]
Helix also provides a [Tree-sitter] module `"helix/treesitter.scm"`. There are two ways of using the [Tree-sitter] module, you can either use an existing [Tree-sitter] parse tree from a document, or construct your own syntax tree. You can see the full reference doc for more information. For examples of plugins that use these bindings, you can can look at [context.hx] and [breadcrumbs.hx].

[Steel]: https://github.com/mattwparas/steel
[tree-sitter-queries]: https://tree-sitter.github.io/tree-sitter/using-parsers/queries/1-syntax.html
[tree-sitter-captures]: https://tree-sitter.github.io/tree-sitter/using-parsers/queries/2-operators.html#capturing-nodes
[textobject-examples]: https://github.com/search?q=repo%3Ahelix-editor%2Fhelix+path%3A%2A%2A/textobjects.scm&type=Code&ref=advsearch&l=&l=
[Tree-sitter]: https://tree-sitter.github.io/tree-sitter/
[Scheme]: https://www.scheme.org/
[steel-docs]: https://github.com/mattwparas/helix/blob/steel-event-system/steel-docs.md
[STEEL.md]: https://github.com/mattwparas/helix/blob/steel-event-system/STEEL.md
[context.hx]: https://codeberg.org/gwid/context.hx
[breadcrumbs.hx]: https://codeberg.org/gwid/breadcrumbs.hx
