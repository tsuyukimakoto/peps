PEP: 591
Title: Adding a final qualifier to typing
Author: Michael J. Sullivan <sully@msully.net>, Ivan Levkivskyi <levkivskyi@gmail.com>
BDFL-Delegate: Guido van Rossum <guido@python.org>
Discussions-To: typing-sig@python.org
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 15-Mar-2019
Post-History:


Abstract
========

This PEP proposes a "final" qualifier to be added to the ``typing``
module---in the form of a ``final`` decorator and a ``Final`` type
annotation---to serve three related purposes:

このPEPは3つの関連する目的を果たすために `` final``デコレータと `` Final``型注釈の形で `` typing``モジュールに追加される "final"修飾子を提案します

* Declaring that a method should not be overridden

  メソッドをオーバーライドしないように宣言する

* Declaring that a class should not be subclassed

  クラスをサブクラス化しないように宣言する

* Declaring that a variable or attribute should not be reassigned

  変数または属性を再割り当てしないように宣言する

Motivation
==========

The ``final`` decorator
-----------------------
The current ``typing`` module lacks a way to restrict the use of
inheritance or overriding at a typechecker level. This is a common
feature in other object-oriented languages (such as Java), and is
useful for reducing the potential space of behaviors of a class,
easing reasoning.

現在の ``typing`` モジュールは継承の使用を制限する方法やタイプチェッカーレベルでオーバーライドする方法を欠いています。 これは他のオブジェクト指向言語（Javaなど）に共通の機能であり、クラスの振る舞いの潜在的なスペースを減らし、推論を容易にするのに役立ちます。

Some situations where a final class or method may be useful include

最終的なクラスやメソッドが役に立つかもしれないいくつかの状況は:

* A class wasn’t designed to be subclassed or a method wasn't designed
  to be overridden. Perhaps it would not work as expected, or be
  error-prone.

  クラスがサブクラス化されるように設計されていないか、またはメソッドがオーバーライドされるように設計されていません。 おそらくそれは期待通りには動かないか、エラーを起こしやすいでしょう。

* Subclassing or overriding would make code harder to understand or
  maintain. For example, you may want to prevent unnecessarily tight
  coupling between base classes and subclasses.

  サブクラス化またはオーバーライドすると、コードの理解や維持が難しくなります。 たとえば、基本クラスとサブクラスの間の不必要に密接な結合を防ぐことができます。

* You want to retain the freedom to arbitrarily change the class
  implementation in the future, and these changes might break
  subclasses.

  将来、クラス実装を任意に変更する自由を保持したいのですが、これらの変更はサブクラスを壊すかもしれません。

The ``Final`` annotation
------------------------

The current ``typing`` module lacks a way to indicate that a variable
will not be assigned to. This is a useful feature in several
situations

現在の ``typing`` モジュールは変数が代入されないことを示す方法を欠いています。 これはいくつかの状況で便利な機能です。:

* Preventing unintended modification of module and class level
  constants and documenting them as constants in a checkable way.

  Preventing unintended modification of module and class level  constants and documenting them as constants in a checkable way.

* Creating a read-only attribute that may not be overridden by
  subclasses. (``@property`` can make an attribute read-only but
  does not prevent overriding)

  サブクラスによってオーバーライドされない可能性がある読み取り専用属性を作成します。 （ `` @ property``は属性を読み込み専用にすることができますが、上書きを妨げることはありません）

