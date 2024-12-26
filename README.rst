This is a fork of of the LISA system by David E. Young (from https://lisa.sourceforge.net/).

The LISA Reference Guide
------------------------

This guide describes in detail the programming interface to LISA. It is
*not* a treatise on production-rule technology; readers are assumed to
have a working knowledge of rule-based systems and their development.
This document also avoids any detailed description of LISA's
implementation of the Rete algorithm; perhaps at some future date I'll
make an attempt.

Abbreviated Table of Contents
-----------------------------

| `The Programming Language <#The%20Programming%20Language>`__
| `The Environment <#The%20Environment>`__
| `Contexts <#Contexts>`__
| `Dynamic Rule Definition <#Dynamic%20Rule%20Definition>`__
| `Queries <#Queries>`__
| `Conflict Resolution <#Conflict%20Resolution>`__\ `
  The LISA Debugger
   <#The%20LISA%20Debugger>`__\ `Auto Notification <auto-notify.html>`__
| `Getting Started <#Getting%20Started>`__
| `Things Yet to Do <#Things%20Yet%20to%20Do>`__
| `Supported Platforms <#Supported%20Platforms>`__
|  

I. The Programming Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section describes the publicly-available operators in the LISA
language, separated into various categories:

   `Fact-Related Operators <#Fact-Related%20Operators>`__

      Language elements dealing with facts.

   `Rule-Related Operators <#Rule-Related%20Operators>`__

      Language elements dealing with rules.

   `CLOS-Related Operators <#CLOS-Related%20Operators>`__

      Language elements dealing with CLOS instances.

   `Engine-Related Operators <#Engine-Related%20Operators>`__

      Language elements dealing with operations on an inference engine.

   `Environment-Related Operators <#Environment-Related%20Operators>`__

      Language elements dealing with the LISA environment.

   `Debugging-Related Operators <#Debugging-Related%20Operators>`__

      Language elements useful during system development.

Fact-Related Operators
^^^^^^^^^^^^^^^^^^^^^^

 

======================================= =====
(deftemplate *name* () (*slot-name\**)) Macro
======================================= =====

..

   Creates an internal LISA class identified by *name* that can be
   instantiated as a fact within the knowledge base. The *slot-name*\ s
   are analogous to class slots, but without any of the keyword
   arguments. Templates are a convenient way of specifying concepts that
   don't need the full support of CLOS, but frankly they're really only
   in place to ease the transition from CLIPS and Jess. 

======================== =====
(defimport *class-name*) Macro
======================== =====

..

   A convenience macro that imports the symbols associated with a class
   into whatever LISA-related package the developer wishes. The symbols
   imported reflect the name of the class name and all of its immediate
   slots. If taxonomic reasoning is enabled, then the class's ancestors
   are imported as well.

===================================================== =====
(deffacts *deffact-name* (*key*\ \*) *fact-list*\ \*) Macro
===================================================== =====

..

   Registers a list of facts that will be automatically inserted into
   the knowledge base upon each RESET. The *deffact-name* is the
   symbolic name that will be attached to this group of facts;
   *fact-list* is a list of fact specifiers. The format of each fact
   specifier is identical to that found in an ASSERT form, minus the
   *assert* keyword. There are currently no supported keywords for this
   macro.

=========================== =====
(assert (*fact-specifier*)) Macro
=========================== =====

..

   Inserts a fact identified by *fact-specifier* into the knowledge
   base. There are two forms of ASSERT; the first operates on
   template-based facts, the other on CLOS instances. For templates,
   ASSERT takes a symbol representing the name of the template, followed
   by a list of (*slot-name value*) pairs:

   (assert (frodo (name frodo) (age 100))

   If the template associated with a fact has not been declared prior to
   its assertion, LISA will signal a continuable error.

   For instances of user-defined classes, ASSERT takes a form that must
   evaluate to a CLOS instance:

   (assert ((make-instance 'frodo :name 'frodo :age 100)))

   or:

   (let ((?instance (make-instance 'frodo :name 'frodo)))

       (assert (?instance)))

   or:

   | (defun add-my-instance (frodo-object)
   |   (assert (#?frodo-object)))

   This last example makes use of the #? reader macro, which LISA offers
   as a user-customisable feature. It's simply a short-hand notation for
   *(identity frodo-object)*.

============================ ========
(retract *fact-or-instance*) Function
============================ ========

..

   Removes a fact or instance from the knowledge base. In the case of a
   template-based fact, *fact-or-instance* may be either a symbol
   representing the name of the fact, or an integer mapping to the fact
   identifier; for CLOS objects *fact-or-instance* must be an instance
   of STANDARD-OBJECT.

============================ ========
(assert-instance *instance*) Function
============================ ========

..

   Inserts a CLOS instance into the knowledge base.

============================= ========
(retract-instance *instance*) Function
============================= ========

..

   Removes a CLOS instance from the knowledge base.

===================================== =====
(modify *fact* (*slot-name value*)\*) Macro
===================================== =====

..

   Makes changes to the fact instance identified by *fact*. Affected
   slots and their new values are specified by (*slot-name value*). Note
   that *value* can be an arbitrary Lisp expression that will be
   evaluated at execution time.

Rule-Related Operators
^^^^^^^^^^^^^^^^^^^^^^

 

================================================ =====
(defrule *name (key\*) pattern\** => *action\**) Macro
================================================ =====

..

   Creates a rule identified by *name* and compiles it into the Rete
   network. *Name* is any Lisp form that evaluates to a symbol. The
   keyword arguments modify the rule as follows:

   ::

      :salience integer

   ..

      Assigns a priority to the rule that will affect the firing order.
      The salience value is a small integer in the range (-250, 250). By
      default, all rules have salience 0.

   ::

      :context name

   ..

      Binds the rule to a context identified by the symbol *name*. The
      context must have been previously defined, or LISA will signal an
      error.

   ::

      :auto-focus t

   ..

      Identifies the rule as requiring "auto focus" behavior. This means
      that whenever the rule is activated, its context will be made the
      active context after the rule firing completes.

   If the rule identified by *name* already exists in the Rete network
   it is replaced by the new definition.

   .. rubric:: Patterns
      :name: patterns

   Each rule consists of zero or more *pattern*\ s, or Conditional
   Elements (CEs). Collectively these patterns are known as the rule's
   Left Hand Side (LHS), and are the entities that participate in the
   pattern-matching process. LISA currently defines three pattern types:

   ::

      generic pattern

   ..

      This pattern type matches against facts in the knowledge base. The
      head of the pattern matches equivalently-named facts; the pattern
      body is optionally composed of slot-names, values, variables and
      predicates. The best way to understand these things is to look at
      some examples:

      ::

         (simple-pattern)

      ..

         The simplest type of pattern. This example will match any fact
         of class *simple-pattern*.

      ::

         (goal-is-to (action unlock))

      ..

         This pattern matches facts of class *goal-is-to*. In addition,
         it specifies that the slot named *action* must have as its
         value the symbol *unlock*.

      ::

         (thing (name ?chest) (on-top-of (not floor)))

      ..

         A bit more interesting. Matches facts of class *thing*;
         assuming this is the first appearance of the variable *?chest*,
         binds it to the value of the slot *name*; specifies that the
         slot *on-top-of* should not have as its value the symbol
         *floor*.

      ::

         (?monkey (monkey (holding ?chest)))

      ..

         Assuming the variable *?chest* was bound in a previous pattern,
         matches facts of class *monkey* whose slot *holding* has the
         same value as *?chest*. Additionally, if the pattern is
         successfully matched, binds the fact object to the variable
         *?monkey*. The variable *?monkey* is called a *pattern
         binding*.

      ::

         (pump (flow-rate ?flow-rate (< ?flow-rate 25)))

      ..

         More interesting still. This pattern matches facts of class
         *pump*, and binds the value of the slot *flow-rate* to the
         variable *?flow-rate*. In addition, there is a constraint on
         this slot declaring that the value of *?flow-rate* must be less
         than 25. In general, constraints can be arbitrary Lisp
         expressions that serve as predicates.

      ::

         (fact-with-list (list '(1 2 three)))

      ..

         Patterns can perform matching on lists as well as simpler data
         types. Here, the slot *list* must have the value *'(1 2
         three)*. More complicated list analysis can be done using
         user-defined predicates.

   ::

      negated pattern

   ..

      This pattern type is the complement of most variations of the
      generic pattern. Negated patterns have the symbol *not* as their
      head, and match if a fact satisfying the pattern is *not* found.
      For example:

      ::

         (not (tank-level-warning (tank ?tank) (type low)))

      Note that negated patterns are not allowed to have pattern
      bindings.

   ::

      test pattern

   ..

      The *test* conditional element allows one to evaluate arbitrary
      Lisp code on the rule LHS; these Lisp forms serve as a predicate
      that determines whether or not the pattern will match. For
      example, the pattern

      ::

         (test (and (high-p ?tank) (intact-p ?tank)))

      will succeed if the AND form returns non-*nil*; i.e. the functions
      HIGH-P and INTACT-P both return non-*nil* values.

   ::

      or pattern

   ..

      The *or* conditional element collects any number of patterns into
      a logical group, and matches if any of the patterns inside the
      *or* match. If more than one of the sub-patterns matches, the *or*
      group matches more than once. LISA implements a rule containing an
      *or* CE as a collection of related rules, with each rule
      representing exactly one branch. For example, given the following
      DEFRULE form:

      (defrule frodo () 

          (frodo) 

          (or (bilbo) 

               (gandalf)) 

          (samwise)

      =>)

      LISA will generate two rules into the rete network, a primary rule
      and a single sub-rule:

         ::

            frodo: (frodo), (bilbo), (samwise)

         ::

            frodo~1: (frodo), (gandalf), (samwise)

      Notice that LISA separates the example DEFRULE into the primary
      rule *frodo*, and a single sub-rule, *frodo~1*. LISA maintains the
      relationship between a primary rule and its sub-rules; if a
      primary rule is removed, every related sub-rule is also
      eliminated.

   ::

      logical pattern

   ..

      The *logical* conditional element implements LISA's notion of
      truth maintenance. Patterns appearing within a LOGICAL form in a
      rule are conditionally bound to facts asserted from that rule's
      RHS. If during inferencing one or more logical facts are retracted
      (or asserted in the case of negated patterns), all facts bound to
      those logical facts are retracted. Here's an example:

      (defrule frodo ()

          (logical 

            (bilbo) 

            (not (gandalf)))

                    (frodo)

      =>

      (assert (pippin)))

      When rule FRODO fires, it asserts a PIPPIN fact that is dependent
      on the existence of BILBO and the absence of GANDALF. If either
      BILBO is retracted or GANDALF asserted, PIPPIN will be removed as
      a consequence.

      A LOGICAL conditional element must be the first pattern in a rule.
      Multiple LOGICAL forms within the same rule are allowed, but they
      must be contiguous.

      **NB**: A rule beginning with the LOGICAL conditional element
      implicitly matches the INITIAL-FACT; thus, in order for rules
      employing truth maintenance to function correctly, a RESET must be
      always be performed prior to any operation affecting working
      memory. LISA's behavior is undefined otherwise.

   ::

      exists pattern

   ..

      The EXISTS conditional element performs an existential test on a
      pattern. The pattern will match exactly once, even if there are
      many facts that might satisfy it. For example, this rule:

      | (defrule frodo ()
      |     (exists (frodo (has-ring t)))
      |     =>)

      will activate just once if there is at least one FRODO fact whose
      HAS-RING slot has the value T.

   ::

      The initial fact

   If a rule provides no conditional elements, then it is said to match
   the *initial-fact*, which is asserted as the result of a call to
   *reset*. Thus, the following rule will always activate after each
   reset:

   (defrule always-fires ()

      =>

      (format t "always-fires fired!~%"))

   ::

      CLOS instances

   Every fact asserted into working memory is backed by a corresponding
   CLOS instance. In the case of DEFTEMPLATEs, LISA creates an internal
   class mirroring the template; user-defined class instances are simply
   bound to a fact during assertions. Instances associated with facts
   are accessible on rule LHSs via the :OBJECT special slot:

   (tank (name ?name) (:object  ?tank-object))

   Once bound, method and function calls can be made on this object from
   the rule's LHS and RHS.

   | When reasoning over CLOS objects, LISA is capable of considering an
     instance's object hierarchy during pattern matching. In other
     words, it is possible to write rules that apply to many facts that
     share a common ancestry. The following code fragment provides an
     example:
   |  

   (defclass fundamental () ())

   (defclass rocky (fundamental) ())

   (defclass boris (fundamental) ())

    

    

   (defrule cleanup (:salience -100)

       (?fact (fundamental))

        =>

       (retract ?fact))

    

   The rule *cleanup* will fire for every instance of *rocky* and
   *boris* in the knowledge base, retracting each in turn. Note that
   taxonomic reasoning is disabled by default. To use the feature,
   evaluate (setf (lisa:consider-taxonomy) t).

   .. rubric:: Actions
      :name: actions

   Following any conditional elements are the rule's actions, if any.
   Collectively known as the Right Hand Side (RHS), actions consist of
   arbitrary Lisp forms. All variables declared on the LHS are
   available, along with the special operator *engine*, which evaluates
   to the rule's inference engine object. Currently, each rule's RHS is
   given to the Lisp compiler during rule compilation, and executes
   within a special lexical environment established by LISA for each
   firing.

======================= =====
(undefrule *rule-name*) Macro
======================= =====

..

   Undefines, or removes, a rule from the Rete network. *Rule-name* is a
   symbol representing the name of the rule. If *rule-name* is not
   qualified with a context name (e.g. *context.rulename*), then the
   Initial Context is assumed.

CLOS-Related Operators
^^^^^^^^^^^^^^^^^^^^^^

| There are a few special features that LISA provides for keeping CLOS
  instances synchronised with their corresponding facts in working
  memory. If an instance is altered outside of LISA's control, then LISA
  must somehow be informed of the change to maintain working memory
  consistency. The basic mechanism is manual notification, in which an
  application explicitly invokes a special function to initiate
  synchronisation. Users of the two commercial Lisps supported by LISA
  also have the option of employing `Auto
  Notification <auto-notify.html>`__, an experimental feature that
  removes the burden of synchronisation from the application.
|  

====================================================== ========
(mark-instance-as-changed *instance* &key *slot-name*) Function
====================================================== ========

..

   Notifies LISA that a change has been made to *instance* outside of
   the knowledge-base (i.e. not via the *modify* operator), and
   synchronizes the instance with its associated fact. *Slot-name* is
   either the symbolic name of a slot belonging to *instance* that has
   changed value, or NIL (the default), in which case all slots are
   synchronized. An application *must* call this method whenever a slot
   change occurs outside of LISA's control.

Engine-Related Operators
^^^^^^^^^^^^^^^^^^^^^^^^

| These operators provide an interface to instances of the inference
  engine itself.

================== ========
(inference-engine) Function
================== ========

..

   Evaluates to the currently active instance of the inference engine.

======= ========
(reset) Function
======= ========

..

   Re-initializes the knowledge base, removing facts, clearing all
   context agendas, and asserting the *initial-fact*.

======= ========
(clear) Function
======= ========

..

   Re-initializes the LISA environment, mostly by creating a new
   instance of the default inference engine.

============================ ========
(run &optional *focus-list)* Function
============================ ========

..

   Runs the inference engine, optionally pushing the context names on
   *focus-list* onto the focus stack before doing so. Execution will
   continue until either all agendas are exhausted or a rule calls
   (halt).

======================= ========
(walk &optional *step*) Function
======================= ========

..

   Runs the engine in *step* increments, single-stepping by default.
   Here, "single-stepping" means "one rule at a time".

====== ========
(halt) Function
====== ========

..

   Halts the inference engine, even if the agendas still have
   activations. Typically used only on rule RHSs.

Environment-Related Operators
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

   These operators are used to manipulate and inspect the LISA
   environment.

============================================== =====
(with-inference-engine (*engine*) *forms*\ \*) Macro
============================================== =====

..

   Evaluates *forms* within the context of the inference engine
   *engine*. Under most circumstances, use this macro in a
   multi-processing environment to safely load a knowledge base into
   *engine*.

======================= ========
(make-inference-engine) Function
======================= ========

..

   Creates an instance of LISA's default inference engine.

====== ========
(rule) Function
====== ========

..

   Within the context of an executing rule, returns the CLOS object
   representing that rule.

=================== ========
(consider-taxonomy) Function
=================== ========

..

   Returns the current setting for taxonomic reasoning. Use (setf
   (consider-taxonomy) *value*) to change the setting. The default (NIL)
   means LISA ignores class taxonomy during pattern matching.

======================= ========
(allow-duplicate-facts) Function
======================= ========

..

   Returns the current setting for duplicate fact checking. Use (setf
   (allow-duplicate-facts) *value*) to change the setting. By default,
   LISA allows duplicate facts to be asserted. If checking is enabled
   and an application attempts to assert a duplicate fact, LISA signals
   a DUPLICATE-FACT error.

================== ========
(use-fancy-assert) Function
================== ========

..

   Returns the current setting for fancy assertions. If enabled (the
   default), the #? reader macro is installed in the global readtable.

Debugging-Related Operators
^^^^^^^^^^^^^^^^^^^^^^^^^^^

| These operators are typically used interactively to inspect the state
  of an inference engine. Some of these operators are only loosly
  defined and need further work.
|  

======= ========
(facts) Function
======= ========

..

   Prints on *trace output* the contents of the active inference
   engine's fact base.

================================ ========
(rules &optional *context-name*) Function
================================ ========

..

   Prints on *trace output* the contents of the active inference
   engine's rule base. By default, all rules and all contexts will be
   printed. If *context-name* is provided, then only those rules in that
   context are printed.

================================= ========
(agenda &optional *context-name*) Function
================================= ========

..

   Prints on *trace output* the contents of the active inference
   engine's agenda. By default, the agendas for all contexts will be
   printed, unless *context-name* is supplied.

=============== ========
(watch *event*) Function
=============== ========

..

   Asks LISA to report the occurrence of *event* to *trace output*.

   Currently, LISA allows monitoring of these events:

   ::

      :facts

   ..

      Triggers an event each time a fact is asserted or retracted.

   ::

      :activations

   ..

      Triggers an event each time a rule is added to or removed from the
      agenda.

   ::

      :rules

   ..

      Triggers an event each time a rule is fired.

   ::

      :all

   ..

      Watch all allowable events.

================= ========
(unwatch *event*) Function
================= ========

..

   Disables the monitoring of *event*. See the documentation for *watch*
   to see the allowable event types.

========== ========
(watching) Function
========== ========

..

   Displays the list of events currently being monitored.

II. The Environment
~~~~~~~~~~~~~~~~~~~

For application developers, LISA makes available two different types of
environments. Package LISA-USER contains all of LISA's exported symbols,
plus those of COMMON-LISP. User-level work can safely be done in this
package. Alternatively, package LISA-LISP can be used with DEFPACKAGE
forms to import a LISA environment into user-defined packages:

   | (defpackage "FRODO"
   |   (:use "LISA-LISP"))

As with LISA-USER, LISA-LISP exports all external symbols in the LISA
and COMMON-LISP packages. See the various examples provided in
"lisa:misc;".

There are a few aspects of LISA that may be customised prior to
building. The file "lisa:src;config;config.lisp" contains a set of
default behaviors; feel free to edit this file to your liking.

III. Contexts
~~~~~~~~~~~~~

LISA contexts are a way of partitioning a knowledge base into distinct
groups of rules. The mechanism is similar to modules in CLIPS and recent
versions of Jess, in that individual rule groups can be invoked
"procedurally" without resorting to the use of control facts to
manipulate firing order. Contexts can also serve as an organizational
construct when working with larger knowledge bases. Each context has its
own agenda and conflict resolution strategy.

Each inference engine instance created in LISA contains a default
context, named "The Initial Context" (or INITIAL-CONTEXT). Unless
arrangements are made otherwise, all rules will reside in this context. 
The DEFCONTEXT macro creates a new context; rules may then be loaded
into the new context by supplying the :CONTEXT keyword to DEFRULE.
Contexts serve as a form of namespace for rules; thus, it is legal for
identically named rules to reside in different contexts. Rules are
distinctly identified by qualifying the rule name with the context name;
for example, rule *wizards.gandalf* is a rule named *gandalf* that
resides within the *wizards* context.

Activations in the Initial Context are always available for firing.
Otherwise, activations in other contexts will only fire if those
contexts are explicitly given control, via the FOCUS operator. Each
inference engine maintains its own focus stack; before a new context is
given control, the active context is pushed onto the focus stack. The
REFOCUS operator may be used on a rule's RHS (or, perhaps,
interactively) to leave the active context and return control to the
previous context on the stack. Control automatically returns to the
previous context if the active context runs out of activations. When all
contexts have exhausted their activations, the inference engine halts.

A rule can be tagged with the *auto-focus* attribute by supplying the
AUTO-FOCUS keyword to DEFRULE. If an auto-focus rule activates, that
rule's context is automatically pushed onto the focus stack and given
control when the rule completes its firing.

Note carefully that while the *rules* in a knowledge base may be
partitioned, there remains a *single working memory* per inference
engine. At any given time, all facts in a knowledge base are visible to
all rules in that knowledge base, regardless of context.

=========================================== =====
(defcontext *context-name* &key *strategy*) Macro
=========================================== =====

..

   Creates a new context identified by *context-name*, which must be a
   string designator. If *strategy* is non-NIL then it must be an object
   implementing a suitable conflict resolution strategy.

============================= =====
(undefcontext *context-name*) Macro
============================= =====

..

   Destroys the context identified by *context-name*, which must be a
   string designator. All rules bound to the context are removed from
   the Rete network, along with their activations, if any.

============================= =====
(focus &rest *context-names*) Macro
============================= =====

..

   If *context-names* is non-NIL, it should be a collection of context
   names that will be added to the focus stack. Contexts are pushed onto
   the focus stack in right-to-left order. If no names are specified,
   then FOCUS returns the active context object.

========= =====
(refocus) Macro
========= =====

..

   Activates the next available context.

============= =====
(focus-stack) Macro
============= =====

..

   Returns the inference engine's focus stack.

========== =====
(contexts) Macro
========== =====

..

   Returns a list of all the inference engine's defined contexts.

====================================== =====
(with-context (*context* &body *body*) Macro
====================================== =====

..

   Evaluates the forms contained in *body* within the context *context*.

IV. Dynamic Rule Definition
~~~~~~~~~~~~~~~~~~~~~~~~~~~

| In addition to statically declared rules, LISA supports the definition
  of rules at runtime. That is, it is possible to create new rules from
  the RHSs of existing rules as they fire. These *dynamically defined*
  rules become immediately available to the inference engine for
  potential activation. As a simple example, consider the following
  rule:

   (defrule rocky ()
     (rocky (name ?name))
     =>
     (defrule boris ()
       (boris (name ?name))
       =>
       (format t "Dynamic rule BORIS fired; NAME is ~S~%" ?name)))

When rule ROCKY fires, its RHS creates a dynamically defined rule named
BORIS. This new rule is inserted into the inference engine and
immediately becomes part of the Rete network. Variables bound on the LHS
of ROCKY behave as expected within the context of BORIS; this means that
?NAME in BORIS is bound to the same object as that found in
ROCKY\ :sup:`\*`.

Here's a more complicated example:

   (defrule rocky ()
     (rocky (name ?name))
     =>
     (let ((dynamic-rule
                (defrule (gensym) ()
                  (boris (name ?name))
                  =>
                  (format t "Dynamic rule ~A fired; name is ~S~%"
   (get-name (rule)) ?name))))
       (format t "Rule ROCKY added dynamic rule ~A~%" (get-name
   dynamic-rule))))

As before, rule ROCKY creates a dynamic rule when fired. However, this
time the new rule is given a unique name by evaluating the form
(GENSYM); either the name or the instance can then be remembered for use
later. These two rules also introduce functions in the LISA API for
retrieving both the currently executing rule and its name (RULE and
GET-NAME, respectively).

:sup:`\*` Actually, this isn't precisely true. Whenever LISA encounters
a dynamic rule during parsing it looks at all the rule's variables and
substitutes any bound values. Thus, in rule BORIS the variable ?NAME
would be replaced by the value of ?NAME as bound in rule ROCKY.

V. Queries
~~~~~~~~~~

As of LISA version 1.2, a simple query language is supported that allows
retrieval of facts from a knowledge base, either interactively via a
Lisp listener or programmatically. The query engine leverages the
inferencing component by transforming query expressions into rules and
inserting them, at runtime, into the Rete network. Each query is
assigned a unique identifier and cached upon first appearance;
subsequent queries with semantically equivalent bodies will find the
cached instance and execute with substantially improved performance,
especially for larger knowledge bases. As an example, consider the
following two query forms:

|   (retrieve (?x ?y)
|     (?x (hobbit (name ?name)))
|     (?y (ring-bearer (name ?name))))

|   (retrieve (?h1 ?h2)
|     (?h1 (hobbit (name ?hobbit-name)))
|     (?h2 (ring-bearer (name ?hobbit-name))))

The variables appearing in the first argument to RETRIEVE (e.g. '(?x
?y)) are used to establish bindings for each firing of the query; each
variable must also appear in the query body as an appropriate pattern
binding. Other than this requirement, query bodies are structurally
identical to rule bodies; anything that is legal in a rule LHS is legal
in a query body. Note that these two examples are semantically
equivalent; although the variable names are different, the patterns
appear in the same order and the relationships among variables are
identical. Thus, firing either query will yield the same set of fact
bindings. LISA is able to recognize such similarities in queries and
implements a caching scheme that minimizes unnecessary dynamic rule
creation.

RETRIEVE returns two values; a list of bindings for each query firing,
and the symbolic name assigned by LISA to the query instance. The second
value is probably only useful while developing/testing queries; using
this symbol one can ask LISA to forget about a query by removing it from
the cache and the Rete network. The binding list is the principal value
of interest. Since a query is really a rule, it can fire an arbitrary
number of times for each invocation. Each firing is represented as a
list of CONS cells. The CAR of each cell is one of the variables
specified in the query's binding list; its CDR is a fact bound to that
variable that satisfies the variable's associated pattern. For example,
assuming that the first of the above query examples fires twice during a
certain invocation, RETRIEVE would return something like:

|   (((?X . <HOBBIT INSTANCE 1>) (?Y . <RING-BEARER INSTANCE 1>))
|     ((?X . <HOBBIT INSTANCE 2>) (?Y . <RING-BEARER INSTANCE 2)))
|   #:G7777

As explained previously, the second value is the symbolic name LISA
assigned to the query when its rule instance was initially created. Most
of the time it will be ignored.

As of release 1.3, LISA incorporates a unified view of template- and
CLOS-based facts; as a result queries now function for both types of
facts. In the former case, LISA creates a class modeled around the
template. Class and slot names are taken directly from the DEFTEMPLATE
form; each slot is given a reader method named according to DEFSTRUCT
conventions (i.e. *class name-slot name*). For example, 

   | (deftemplate frodo ()
   |   (slot companion (default merry)))

Will yield the following class specification:

   | (defclass frodo (inference-engine-object)
   |   ((companion :initform 'merry :initarg :companion :reader
     frodo-companion)))

| These functions and macros comprise the current interface to the query
  engine:
|  

=========================================== =====
(retrieve (*variables*\ \*) *patterns*\ \*) Macro
=========================================== =====

..

   Initiates a query against the knowledge base. Variable bindings for
   the query are found in *variables; patterns* consists of matching
   forms that comprise the body of the query rule. RETRIEVE returns two
   values: a list of CONS cells for each firing and the symbolic name
   LISA assigned to the query when it was initially constructed. The CAR
   of each CONS cell is one of the binding variables; the CDR is the
   CLOS instance bound to that variable.

===================== ========
(forget-query *name*) Function
===================== ========

..

   Instructs LISA to forget about the query identified by the symbol
   *name*. Doing so removes the query's rule instance from the Rete
   network and the query itself from the cache. Useful only during query
   development, probably.

============================================================= =====
(with-simple-query ((*var value*) *query-form* &body *body*)) Macro
============================================================= =====

..

   Evaluates *query-form*. Then, iterates over the resulting list
   structure, binding each variable and fact to *var* and *value*,
   respectively, and evaluating *body*. This macro is useful if one is
   interested in just the individual variable/fact pairs and doesn't
   care much about the binding context that occurred during query
   firing.

VI. Conflict Resolution
~~~~~~~~~~~~~~~~~~~~~~~

| Conflict Resolution (CR) is the mechanism LISA employs to determine
  the order in which multiple activations will fire. Currently, LISA
  offers two "built-in" strategies; *breadth-first* and *depth-first*.
  It is possible to implement new CR algorithms by creating a class
  derived from *lisa:strategy* and implementing a few generic functions;
  instances of this new strategy can then be given to
  *make-inference-engine*.
|  

======================================== ================
(add-activation *strategy* *activation*) Generic Function
======================================== ================

..

   Makes a new *activation* eligible for firing.

=========================================== ================
(find-activation *strategy* *rule* *token*) Generic Function
=========================================== ================

..

   Locates an activation associated with *rule* and *token*.

============================ ================
(next-activation *strategy*) Generic Function
============================ ================

..

   Returns the next eligible activation.

============================= ================
(list-activations *strategy*) Generic Function
============================= ================

..

   Returns a list of eligible activations.

Documentation for the CR interface is still fairly light. Look for
improvements in upcoming releases.

VII. The LISA Debugger
~~~~~~~~~~~~~~~~~~~~~~

New as of 2.0 alpha 4, the LISA debugger is a simple monitoring and
inspection utility that may be used to "debug" production rules.
Although one cannot step through a rule pattern by pattern, breakpoints
may be set to trigger just before a rule fires. When a breakpoint is
reached, one can then interactively examine the token stack, display all
pattern bindings and their values, single-step into the next activation,
etc.

By default, LISA builds without the debugger loaded to avoid a slight
performance drag on rule firings. To use the debugger, in the CL-USER
package evaluate the form (require 'lisa-debugger (lisa-debugger)). If
you're running Allegro Common Lisp, LISA understands how to hook into
the module search list; thus you may instead evaluate (require
'lisa-debugger).

The functionality available via the LISA debugger may increase as user
needs dictate; here is the command set as of this writing:

======================= ========
(set-break *rule-name*) Function
======================= ========

..

   Sets a breakpoint in the rule identified by the symbol *rule-name.*

========================= ========
(clear-break *rule-name*) Function
========================= ========

..

   Clears the breakpoint previously set on the rule identified by the
   symbol *rule-name*.

============== ========
(clear-breaks) Function
============== ========

..

   Removes all breakpoints.

===================== ================
\*break-on-subrules\* Special variable
===================== ================

..

   Setting this variable to a non-NIL value will cause the debugger to
   manage breakpoints for a primary rule and all of its subrules (see
   the section on the *or* conditional element for an explanation of
   primary rules).

====== ========
(next) Function
====== ========

..

   Fires the currently suspended rule, then single-steps into the next
   activation, if there is one. If there isn't one, the debugger exits.

======== ========
(resume) Function
======== ========

..

   Resumes normal execution, until the next breakpoint is reached.

============================= ========
(tokens &key (*verbose nil*)) Function
============================= ========

..

    Displays the token stack, which contains the facts that activated
   this particular rule. If *verbose* is non-nil, then the fact
   instances themselves are printed; otherwise, a shorthand notation is
   used.

========== ========
(bindings) Function
========== ========

..

   Displays the bindings (pattern variables) found on the rule's LHS,
   along with their values.

================ ========
(fact *fact-id*) Function
================ ========

..

   Returns the fact instance associated with *fact-id*, a small integer
   assigned to each fact by the inference engine.

============= ========
(breakpoints) Function
============= ========

..

   Displays all breakpoints.

====== ========
(rule) Function
====== ========

..

   Returns the rule instance representing the suspended activation.

VIII. Getting Started
~~~~~~~~~~~~~~~~~~~~~

LISA requires the Portable Defsystem as maintained by the CLOCC project;
for your convenience, a copy is included in the distribution. Building
LISA should be straight-forward. First, either load "lisa:lisa.system"
or change your working directory to the LISA root directory; then,
evaluate (mk:compile-system :lisa). Note that LISA uses logical
pathnames in its defsystem, and translations that are suitable for a
Linux (or Cygwin/Windows) environment are established there. They might
work for you; perhaps not. Until I figure out how to correctly place
default translations you might have to do some hand editing. Sorry.

To build a knowledge base, write your production rules using the various
source examples (and this document) as your guide and load the file(s)
into Lisp. You can then change to the LISA-USER package and experiment.
Look for the examples in "lisa:misc;".

A note to CLISP users. LISA requires that CLISP be run with full ANSI
support enabled. Also, the baseline version with which LISA has been
tested is 2.25.1. Earlier releases might work as well, but no
guarantees.

IX. Things Yet to Do.
~~~~~~~~~~~~~~~~~~~~~

This section is a list (albeit incomplete) of features that would
improve LISA significantly.

#. *Backward chaining*: Perhaps an implementation of Prolog's
   backchaining algorithm that has concurrent access to working memory
   (i.e. along with Rete).

X. Supported Platforms.
~~~~~~~~~~~~~~~~~~~~~~~

LISA has been tested, and is known to run, on the following Common Lisp
implementations:

-  Allegro Common Lisp, versions 5.0.1 and 6.x, Linux and Windows 2000.
-  Xanalys LispWorks, versions 4.1.20 and 4.2, Linux and Windows 2000.
-  CLISP, version 2.27 and newer, Linux and Windows 2000.
-  CMUCL, version 18c, Linux.
