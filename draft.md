# Roverbanks

## What is Roverbanks

It's a Turing-complete language for string content transformation and flow.

String content is called a frame. During execution, this content will be transformed into other content, continuously, forming a real-time flow.

For this flow, a form of observer pattern is used, somewhat like a real-time spreadsheet.

In Roverbanks, there are several "banks", holding content. A bank is just a folder containing the source code files of the system. In each bank, there's an "index" file that acts as the entry point for the bank.

There are 3 types of files:

- *.ZRFF - Roverbanks Frame Format,
- *.ZRSF - Roverbanks Script Format,
- *.ZRTF - Roverbanks Table Format

The entry point of a bank depends on the type of its index file:

- index.zrff -> the bank is a frame,
- index.zrsf -> the bank is a script,
- index.zrtf -> the bank a table.



## Data types

| Type | Syntax | Notes |
|---|---|---|
| Table | `( )` | array, object/dictionary, and set at once - access style determines which |
| Script | `{ }` | a sequence of computations |
| Frame | `[ ]` | text content, with scripts and tables inclusion |
| Source | `$`prefix | source, a reference to external content (see below) |

Texts and numbers belong to the "frame" type.



## Sources

A source is a resource containing Roverbanks code, an I/O stream, or something else.

It is identified by a `$` prefixing a URL, that can be given as a literal or by a variable.

Banks are themselves sources too. Generally speaking, a source is "what's outside" a given bank.

For external connections, sources can be viewed as a local repository mirroring content that is elsewhere, not inside the Roverbanks engine.

To access the content of a source for reading or writing, you give the URL of the source prefixed with `$`.

```
    $[lib/my source code.zrsf] -> [my_code]
```

Where a source is read, it is considered as "observed" in the context of the execution model.



## Execution model

When the value of a bank (as defined by its index) is updated, it triggers an update of all banks observing it.

The Roverbanks engine is built around a central task queue. When an update is triggered, a task identifying that update is simply added to the queue, if not already present.

Tasks are executed serially.

When a table, a script, or a frame, gets updated, its new value is computed and available for other parts of the system. This new value includes the URLs of the sources this table/script/frame reads from or writes to, if any.



## Tables

A table is at once an array, an object or dictionary, and a set. The distinction is made by the choice of functions used to access it for reading/writing.

The key of a table is always converted to a frame.

```
    (
        one,
        [two and a half],
        3, 4, 5,
        seven: 7,
        {4 + 4}: [eight],
        nine: ([some], [sub], [table])
    )
```

As syntactic sugar, square brackets are not required around a key if it contains a frame only made of alphanumeric characters and underscores.



## Scripts

A script is denoted between curly braces.

The value of a script is a frame containing the results of all its computations (and not simply the last one obtained, or the value indicated by a "return" as is often the case).

However, if the script manipulates multiple universes (sets of values), the resulting value will be a table of frames, rather than a single frame.

```
    {
        [sum] <- [{ {0} + {1} }]

        [MATH:]
        [4 + 4 =]
        sum((4, 4))
    }[0]

=>  [MATH: 4 + 4 = 8]
```

- First the text `{ {0} + {1} }` is assigned to a variable identified by the string `sum`. This assignment doesn't append any value by itself.
- Then the text `MATH:` is appended as the first part of the resulting value of the script.
- Same for the text `[4 + 4 =]`.
- Then, a table of tables `((4, 4))` is applied to the value of the `sum` variable, which contains the source code of the script `{ {0} + {1} }`.
- This table of tables contains only one table, which is interpreted as one universe (one set of values).
- The application of this table of 1 universe to this script triggers the evaluation of the script, which is rendered as a table of 1 solution corresponding to the only given universe. Conceptually, this event *forks* the containing script evaluation (but in this case obviously, the fork contains only one universe).
- The main containing script is then rendered as a table of values, containing the only value produced: `([MATH: 4 + 4 = 8])`
- We then ask for the first of these values using `[0]`, which renders the final `[MATH: 4 + 4 = 8]`.

When a script is evaluated and thus rendered as a frame, multiple consecutive newlines and spaces are replaced by a single space. That's why in our example, `MATH:` and `[4 + 4 =]` end up separated only by a space, instead of a newline plus indentation.



## Frames

A frame is denoted between brackets.

The frame is a value of type string.

It can contain:
- text,
- a number,
- a table,
- scripts, which will be replaced by their value, and
- nested frames, which will be embedded as-is.

