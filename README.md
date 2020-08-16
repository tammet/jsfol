
JSON-LD-LOGIC: JSON representation of first-order logic formulas
=================================================================

*Draft version 0.7: looking for discussion and feedback*


tanel.tammet@gmail.com

with contributions from schulz@eprover.org and geoff@cs.miami.edu


Goals and summary
------------------

JSON-LD-LOGIC is a simple JSON syntax for writing
first order logic [FOL](https://en.wikipedia.org/wiki/First-order_logic "FOL") 
formulas in a machine-processable manner suitable for automated reasoners.

The main goals of JSON-LD-LOGIC are:
* Ease of understanding and writing.
* Compatibility with the [TPTP](http://tptp.org "TPTP") language used by 
  most high-end full FOL reasoners, as described in the  
  [TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual").
* Compatibility with [JSON-LD](https://json-ld.org/), 
  see the [latest W3C draft](https://w3c.github.io/json-ld-syntax/). 

An implementation of JSON-LD-LOGIC is available: the [gkc](https://github.com/tammet/gkc/)
reasoner and toolkit. Gkc is able to both answer the questions and present proofs,
clausify JSON-LD-LOGIC expressions and convert them to or from the TPTP language. 
Check out a [web playground][gkc](http://logictools.org/json.html) for JSON-LD-LOGIC: 
it runs gkc in the browser using WASM.

The conversion and proof examples in this document are made using 
[gkc](https://github.com/tammet/gkc/).

A simple unsatisfiable example using core JSON-LD-LOGIC stating some facts, rules and a question:

    [
      ["brother","john","mike"],
      ["brother","john","pete"],
      [["brother","?:X","?:Y"], "=>", ["brother","?:Y","?:X"]],
      [[["brother","?:X","?:Y"], "&",  ["brother","?:Y","?:Z"]], "=>", ["brother","?:X","?:Z"]],
      ["~brother","mike","pete"]
    ]

Another example using full JSON-LD-LOGIC demonstrating the combination of standard logic syntax
with the JSON-LD syntax, using both explicit quantifiers and free variables, using
convenience operators like if ... then ..., indicating a question to be answered:

    [
      {
        "@context": {    
          "@vocab":"http://foo.org/"
        },
        "@id":"pete", 
        "@type": "male",
        "father":"john",
        "son": ["mark","michael"],
        "@logic": [
          "forall",["X","Y"],
           [[{"@id":"X","son":"Y"}, "&", {"@id":"X","@type":"male"}], 
            "=>", 
            {"@id":"Y","father":"X"}]
        ]
      },   

      [
       "if", 
         {"@id":"?:X","http://foo.org/father":"?:Y"}, 
         {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
       "then", 
         {"@id":"?:X","grandfather":"?:Z"}        
      ],

      {"@question": {"@id":"?:X","grandfather":"john"}}
    ]


The main features of the syntax are:

* Terms and atoms are represented as JSON lists with predicate/function symbols 
  in the first position (prefix form) like ["brother","john","pete"],
* JSON-LD semantics in RDF is represented by a special $arc predicate for triplets
  like ["$arc","pete","father","john"] and an
  $narc predicate for named triplets aka quads, like ["$narc","pete","father","john","eveknows"].
* JSON strings can represent ordinary constant/function/predicate symbols like "foo",
  free variables like "?:X", blank nodes like "_:b0" and distinct symbols like "#:bar",  
  using a special JSON-LD-style *prefix*. 
* Numeric arithmetic, string operations on distinct symbols and a list type are provided.  
* JSON lists in JSON-LD like {"@list":["a",4]} are translated to nested typed terms
  using the "$list" and "$nil" functions, like ["$list","a",["$list",4,$nil]].
* JSON maps like  `{"@name": "example", "@logic": ["p","?X",1]}` are used for 
  inserting logic into JSON-LD expressions and adding metainformation to logic formulas.

The semantics of most JSFOL constructions stems directly from the semantics of the 
corresponding TPTP constructions. The semantics of expressions like lists, null, 
if ... then ... etc not present in TPTP are presented explicitly in the current document.


Initial conversion and proof examples
-------------------------------------

JSON-LD-LOGIC should guarantee clear JSON <-> TPTP transformation 
while not covering all of the more complex TPTP sublanguages with features
like higher order formulas.

TPTP <-> JSON conversion is currently defined only for the non-higher order 
fragment (FOF and CNF sublanguages) of TPTP along with a both limited and extended
subset of the type fragment - TFF with arithmetic - of TPTP.

The first example above can be converted to TPTP as:

    fof(frm_1,axiom,brother(john,mike)).
    fof(frm_2,axiom,brother(john,pete)).
    fof(frm_3,axiom,(! [X,Y] : (brother(X,Y) => brother(Y,X)))).
    fof(frm_4,axiom,(! [X,Y,Z] : ((brother(X,Y) & brother(Y,Z)) => brother(X,Z)))).
    fof(frm_5,axiom,~brother(mike,pete)).

The TPTP syntax wraps formulas into the `fof(name,role,formula).`
construction terminated by the period. 
The exclamation mark `!` denotes universal quantifier (forall)
and the question mark `?` denotes existential quantifier (exists).
Variables have to start with the uppercase letter, non-variable symbols
with a lowercase letter, underscore `_` or dollar `$`. The logical
operator `&` is conjunction (and), `=>` is implication, `~` is negation.

We obtain the following refutation proof in json:

    {"result": "proof found",

    "answers": [
    {
    "proof":
    [
    [1, ["in", "frm_4"], [["-brother","?:X","?:Y"], ["-brother","?:Z","?:X"], 
                          ["brother","?:Z","?:Y"]]],
    [2, ["in", "frm_1"], [["brother","john","mike"]]],
    [3, ["mp", 1, 2], [["-brother","?:X","john"], ["brother","?:X","mike"]]],
    [4, ["in", "frm_3"], [["-brother","?:X","?:Y"], ["brother","?:Y","?:X"]]],
    [5, ["in", "frm_2"], [["brother","john","pete"]]],
    [6, ["mp", 4, 5], [["brother","pete","john"]]],
    [7, ["mp", 3, 6], [["brother","pete","mike"]]],
    [8, ["in", "frm_5"], [["-brother","mike","pete"]]],
    [9, ["mp", 7, 4, 8], false]
    ]}
    ]}

The proofs in this document are json objects with two keys:

* `result` indicates whether the proof was found,
* `answers` contains a list of all answers and proofs for these answers. 
  Here we did not ask to find a concrete person, so we have only 
  a single answer containing just a `proof` key indicating a list with the
  numbered steps of the proof found:
  each step is either a used input fact / rule or a derived fact / rule.

The formulas in the proof are always just lists of atoms (called *clauses*) treated
as a disjunction (or). Negation is prefixed as a minus sign `-` to the predicate:
`["-brother","mike","pete"]` means the same as `["not" ["brother","mike","pete"]]`.
Strings prefixed by `?:` like `?:X` are *free variables* implicitly assumed to be quantified by "forall".
`["in","frm_N" ]` means that the clause stems from the fact/rule number *N* was given as input.
`["mp", 1, 2]`  means that this clause was derived by modus ponens (i.e. the 
[resolution rule](https://en.wikipedia.org/wiki/Resolution_(logic))
from previous steps 1 and 2. More concretely, the first literals of both were cut off and
the rest were glued together. `["mp", 7, 4, 8]` means that the clause was first 
derived from clauses 7 and 4 and then the clause 8 was used for and additional
simplifying resolution step.

The second example above can be converted to TPTP as:
    
    fof(frm_1,axiom,
      ((! [X,Y] : 
         (($arc(X,'http://foo.org/son',Y) & 
           $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)) 
          => 
          $arc(Y,'http://foo.org/father',X))) & 

         ($arc(pete,'http://foo.org/son',michael) & 
           ($arc(pete,'http://foo.org/son',mark) & 
            ($arc(pete,'http://foo.org/father',john) & 
             $arc(pete,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)))))).

    fof(frm_2,axiom,
      (! [X,Y,Z] : 
         (($arc(Y,'http://foo.org/father',Z) & $arc(X,'http://foo.org/father',Y)) 
          => 
          $arc(X,grandfather,Z)))).

    fof(frm_3,negated_conjecture,(! [X] : ($arc(X,grandfather,john) => $ans(X)))).

and given the following refutation proof in json:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","michael"]],
    "proof":
    [
    [1, ["in", "frm_1"], [["-$arc","?:X","http://foo.org/son","?:Y"], 
                          ["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"],
                          ["$arc","?:Y","http://foo.org/father","?:X"]]],
    [2, ["in", "frm_1"], [["$arc","pete","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"]]],
    [3, ["mp", [1,1], 2], [["-$arc","pete","http://foo.org/son","?:X"], 
                           ["$arc","?:X","http://foo.org/father","pete"]]],
    [4, ["in", "frm_1"], [["$arc","pete","http://foo.org/son","michael"]]],
    [5, ["mp", 3, 4], [["$arc","michael","http://foo.org/father","pete"]]],
    [6, ["in", "frm_2"], [["-$arc","?:X","http://foo.org/father","?:Y"], 
                          ["-$arc","?:Z","http://foo.org/father","?:X"], 
                          ["$arc","?:Z","grandfather","?:Y"]]],
    [7, ["in", "frm_1"], [["$arc","pete","http://foo.org/father","john"]]],
    [8, ["mp", 6, 7], [["-$arc","?:X","http://foo.org/father","pete"], 
                       ["$arc","?:X","grandfather","john"]]],
    [9, ["mp", 5, 8], [["$arc","michael","grandfather","john"]]],
    [10, ["in", "frm_3"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [11, ["mp", 9, 10], [["$ans","michael"]]]
    ]}
    ]}


Two layers of JSON-LD-LOGIC
-----------------------------

`Core fragment` specifies minimal JSON syntax for writing logic formulas 
which can be translated to the TPTP FOF and CNF sublanguages. All the TPTP FOF and CNF 
formulas are directly convertible to the core JSON-LD-LOGIC. Both these
TPTP sublanguages cover standard first order logic with functions and 
the equality predicate.

`Full language` adds compatibility with JSON-LD along with support for
an arithmetic-plus-distinct-symbols part of the TPTP typed sublanguage TFF along
with an additional list type, several additional pre-defined functions, 
convenience functions and operators. Full JSON-LD-LOGIC can be directly
converted to the core fragment, except for the typed part using arithmetic, lists
and distinct symbols.

We will first present the core fragment and thereafter the full language.


Core fragment
-------------

The core fragment is essentially a JSON representation of the combined TPTP
FOF (first order formula) and CNF (clause normal form) sublanguages by
allowing the introduction of free variables in any formulas and not
requiring the formulas to be embedded in special `fof` or `cnf` terms.

Arithmetic, distinct symbols, lists, null and JSON-LD constructions are
not included in the core fragment.

Core fragment constructions can be viewed as falling into one of five categories,
from bottom to top:

* `Primitives` are constants, function/predicate symbols and variables 
   represented by JSON primitives.
* `Terms and atoms` are built of primitives and terms as nested JSON lists. 
   A `literal` is either an atom or a negated atom.
* `Formulas` are built of literals using logical contructors like "&", "exists", etc
   using nested JSON lists and JSON maps as used in JSON-LD.
* `Formula lists` are JSON lists of fully formed, independent standalone formulas.


### Primitives in the core fragment


* JSON strings have multiple uses covered in the following list:

    * Special predefined strings in TPTP are used for constructing formulae.
      The full list is:
      `"~", "|", "&",  "<=>", "=>", "<=", "<~>", "~|", "~&", "@", 
      "forall", "exists"`.
      `"=", "!="` are pre-defined equality and inequality predicates.      

    * Strings bound by a quantifier in JSON stand for corresponding variables in
      FOL and TPTP. A bound variable *must* start with an upper case letter.     
      In the following example all the variables `X1, X2, U` 
      are existentially quantified:        

      `["exists",["X1","X2","U"],["p","X1","X2","U"]]`
   
    * Strings starting with a question mark and colon like  `"?:X1"`
      and *not bound by a quantifier* 
      stand for free variables in FOL and are assumed to be universally quantified
      at the top level of a formula in TPTP, except in the formulas with the role 
      *conjecture*, as explained later. Examples: 

      `"?:X", "?:Y1", "?:Some car"`    

    * Strings prefixed by `"~"` and located in the first position of a list in a
      formula context construct a negated atom led by a symbol
      constructed from the rest of the string. 
      For example, the arguments of the following "&" are equivalent:

      `[["~p", 1], "&", ["~", ["p",1]]`
      
      Note: multiple "~" like "~~p" do not create a nested negation.

    * Strings starting with a hash sign and colon like  `"#:foo"` are distinct
      symbols used in the full language and should not be used in the core fragment.

    * Strings starting with an underscore sign and colon like  `"_:bar"` are 
      blank nodes of JSON-LD and should not be used in the core fragment.   
      
    * Strings without any of the previously described prefixes `?:`, `#:`, `_:`, `~` 
      stand for ordinary predicate, function or constant symbols, unless
      they are one of the predefined strings in the full language 
      like `"&", "$sum", "$is_real", "+"` etc. Examples:

      `"p", "Foo", "'bar' 23", "_here", "++something", "1 to 3"`

* `true` and `false` in JSON stand for the corresponding logical values
  in TPTP: `$true` and `$false`.

* Other primitive JSON datatypes - numbers and null - are a part
  of the full language and should not be be used in the core fragment.

* An empty JSON list `[]` stands for a symbol `$nil` in the full language
  and should not be used in the core fragment.

* Objects (aka maps) are a part of the full language and should be used
  only in the restricted way at as top level formulas in the core fragment. 
   
   
### Terms and atoms in the core fragment


Terms and atoms are represented as JSON lists, using the prefix form analogous to
lisp s-expressions: the first element of a list stands for a predicate or a function symbol.

Example:

    `["foo","bar","?:Y","X",["f",["f","X]]]`

stands for a TPTP term or atom 

    `foo(bar,'X',Y,f(f('X')))` 

where `Y` is a free variable.

Important aspects of the TPTP conversion:

* The TPTP FOF sublanguage does not allow free variables, whereas the clausal CNF sublanguage allows 
  only free variables. JSON-LD-LOGIC merges the FOF and CNF sublanguages into one, allowing
  free variables in both simple clauses and arbitrary formulas.

* TPTP requires symbols to either 

  * (a) consist of alphanumerics, underscore `_` and dollar `$` and not start with a capital letter,

  * (b) or to be surrounded by single quotes like `'Here is "John"'`. 
  
  JSON-LD-LOGIC treats all non-special strings not prefixed by `?:`, `#:` or `_:`
  and not bound by a quantifier as ordinary symbols. Thus the JSON-LD-LOGIC symbols which do not
  satisfy the TPTP requirement (a) are converted to TPTP by surrounding them with single quotes
  and adding a "\" character in front of all internal quote characters. The TPTP symbols surrounded
  by quotes are converted to JSON-LD-LOGIC by removing the surrounding quotes and adding 
  a "\" character in front of internal double quotes.

JSON-LD-LOGIC does not require that predicate or function symbols have a unique arity, although
applications may pose such restrictions.

The first element of an atom or a function term should be a string, except for
atoms constructed by infix `"="` or `"!="`. 


### Equality in the core fragment


Predicates `"="` and `"!="` stand for equality and inequality and can occur
only in the middle of a three-element list (infix form), like

  `["a","=","b"]`
  `["?:X","!=",["foo","?:X"]]`

The following example uses equality:

    [
      [["father","john"],"=","pete"],
      [["father","mike"],"=","pete"],
      [["mother","john"],"=","eve"],
      [["mother","mike"],"=","eve"],
      [["father","pete"],"=","mark"],
      [["mother","eve"],"=","mary"],
      ["grandfather",["father",["father","?:0"]],"?:0"],
      ["grandfather",["father",["mother","?:0"]],"?:0"],
      ["~grandfather","mark","?:0"]
    ]

Conversion to the TPTP form is

    fof(frm_1,axiom,(father(john) = pete)).
    fof(frm_2,axiom,(father(mike) = pete)).
    fof(frm_3,axiom,(mother(john) = eve)).
    fof(frm_4,axiom,(mother(mike) = eve)).
    fof(frm_5,axiom,(father(pete) = mark)).
    fof(frm_6,axiom,(mother(eve) = mary)).
    fof(frm_7,axiom,(! [X0] : grandfather(father(father(X0)),X0))).
    fof(frm_8,axiom,(! [X0] : grandfather(father(mother(X0)),X0))).
    fof(frm_9,axiom,(! [X0] : ~grandfather(mark,X0))).

The proof of unsatisfiability we get uses equality. 
Notice that the clauses in the proof do not strictly correspond
to the core fragment, since (a) equality is written in prefix form
as allowed only in the full language and (b) minus sign is
used instead of tilde to indicate negation, also allowed
in the full language.

    {"result": "proof found",

    "answers": [
    {
    "proof":
    [
    [1, ["in", "frm_2"], [["=",["father","mike"],"pete"]]],
    [2, ["in", "frm_7"], [["grandfather",["father",["father","?:X"]],"?:X"]]],
    [3, ["=", 1, [2,0,2]], [["grandfather",["father","pete"],"mike"]]],
    [4, ["in", "frm_5"], [["=",["father","pete"],"mark"]]],
    [5, ["simp", 3, 4], [["grandfather","mark","mike"]]],
    [6, ["in", "frm_9"], [["-grandfather","mark","?:X"]]],
    [7, ["mp", 5, 6], false]
    ]}
    ]}


### Formulas in the core fragment


Formulas are built from booleans, literals and formulas using ordinary logical connectives,
used in the infix form in the core fragment.

The following connectives are predefined, whereas `"~"`
may occur as a first element of a two-element list and all the other connectives 
may only occur between elements of a list (infix form). A list may not contain different 
connectives: instead of

  `["a","&","b","=>","c"]`

  one must use
  
  `[["a","&","b"],"=>","c"]`

The binary connectives are left associative. For example,

  `["a","=>","b","=>","c"]`

is interpreted as

  `[["a","=>","b"],"=>","c"]` 

The connectives are:  

* `"~"` stands for negation and takes a single argument. Note that as we have said before,
    `["~p",1]` stands for `["~",["p",1]]`
     
* Based on the TPTP language, the following connectives are available, all of them
   used only in the infix form:

    * infix `"|"` for disjunction,
    * infix `"&"` for conjunction, 
    * infix `"<=>"` for equivalence, 
    * infix `"=>"` for implication, 
    * infix `"<="` for reverse implication, 
    * infix `"<~>"` for non-equivalence (XOR), 
    * infix `"~|"` for negated disjunction (NOR), 
    * infix `"~&"` for negated conjunction (NAND), 
    * infix `"@"` for application, used mainly in the higher-order context in TPTP.

* The quantifiers `"exists"` and `"forall"` 
  are represented as a list of three elements: quantifier, list of variables, formula.
  Examples:

  `["exists",["X"],["p","X"]]`

  `["forall",["X","Y2"],["p","Y2","X"]]`

There is a simplified alternative syntax for clauses (disjunctions of literals):
a list `[L1, ... ,LN]` occurring in the  top-level formula formula list (described next)
is treated as a disjunction of elements `[L1,"|", ... ,"|",LN]` if

* `L1` is a list,

* no element `Li` in the list is a connective or a positive or negative equality predicate.


### Formula list


The JSON-LD-LOGIC core fragment document must be a list of formulas like this:
 
    [      
      ["brother","john","mike"],
      ["brother","john","pete"],
      [["brother","?:X","?:Y"], "=>", ["brother","?:Y","?:X"]],
      [["is_father","?:X"], "<=>", ["exists",["Y"],["father,"?:X","Y"]]]      
    ]

The elements of the formula list are translated as a conjunction
of the elements, with the following differences from a simple conjunction:

Each formula in the list containing free variables
is translated by binding the free variables in this formula by
a "forall" quantifier at the top level of this formula, except for
the formulas with the *conjecture* role (to be described later).
The previous example is thus equivalent to:

    [
      ["brother","john","mike"], "&", 
      ["brother","john","pete"], "&",
      ["forall",["X","Y"], [["brother","X","Y"], "=>", ["brother","Y","X"]]], "&",
      ["forall",["X"], [["is_father","X"], "<=>", ["exists",["Y"],["father,"X","Y"]]]]      
    ]
 
Notice that 

* The existentially quantified "Y" variable in the last formula is dependent on
  the leading "X" variable of the same formula.

* The free variables "?:X" in the last two formulas of the example before conversion 
  are distinct from each other.
  

   
### Metainformation and roles in the core fragment


Metainformation is represented using JSON objects and can be attached to formulas in
top level formula lists. Example:

    {
     "@name":"example formula",    
     "@role", "axiom",
     "@logic": ["p",1,"a"] 
    }
 
 
where the predefined keys correspond to the TPTP positional metainformation values:

* `"@name"` for the construction name: the value is a formula name without a logical meaning,
  useful for increasing the readability of proofs.

* `"@role"` value should be normally either "axiom", "assumption", "conjecture" or a
  "negated_conjecture": normally used for annotating formulas for their intended use.
  A longer description and more options are given below.  

* `"@logic"` for the JSON-LD-LOGIC formula with the logical content as described before.

All of these key/value pairs are optional. If "@logic" is not present, the object should
be (normally) just ignored by the application.

In case a formula list contains such a JSON object, then

* If `"name"` is not present, then the system may optionally construct a new name.

* If `"role"` is not present, then its value is assumed to be "axiom". 

The special `"conjecture"` role value in TPTP forces the content to be negated when asking
for unsatisfiability of a formula list. 

We bring two equivalent example formulas, where in the first case "conjecture" is used with
the positive `["p","a"]` and in the second "negated_conjecture" with the negative `["~p","a"]`:

    [
      ["p","a"],
      ["p","b"],
      {"@role": "conjecture", "@logic": ["p","a"]}
    ]

represents a provable formula `(p(a) & p(b)) => p(a)`, the negation of which
is equivalent to an unsatisfiable `p(a) & p(b) & ~p(a)`

whereas 

    [
      ["p","a"],
      ["p","b"],
      {"@role": "negated_conjecture", "@logic": ["~p","a"]}
    ]

also represents an unsatisfiable formula `p(a) & p(b) & ~p(a)`.

As said before, *free variables* are not implicitly bound by a "forall" quantifier
in the formula with a *conjecture* role: instead, they are implicitly
bound by the "exists" quantifier: this corresponds better to the intuitive 
understanding of free variables in the conjecture, especially in the light
of the `$ans`predicate and the `@question` key in the full language. For example,

    [
      ["p","a"],
      ["p","b"],
      {"@role": "conjecture", "@logic": ["p","?:X"]}
    ]

is equivalent to

    [
      ["p","a"],
      ["p","b"],
      {"@role": "conjecture", "@logic": ["exists",["X"],["p","X"]]}
    ]

which is equivalent to

    [
      ["p","a"],
      ["p","b"],
      {"@role": "negated_conjecture", "@logic": ["forall",["X"],["~p","X"]]}
    ]


For other *role* values we cite "The Formulae Section" of 
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual"): 
the role gives the user semantics of the formula, one of `axiom, hypothesis, definition,
assumption, lemma, theorem, corollary, conjecture, negated_conjecture, plain, type`, and `unknown`. ..."

Treating of any other key values except these of the predefined keys is up to the application.


Included files
--------------

A document may include other files/documents, treated by appending the included formula list and the formula
list of the main document. A special atom with the form `["include", filename]` in the top formula list
is interpreted as an include command:

    [
      ["include","foo.js"],
      ...     
    ]

will cause the file "foo.js" to be included. The file is searched for in places specified by TPTP as follows: 
include files with relative path names are expected to be found either under the directory of the current file,
or if not found there then under the directory specified in the $TPTP environment variable.

The languages supported by the include command are implementation specific: other languages than JSON-LD-LOGIC
might be allowed to be imported.



Full JSON-LD-LOGIC
------------------

The main additional features of the full language above the core fragment are:

* JSON-LD objects aka maps are interpreted as logic and can be intermingled with the
  core fragment constructions. 

* The semantics in RDF is represented by a special $arc predicate for triplets
  like ["$arc","pete","father","john"] and an
  $narc predicate for named triplets aka quads, like ["$narc","pete","father","john","eveknows"].

* Numeric arithmetic, distinct symbols, string operations and a list type are provided.  

* JSON lists in JSON-LD like {"@list":["a",4]} are translated to nested typed terms
  using the "$list" and "$nil" functions, like ["$list","a",["$list",4,$nil]].

* Several convenience operators are introduced.


### JSON objects aka maps

JSON objects are interpreted according to the RDF semantics of JSON-LD, 
with a few modifications described later in this section.

The core principle is that each RDF triple `subject, predicate, object` is 
converted to an atom `["$arc",subject,predicate,object]` indicating that
there is an *arc* with the label *predicate* from the *subject* to *object*.

The `"$arc"` symbol has no special semantics except that it is used for
translating the triplets.

Consider the following JSON-LD example:

    [
      {"@id":"pete", "father":"john", "age":40},
      {"@id":"mark", "father":"pete", "age":10}
    ]

This formula list is a valid JSON-LD-LOGIC list and can be converted to TPTP as

    fof(frm_1,axiom,($arc(pete,age,40) & $arc(pete,father,john))).
    fof(frm_2,axiom,($arc(mark,age,10) & $arc(mark,father,pete))).

We say *can be converted* to indicate that the exact form of the conversion
is implementation-specific, but the logical semantics should be exactly
as indicated in the TPTP conversion.


#### Ordinary and distinct symbols

Importantly, both the *subject*, *predicate* and *object* strings 
like "john" and "pete" in the example above are treated as ordinary
symbols in logic, i.e. "john" and "pete" are not automatically considered
to be distinct: they both could be potentially equal to some "John Pete Smith",
for example.

JSON-LD-LOGIC uses a special prefix `"#:` to indicate that a symbol
is not equal to any other syntactically different symbols with the `"#:`
prefix and not equal to any numbers or lists.

For example, both `["#:pete","=","#:john"] and `["#:pete","=",2]`
must be evaluated to *false*, while `["pete","=","#:john"] and `["pete","=",2]`
are not evaluated to false.

The *distinct symbols* can be seen as an extension of the *string* type; 
string functions `"$substr"` and `"$substrat"` in JSON-LD-LOGIC can be
computed on distinct symbols.

The distinct symbols are translated to the distinct symbols of the
TPTP language with the same semantics as described above. The distinct
symbols of TPTP are surrounded by double quotes.

The following example

    [
      {"@id":"#:pete", "father":"#:john", "age":40},
      {"@id":"mark", "father":"#:pete", "age":10}
    ]

 can be converted to TPTP containing the distinct symbols `"pete"` and `"john"`
 as well as ordinary symbols `mark` , `father` and `age`:

    fof(frm_1,axiom,($arc("pete",age,40) & $arc("pete",father,"john"))).
    fof(frm_2,axiom,($arc(mark,age,10) & $arc(mark,father,"pete"))).

We note that the TPTP language uses the single quotes like `'John Smith'`
to denote ordinary symbols containing characters which are neither alphanumeric nor
`_` or `$`.

The motivation for treating JSON strings as ordinary symbols by default and using
a special prefix for distinct symbols (strings, if so called), and not vice versa,
stems from the following observations:

* The vast majority of symbols occurring in the TPTP library with tens of thousands
  of problems are ordinary symbols, with distinct symbols being a rarity.

* The difference between ordinary and distinct symbols in logic only appears in
  the presence of the equality predicate or other typed objects in the context of
  special string/arithmetic/list operations: otherwise there is no difference
  in the logical meaning.


### Datatypes via @type and typed symbols

JSON-LD-LOGIC does not currently specify the treatment of `@type` as a datatype
(say, XML Schema type as used in RDF). The special types of strings like
dates etc could be treated, for example, by prepending `^^typename` to a 
typed string and considering such typed strings to be unequal to syntactically
different typed symbols.

The suitable way of converting typed strings to TPTP may be determined in the
later revisions.

JSON-LD-LOGIC does, however, treat numeric JSON values as well as lists
indicated by the `"@list"` key (described later) and distinct symbols analogous
to strings as built-in types.


### Missing @id and blank nodes

In case a JSON object does not contain an `"@id"` key, a new symbol will 
be automatically generated for the *object*.

The following example 
    
    [
      {"father":"john", "age":40},
      {"father":"pete", "age":10}
    ]

can be converted to TPTP as

    fof(frm_1,axiom,($arc('_:crtd_1',age,40) & $arc('_:crtd_1',father,john))).
    fof(frm_2,axiom,($arc('_:crtd_2',age,10) & $arc('_:crtd_2',father,pete))).

where `_:crtd_1` and `_:crtd_2` are new symbols not occurring anywhere else,
corresponding to the *blank nodes* of JSON-LD and RDF. The blank nodes
behave similarly to the existentially quantified variables, converted to
the *skolem functions* by the clausification algorithms used by resolution
theorem provers.

The following JSON-LD example containing *blank nodes* 
    
    [
      {"@id":"_:pete", "father":"john"},
      {"@id":"_:pete", "age":10}
    ]

can be converted to TPTP as

    fof(frm_1,axiom,$arc('_:pete',father,john)).
    fof(frm_2,axiom,$arc('_:pete',age,10)).

The interpretation of blank nodes is the same as in JSON-LD: 
syntactically identical blank nodes in different documents/files should be treated
as different symbols. This could be achieved by renaming all the blank nodes
in the documents to unique values in the scope of a document/file whenever several
documents/files are merged in some way.



## @logic: introducing logic to JSON-LD 

The value of the `"@logic"` key in an object/map is treated as a logical formula,
using either the core fragment described before or the full JSON-LD-LOGIC language.

The value of the `"@logic"` key should not contain `"@logic"`, i.e. it should
not be nested. It can contain other objects/maps though, interpreted in the same
was as defined in the current specification: objects/maps can be nested in the
the logical formula.

The free variables occurring in the value of the `"@logic"` key and/or elsewhere
in the same top-level list element have the whole element as its scope.

The following example demonstrates a simple use of `"@logic"` along with the
convenience operator `if ... then ...` and the `"@question"`
key described in the next section:

    [
      {"@id":"pete", "father":"john"},
      {"@id":"mark", "father":"pete"},
      {"@name":"gfrule",
      "@logic": ["if", 
                    ["$arc","?:X","father","?:Y"],
                    ["$arc","?:Y","father","?:Z"], 
                  "then", 
                    ["$arc","?:X","grandfather","?:Z"]]
      },
      {"@question": ["$arc","?:X","grandfather","john"]}
    ]

The example can be converted to TPTP as 

    fof(frm_1,axiom,$arc(pete,father,john)).
    fof(frm_2,axiom,$arc(mark,father,pete)).
    fof(gfrule,axiom,(! [X,Y,Z] : 
                       (($arc(Y,father,Z) & $arc(X,father,Y)) 
                       => 
                       $arc(X,grandfather,Z)))).
    fof(frm_4,negated_conjecture,
          (! [X] : ($arc(X,grandfather,john) => $ans(X)))).

and given a proof as

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","mark"]],
    "proof":
    [
    [1, ["in", "gfrule"], [["-$arc","?:X","father","?:Y"], 
                           ["-$arc","?:Z","father","?:X"], 
                           ["$arc","?:Z","grandfather","?:Y"]]],
    [2, ["in", "frm_1"], [["$arc","pete","father","john"]]],
    [3, ["mp", 1, 2], [["-$arc","?:X","father","pete"], ["$arc","?:X","grandfather","john"]]],
    [4, ["in", "frm_2"], [["$arc","mark","father","pete"]]],
    [5, ["mp", 3, 4], [["$arc","mark","grandfather","john"]]],
    [6, ["in", "frm_4"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [7, ["mp", 5, 6], [["$ans","mark"]]]
    ]}
    ]}

The following example using objects/maps inside the value of the `"@logic"` key
along with the free variables has an exactly same translation and 
proof as the previous example:

    [
    {"@id":"pete", "father":"john"},
    {"@id":"mark", "father":"pete"},
    {"@name":"gfrule",
     "@logic": ["if", 
                  {"@id":"?:X","father":"?:Y"}, 
                  {"@id":"?:Y","father":"?:Z"}, 
                "then", 
                  {"@id":"?:X","grandfather":"?:Z"}]
    },
    {"@question": {"@id":"?:X","grandfather":"john"}}
    ]

The use of free variables in the last example illustrates that their
scope is limited to the top level list element: `"?:X"` in
`{"@id":"?:X","father":"?:Y"}` is different from `"?:X"` in
 `{"@id":"?:X","grandfather":"john"}`.


### $ans and "@question"

A conventional way to find out interesting substitutions for variables -
the answers we are looking for - is to use the special `$ans` predicate
with the meaning that deriving a clause containing only atoms with this predicate
is equivalent to deriving an empty clause, i.e. false, meaning the proof is
found. The arguments of `$ans` are then the answers we look for.
The presence of several `$ans` atoms in the clause indicates an
indefinite answer.

JSON-LD-LOGIC uses `"$ans"` as such a predicate.

Example:

    [
      ["father","john","pete"],
      ["father","pete","mark"],
      ["forall", ["X","Y","Z"], [[["father","X","Y"], "&", ["father","Y","Z"]], 
                                 "=>", 
                                ["grandfather","X","Z"]]],
      ["forall", ["X"], [["grandfather","john","X"],"=>",["$ans","X"]]]
    ]

gives a proof containing an answer:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","mark"]],
    "proof":
    [
    [1, ["in", "frm_3"], [["-father","?:X","?:Y"], 
                          ["-father","?:Z","?:X"], 
                          ["grandfather","?:Z","?:Y"]]],
    [2, ["in", "frm_2"], [["father","pete","mark"]]],
    [3, ["mp", 1, 2], [["-father","?:X","pete"], ["grandfather","?:X","mark"]]],
    [4, ["in", "frm_1"], [["father","john","pete"]]],
    [5, ["mp", 3, 4], [["grandfather","john","mark"]]],
    [6, ["in", "frm_4"], [["-grandfather","john","?:X"], ["$ans","?:X"]]],
    [7, ["mp", 5, 6], [["$ans","mark"]]]
    ]}
    ]}

It may be possible to indicate to the reasoner that several different
answers are looked for. The ways to do this are implementation-specific.

JSON-LD-LOGIC introduces a convenience key `"@question"` automatically
forming the implication with the `"$ans"` predicate with all the free
variables in its value as arguments.

The following example

    [
      ["father","john","pete"],
      ["father","pete","mark"],
      ["forall", ["X","Y","Z"], [[["father","X","Y"], "&", ["father","Y","Z"]], 
                                "=>", 
                                ["grandfather","X","Z"]]],
      {"@question": ["grandfather","john","?:X"]}
    ]

is thus equivalent to the previous example with the explicit `"$ans"`, plus
it gives the formula the `negated_conjecture` role telling the reasoner that
this particular formula is a goal to be focused on: this information often
leads to a much faster proof search. The conversion to TPTP is thus:


    fof(frm_1,axiom,father(john,pete)).
    fof(frm_2,axiom,father(pete,mark)).
    fof(frm_3,axiom,(! [X,Y,Z] : ((father(X,Y) & father(Y,Z)) => grandfather(X,Z)))).
    fof(frm_4,negated_conjecture,(! [X] : (grandfather(john,X) => $ans(X)))).


### Convenience connectives

The following "convenience" connectives and constants 
`"-", "not", "and", "or", "if" ... "then" ..., null`
are translated to the core fragment constructions as follows.

The minus sign `-` can be used instead of the tilde `~` both as a logical negation and
as a prefix of a string, indicating that the string is a predicate which has to be
negated.

The `"not"` connective can be used as a negation connective instead of the tilde `"~"`.

The `"and"` and `"or"` connectives can be used instead of the `"&"` and `"|"` correspondingly,
whereas they can be used both in the infix and prefix form, the latter form taking an 
arbitrary number of arguments like this:

  `["and", ["p",1], ["foo","?:X", "a], ["bar"]]`


The `"and"` with no arguments is equivalent to `true` and the `"or"` with no arguments to `false`.

The mixfix operator `if..then..` can be used. The list of formulas before `then` is treated
as a conjunction premiss of the implication and the list after `then` as a disjunction
consequent of the implication.  Example:

  `["if", "a", "b", "c", "then", "d", "e"]`

  is translated as 

  `[["a","&","b","&","c"], "=>", ["d","|","e"]]`  


The `null` symbol of JSON is treated analogously to the SQL `null` representing a missing
or unknown value. The translation mechanism of eliminating a `null` inside some formula
`F` is as follows:

* Create a new variable `V` not occurring in `F`. I.e. `V` is a new string.
* Replace an atom `[...null....]` where the eliminated `null` occurs by a formula

   `["exists", [V], [....V....]]`

where this particular occurrence of `null` is replaced by the variable `V'.

Example:

   `["father","john",null]` 

is translated as 

   `["exists", ["X"], ["father","john","X"]]`. 

Example:

   `[["is_father","?:X"], "<=>", ["father","?:X",null]]` 

is translated as 

   `[["is_father","?:X"], "<=>", ["exists", ["Y"], ["father","?:X","Y"]]]`
    

### Multiple values, @context and namespaces, @base, @type.

Multiple values like given in `"son": ["mark","michael"]` in the
following example are interpreted according to the JSON-LD RDF
interpretation as leading to multiple triplets.

An important feature of JSON-LD is the use of the `"@context"` key to specify,
for example, namespaces, symbol mapping and types.

JSON-LD-LOGIC expands strings in the object/map by the JSON-LD `"@context"` 
expansion rules.

Notice that the example uses inline `"@context"` value, not one read
from an URL: reading the `"@context"` value from the
URL would make the interpretation of the formula indeterminate.

The question as for what is the default base URI for `"@id"` key values
is left open in JSON-LD-LOGIC. In the following we assume it is an empty
string.

The `"@type"` key is converted to the RDF value string
`"http://www.w3.org/1999/02/22-rdf-syntax-ns#type"`.

Consider the following example using multiple values and the JSON-LD `"@context"` 
`"@base"` and  `"@type"` keys along with the JSON-LD-LOGIC `"@logic"` key and 
the convenience operator  `if ... then ...` and the `"@question"` key described later:

    [
    { "@context": {
        "@base": "http://people.org/",
        "@vocab":"http://bar.org/", 
        "father":"http://foo.org/father"
      },
      "@id":"pete", 
      "@type": "male",
      "father":"john",
      "son": ["mark","michael"],
      "@logic": ["forall",["X","Y"],
                  [[{"@id":"X","son":"Y"}, "&", {"@id":"X","@type":"male"}], 
                  "=>", 
                  {"@id":"Y","father":"X"}]
      ]
    },      
    ["if", 
       {"@id":"?:X","http://foo.org/father":"?:Y"},
       {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
      "then", 
      {"@id":"?:X","grandfather":"?:Z"}],
    {"@question": {"@id":"?:X","grandfather":"john"}}
    ]

The formula can be translated to TPTP as


    fof(frm_1,axiom,
        ((! [X,Y] : 
           (($arc(X,'http://bar.org/son',Y) & 
             $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)) 
             => 
             $arc(Y,'http://foo.org/father',X))) 
         & 
         ($arc('http://people.org/pete','http://bar.org/son',michael) & 
         ($arc('http://people.org/pete','http://bar.org/son',mark) & 
         ($arc('http://people.org/pete','http://foo.org/father',john) & 
         $arc('http://people.org/pete','http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)))))).
    fof(frm_2,axiom,
        (! [X,Y,Z] : 
          (($arc(Y,'http://foo.org/father',Z) & $arc(X,'http://foo.org/father',Y)) 
          => 
          $arc(X,grandfather,Z)))).
    fof(frm_3,negated_conjecture,(! [X] : ($arc(X,grandfather,john) => $ans(X)))).


Observe that unless a formula in the top level list is in the scope of the 
same `"@context"` key value, the namespaces of symbols should be defined either
with a copy of the `"@context"` key value or as absolute values, like done in
this example.

Proof of the example:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","michael"]],
    "proof":
    [
    [1, ["in", "frm_1"], [["-$arc","?:X","http://bar.org/son","?:Y"], 
                          ["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"],
                          ["$arc","?:Y","http://foo.org/father","?:X"]]],
    [2, ["in", "frm_1"], [["$arc","http://people.org/pete","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"]]],
    [3, ["mp", [1,1], 2], [["-$arc","http://people.org/pete","http://bar.org/son","?:X"], 
                           ["$arc","?:X","http://foo.org/father","http://people.org/pete"]]],
    [4, ["in", "frm_1"], [["$arc","http://people.org/pete","http://bar.org/son","michael"]]],
    [5, ["mp", 3, 4], [["$arc","michael","http://foo.org/father","http://people.org/pete"]]],
    [6, ["in", "frm_2"], [["-$arc","?:X","http://foo.org/father","?:Y"], 
                          ["-$arc","?:Z","http://foo.org/father","?:X"],
                          ["$arc","?:Z","grandfather","?:Y"]]],
    [7, ["in", "frm_1"], [["$arc","http://people.org/pete","http://foo.org/father","john"]]],
    [8, ["mp", 6, 7], [["-$arc","?:X","http://foo.org/father","http://people.org/pete"], 
                       ["$arc","?:X","grandfather","john"]]],
    [9, ["mp", 5, 8], [["$arc","michael","grandfather","john"]]],
    [10, ["in", "frm_3"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [11, ["mp", 9, 10], [["$ans","michael"]]]
    ]}
    ]}

### Nested objects/maps

JSON-LD interprets JSON objects/maps as key values as new things with their
own id-s and properties.

JSON-LD-LOGIC interprets such values correspondingly.

Here is a bit more complex example with a nested `"child"` value indicating that
the person we describe (with no id given) has two children with ages
10 and 2, but nothing more is known about them. We also know the person
has a father `john`. The rules state the son/daughter and mother/father
correspondence, define children with ages between 19 and 6 as schoolchildren
and define both the maternal and paternal grandfather. The question is to 
find a schoolchild with `john` as a grandfather. Notice the use of arithmetic
predicate `"$less"`. 

    [
    { "@context": {    
        "@vocab":"http://foo.org/",   
        "age":"hasage"
      },
      "father":"john",
      "child": [
        {"age": 10}, 
        {"age": 2} 
      ],      
      "@logic": ["and",    
        ["forall",["X","Y"],[{"@id":"X","child":"Y"}, 
                            "<=>", 
                            ["or",{"@id":"Y","father":"X"},{"@id":"Y","mother":"X"}]]],     
        ["forall",["X","Y"],[{"@id":"Y","child":"X"}, 
                            "<=>", 
                            ["or",{"@id":"Y","son":"X"},{"@id":"Y","daughter":"X"}]]]
      ]  
    },      
    [
      "if", 
        {"@id":"?:X","http://foo.org/hasage":"?:Y"}, 
        ["$less",6,"?:Y"], 
        ["$less","?:Y",19], 
      "then", 
        {"@id":"?:X", "@type": "schoolchild"}
    ],
    [
      "if", 
        {"@id":"?:X","http://foo.org/father":"?:Y"}, 
        {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
      "then",
        {"@id":"?:X","grandfather":"?:Z"}
    ],
    [
      "if", 
        {"@id":"?:X","http://foo.org/mother":"?:Y"}, 
        {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
      "then", 
        {"@id":"?:X","grandfather":"?:Z"}
    ],
    {"@question": [{"@id":"?:X","grandfather":"john"}, 
                    "&", 
                   {"@id":"?:X","@type":"schoolchild"}]}
    ]

The conversion to TPTP creates new blank nodes `_:crtd_1` for
the person and `_:crtd_2` and `_:crtd_3` for the children:

    fof(frm_1,axiom,(((! [X,Y] : ($arc(X,'http://foo.org/child',Y) 
                                  <=> 
                                  ($arc(Y,'http://foo.org/father',X) | 
                                   $arc(Y,'http://foo.org/mother',X)))) & 
                     (! [X,Y] : ($arc(Y,'http://foo.org/child',X) 
                                 <=> ($arc(Y,'http://foo.org/son',X) | 
                                     $arc(Y,'http://foo.org/daughter',X))))) &
                     ($arc('_:crtd_3','http://foo.org/hasage',2) &
                     ($arc('_:crtd_1','http://foo.org/child','_:crtd_3') &
                     ($arc('_:crtd_2','http://foo.org/hasage',10) & 
                     ($arc('_:crtd_1','http://foo.org/child','_:crtd_2') & 
                     $arc('_:crtd_1','http://foo.org/father',john))))))).

    fof(frm_2,axiom,(! [X,Y] : ((($less(Y,19) &
                                  $less(6,Y)) & 
                                  $arc(X,'http://foo.org/hasage',Y)) 
                     =>
                     $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',schoolchild)))).
    fof(frm_3,axiom,(! [X,Y,Z] : (($arc(Y,'http://foo.org/father',Z) & 
                                  $arc(X,'http://foo.org/father',Y)) 
                                  => 
                                 $arc(X,grandfather,Z)))).
    fof(frm_4,axiom,(! [X,Y,Z] : (($arc(Y,'http://foo.org/father',Z) &
                                   $arc(X,'http://foo.org/mother',Y)) 
                                  => 
                                  $arc(X,grandfather,Z)))).
    fof(frm_5,negated_conjecture,
          (! [X] : (($arc(X,grandfather,john) & 
                     $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',schoolchild)) 
                     => 
                     $ans(X)))).

and the proof found gives a newly created blank node for one of the children as
an answer:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","_:crtd_2"]],
    "proof":
    [
    [1, ["in", "frm_2"], [["-$arc","?:X","http://foo.org/hasage","?:Y"], 
                          ["$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"], 
                          ["-$less",6,"?:Y"], ["-$less","?:Y",19]]],
    [2, ["in", "frm_1"], [["$arc","_:crtd_2","http://foo.org/hasage",10]]],
    [3, ["mp", 1, 2], [["$arc","_:crtd_2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"]]],
    [4, ["in", "frm_1"], [["-$arc","?:X","http://foo.org/child","?:Y"],
                          ["$arc","?:Y","http://foo.org/mother","?:X"], 
                          ["$arc","?:Y","http://foo.org/father","?:X"]]],
    [5, ["in", "frm_1"], [["$arc","_:crtd_1","http://foo.org/child","_:crtd_2"]]],
    [6, ["mp", 4, 5], [["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"], 
                       ["$arc","_:crtd_2","http://foo.org/mother","_:crtd_1"]]],
    [7, ["in", "frm_4"], [["-$arc","?:X","http://foo.org/father","?:Y"], 
                          ["-$arc","?:Z","http://foo.org/mother","?:X"], 
                          ["$arc","?:Z","grandfather","?:Y"]]],
    [8, ["in", "frm_1"], [["$arc","_:crtd_1","http://foo.org/father","john"]]],
    [9, ["mp", 7, 8], [["-$arc","?:X","http://foo.org/mother","_:crtd_1"],
                       ["$arc","?:X","grandfather","john"]]],
    [10, ["mp", [6,1], 9], [["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"],
                            ["$arc","_:crtd_2","grandfather","john"]]],
    [11, ["in", "frm_5"], [["-$arc","?:X","grandfather","john"],
                           ["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"],
                           ["$ans","?:X"]]],
    [12, ["mp", [10,1], 11], [["-$arc","_:crtd_2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"], 
                              ["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"],
                              ["$ans","_:crtd_2"]]],
    [13, ["mp", 3, 12], [["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"], 
                         ["$ans","_:crtd_2"]]],
    [14, ["in", "frm_3"], [["-$arc","?:X","http://foo.org/father","?:Y"],
                           ["-$arc","?:Z","http://foo.org/father","?:X"],
                           ["$arc","?:Z","grandfather","?:Y"]]],
    [15, ["mp", 14, 8], [["-$arc","?:X","http://foo.org/father","_:crtd_1"],
                         ["$arc","?:X","grandfather","john"]]],
    [16, ["mp", 13, 15], [["$arc","_:crtd_2","grandfather","john"],
                          ["$ans","_:crtd_2"]]],
    [17, ["mp", 1, 2], [["$arc","_:crtd_2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"]]],
    [18, ["mp", 16, 11, 17], [["$ans","_:crtd_2"]]]
    ]}
    ]}



## Graphs, named graphs and quads: the $narc predicate

JSON-LD uses the `"@graph"` key for two objectives:

* If an object does not have an id but contains `"@graph"`, the value of the latter is 
  simply a list of objects in the scope of the `"@context"` of the object.
  JSON-LD-LOGIC converts such lists to conjunctions, each element using
  the same expansion rules.

* If an object has an id, the conjunction elements are not interpreted as RDF triplets, 
  but *quads* with the object id as a fourth element: the id indicates which graph the
  arc belongs to. JSON-LD-LOGIC converts such lists using the four-argument `"$narc"`
  predicate (named arc).
   
A grandfather example with two trivial graphs and a rule merging all the names graphs into one
unnamed graph:   

    [
    { 
      "@id":"bobknows",
      "@graph": [  
        {"@id":"pete", "father":"john"}
      ]
    },    
    { 
      "@id":"eveknows",
      "@graph": [  
        {"@id":"mark", "father":"pete"}
      ]
    }, 
    {
      "@name":"mergegraphs",
      "@logic": [["$narc","?:X","?:Y","?:Z","?:U"],"=>",["$arc","?:X","?:Y","?:Z"]]
    },  
    {
      "@name":"gfrule",
      "@logic": ["if", 
                    {"@id":"?:X","father":"?:Y"}, 
                    {"@id":"?:Y","father":"?:Z"}, 
                 "then", 
                    {"@id":"?:X","grandfather":"?:Z"}]
    },      
    {"@question": {"@id":"?:X","grandfather":"john"}}
    ]

Conversion to TPTP gives

    fof(frm_1,axiom,$narc(pete,father,john,bobknows)).
    fof(frm_2,axiom,$narc(mark,father,pete,eveknows)).
    fof(mergegraphs,axiom,(! [X,Y,Z,U] : ($narc(X,Y,Z,U) => $arc(X,Y,Z)))).
    fof(gfrule,axiom,(! [X,Y,Z] : (($arc(Y,father,Z) & $arc(X,father,Y))
                                   => 
                                   $arc(X,grandfather,Z)))).
    fof(frm_5,negated_conjecture,(! [X] : ($arc(X,grandfather,john) => $ans(X)))).

And we get the expected result

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","mark"]],
    "proof":
    [
    [1, ["in", "gfrule"], [["-$arc","?:X","father","?:Y"], 
                           ["-$arc","?:Z","father","?:X"],
                           ["$arc","?:Z","grandfather","?:Y"]]],
    [2, ["in", "mergegraphs"], [["-$narc","?:X","?:Y","?:Z","?:U"], 
                                ["$arc","?:X","?:Y","?:Z"]]],
    [3, ["in", "frm_1"], [["$narc","pete","father","john","bobknows"]]],
    [4, ["mp", 2, 3], [["$arc","pete","father","john"]]],
    [5, ["mp", 1, 4], [["-$arc","?:X","father","pete"],
                       ["$arc","?:X","grandfather","john"]]],
    [6, ["in", "frm_2"], [["$narc","mark","father","pete","eveknows"]]],
    [7, ["mp", 2, 6], [["$arc","mark","father","pete"]]],
    [8, ["mp", 5, 7], [["$arc","mark","grandfather","john"]]],
    [9, ["in", "frm_5"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [10, ["mp", 8, 9], [["$ans","mark"]]]
    ]}
]}


## Numbers and arithmetic


* JSON numbers without a fractional part and exponent stand for integers in the typed
  fragment of TPTP. Different integers are assumed to be inequal.
  Examples:

  `23, -1, 0`

* JSON numbers with a fractional part stand for reals in the typed
  fragment of TPTP. Integers are separate from reals, thus `4` is different 
  from `4.0000`. Examples: 
  
  `23.45, 0.0, -4.0000000, 1.0E+2`  

* The following TPTP prefix form arithmetic predicates and functions on
  integers and reals are used with the same meaning as in TPTP:
  
   * Type detection predicates "$is_int", "$is_real".
   
   * Comparison predicates "$less", "$lesseq","$greater", "$greatereq".

   * Type conversion functions "$to_int", "$to_real".

   * Arithmetic functions on integers and reals:
      "$sum", "$difference", "$product", 
      "$quotient", "$quotient_e",
      "$remainder_e", "$remainder_t", "$remainder_f", 
      "$floor", "$ceiling",
      "$uminus", "$truncate", "$round",       
      "$is_number"`
    
    Note: the arithmetic functions take either two arguments or one argument:
    they are not allowed to be applied to long lists of arguments.

    Example: `["$less",["$sum",1,["$to_int",2.1]],["$product",3,3]]`

* Additional convenience predicate is used: "$is_number" is true
  if and only if "$is_int" or "$is_real" is true.

* Additional infix convenience functions `"+", "-", "*", "/"` are
  used with the same meaning as $sum", "$difference", "$product" and 
  "$quotient", respectively.

  Example: `["$less",[1,"+",[1,"+",2]],[3,"*",3]]`

  Note: these arithmetic functions take exactly two arguments and
  can occur only in the lists with the length three.
  


## null

* JSON `null` value stands for an unknown value similar to the SQL null value. 
  The meaning for `null` will be given in a later chapter.



      ' "$sum", "$less", "$is_int"` are used as function, predicate or type symbols
      with a concrete predefined semantics in TPTP. We note that all the
      predefined functions and predicates in TPTP start with `$`.

      The full list of predefined strings in the core fragment:

      `"~", "|", "&",  "<=>", "=>", "<=", "<~>", "~|", "~&", "@", 
      "forall", "exists", 
      "=", "!=",      
      "$less", "$lesseq","$greater", "$greatereq",
      "$sum", "$difference", "$product", "$quotient", "$quotient_e",
      "$remainder_e", "$remainder_t", "$remainder_f", "$floor", "$ceiling",
      "$uminus", "$truncate", "$round", 
      "$is_int", "$is_real",     
      "$to_int", "$to_real".
      "$is_number"`

      The full language adds symbols 

      `"$is_atom",  "$is_distinct", "$is_list",
       "$list", "$nil", "$first", "$rest"
      

    * Strings bound by a quantifier in JSON stand for corresponding variables in
      FOL and TPTP. A bound variable *must* start with an upper case letter.     
      In the following example all the variables are existentially quantified:        

      `["exists",["X1","X2","U"],["p","X1","X2","U"]]`
   
    * Strings starting with a question mark and colon like  "?:X1" 
      and *not bound by a quantifier* 
      stand for free variables in FOL and are assumed to be universally quantified
      in TPTP. Examples: 

      `"?:X", "?:Y1", "?:Some car"`    
   
    * Strings prefixed by `"#:"` (for distinct) stand for distinct objects in TPTP.
      Such "#:someobj" is always considered unequal to:
      
      * any other syntactically different distinct object like "#:otherobj", 
      * any object with either an arithmetic type like integer, real.
      * any list, i.e. either "$nil" or a term led by "$list".

      However, a distinct object like "#:someobj" may be equal to an 'ordinary' object
      like "some_thing".

    * Strings prefixed by `"~"` and located in the first position of a list in a
      formula context are assumed to construct a negated atom led by a symbol
      constructed from the rest of the string. 
      For example, the arguments of the following "&" are equivalent:

      `[["~p", 1], "&", ["~", ["p",1]]`
      
      Note: multiple "~" like "~~p" do create a nested negation.
      

    * Strings without any of the previously described prefixes `d:`, `r:`, `~` and 
      *starting with a lower case or a non-alphabetic letter* 
      in a term stand for ordinary predicate or function symbols, unless
      they are one of the predefined strings like `"&", "$sum", "$real"` etc.
      Examples:

      `"p", "foo", "bar 23", "_here", "++something", "1 to 3"`



The `$distinct` is a special unlimited-length-list prefix predicate for specifying
that many terms are  unequal to each other. Example:

  `["$distinct","john","mike","pete","andrew"]` 

The full list of arithmetic predicates and functions:
    
   `"$less", "$lesseq","$greater", "$greatereq",
   "$uminus", "$sum", "$difference", "$product", "$quotient", "$quotient_e",
   "$remainder_e", "$remainder_t", "$remainder_f", "$floor", "$ceiling",
   "$truncate", "$round", "$is_int", "$is_rat", "$to_int", "$to_rat", "$to_real".`

An example using arithmetic:

   `["$less", ["$sum",1,2], ["$to_int",4.56]]`

The full list of predefined types, all separate from each other:

   `"$int", "$rat", "$real", "$i", "$o"`


### Configurable differentiation between variables and constants 

The JSFOL requirement that all variables start with an upper case letter and
constants not, is dropped: any string bound by a quantifier is considered
to be a variable, including strings starting with lower case letters.

The variable/constant distinction and special string prefixes can be
configured in a surrounding metainformation block by using the following 
keys:

*  `"variable_prefix"` : if present, then any string starting with the value
  of the key is considered to be a free variable, all the other strings are 
  considered to be constant/function/predicate symbols unless they are bound,
  predefined or have some other defined prefix.
*  `"distinct_prefix"` : similarly redefines the default `d:` prefix for
  distinct symbols.
*  `"rational_prefix"` : similarly redefines the default `r:` prefix for
   rationals.

In case a `"variable_prefix"` is not defined, the JSFOL upper case first character
convention still holds for distinguishing free variables and constants.

In the following example `"x"` is a quantified variable, `"?something"` 
is a free variable, `"##an_object"`is a distinct constant and
`"John"` is an ordinary constant:

      {"variable_prefix" : "?",
       "distinct_prefix" : "##",
       "content": ["formulas",
         ["forall",["x"],["p","x","?something","##an_object","John"]]
       ]
      }

### Questions


A question indicates that in addition to proving a conjecture, suitable values
for the existentially quantified variables should be found.

Citing TPTP: questions are written like conjectures, but using the *question* role
instead of the *conjecture* role. The outermost existentially quantified variables
are the ones that the user wants values for, i.e., their sets of instatiations are
the answer. Example:

    {"role": "question",
     "content": ["exists","["X","Y"], ["brother","X","Y"]] }


### Additional predefined predicates and functions present in JSFOL+

  
The following infix arithmetic functions and comparison predicates operate on
all number types and are converted to the obviously corresponding functions/predicates
of the core JSFOL:

* Infix `"*","+","-","/"`.

The functions above are left-associative.
  

### Additional logical connectives in JSFOL+


The following "convenience" connectives and constants `-, not, and, or, if ... then ..., null`
are translated to standard JSFOL constructions.

The minus sign `-` can be used instead of the tilde `~` both as a logical negation and
as a prefix of a string, indicating that the string is a predicate which has to be
negated.

The `"not"` connective can be used as a negation connective instead of the tilde `"~"`.

The `"and"` and `"or"` connectives can be used instead of the "&" and "|" correspondingly,
whereas they can be used both in the infix and prefix form, the latter form taking an 
arbitrary number of arguments like this:

  `["and", ["p",1], ["foo","?X", "a], ["bar"]]`


The `"and"` with no arguments is equivalent to `true` and the `"or"` with no arguments to `false`.

The mixfix operator `if..then..` can be used. The list of formulas before `then` is treated
as a conjunction premiss of the implication and the list after `then` as a disjunction
consequent of the implication.  Example:

  `["if", "a", "b", "c", "then", "d", "e"]`

  is translated as 

  `[["a","&","b","&","c"], "=>", ["d","|","e"]]`  


The `null` symbol of JSON is treated analogously to the SQL `null` representing a missing
or unknown value. The translation mechanism of eliminating a `null` inside some formula
`F` is as follows:

* Create a new variable `V` not occurring in `F`. I.e. `V` is a new string.
* Replace an atom `[...null....]` where the eliminated `null` occurs by a formula

   `["exists", [V], [....V....]]`

where this particular occurrence of `null` is replaced by the variable `V'.

Example:

   `["father","john",null]` 

is translated as 

   `["exists", ["X"], ["father","john","X"]]`. 

Example:

   `[["is_father","X"], "<=>", ["father","X",null]]` 

is translated as 

   `[["is_father","X"], "<=>", ["exists", ["Y"], ["father","X","Y"]]]`
    ` 
   

