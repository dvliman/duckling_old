# What is Picsou

Picsou parses text into structured data:

    “in two hours” => {:dim :time 
                       :value {:from 2014-06-09T13:24:06.634-07:00
                               :to 2014-06-09T14:24:06.634-07:00}}

Picsou is agnostic: it makes no assumption on the kind of extracted data. It takes a string as input and produces hashes (more precisely, Clojure maps) as output. Everything else is defined in configuration. Picsou was first written to handle temporal expressions, but it can be used for other purposes.

Picsou is probabilistic. Often a given input string may produce dozens of potential results. Picsou assigns a probability on each result. It decides which results are more probable by using the corpus given in the configuration.

If you know NLP, Picsou is “almost” a PCFG. But formally it’s not, because Picsou tries to be much more flexible and easier to configure than a formal PCFG.

# Getting started

Launch a REPL from the project directory (you need to have Leiningen installed):

```
→ lein repl
nREPL server started on port 59436
REPL-y 0.1.9
Clojure 1.5.1
    Exit: Control+D or (exit) or (quit)
Commands: (user/help)
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
          (user/sourcery function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
Examples from clojuredocs.org: [clojuredocs or cdoc]
          (user/clojuredocs name-here)
          (user/clojuredocs "ns-here" "name-here")
picsou.core=>
```

Load Picsou with the default configuration file:

```
picsou.core=> (load!)
INFO: Loading picsou config  :fr$datetime
INFO: Loading picsou config  :en$datetime
INFO: Loading picsou config  :es$datetime
nil
```

Run the corpus and check that all the tests pass:

```
picsou.core=> (run)
OK  "ahora"
OK  "ya"
OK  "ahorita"
OK  "hoy"
OK  "en este momento"
OK  "ayer"
OK  "anteayer"
OK  "antier"
OK  "mañana"
OK  "pasado mañana"
OK  "lunes"
OK  "lu"
OK  "lun."
OK  "este lunes"
OK  "martes"
[...]
282 examples, 0 failed.
Results :
:es$datetime: 269 examples, 0 failed.
:en$datetime: 264 examples, 0 failed.
:fr$datetime: 282 examples, 0 failed.
Global Failed count:  0
nil
```

See the detailed parsing of a given string like "in two hours":

```
picsou.core=> (play :en$datetime "in two hours")
------------  11 | time      | in/after <duration>       | P =  0.0000 |  + <integer> <unit-of-duration>
   ---        10 | distance  | number as distance        | P =  0.0000 | integer (0..19)
   ---         9 | temperature | number as temp            | P =  0.0000 | integer (0..19)
   ---------   8 | duration  | <integer> <unit-of-duration> | P =  0.0000 | integer (0..19) + hour (unit-of-duration)
   ---         7 | null      | number (as relative minutes) | P =  0.0000 | integer (0..19)
   ---         6 | time      | <integer> (latent time-of-day) | P =  0.0000 | integer (0..19)
   ---         5 | time      | month (numeric)           | P =  0.0000 | integer (0..19)
   ---         4 | time      | day of month (numeric)    | P =  0.0000 | integer (0..19)
       -----   3 | unit-of-duration | hour (unit-of-duration)   | P =  0.0000 |
       -----   2 | cycle     | hour (cycle)              | P =  0.0000 |
   ---         1 | number    | integer (0..19)           | P =  0.0000 |
in two hours
12 tokens in stash
[...]
```

# Workflow

When we extend Picsou's coverage, we follow a typical TDD workflow:

1. Load Picsou
2. Add tests in the corpus
3. Run the corpus: the new tests don’t pass
4. Add or modify rules until the corpus tests pass

The following sections detail these steps.

# Loading

## Configuration file

The default configuration file `resources/default-config.clj` defines three modules (`fr$datetime`, `en$datetime` and `es$datetime`):

```Clojure
{:fr$datetime {:corpus ["fr.time"
                        "fr.numbers"
                        "fr.temperature"
                        "fr.finance"
                        "en.communication"]
               :rules  ["fr.time"
                        "fr.numbers"
                        "fr.cycles"
                        "fr.duration"
                        "fr.temperature"
                        "en.finance" ; sic
                        "en.communication"]} ; sic
 :en$datetime {:corpus ["en.time"
                        "en.numbers"
                        "en.temperature"
                        "en.finance"
                        "en.communication"]
               :rules  ["en.time"
                        "en.numbers"
                        "en.cycles"
                        "en.duration"
                        "en.temperature"
                        "en.finance"
                        "en.communication"]}
 :es$datetime {:corpus ["es.time"
                        "es.numbers"
                        "es.temperature"
                        "es.finance"
                        "en.communication"]
               :rules  ["es.time"
                        "es.numbers"
                        "es.cycles"
                        "es.duration"
                        "es.temperature"
                        "en.finance"
                        "en.communication"]}}
```