```
    [The result of [{ 4 + 4 }] is { 4 + 4 }.]

=>  [The result of { 4 + 4 } is 8.]
```

A frame is therefore a *quasi-quoting* element, with included scripts acting as the "unquoted parts" of the quoted content.

When a frame is rendered, newlines and spaces are preserved exactly (unlike what happens when a script is converted to a frame).

A frame containing a number will be treated as a number when the context needs a number, for example when doing an addition.



## Scopes

Scripts and frames have zero or more scopes.

A scope is a table that contains the variables that are available.

We sometimes use the term "multiscope", because when several scopes are present, they provide different sets of values for the variables, essentially forming a "multiverse" of possibilities.



## Variables

Inside a frame, there are variables that can be called by their name. To assign a value to a variable, we use assignment `->` or `<-`, works both ways.

```
    [
        { [total] <- 5 + 5 }

        The result is {total}.
    ]

=>  [
    

        The result is 10.
    ]
```

As can be seen, assignment is not a special form but a normal operator, which is why the variable name has to be given as text.



## Accessing the content of a table: `(...)[]`

You can access the content of a table by following it with a frame with no space separating them.

```
    (
        one,
        [two and 3],
        4, 5,
        seven: 7,
        {4 + 4}: [eight],
        nine: ([some], [sub], [table], [= # < 0]: [negative])

    )[/nine/0]

=>  [some]
```

If the 1st character of the address is a slash, this activates graphmaster mode. In this mode:
- The address is slash/separated. At each slash: child node.
- The keys of the tables read can be treated as wildcards.

Wildcards below are listed in increasing priority - an exact match always wins over a fallback, except a formula:

| Wildcard | Description (increasing priority) |
|---|---|
| `*` | Default key: used if no match is found |
| `%` | Default index: used if no numeric key is found |
| `\|a\|b` | Synonyms: a key having several "names" |
| `abc123` | Exact key or index |
| `=(n < # * 2.5)` | A boolean formula testing the key denoted by `#` |



## Calling a script with arguments: `{...}()`

This is the same as applying a table to a frame, see next section.



## Applying a table to a frame: `[...]()`

It is possible to "apply" a table to a frame, by following the frame with the table with no space separating them.

```
    [
        This {bar} a {foo}.
    ](
        ( foo: [pipe], bar: [is not] ),
        ( foo: [sentence], bar: [is] ),
    )

=>  (
        [This is not a pipe.],
        [This is a sentence.]
    )
```

The tables provided as a suffix are accessible throughout the frame, forming the "local scope" of the frame. This is therefore a multi-layer scope, each layer containing a possible universe. This is the frame's "multiscope".

It can also be done anonymously.

```
    [
        This {1} a {0}.
    ](
        ( [pipe], [is not] ),
        ( [sentence], [is] ),
    )

=>  (
        [This is not a pipe.],
        [This is a sentence.]
    )
```



## Applying a table to a table of frames: `(...)()`

A table can be applied to a list of frames to deduce a frame and its multiscope. This is a pattern generator.

```
    (
        [This is not a pipe.],
        [This is a sentence.]

    )()

=>  (
        [
            This {0} a {1}
        ],
        (
            ( [is not], [pipe.] ),
            ( [is], [sentence.] ),
        )
    )
```

Names can also be provided for the lvars. Order is preserved. If there aren't enough names, it continues in anonymous mode (e.g. foo, bar, 0, 1, 2...).

```
    (
        [This is not a pipe.],
        [This is a sentence.]

    )([foo], [bar])

=>  (
        [
            This {foo} a {bar}
        ],
        (
            ( foo: [is not], bar: [pipe.] ),
            ( foo: [is], bar: [sentence.] ),
        )
    )
```



## Manipulating local multiscope variables

A script contained in a frame can also read and write variables in the scope of that frame.

Example:
```
    [
        {
            [result] <- {0} * 2 ;
        }
        The double of {{0}} is {result}.
    ]((4))

=>  [
    
        The double of 4 is 8.
    ]
```



### Variables and literals

In the script, we need:
- a way to call a variable (whose name is a text)
- a way to express a value (of text type)

variable: `my_variable`
value: `[my_value]`



### Assignment

Next we need:
- a way to assign a value to a variable

we use:
- an infix operator `->`: the variable on the right takes the value on the left
- an infix operator `<-`: the variable on the left takes the value on the right

Returns the value: `{}`
That is: nothing, which is not the same thing as `()`.



### Separator

Next we need:
- a way to separate expressions

