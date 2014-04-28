---
layout: post
title:  "Hacking (Technical) Writing - Mechanics"
tags: ["technical", "writing"]
---

## Split long sentences

The following sentence was very long. Its first half described valid date representations, while its second half described valid time representations.

> _Before_: It permits removing the least significant values, so that `YYYY-MM` can represent a month in a year, and `YYYY` can represent just a year, while `hh:mm` can represent a minute in an hour, and `hh` can represent just an hour.
>
> _After_: It permits removing the least significant values, so that `YYYY-MM` represents a month in a year, and `YYYY` represents just a year. Similarly, `hh:mm` represents a minute in an hour, and `hh` represents just an hour.

The following sentence has two commas by the time the following sentence describes the conditions under which the algorithm terminates. We therefore split it:

> _Before_: The process repeats with the child node, and then with its child node, and so on, continuing until the value is found, or we cannot continue because the child pointer to follow is null.
>
> _After_: The process repeats with the child node, and then with its child node, and so on. It terminates when either it finds the value, or the child pointer to follow is null.

Finally, TODO

> _Before_: The following implementation of the `Player` class hides the `moneyRemaining` data member, forcing a client to read its value through method `getMoneyRemaining` and write its value through method `setMoneyRemaining`.
>
> _After_: The following implementation of the `Player` class hides the `moneyRemaining` data member. This forces a client to read its value through method `getMoneyRemaining`, and to write its value through method `setMoneyRemaining`.

## Remove _by_ and _through_ constructions

As you may have learned in school, a _by_ construction is the canonical definition of passive voice.

> _Before_: The running time of method `average` *can be defined by* the function f(n) = t<sub>1</sub> + (t<sub>2</sub> &times; n) + t<sub>3</sub>.
>
> _After_: The function f(n) = t<sub>1</sub> + (t<sub>2</sub> &times; n) + t<sub>3</sub> *defines* the running time of method `average`.

> _Before_: This *is implemented by* the following Python program.
>
> _After_: The following Python program *implements* this.

> _Before_: A collection of problems and their subproblems *can be grouped by* a package or module.
>
> _After_: A package or module *can group* a collection of problems and their subproblems.

> _Before_: How these seemingly arbitrary arbitrary size thresholds *were undoubtedly chosen through* tireless profiling.
>
> _After_: Tireless profiling *undoubtedly chose* these seemingly arbitrary size thresholds.

## Remove _to be_ verbs

> _Before_: A compiled form of the tz database *is usually found in* the directory `/usr/share/zoneinfo`.
>
> _After_: The directory `/usr/share/zoneinfo` *usually contains* a compiled form of the tz database.

TODO: Make this noun to verb?

> `Filter` *is an enumeration over* all possible image filters.
>
> `Filter` *enumerates* all possible image filters.

## Change the subject

> _Before_: Through the Facebook Graph API, *my profile is represented by* the following collection of key-pairs.
>
> _After_: The Facebook Graph API *represents my profile* using the following key-value pairs.

> _Before_: With a doubly-linked list, *we can efficiently* implement a queue.
>
> _After_: A *doubly-linked list can efficiently* implement a queue.

> _Before_: By using an in-order traversal on the backing tree, *it is efficient to find* the *k* mappings with the smallest or largest keys.
>
> _After_: An in-order traversal of the backing tree *efficiently finds* the *k* mappings with the smallest or largest keys.

## Create a subject

> _Before_: If these instances are all added to the same hash table...
>
> _After_: If we add all these instances to the same hash table...

> _Before_: Each implementation was free to change because...
>
> _After_: The developers could freely change each implementation because...

## Convert nouns to verbs:

When you convert nouns into verbs, your writing is more active. It is more vigorous.

We can replace some expressions of the form `[verb] of [noun]` with a single verb that is derived from the noun.

> _Before_: The list constructor *creates a copy of* the array in O(n) time.
>
> _After_: The list constructor *copies* the array in O(n) time.

Below, the noun *implementation* becomes the verb *implements*:

> _Before_: Class `AbstractCollection` *provides implementations of* trivial methods in the `Collection` interface.
>
> _After_: Class `AbstractCollection` *implements* trivial methods in the `Collection` interface.

Finally, the noun *complexity* becomes the verb *complicates*:

> _Before_: With cavalier use, inheritance *increases the complexity of* our code.
>
> _After_: With cavalier use, inheritance *complicates* our code.

## Use stronger verbs

> _Before_: This behavior *does not comply* with the POSIX standard.
>
> _After_: This behavior *violates* the POSIX standard.

TODO

## Use appropriate jargon

> _Before_: That method *invokes and returns any value from* the corresponding method...
>
> _After_: That method *delegates to* the corresponding method...

TODO

> _Before_: The framework passes the `Request` instance to the *method that processes the request and generates the response* for this URL.
>
> _After_: The framework passes the `Request` instance to the *request handler* for this URL.

TODO

> _Before_: TODO
>
> _After_: The hash table then rehashes its elements.

## Don't bury the subject

The subject of the following sentence is the list of *n* elements. The _before_ construction surrounds it with the descriptions of its first and last elements, while the _after_ construction moves it to the front:

> _Before_: The first element in the list has an index of *0*, and if the list has *n* elements, then the last element in the list has an index of *n-1*.
>
> _After_: In a list with *n* elements, the first element has an index of *0*, and the last element has an index of *n-1*.

TODO

> _Before_: The primary disadvantages is devoting resources to the task queue, along with its maintenance.
>
> _After_: The primary disadvantages of a task queue are its provisioning and maintenance.

TODO

> _Before_: There is exactly one node that is not the child of any other node, which is called the root node.
>
> _After_: The root node is the only node that is not the child of another node.

## Use parallel constructions

Consider the following list:

> * Strive for loosely-coupled subsystems...
> * Refer to suitable supertypes and interfaces...
> * Breaking a large program into smaller programs can simplify the design.

The structure of the last list item differs from that of the two preceding items. We can rewrite the last list item as:

> * Strive for loosely-coupled subsystems...
> * Refer to suitable supertypes and interfaces...
> * Break a large program into smaller programs if it simplifies the design.

TODO

> * In JavaScript, the expression `[] + []` evaluates as the empty string.
> * In Ruby, the empty string is evaluated as `true`.

TODO

> * In JavaScript, the expression `[] + []` evaluates as the empty string.
> * In Ruby, the empty string evaluates as `true`.

TODO

> _Before_: We increase the value of `num_bytes_buffered` every time the networking layer appends data to the buffer. When the client calls method `read_more()`, we must decrease the value of `num_bytes_buffered` accordingly.
>
> _After_: When the networking layer calls method `append`, we increase the value of `num_bytes_buffered` accordingly. When the client calls method `read_more()`, we decrease the value of `num_bytes_buffered` accordingly.

## Lead in and lead out

The code that demonstrated information hiding was suspended in a vacuum.

> The following implementation of the `Table` class hides the `players` data member. This forces a client to add players through method `addPlayer`:

Similarly, I had three examples of how the C++ keyword `const` can enforce data integrity. Instead of concluding the section with the last example, I followed the last example with the following sentence:

> Collectively, these uses of `const` ensure the data integrity of the `PanoramaBuilder` class.

## Remove wordiness

> _Before_: This graph *gives us a decent perspective of the values of*...
>
> _After_: This graph *illustrates*...

> _Before_: An infinite number of variations and combinations of these exist.
>
> _After_: We can combine and vary them in endless ways.

> _Before_: Now say while reading more about Python, we discover that it's list implementation...
>
> _After_: Now say we later learn that Python's list implementation...

## Place _only_, _either_, and _just_

TODO

