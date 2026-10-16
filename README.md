<!-- Generated from pep-9999.rst — do not edit by hand. -->

# `__parse__`: constructing instances from strings

| Field          | Value                                            |
|----------------|--------------------------------------------------|
| PEP            | 9999                                             |
| Title          | `__parse__`: constructing instances from strings |
| Author         | Christiane Ruetten \<cr@23bit.net\>              |
| Sponsor        | TBD                                              |
| Discussions-To | Pending                                          |
| Status         | Draft                                            |
| Type           | Standards Track                                  |
| Created        | 16-Oct-2026                                      |
| Python-Version | TBD                                              |
| Post-History   | —                                                |

## Abstract

This PEP specifies an optional, discoverable, class-level special method, `__parse__`, by which a type declares that it attempts to construct its instances from a `str`. It standardizes the interface of the operation that `int`, `float`, `Decimal`, `UUID`, `datetime.fromisoformat()` and third-party types already provide under mutually incompatible conventions. It adds no syntax, alters no constructor, and defines no serialization format.

## Motivation

Python standardizes the map `T` → `str` (`__str__`), but not its partial inverse `str` ⇀ `T`. Given a type `T` and `s: str`, nothing in the language answers:

1.  Does `T` define a parse from `str`?
2.  If so, how is it invoked?

Constructors do not answer them. `T(s)` is construction. Whether it also parses is a property of the type, not of the language:

    int("42")                 # 42
    bool("False")             # True - truth test
    list("[1, 2]")            # ['[', '1', ',', ' ', '2', ']']
    datetime("2026-10-16")    # TypeError - see fromisoformat()

`try: T(s)` succeeding does not establish that `s` was interpreted as a value of `T`. Failing does not establish that `T` cannot parse. The attempt itself may have side effects.

As a consequence, every consumer of runtime type information (configuration, CLI argument parsing, environment variables, ORMs, RPC) must maintain a private registry of `type → parser` and a private convention (`from_string`, `parse`, `from_str`, …) for user-defined types. A type cannot be portable across consumers without per-consumer adaptation.