We use:
- the semicolon
- the newline



### Conversions

Now we have conversion choices to make.

| Old type | New type | Description |
|---|---|---|
| none | script | The empty script. |
| none | table | The empty table. |
| none | frame | The empty frame. |
| script | table | An array of values (each expression). |
| script | frame | A concatenation of values (each expression). |
| table | script | The table itself, as a literal value. |
| table | frame | The table itself, as a literal value. |
| frame | script | The frame itself, as a literal value. |
| frame | table | A 1-item table. |

`{}` is the literal meaning "nothing".

`[]` is a frame that is empty.

`()` is a table that is empty.



### Applications

When one type is applied to another, the application itself is semantically meaningful, and has priority over the semantics of the 2 types.

| Application | Description |
|---|---|
| `{script}(table)` | Provides a multiscope to a script. |
| `(table)(table)` | Deduces a multiscope and a table (pattern gen). |
| `(table)[frame]` | Accesses an element of the table + graphmaster mode. |
| `[frame](table)` | Provides a multiscope to a frame. |
| `[frame][frame]` | Searches for a pattern in a frame. |

For example, when a table is applied to a script, the engine first handles the application, and only *in the context of this application* later handles the table and the script.

```
( [], [], [], ...)( ) == ( [], ( (), (), (), ...) )
<list of strings>  =>  <1 pattern> + <N universes>

[]( (), (), (), ...) == ( [], [], [], ...)
<1 pattern> + <N universes> => <list of strings>
```



## Loop: `#` and `@`

A loop is built from three ingredients:

1. An expression: what's calculated on each pass, any script, frame, or table, written using `#`.
2. `#`, the current-item reference: inside this expression, `#` stands the current item being processed. It changes on every pass, the way a loop variable would in another language.
3. `@ <source>`, the table being walked: The loop runs once per item in this table, in order.

The general shape is:

```
<expression using #> @ <table>
```

The loop's overall value is a table, as in usual iteration.

**Simplest case - a straight numeric transform:**

```
    # / (1 + #) @ (1, 2, 3)

=> (0.5, 0.666, 0.75)
```

Here `@ (1, 2, 3)` supplies three items. On each pass `#` is bound to `1`, then `2`, then `3`, and the result expression `# / (1 + #)` is evaluated with that binding, producing one output per item.

**Templated case - substituting into frames:**

```
{
    [test] <- (
        [the car is {what} now],
        [{what} is here]
    )

    #((what: mine)) @ test
}

=>  (
        [the car is mine now],
        [mine is here]
    )
```

Here the source is `test`, a table of two template frames. On each pass, `#` is the current template frame, and `[mine]` is applied to it as the substitution value - filling in the `{what}` placeholder - giving one filled-in frame per item of `test`.



## Operators

WIP

Listed from lowest to highest precedence.

| Precedence | Operator | Description |
|---|---|---|
| 0 (lowest) | `#` `@` | Loop: `#` is the current-item reference, `@` binds the source table |
| 1 | `->` `<-` | Assignment. Value flows in the direction of the arrow. |
| 2 | `?` `:` | Ternary: `(cond ? then : else)` |
| 3 | `\|\|` | Logical OR, lazy |
| 4 | `&&` | Logical AND, lazy |
| 5 | `!` | Logical NOT (unary) |
| 6 | `==` `!=` `<` `>` `<=` `>=` | Equality and comparison (same tier; left-to-right). When texts are compared, the result indicate inclusion |
| 7 | `+` `-` | Addition, subtraction, concatenation, removal |
| 8 | `*` `/` `%` | Multiplication, division, modulo. When text is divided, it is split by text and results in a table |
| 9 | `^` | Exponentiation (right-associative) |
| 10 | `-` (unary) | Numeric negation, e.g. `-4` |
| 11 (highest) | `()` `[]` `{}` | Grouping / table access / frame application (see relevant sections above) |



## Using graphmaster mode in a scope

It is legal to use graphmaster mode for accessing variables in the scope. In the example below, we access the variable `15` in graphmaster mode.

```
    {
        [=(# > threshold)] <- [above]
        [=(# <= threshold)] <- [not above]

        [threshold] <- 10
        
        [Before, we're {/15}.]

        [threshold] <- 20
        
        [After, we're {/15}.]
    }

=>  [Before, we're above. After, we're not above.]
```



## Using a table as a graph language

A table can easily be used to describe a directed graph, like the one that constitutes the architecture of the system itself, simply by declaring the edges as key-value pairs.

