
JSFOL: JSON representation of first-order logic formulas
============================================================

*Draft version 0.2: looking for discussion and feedback*

tanel.tammet@gmail.com


Goals and summary
------------------


The goal of the JSFOL draft is proposing a simple JSON syntax for writing
first order logic [FOL](https://en.wikipedia.org/wiki/First-order_logic "FOL") 
formulas in a machine-processable manner. A quick example containing some
facts, rules and a conjecture to be proved from those:

    {"name": "grandfather example",
     "role": "query",      
     "content": ["and",
      ["father","John","Mark"],
      ["mother","Eve","Mark"],
      ["father","Donald","Eve"],
      ["if" ["father","?X","?Y"], ["father","?Y","?Z"], "then", ["grandfather","?X","?Z"]],
      ["if" ["father","?X","?Y"], ["mother","?Y","?Z"], "then", ["grandfather","?X","?Z"]],
      {"role": "conjecture",
       "content": ["exists",["X"],["grandfather","X","Mark"]] 
      }]
    }

The syntax is inspired by and is intendeded to be compatible with the 
[TPTP](http://tptp.org "TPTP") syntax, see the
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual").

In particular, the proposed syntax should guarantee clear JSON->TPTP transformation, 
while not covering all the TPTP features like higher order formulas; 
the TPTP->JSON conversion is currently possible only for the non-higher order 
fragment (FOF and CNF fragments) of TPTP with a small subset of the typed
fragment (TFF) of TPTP being covered as well.

The main proposals of the syntax are:

* Terms and atoms are represented as JSON lists with predicate/function symbols 
  in the first position (prefix form).
* JSON strings can represent either variables, constant/function/predicate symbols,  
  strings-as-data, unique symbols or typed data, using a special optional *prefix* 
  of a JSON string.
* Booleans, integers, floats, strings and datetimes are predefined types. 
  Other types could be added, but the syntactic mechanisms have not been worked out yet.
* JSON objects like  `{"name": "example", "content": ["p","?X",1]}` are used for 
  adding metainformation to terms, atoms and formulas.

The semantics of JSFOL constructions is only partially defined in the current draft:
we assume "standard" semantics of most of the FOL constructions and leave the semantics
of any  special cases intentionally open,to be determined by the applications. 
For example, the JSFOL specification does not determine whether applications should
use classical or intuitionistic logic for proofs.

Primitives
----------

* JSON numbers without a fractional part stand for integers in FOL, 
  translated to the TPTP `$int` type. 
  Examples:

  `23, -1`

* JSON numbers with a fractional part stand for floating point numbers (floats) in FOL,
  assumed to be a subset of the TPTP `$real` type. A float with
  only zeros after a period is equal to a corresponding integer.
  Examples: 
  
  `23.45, -4.0000000`  
  
* JSON booleans `true` and `false` stand for corresponding truth values, represented
  as `$true` and `$false` in TPTP.
 
* JSON `null` value stands for a constant symbol `null` without any special
  properties.

* JSON strings have multiple uses, described in the following list:

    * Several special predefined strings like `"not", "or", "and"` at the formula level are used
      for constructing formulae, while some other predefined strings like `"=", "+", "<"` are
      used as function or predicate symbols with a concrete predefined semantics.

    * Strings bound by a quantifier in JSON stand for corresponding variables in
      FOL and TPTP, regardless of the form or prefix of the string. In the
      following example all the variables are existentially quantified 
      and have no associated types:      

      `["exists",["X1","X2","?U", "s:variable"],["p","X1","X2","?U","s:variable"]]`
   
    * Strings prefixed by `?` like `"?X1"` stand for free variables in FOL 
      and are assumed to be universally quantified in TPTP, unless already explicitly
      bound by a quantifier in JSON.
      Example: 

      `["r","?X1,"?y"]`

      A single question mark "?" also stands for a free variable. 
      In the following example the first and second "?" represent the same variable:

      `["=","?","?"]`
   
    * Strings prefixed by `"u:"` (for unique) stand for distinct objects in TPTP,
      where they are written in double quotes. Citing TPTP: all "distinct object"s are 
      unequal to all "different distinct object"s (but not necessarily unequal to
      any other constants), Example:

      `"u:there_is_nobody_like_me"`
   
    * Strings prefixed by `"s:"` (for string) stand for objects of a character
      string type, which are distinct between each other and unequal to any booleans,
      integers, floats, datetimes. Example:

      `"s:I am a real, distinct string"`
   
    * Strings prefixed by `"d:"` (for date) stand for datetimes or their intervals, 
      represented as an ISO datetime or an initial part of an ISO datetime string.
      Examples: 

      `"d:2015-12-10T24:12:34",  "d:2015-12-10", "d:2015"`

      Exact semantics of datetimes and their intervals is left to applications.
   
    * Strings prefixed by `"#some_name:"` where `some_name` is an arbitrary string,
      stand for objects of type `some_name`, represented as as a string. 
      Using such tupes typically assumes that the type has been defined:
      however, we will not cover the type definition syntax in this draft.
      Note that predefined type prefixes "s:" and "d:" do not use the "#" prefix.
      Example:

      `"#animal:dog"`

    * Strings prefixed by `"-"` and located in the first position of a list in a
      formula context are assumed to construct a negated atom led by a symbol
      constructed from the rest of the string. 
      For example, the arguments of the following "and" are equivalent:

      `["and" ["-p", 1], ["not", ["p",1]]`
      
      Note: multiple "-" like "--p" do not create a nested negation.

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



Predefined predicates and functions
-----------------------------------

`"="` and `"!="` stand for equality and inequality, if used either as:

* the first element of a three-element list, like
  
  `["=","a","b"]`

* or the middle element of a three-element list, like

  `["a","=","b"]`
  
Note: `"-="` is considered equivalent to `"!="`.  

`"*","+","-","/"`stand for corresponding functions on numbers, if used either 

* as a first element of a list with length N
* as a second element of a list with length 3.

`">"`, `">=="` and `"<", "<=="` stand for corresponding functions on numbers, dates and strings, and they
can be used either 

* as a first element of a list with length 3
* as a second element of a list with length 3.

Notice that we use two equality symbols to avoid confusion with backward implication `"<="`.
  
TPTP functions described in the chapter "The TPTP Arithmetic System" of the
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml "TPTP technical manual")
like `"$quotient"` are represented as strings in exactly the same form as in TPTP.   

The question whether these predefined functions can be applied only to some types 
is left for the applications to decide. JSFOL syntax does not prohibit constructions
like 

`[1, ">", ["a","+",3.4]]`

although such constructions may be either disallowed by some applications or have
special semantics.




Formulas
--------

Formulas are built from booleans, atoms and formulas using ordinary logical constructors, which are, 
depending on the constructor, used either as a first element of a list (prefix) or between list elements
(infix) or in both ways.

The following constructors are predefined:

* `"not"` takes a single argument. Note that we have said before:
    * `["-p",1]` stands for `["not",["p",1]]`,
    * Both `"!="` and `"-="` are used for constructing negated equality. For example,
       
       `["a","!=","b"],  ["a","-=","b"], ["!=", "a", "b"], ["-=", "a", "b"]`
       
       are all equivalent to

       `["not",["=","a","b"]`


* `"and"` and `"or"` in a prefix position take N arguments, including no arguments 
  (in the latter case meaning `true` and `false`, correspondingly). Notice 
  the use of `"and"`, `"or"` instead of `"&"`, `"|"` used in TPTP. Example:
  
  `["and", ["p",1], ["foo","?X", "a], ["bar"]]`
  

*  Similarly to the TPTP language, the following connectives are used in the infix form:

    * infix `"or"` for disjunction (TPTP |), additionally to prefix usage
    * infix `"and"` for conjunction, (TPTP &), additionally to prefix usage
    * infix `"<=>"` for equivalence, 
    * infix `"=>"` for implication, 
    * infix `"<="` for reverse implication, 
    * infix `"<~>"` for non-equivalence (XOR), 
    * infix `"~|"` for negated disjunction (NOR), 
    * infix `"~&"` for negated conjunction (NAND), 
    * infix `"@"` for application, used mainly in the higher-order context.

* There is an additional mixfix operator `if..then` not present in TPTP. Example:

    `["if", "a", "b", "c", "then", "d", "e"]`

    is translated as 

    `[["and","a","b","c"], "=>", ["or","d","e"]]`

* The untyped quantifiers `"exists"` and `"forall"` 
  are reprented as a list of three elements: quantifier, list of variables, formula.
  Examples:

  `["exists",["X"],["p","X"]]`

  `["forall",["X","Y2"],["p","Y2","X"]]`
  
* The *typed* quantifiers are represented similarly, with a two-element list standing
  for a typed variable. Examples:
  
  `["exists",[["X","int"]],["p","X"]]`

  `["forall",[["X","int"],["Y2","string"]],["p","Y2","X"]]`
  
  where we could use any type names instead of "int" and "string". 
  
  The predefined type names are: `"bool", "int", "float", "string", "datetime"`


   
Metainformation and file import
-------------------------------

Any kind of metainformation (formula/term names, confidence measures, context indicators, comments etc) 
is represented using JSON objects and can be attached to formulas, parts of formulas and terms. Example:

    {"name":"example formula",
    "confidence": 0.8,
    "context": "examples",
    "include": ["/opt/axioms/a1.ax","http://example.org/ax2"],
    "role", "axiom",
    "content": ["p",1,"a"] }
 
 
where the predefined keys are:

* `"name"` for the construction name: this is a *suggested* key name without logical meaning
* `"include"` value is a list of files and urls containing JSON formulas which should be imported
* `"role"` value should be normally either "query", "axiom", "assumption", "conjecture" or a "negated_conjecture": normally used for annotating formulas for their intended use. A longer description and more options are given below.
* `"content"` for the logical content (formula or a term)

All of these key/value pairs are optional. If "content" is not present, the object should
be (normally) just ignored by the application, unless it contains "include". Example:

`["p",{"comment":"look that!","content": "a"}, {"note": "what?"}]`

is treated as

`["p","a"]`

If `"role"` is not present and the construction is at the top level of a file or a topmost "and" list
of a top-level formula, then its value is assumed to be "axiom". No default value is given to a "role" value
inside a nested formula or term, since it would not have a sensible meaning.

`"query"` is a special *role* value not present in TPTP: the intended meaning is that an annotated
formula represents a full query to be answered, in other words, a theorem intended to be proved or
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
"The role gives the user semantics of the formula, one of `axiom, hypothesis, definition, assumption, lemma, theorem, corollary, conjecture, negated_conjecture, plain, type`, and `unknown`. The axiom-like formulae are those with the roles axiom, hypothesis, definition, assumption, lemma, theorem, and corollary. They are accepted, without proof, as a basis for proving conjectures in THF, TFF, and FOF problems. In CNF problems the axiom-like formulae are accepted as part of the set whose satisfiability has to be established. There is no guarantee that the axiom-like formulae of a problem are consistent. hypothesis are assumed to be true for a particular problem. definitions are used to define symbols. assumptions must be discharged before a derivation is complete. lemmas and theorems have been proven from the other axiom-like formulae, and are thus redundant wrt those axiom-like formulae. theorem is used also as the role of proven conjectures, in output. A problem containing a lemma or theorem that is not redundant wrt the other axiom-like formulae is ill-formed. theorems are more important than lemmas from the user perspective. corollarys have been proven from the axioms and a theorem, and are thus redundant wrt the other axiom-like and theorem formulae. A problem containing a corollary that is not redundant wrt the other axiom-like formulae and theorem formulae is ill-formed. conjectures occur in only THF, TFF, and FOF problems, and are to all be proven from the axiom(-like) formulae. A problem is solved only when all conjectures are proven."


Treating of any other key values except "include", "content" and "role", is up to the application.

In case "include" occurs inside atoms/terms and not at the top of the formula, then these
files/urls should be included as well. In other words, "include" is a global operator.
The presence of "include" without any "content" key should still trigger import of the
indicated files/urls.

Objects, i.e. metainformation may be nested.

JSON streaming
--------------

The files in the form of several sequential JSON constructions 
(see [JSON streaming](https://en.wikipedia.org/wiki/JSON_streaming "JSON streaming")): 
should be preferrably treated as:

* A conjunction of all the constructions, i.e. each separate JSON construction is
  in the topmost "and" list. 

* In case the stream starts with a JSON object without the "content" key, the
  whole following "and" list of the stream is treated as a value of the "content"
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
     "content":, ["and",

    ["forall",[["X","int"],["Y2","string"]],["p","Y2","X"]],
    {"name":"example formula",
    "confidence": 0.8,
    "context": "examples",
    "include": ["/opt/axioms/a1.ax","http://example.org/ax2"],
    "content": ["p",1,"a"] },

    ["exists",[["X","int"]],["p","X"]]

    ]
    }