Precedent: [PEP 519](https://peps.python.org/pep-0519/) (`__fspath__`) standardized an existing operation with several conventions by naming it.

## Specification

A class MAY define:

    @classmethod
    def __parse__(cls, text: str, /) -> Self: ...

1.  **Domain**: `text` MUST be a `str`. For any other argument, `__parse__` MUST raise `TypeError`.
2.  **Codomain**: On success, `__parse__` MUST return an instance of `cls` (subclass instances included).
3.  **Failure**: If `text` is not a representation of a value of `cls`, `__parse__` MUST raise `ValueError` or a subclass thereof.
4.  **Partiality**: `__parse__` is a partial function `str` ⇀ `cls`. No implementation is required to accept every `str`.
5.  **Discovery**: `T` supports the protocol iff `callable(getattr(T, "__parse__", None))`. Discovery MUST NOT invoke `__parse__`. `__parse__ = None` disables the protocol for that class, per the existing `__hash__ = None` convention.
6.  **Non-inference**: A consumer MUST NOT infer support from a constructor accepting `str`, from `__str__`, or from `__repr__`. Absent `__parse__`, a consumer SHOULD raise `TypeError`.
7.  **Inheritance**: `__parse__` is inherited. An inherited implementation MUST construct the class it was invoked on, never a base class. A subclass for which the inherited implementation is invalid SHOULD override it or set it to `None`.
8.  **Syntax**: Accepted representations (whitespace, case, alternative spellings) are defined by each type; the protocol defines none.
9.  **Round-trip**: For a type defining both methods, `type(x).__parse__(str(x)) == x` SHOULD hold whenever `str(x)` determines `x` up to `==`. It is not required. It CAN accept `repr(x)`, but it is not required.
10. **Dispatch**: No syntax or built-in invokes `__parse__` implicitly. `T(text)`, `str()` and annotation evaluation are unchanged.

The protocol is expressible as a `typing.Protocol` ([PEP 544](https://peps.python.org/pep-0544/)). A conforming consumer, not proposed as a built-in:

    def parse_as(cls, text):
        parser = getattr(cls, "__parse__", None)
        if not callable(parser):
            raise TypeError(f"{cls.__qualname__} does not support __parse__")
        return parser(text)

## Standard library

Initial implementations, each delegating to existing parsing code:

| Type | Delegates to |
|----|----|
| `str` | identity |
| `int`, `float`, `complex` | constructor (`int`: base 10 only) |
| `bool` | new: `"True"` → `True`, `"False"` → `False`, else `ValueError` |
| `Decimal`, `Fraction`, `UUID` | constructor |
| `IPv4Address`, `IPv6Address`, `IPv4Network`, `IPv6Network` | constructor |
| `date`, `time`, `datetime` | `fromisoformat()` |

Types without a canonical textual representation (`timedelta`, `list`, `dict`, …) are excluded. The protocol names existing grammars; it does not define new ones.

## Non-goals

- No common serialization format: each type keeps its own grammar.
- No recovery of the target type from the input: the type is a parameter of the operation.
- No containers, object graphs or recursion: `list[T]` is a consumer's grammar applied over `T.__parse__`.
- No automatic conversion driven by annotations.
- No relation to `pickle`, `__reduce__`, or proposed `__json__` / `__deserialize__` methods. Those must solve type recovery and format definition, and have failed on them. This PEP has neither problem.

## Backwards Compatibility

Opt-in. No existing behavior changes. Two items require review before acceptance:

- Existing third-party attributes named `__parse__` with incompatible semantics. Fallback name: `__fromstr__`.
- Subclasses of built-ins with incompatible constructors inherit the built-in implementation (item 7). Built-in implementations MUST construct `cls` or raise; they MUST NOT return the base type.

## Rejected Ideas

| Alternative | Rejected because |
|----|----|
| Constructor as parser | `bool("False") is True`. Accepting `str` ≠ interpreting it; undecidable without a declaration. |
| Ordinary class method `from_string()` | Viable; closest contender. A special-method name places the operation in the data model, cannot collide with ordinary API names, and ends the `parse` / `from_str` / `from_text` proliferation. |
| Built-in `parse(T, text)` | Requires this protocol underneath. May follow as a separate PEP. |
| Per-consumer registries | The status quo that leads to the problem statement. |
| General deserialization protocol | Different problem (see Non-goals). |

## Open Issues

1.  **Name**: Special methods usually back implicit interpreter operations; this one is never invoked implicitly. If a plain class method is preferred, the specification is unchanged under renaming.
2.  **Round-trip strength**: Recommended (item 9) versus required. Requiring it excludes types with lossy `__str__`.
3.  **Initial coverage**: The table above is a floor. `timedelta`, `Path`, `Enum` are candidates once a grammar is agreed.

## How to Teach This

`__str__`: object → text. `__parse__`: text → instance of the type. Both are optional and do not imply each other.

    class Color:
        def __new__(cls, r=0, g=0, b=0, /):
            if isinstance(r, str):            # legacy: Color("#ff8000")
                return cls.__parse__(r)
            self = super().__new__(cls)
            self.r, self.g, self.b = r, g, b
            return self

        def __str__(self):
            return f"#{self.r:02x}{self.g:02x}{self.b:02x}"

        @classmethod
        def __parse__(cls, text, /):
            if len(text) != 7 or text[0] != "#":
                raise ValueError("expected #RRGGBB")
            v = int(text[1:], 16)
            return cls(v >> 16 & 255, v >> 8 & 255, v & 255)

Porting an existing `str`-accepting constructor: move the parsing body into `__parse__` and have the constructor delegate to it as shown above. The public API is unchanged and the capability becomes discoverable. Delegation from `__new__` rather than `__init__` avoids re-running initialization on the returned instance.

## References

- [PEP 519](https://peps.python.org/pep-0519/) – `__fspath__`
- [PEP 544](https://peps.python.org/pep-0544/) – structural protocols
- [Data model](https://docs.python.org/3/reference/datamodel.html)

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