## Modules

Each module has a name (`en$datetime`), with which it is referred to when you want to use it at runtime, or reload it.

Each module refers to a set of corpus files and rules files (more on this in the following sections).

Each module is run by Picsou in a separate “sandbox”, so for example, rules in module A cannot expect to match tokens created by rules in module B. There’s typically one module per language, but nothing prevents you to use several modules for a given language, as long as these modules don't need to interact with each other.

## Loading modules

To load Picsou with all the modules defined in the default configuration file `resources/default-config.clj`:

```
picsou.core=> (load!)
INFO: Loading picsou config  :fr$datetime
INFO: Loading picsou config  :en$datetime
INFO: Loading picsou config  :es$datetime
nil
```

Alternatively, to load Picsou without using a configuration file, you can define modules directly in the `load!` function arguments:

```
(load! {:en$location {:corpus ["en.location"] :rules ["en.location"]}})
INFO: Loading picsou config  :en$location
```

# Corpus

Corpus files are located in `resources/corpus`. You can either edit existing files or create new files. If you create new files, don’t forget to load them by referencing them in your configuration file or `load!` command (see above). **Once you’ve modified corpus files, you must reload to take the changes into account.**

Here is an example corpus file with two test groups:

```Clojure
(
  {} ; Context map

  "0"
  "naught"
  "zero"
  (number 0)

  "1"
  "one"
  (number 1)
)
```

Each test group is described by one or more strings and a function. To run the group Picsou will take each string one by one, analyze it, a call the function on the output. The test passes if the function returns true (or a truthy value).

For instance, to test that “0”, “naught” and "zero" will all produce the output `{:dim :number :value 0}`, we can use:

```Clojure
“0”
“naught”
“zero”
(fn [token context] (and (= :number (:dim token)) (= 0 (:value token))))
```

For now, the context is just used for date and times, in order to solve relative dates like “tomorrow”. You can provide a context map at the beginning of your corpus file, and this map will be provided to the test function. In most cases, you shouldn’t need to use context.

In practice, we use helpers to generate easy to read test functions. In the previous example, we use a helper `number` defined in `src/picsou/corpus.clj`:

```Clojure
(defn number
  "check if the token is a number equal to value.
  If value is integer, it also checks :integer true"
  [value]
  (fn [token _] (and
                  (= :number (:dim token))
                  (or (not (integer? value)) (:integer token))
                  (= (:val token) value))))
```

So that the test becomes just `(number 0)`, which is easy to read and reusable.

Picsou will frequently generate several possible results for a given input. In this case, each result is tested by the test function. If the function returns true for at least one result, then the test passes.

Once you’ve added your tests, reload your module (see above) and run the corpus:

```
picsou.core=> (run :en$datetime)
OK  "now"
OK  "right now"
OK  "just now"
[...]
OK  "0"
OK  "naught"
OK  "nought"
OK  "zero"
FAIL"boule a zero" none of the 0 winners did pass the test
OK  "1"
OK  "one"
[...]
265 examples, 1 failed.
[:en$datetime 265 1]
```

Make sure the tests don’t pass anymore (if they do, either you’re very lucky and the existing rules actually cover your new tests, or you did not reload the corpus -- usually it’s the latter!). Now you’re ready to write rules.

# Rules

Rules files are located in `resources/rules`. You can either edit existing files or create new files. If you create new files, don’t forget to load them by referencing them in the configuration file or `load!` command. **Once you’ve modified rules files, you must reload to take the changes into account.**

Here is an example file with just one rule:

```Clojure
("zero"                                ; _label_ of the rule, useful for debugging
 #”0|zero|naught”                      ; _pattern_, here it’s a simple regex
 {:dim number :integer true :val 0})   ; _production_ token, it can be any map
```

When the pattern is matched, the production token is produced. Picsou adds this new token to its collection of tokens, which is called the “stash”. Then other rules can try to match this token and produce other tokens that are added to the stash, and so on. All rules are tried again and again until no more token is produced.

Here is an illustration of this process, with a stash containing 11 tokens:

```
picsou.core=> (play :en$datetime "in two hours")
------------  11 | time      | in/after <duration>       | P =  0.0000 |  + <integer> <unit-of-duration>
   ---        10 | distance  | number as distance        | P =  0.0000 | integer (0..19)
   ---         9 | temperature | number as temp            | P =  0.0000 | integer (0..19)
   ---------   8 | duration  | <integer> <unit-of-duration> | P =  0.0000 | integer (0..19) + hour (unit-of-duration)
   ---         7 | null      | number (as relative minutes) | P =  0.0000 | integer (0..19)
   ---         6 | time      | <integer> (latent time-of-day) | P =  0.0000 | integer (0..19)
   ---         5 | time      | month (numeric)           | P =  0.0000 | integer (0..19)
   ---         4 | time      | day of month (numeric)    | P =  0.0000 | integer (0..19)
       -----   3 | unit-of-duration | hour (unit-of-duration)   | P =  0.0000 |
       -----   2 | cycle     | hour (cycle)              | P =  0.0000 |
   ---         1 | number    | integer (0..19)           | P =  0.0000 |
in two hours
[...]
```

## Patterns

### Base patterns

There are two types of base patterns:
- regular expressions that try to match the input text
- functions that try to match tokens in the stash

Any function accepting one token as argument (a Clojure map) can work as a pattern. It must return true when the token matches. For example:

```Clojure
; this pattern will match a token with :dim :number whose :val is 0
(fn [token] (and (= :number (:dim token)) (= 0 (:val token))))
```

**Protip:** These patterns are very close, but should not be confused with Corpus test patterns. We might merge them later.

### Helpers

Like for corpus test functions, you’ll find yourself using the same patterns again and again. We use helpers that produce pattern functions. For instance

```Clojure
(number 3) ; => (fn [token] (and (= :number (:dim token)) (= 3 (:val token))))

(dim :number) ; => (fn [token] (= :number (:dim token)))
```

You should reuse existing helpers or define your own as much as possible, as it makes the rules much easier to read.

**Protip:** Using (dim :number) is better than a regex like #”\d+”, because if will match any number even “twenty”, “minus six”, “2M”, etc. You actually leverage other Picsou rules that are just responsible to recognize numbers.

### Slots

Let’s say you want to parse something like “10 degrees”, “twenty degrees”, and “30°”. The right approach is to look for a token of `:dim :number`, immediately followed by a word like “degrees” or “°”. In this case, we say the pattern has two *slots*. It is written like this:

```Clojure
[(dim :number)   ; first slot is a token with :dim :number
 #”degrees?|°”]  ; second slot is the string "degree", "degrees" or "°"" in the input string
```

## Production

Once a rule’s pattern matches, Picsou creates a token and adds it to the stash.

In its simplest form, the production is just the token to produce:

```Clojure
{:dim :number 
 :integer true
 :val 0}
```

But what if the product token is a function of a token matched by the pattern? You can use %1, %2, ... %S to represent the tokens matched in the S slots:

```Clojure
“<n> degrees"                ; label
[(dim :number) #”degrees?”]  ; pattern (2 slots)
{:dim :temperature           ; production
 :degrees (:val %1)}
```

**Protip:** Internally, the production form is expanded with `#(...)`. It becomes a function, which is called with the matching tokens as arguments.

**Warning:** If the pattern has S slots, you MUST use `%S` (even if you don't need it) if you need any %i. That will set the right arity to the production function.

### Special case of regex patterns

If the base pattern is a regex and you need to use the groups matched by the regex in the production, you use the `:groups` key:

```Clojure
“international phone number”
#”\+(\d+) (\d+)” ; regex capturing two groups
{:dim :phone-number 
 :country-code (-> %1 :groups first)
 :number (-> %1 :groups second)}
```

# Debugging

When a corpus test doesn’t pass and you don’t understand why, you can have a closer look at what happens with `play`:

```
picsou.core=> (play :en$datetime "45 degrees")
----------   5 | temperature | <latent temp> degrees     | P =  0.0000 | number as temp +
--           4 | distance    | number as distance        | P =  0.0000 | integer (numeric)
--           3 | temperature | number as temp            | P =  0.0000 | integer (numeric)
--           2 | null        | number (as relative minutes) | P =  0.0000 | integer (numeric)
--           1 | number      | integer (numeric)         | P =  0.0000 |
45 degrees
6 tokens in stash
```

Each line represents a token in the stash. The input string is at the bottom.

Columns:

1. The `--` represent the span in the text input
2. Token index (starting at 1, since the input string itself is token 0)
3. :dim
4. Label of the rule that produced the token (that’s why labeling your rules clearly is important)
5. Probability
6. Labels of the rules that produced the tokens in the slots below

If you need more information about a specific token, call the `details` function with the token index:

```
(details 3)
```

# FAQ

### How to add a new language?

To add support for a new language, follow these steps:

1. Create at least one corpus file in `resources/corpus` (you might duplicate an existing file to get started).
2. Create at least one rules file in `resources/rules`.
3. Edit the configuration file `resources/default-config.clj` and add your module. Have your module refer to your corpus and rules files.
4. Reload (see above)
5. Run your module corpus, add tests, see them fail, add rules until everything passes.