* Allowing a name to be used in situations where ordinarily a literal
  is expected (for example as a field name for ``NamedTuple``, a tuple
  of types passed to ``isinstance``, or an argument to a function
  with arguments of ``Literal`` type [#PEP-586]_).

  通常はリテラルが必要とされる状況で名前を使用できるようにします（たとえば、NamedTupleのフィールド名、isinstanceに渡された型のタプル、またはリテラル型の引数を持つ関数への引数として）[＃PEP-586]_ ）

Specification
=============

The ``final`` decorator
-----------------------

The ``typing.final`` decorator is used to restrict the use of
inheritance and overriding.

typing.finalデコレータは、継承とオーバーライドの使用を制限するために使用されます。

A type checker should prohibit any class decorated with ``@final``
from being subclassed and any method decorated with ``@final`` from
being overridden in a subclass. The method decorator version may be
used with all of instance methods, class methods, static methods, and properties.

型チェッカーは、@finalで装飾されたクラスがサブクラス化されること、および@finalで装飾されたメソッドがサブクラスでオーバーライドされることを禁止する必要があります。 メソッドデコレータバージョンは、すべてのインスタンスメソッド、クラスメソッド、静的メソッド、およびプロパティで使用できます。

For example::

    from typing import final

    @final
    class Base:
        ...

    class Derived(Base):  # Error: Cannot inherit from final class "Base"
        ...

and::

    from typing import final

    class Base:
        @final
        def foo(self) -> None:
            ...

    class Derived(Base):
        def foo(self) -> None:  # Error: Cannot override final attribute "foo"
                                # (previously declared in base class "Base")
            ...


For overloaded methods, ``@final`` should be placed on the
implementation (or on the first overload, for stubs)

オーバーロードされたメソッドの場合は、@finalを実装に（またはスタブの場合は最初のオーバーロードに）配置する必要があります::

   from typing import Any, overload

   class Base:
       @overload
       def method(self) -> None: ...
       @overload
       def method(self, arg: int) -> int: ...
       @final
       def method(self, x=None):
           ...

It is an error to use ``@final`` on a non-method function.

メソッド以外の関数で@finalを使用するとエラーになります。

The ``Final`` annotation
------------------------

The ``typing.Final`` type qualifier is used to indicate that a
variable or attribute should not be reassigned, redefined, or overridden.

typing.Final型修飾子は、変数または属性を再割り当て、再定義、または上書きしてはいけないことを示すために使用されます。

Syntax
~~~~~~

``Final`` may be used in in one of several forms

ファイナルはいくつかの形式のうちの1つで使用されるかもしれません:

* With an explicit type, using the syntax ``Final[<type>]``. Example

  明示的な型で、構文Final [<type>]を使用する。 例::

    ID: Final[float] = 1

* With no type annotation. Example

  型注釈なし。 例::

    ID: Final = 1

  The typechecker should apply its usual type inference mechanisms to
  determine the type of ``ID`` (here, likely, ``int``). Note that unlike for
  generic classes this is *not* the same as ``Final[Any]``.

  タイプチェッカーは、通常の型推論メカニズムを使ってIDの型（ここではおそらくint）を決定します。 ジェネリッククラスとは異なり、これはFinal [Any]と同じではないことに注意してください。

* In class bodies and stub files you can omit the right hand side and just write
  ``ID: Final[float]``.  If the right hand side is omitted, there must
  be an explicit type argument to ``Final``.

  クラス本体とスタブファイルでは、右側を省略してID：Final [float]と書くだけです。 右側が省略された場合、Finalへの明示的な型引数がなければなりません。

* Finally, as ``self.id: Final = 1`` (also optionally with a type in
  square brackets). This is allowed *only* in ``__init__`` methods, so
  that the final instance attribute is assigned only once when an
  instance is created.

  最後に、self.idとして、Final = 1とします（オプションで、角括弧内の型も指定します）。 これは__init__メソッドでは* only *許可されているため、インスタンスが作成されたときにfinalインスタンス属性は一度だけ割り当てられます。

Semantics and examples
~~~~~~~~~~~~~~~~~~~~~~

The two main rules for defining a final name are

姓を定義するための2つの主な規則は次のとおりです。:

* There can be *at most one* final declaration per module or class for
  a given attribute. There can't be separate class-level and instance-level
  constants with the same name.

  特定の属性について、モジュールまたはクラスごとに*最大1つの*最終宣言があります。 同じ名前のクラスレベル定数とインスタンスレベル定数を別々にすることはできません。

* There must be *exactly one* assignment to a final name.

  最終名への割り当ては*厳密に1つ*でなければなりません。

This means a type checker should prevent further assignments to final
names in type-checked code

これは、型チェッカーが型チェックされたコード内の最終名へのそれ以上の割り当てを防ぐべきであることを意味します::

   from typing import Final

   RATE: Final = 3000

   class Base:
       DEFAULT_ID: Final = 0

   RATE = 300  # Error: can't assign to final attribute
   Base.DEFAULT_ID = 1  # Error: can't override a final attribute

Note that a type checker need not allow ``Final`` declarations inside loops
since the runtime will see multiple assignments to the same variable in
subsequent iterations.

ランタイムは後続の反復で同じ変数への複数の代入を参照するため、型チェッカーはループ内でFinal宣言を許可する必要はありません。

Additionally, a type checker should prevent final attributes from
being overridden in a subclass

さらに、型チェッカーは、最終的な属性がサブクラスでオーバーライドされるのを防ぐべきです。::

   from typing import Final

   class Window:
       BORDER_WIDTH: Final = 2.5
       ...

   class ListView(Window):
       BORDER_WIDTH = 3  # Error: can't override a final attribute

A final attribute declared in a class body without an initializer must
be initialized in the ``__init__`` method (except in stub files)

初期化指定子なしでクラス本体で宣言された最後の属性は、__ init__メソッドで初期化する必要があります（スタブファイルを除く）。::

   class ImmutablePoint:
       x: Final[int]
       y: Final[int]  # Error: final attribute without an initializer

       def __init__(self) -> None:
           self.x = 1  # Good

Type checkers should infer a final attribute that is initialized in
a class body as being a class variable. Variables should not be annotated
with both ``ClassVar`` and ``Final``.

型チェッカーは、クラス本体内でクラス変数として初期化される最終属性を推測する必要があります。 変数にClassVarとFinalの両方のアノテーションを付けないでください。

``Final`` may only be used as the outermost type in assignments or variable
annotations. Using it in any other position is an error. In particular,
``Final`` can't be used in annotations for function arguments

finalは代入または変数注釈の最も外側の型としてのみ使用できます。 他の場所で使用するとエラーになります。 特に、Finalは関数の引数のアノテーションには使えません。::

   x: List[Final[int]] = []  # Error!

   def fun(x: Final[List[int]]) ->  None:  # Error!
       ...

Note that declaring a name as final only guarantees that the name will
not be re-bound to another value, but does not make the value
immutable. Immutable ABCs and containers may be used in combination
with ``Final`` to prevent mutating such values

名前をfinalとして宣言しても、その名前が別の値に再バインドされないことが保証されるだけで、その値が不変になるわけではありません。 不変のABCとコンテナは、そのような値の変異を防ぐために、Finalと組み合わせて使用することができます。::

   x: Final = ['a', 'b']
   x.append('c')  # OK

   y: Final[Sequence[str]] = ['a', 'b']
   y.append('x')  # Error: "Sequence[str]" has no attribute "append"
   z: Final = ('a', 'b')  # Also works


Type checkers should treat uses of a final name that was initialized
with a literal as if it was replaced by the literal. For example, the
following should be allowed

型チェッカーは、リテラルで初期化された最終名の使用を、リテラルで置き換えられたかのように扱う必要があります。 たとえば、次のことが許可されるべきです。::

   from typing import NamedTuple, Final

   X: Final = "x"
   Y: Final = "y"
   N = NamedTuple("N", [(X, int), (Y, int)])


Reference Implementation
========================

The mypy [#mypy]_ type checker supports `Final` and `final`. A
reference implementation of the runtime component is provided in the
``typing_extensions`` [#typing_extensions]_ module.


Rejected/deferred Ideas
=======================

The name ``Const`` was also considered as the name for the ``Final``
type annotation. The name ``Final`` was chosen instead because the
concepts are related and it seemed best to be consistent between them.

Constという名前も、Final型注釈の名前と見なされました。 概念が関連していて、それらの間で一貫しているのが最も良いように思われたので、代わりに名前Finalが選ばれました。

We considered using a single name ``Final`` instead of introducing
``final`` as well, but ``@Final`` just looked too weird to us.

finalを導入するのではなく、Finalという単一の名前を使用することも検討しましたが、@ Finalは見た目が奇妙すぎました。

A related feature to final classes would be Scala-style sealed
classes, where a class is allowed to be inherited only by classes
defined in the same module. Sealed classes seem most useful in
combination with pattern matching, so it does not seem to justify the
complexity in our case. This could be revisisted in the future.

最終クラスに関連する機能は、Scalaスタイルのシールクラスです。ここで、クラスは、同じモジュール内で定義されたクラスによってのみ継承されることが許可されています。 密封されたクラスはパターンマッチングと組み合わせると最も便利に見えるので、私たちの場合の複雑さを正当化するようには思われません。 これは将来修正されるかもしれません。

It would be possible to have the ``@final`` decorator on classes
dynamically prevent subclassing at runtime. Nothing else in ``typing``
does any runtime enforcement, though, so ``final`` will not either.
A workaround for when both runtime enforcement and static checking is
desired is to use this idiom (possibly in a support module)

実行時にクラスの@finalデコレータが動的にサブクラス化を防ぐようにすることは可能です。 とはいえ、実行時に強制することができるものは他に何もありませんので、finalもそうではありません。 実行時の強制と静的チェックの両方が望まれる場合の回避策は、このイディオムを使用することです（おそらくサポートモジュール内で）。::

  if typing.TYPE_CHECKING:
      from typing import final
  else:
      from runtime_final import final


References
==========

.. [#PEP-484] PEP 484, Type Hints, van Rossum, Lehtosalo, Langa
   (http://www.python.org/dev/peps/pep-0484)

.. [#PEP-526] PEP 526, Syntax for Variable Annotations, Gonzalez,
   House, Levkivskyi, Roach, van Rossum
   (http://www.python.org/dev/peps/pep-0526)

.. [#PEP-586] PEP 486, Literal Types, Lee, Levkivskyi, Lehtosalo
   (http://www.python.org/dev/peps/pep-0586)

.. [#mypy] http://www.mypy-lang.org/

.. [#typing_extensions] https://github.com/python/typing/typing_extensions

Copyright
=========

This document has been placed in the public domain.

..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
