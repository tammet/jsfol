
JSFOL: JSON representation of first-order logic formulas
============================================================

*Draft version 0.3: looking for discussion and feedback*


tanel.tammet@gmail.com

with contributions from schulz@eprover.org and geoff@cs.miami.edu

Changes in 0.3
--------------

* Two separate language layers introduced: core JSFOL and an extended JSFOL+ containing
  additional convenience functions/predicates/connectives.
* Rationals pulled in from TPTP.
* Integers, rationals and reals considered separate.
* Unique symbols corresponding to distinct constants in TPTP considered separate
  from integers, rationals, reals.
* Strings thrown out.
* Predefined functions/predicates/connectives in JSFOL follow TPTP exactly.
* "formulas" list introduced for independent self-contained formulas, like clauses.
* JSFOL+ introduces type-independent predefined arithmetic functions/predicates and
  several convenience logical connectives like -, and, or, if ... then ...
* JSFOL+ introduces SQL-inspired handling of JSON `null`




Goals and summary
------------------


The goal of the JSFOL draft is proposing a simple JSON syntax for writing
first order logic [FOL](https://en.wikipedia.org/wiki/First-order_logic "FOL") 
formulas in a machine-processable manner. 

A simple example stating some facts and rules:

    ["formulas",
      ["brother","John","Mike"],
      ["brother","John","Pete"],
      [["brother","?X","?Y"], "=>", ["brother","?Y","?X"]],
      [[["brother","?X","?Y"], "&",  ["brother","?Y","?Z"]] "=>", ["brother","?X","?Z"]]
    ]

Another example with metainformation added, using a convenience connective `if...then...` 
and indicating a conjecture to be proved:

    {"name": "grandfather example",
      "role": "query",      
      "content": ["formulas",
      ["father","John","Mark"],
      ["mother","Eve","Mark"],
      ["father","Donald","Eve"],
      ["if" ["father","?X","?Y"], ["father","?Y","?Z"], "then", ["grandfather","?X","?Z"]],
      ["if" ["father","?X","?Y"], ["mother","?Y","?Z"], "then", ["grandfather","?X","?Z"]],
      {"role": "conjecture",
        "content": ["exists",["X"],["grandfather","X","Mark"]] 
      }]
    }

The third example encodes the last problem using equality and explicit quantifiers:

    {"name": "grandfather example",
     "role": "query",      
     "content": ["formulas",
      [["father","Mark"], "=", "John"],
      [["mother","Mark"], "=", "Eve"],
      [["father","Eve"], "=", "Donald"],
      ["forall",["gchild"], ["grandfather", ["father",["father","gchild"]], "gchild"]],
      ["forall",["gchild"], ["grandfather", ["father",["mother","gchild"]], "gchild"]],
      {"role": "conjecture",
       "content": ["exists",["X"],["grandfather","X","Mark"]] 
      }]
    }

The JSFOL syntax is inspired by and is intendeded to be compatible with the 
[TPTP](http://tptp.org "TPTP") syntax used by automated reasoners, see the
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual").

In particular, the proposed syntax should guarantee clear JSON->TPTP transformation, 
while not covering all the TPTP features like higher order formulas; 
the TPTP->JSON conversion is currently possible only for the non-higher order 
fragment (FOF and CNF fragments) of TPTP along with a small subset of the typed
fragment (TFF with arithmetic) of TPTP.

The main proposals of the syntax are:

* Terms and atoms are represented as JSON lists with predicate/function symbols 
  in the first position (prefix form).
* JSON strings can represent either variables, constant/function/predicate symbols,  
  unique symbols or typed data, using a special optional *prefix* 
  of a JSON string.
* JSON objects like  `{"name": "example", "content": ["p","?X",1]}` are used for 
  adding metainformation to terms, atoms, formulas and formula lists.

The semantics of JSFOL constructions is only partially defined in the current draft:
we assume "standard" semantics of most of the FOL constructions and leave the semantics
of any  special cases intentionally open,to be determined by the applications. 
For example, the JSFOL specification does not determine whether applications should
use classical or intuitionistic logic for proofs.

Layers of JSFOL
---------------

The `core JSFOL` specifies minimal JSON syntax for writing logic formulas 
which can be translated to the TPTP language. 

The `JSFOL+` extension layer adds several convenience functions and operators
which can be converted to core JSFOL.

We will first present the core JSFOL and thereafter two separate chapters with JSFOL+ extensions.

JSFOL objects can be viewed as falling into one of five categories,
from bottom to top:

* `Primitives` are constants, function/predicate symbols and variables 
   represented by JSON primitives.
* `Terms and atoms` are built of primitives and terms as nested JSON lists.
* `Formulas` are built of atoms using logical contructors like "&", "exists",
   using nested JSON lists.
* `Formula lists` are JSON lists of fully formed, independent standalone formulas.
* `Metainformation` can be attached to any of the previous categories, represented as
   JSON objects consisting of key/value pairs.



Primitives
---------- 

* JSON numbers without a fractional part and exponent stand for integers in FOL, 
  translated to the TPTP `$int` type. 
  Examples:

  `23, -1, 0`

* JSON numbers with a fractional part or an exponent stand for floating point 
  numbers in FOL, translated to the TPTP `$real` type. Integers and rationals 
  are separate from reals, thus `4` is different from `4.0000`.
  Examples: 
  
  `23.45, 0.0, -4.0000000, 1.0E+2`  
  
* JSON booleans `true` and `false` stand for corresponding truth values, represented
  as `$true` and `$false` in TPTP.
 
* JSON `null` value can be used only in JSFOL+ where is stands for 
  an unknown value similar to the SQL null value. It is prohibited
  in pure core JSFOL. JSFOL+ meaning for `null` will be given in a later chapter.

* JSON strings have multiple uses covered in the following list:

    * Special predefined strings like `"&", "|", "=>"` at the formula level are used
      for constructing formulae, while other predefined strings like
      `"=", "+", "$less"` are used as function or predicate symbols with a concrete 
      predefined semantics.

    * Strings bound by a quantifier in JSON stand for corresponding variables in
      FOL and TPTP, regardless of the form or prefix of the string. In the
      following example all the variables are existentially quantified 
      and have no associated types:      

      `["exists",["X1","X2","?U", "r:variable"],["p","X1","X2","?U","r:variable"]]`
   
    * Strings prefixed by `?` like `"?X1"` stand for free variables in FOL 
      and are assumed to be universally quantified in TPTP, unless already explicitly
      bound by a quantifier in JSON.
      Example: 

      `["r","?X1,"?y"]`

      A single question mark "?" also stands for a free variable. 
      In the following example the first and second "?" represent the same variable:

      `["=","?","?"]`
   
    * Strings prefixed by `"u:"` (for unique) stand for distinct objects in TPTP,
      where they are written in double quotes. Such "u:someobj" is always considered
      unequal to:
      
      * any other syntactically different distinct object like "u:otherobj", 
      * any typed object like integer, rational, real, string, datetime, etc.

      However, a distinct object like "u:someobj" may be equal to an 'ordinary' object
      like "some_thing".
      
      Example:

      `"u:there_is_nobody_like_me"`
   
    * Strings prefixed by `"r:"` (for rational) stand for rational numbers (`$rat` type
      in TPTP), represented as two integers separated by a slash "/". 
      
      Example:

      `"r:2/30"`   

    * Strings prefixed by "~" and located in the first position of a list in a
      formula context are assumed to construct a negated atom led by a symbol
      constructed from the rest of the string. 
      For example, the arguments of the following "and" are equivalent:

      `["and" ["~p", 1], ["not", ["p",1]]`
      
      Note: multiple "~" like "~~p" do create a nested negation.
      

    * Strings without any of the previously described prefixes in a term stand for
      ordinary predicate or function symbols.
      Examples:

      `"p", "foo", "bar23"`

      Note: this category includes an empty string "", as well as strings containing
      ":" or "#" like "here:something" or "#note", unless they fall into some of the
      previous categories.
  
   
   
Terms and atoms
---------------

Terms and atoms are represented as JSON lists, using the prefix form analogous to
s-expressions: the first element of a list stands for a predicate or a function symbol.

Example:

`["foo",1,"?X",["f",["f","?X]]]`

stands for a TPTP atom or term 

`foo(1,?X,f(f(?X)))` 

where `?X` is a free variable.

An empty JSON list `[]` is assumed to be an ordinary constant symbol and is distinct
from the JSON `null`.

JSFOL does not require that predicate or function symbols have a unique arity, although
applications may pose such restrictions.

The question as for which types of values are allowed at what position is left
to applications. 

For example, neither classical FOL nor the CNF and FOF fragments of TPTP allow variables
as a leading element of a term, i.e. in a function / predicate symbol position. However,
JSFOL itself does not prohibit such usage.


Predefined predicates and functions in core JSFOL
-------------------------------------------------


All the TPTP predicates and functions described in the chapters "The Formulae Section"
and "The TPTP Arithmetic System" of the
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual")
like `"$quotient"` are represented as strings in exactly the same form as in TPTP.  

Most importantly, `"="` and `"!="` stand for equality and inequality and can occur
only in the middle of a three-element list (infix form), like

  `["a","=","b"]`
  `[1,"!=",2]`

The `$distinct` is a special unlimited-length-list predicate for specifying that many terms are 
unequal to each other. Example:

  `["$distinct","John","Mike","Pete","Andrew"]`  

All the other predefined predicates and functions are used only for arithmetic and may occur only as a first element
(prefix form) of a three- or two-element list, like

   `["$less", ["$sum",1,2], ["$to_int",4.56]]`


Formulas
--------

Formulas are built from booleans, atoms and formulas using ordinary logical connectives,
used in the infix form in core JSFOL.

The following connectives are predefined, whereas `"~"`
may occur as a first element of a two-element list and all the other connectives 
may only occur between elements of a list (infix). A list may not contain different 
connectives: instead of

  `["a","&","b","=>","c"]`

  one must use
  
  `[["a","&","b"],"=>","c"]`

The binary connectives are left associative. For example,

  `["a","=>","b","=>","c"]`

is interpreted as

  `[["a","=>","b"],"=>","c"]` 

The connectives are:  

* `"~"` stands for negation and takes a single argument. Note that as we have said before:
    * `["~p",1]` stands for `["~",["p",1]]`
    * `["~~p",1]` stands for `["~", ["~",["p",1]]]`
     
*  Based on the TPTP language, the following connectives are available, all of them
   used only in the infix form:

    * infix `"|"` for disjunction,
    * infix `"&"` for conjunction, 
    * infix `"<=>"` for equivalence, 
    * infix `"=>"` for implication, 
    * infix `"<="` for reverse implication, 
    * infix `"<~>"` for non-equivalence (XOR), 
    * infix `"~|"` for negated disjunction (NOR), 
    * infix `"~&"` for negated conjunction (NAND), 
    * infix `"@"` for application, used mainly in the higher-order context.

* The untyped quantifiers `"exists"` and `"forall"` 
  are reprented as a list of three elements: quantifier, list of variables, formula.
  Examples:

  `["exists",["X"],["p","X"]]`

  `["forall",["X","Y2"],["p","Y2","X"]]`
  
* The *typed* quantifiers are represented similarly, with a two-element list standing
  for a typed variable. Examples:
  
  `["exists",[["X","$int"]],["p","X"]]`

  `["forall",[["X","$int"],["Y2","$real"]],["p","Y2","X"]]`
  
  where the predefined type names for core JSFOL are: `"$int", "$rat", "$real"




Formula list
------------

A set of formulas can be represented as a list with a first element being `"formulas", 
like this:
 
    ["formulas",
      ["brother","John","Mike"],
      ["brother","John","Pete"],
      [["brother","?X","?Y"], "=>", ["brother","?Y","?X"]],
      [["is_father","?X"], "<=>", [["exists"],["Y"],["father,"?X","Y"]]]      
    ]

The elements of the formula list (except the leading `"formulas"`) are translated as a conjunction
of the elements, with the following differences from a simple conjunction:

Each formula in the list containing free variables (prefixed by "?") is translated by binding
the free variables in this formula by a "forall" quantifier at the top level of this formula.
The previous example is thus translated as:

    ["and",
      ["brother","John","Mike"],
      ["brother","John","Pete"],
      ["forall",["?X","?Y"], [["brother","?X","?Y"], "=>", ["brother","?Y","?X"]]],
      ["forall",["?X"], [["is_father","?X"], "<=>", [["exists"],["Y"],["father,"?X","Y"]]]]      
    ]
 
Notice that 

* The existentially quantified "Y" variable in the last formula is dependent on
  the leading "?X" variable of the same formula.

* The free variables "?X" in the last two formulas are distinct from each
  other.


  

   
Metainformation and file import
-------------------------------

Any kind of metainformation (formula/term names, confidence measures, context indicators, comments etc) 
is represented using JSON objects and can be attached to formula lists, formulas, and terms. Example:

    {"name":"example formula",
    "confidence": 0.8,
    "context": "examples",
    "include": ["/opt/axioms/a1.ax","http://example.org/ax2"],
    "role", "axiom",
    "content": ["p",1,"a"] }
 
 
where the predefined keys are:

* `"name"` for the construction name: this is a *suggested* key name without logical meaning
* `"include"` value is a list of files and urls containing JSON formulas which should be imported
* `"role"` value should be normally either "query", "axiom", "assumption", "conjecture" or a
  "negated_conjecture": normally used for annotating formulas for their intended use.
   A longer description and more options are given below.
* `"content"` for the logical content (formula list, formula or a term)

All of these key/value pairs are optional. If "content" is not present, the object should
be (normally) just ignored by the application, unless it contains "include" or is a member
of a formula list. Example:

`["p",{"comment":"look that!","content": "a"}, {"note": "what?"}]`

is treated as

`["p","a"]`

In case a formula list contains a JSON object, the 

If `"role"` is not present and the construction is at the top level of a file or a "formulas" list,
then its value is assumed to be "axiom". No default value is given to a "role" value inside a
nested formula or term, since it would not have a sensible meaning.

`"query"` is a special *role* value not present in TPTP: the intended meaning is that an annotated
construction represents a full query to be answered, in other words, a theorem intended to be proved or
disproved. In TPTP single problem files stand for the `"query"` role.

We note that `"conjecture"` role value in TPTP forces the content to be negated when asking for unsatisfiability
of a conjunction. We bring two equivalent example formulas, where in the first case "conjecture" is used with the positive "a" and in the second "negated_conjecture" with the negative "-a":

    ["and", 
      "a",
      "b",
      {"role": "conjecture", "content": "a"}]

represents a provable propositional formula `(a & b) => a`, the negation of which
is equivalent to an unsatisfiable `(a & b) & -a`

whereas 

    ["and", 
      "a",
      "b",
      {"role": "negated_conjecture", "content": "-a"}]

also represents an unsatisfiable propositional formula `(a & b) & -a`

For other *role* values we cite "The Formulae Section" of 
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual"): 
"The role gives the user semantics of the formula, one of `axiom, hypothesis, definition, assumption, lemma, theorem, corollary, conjecture, negated_conjecture, plain, type`, and `unknown`. ..."

Treating of any other key values except "include", "content" and "role", is up to the application.

In case "include" occurs inside atoms/terms and not at the top of the formula, then these
files/urls should be included as well. In other words, "include" is a global operator.
The presence of "include" without any "content" key should still trigger import of the
indicated files/urls.

Objects, i.e. metainformation may be nested.

JSON streaming
--------------

The files in the form of several sequential JSON constructions 
(see [JSON streaming](https://en.wikipedia.org/wiki/JSON_streaming "JSON streaming"))
like

    <construction1>
    <construction2>
    ...


should be preferrably treated as a `formula list`, i.e. a list

    ["formulas",
      <construction1>,
      <construction2>,
      ...]

where each separate JSON construction is an independent self-contained formula.

In case the stream starts with a JSON object without the "content" key, the
whole following "formulas" list of the stream is treated as a value of the "content"
key.

This cannot be forced though, in case a JSON parser of an application does not support JSON streaming. 
Example:

    {"name": "streaming example"}

    ["forall",[["X","int"],["Y2","string"]],["p","Y2","X"]]
    {"name":"example formula",
    "confidence": 0.8,
    "context": "examples",
    "include": ["/opt/axioms/a1.ax","http://example.org/ax2"],
    "content": ["p",1,"a"] }

    ["exists",[["X","int"]],["p","X"]]

should be preferrably treated as

    {"name": "streaming example",
     "content":, ["formulas",

    ["forall",[["X","int"],["Y2","string"]],["p","Y2","X"]],
    {"name":"example formula",
    "confidence": 0.8,
    "context": "examples",
    "include": ["/opt/axioms/a1.ax","http://example.org/ax2"],
    "content": ["p",1,"a"] },

    ["exists",[["X","int"]],["p","X"]]

    ]
    }



Additional predefined predicates and functions present in JSFOL+
----------------------------------------------------------------
  
The following arithmetic functions and comparison predicates operate on
all number types and are converted to the obviously corresponding functions/predicates
of the core JSFOL:

* Infix `"*","+","-","/"`.

* Infix `">"`, `">=="` and `"<", "<=="`


Notice that we use two equality symbols to avoid confusion with backward implication `"<="`.
  

Additional logical connectives in JSFOL+
----------------------------------------

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


The "and" with no arguments is equivalent to `true` and the "or" with no arguments to `false`.

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

   `["father","John",null]` 

is translated as 

   `["exists", ["X"], ["father","John","X"]]`. 

Example:

   `[["is_father","?X"], "<=>", ["father","?X",null]]` 

is translated as 

   `[["is_father","?X"], "<=>", ["exists", ["Y"], ["father","?X","Y"]]]`
    ` 
   

