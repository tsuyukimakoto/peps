PEP: 586
Title: Literal Types
Author: Michael Lee <michael.lee.0x2a@gmail.com>, Ivan Levkivskyi <levkivskyi@gmail.com>, Jukka Lehtosalo <jukka.lehtosalo@iki.fi>
BDFL-Delegate: Guido van Rossum <guido@python.org>
Discussions-To: Typing-Sig <typing-sig@python.org>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 14-Mar-2018
Python-Version: 3.8
Post-History: 14-Mar-2018

Abstract
========

This PEP proposes adding *Literal types* to the PEP 484 ecosystem.
Literal types indicate that some expression has literally a
specific value. For example, the following function will accept
only expressions that have literally the value "4"

このPEPはPEP 484エコシステムに*リテラル型*を追加することを提案します。 リテラル型は、ある表現が文字通り特定の値を持つことを示します。 例えば、次の関数は文字通り "4"という値を持つ式だけを受け入れます::

    from typing import Literal

    def accepts_only_four(x: Literal[4]) -> None:
        pass

    accepts_only_four(4)   # OK
    accepts_only_four(19)  # Rejected

Motivation and Rationale
========================

Python has many APIs that return different types depending on the
value of some argument provided. For example

Pythonには、提供されたいくつかの引数の値に応じて異なる型を返す多くのAPIがあります。 例えば:

-  ``open(filename, mode)`` returns either ``IO[bytes]`` or ``IO[Text]``
   depending on whether the second argument is something like ``r`` or
   ``rb``.

   ``open(filename、mode)`` は、2番目の引数が ``r`` または ``rb`` かに応じて、「IO[bytes]」または「IO[Text]」のいずれかを返します

-  ``subprocess.check_output(...)`` returns either bytes or text
   depending on whether the ``universal_newlines`` keyword argument is
   set to ``True`` or not.

   ``subprocess.check_output(...)`` は、 ``universal_newlines`` キーワード引数が ``True``に設定されているかどうかに応じてバイトまたはテキストを返します。

This pattern is also fairly common in many popular 3rd party libraries.
For example, here are just two examples from pandas and numpy respectively

このパターンは、多くの人気のあるサードパーティのライブラリでもかなり一般的です。
例えば、これはそれぞれpandaとnumpyの2つの例です。:

-  ``pandas.concat(...)`` will return either ``Series`` or
   ``DataFrame`` depending on whether the ``axis`` argument is set to
   0 or 1.

   ``pandas.concat(...)``は ``axis`` 引数が0か1のどちらに設定されているかに応じて ``Series`` または ``DataFrame`` を返します

-  ``numpy.unique`` will return either a single array or a tuple containing
   anywhere from two to four arrays depending on three boolean flag values.

   ``numpy.unique`` は単一の配列か、3つのブーリアンフラグ値に応じて2つから4つの配列を含むタプルを返します。

The typing issue tracker contains some
`additional examples and discussion <typing-discussion_>`_.

タイピング問題トラッカーには、 `additional examples and discussion <typing-discussion_>`_ が含まれています。

There is currently no way of expressing the type signatures of these
functions: PEP 484 does not include any mechanism for writing signatures
where the return type varies depending on the value passed in.
Note that this problem persists even if we redesign these APIs to
instead accept enums: ``MyEnum.FOO`` and ``MyEnum.BAR`` are both
considered to be of type ``MyEnum``.

現在、これらの関数の型シグネチャを表現する方法はありません。PEP 484には、渡された値によって戻り型が異なるシグネチャを書き込むためのメカニズムが含まれていません。 enums： ``MyEnum.FOO`` と ``MyEnum.BAR`` はどちらも ``MyEnum`` 型であると見なされます。

Currently, type checkers work around this limitation by adding ad hoc
extensions for important builtins and standard library functions. For
example mypy comes bundled with a plugin that attempts to infer more
precise types for ``open(...)``. While this approach works for standard
library functions, it’s unsustainable in general: it’s not reasonable to
expect 3rd party library authors to maintain plugins for N different
type checkers.

現在、型チェッカーは重要な組み込み関数と標準ライブラリ関数のためのアドホック拡張を追加することによってこの制限を回避しています。 例えば、mypyは ``open(...)`` に対してより正確な型を推論しようとするプラグインをバンドルしています。 このアプローチは標準的なライブラリ関数には有効ですが、一般的には持続不可能です。サードパーティのライブラリ作成者がN種類のチェッカーのプラグインを保守することを期待するのは合理的ではありません。

We propose adding *Literal types* to address these gaps.

これらのギャップに対処するために*リテラル型*を追加することを提案します。

Core Semantics
==============

This section outlines the baseline behavior of literal types.

この節では、リテラル型の基本的な動作について概説します。

Core behavior
-------------

Literal types indicate that a variable has a specific and
concrete value. For example, if we define some variable ``foo`` to have
type ``Literal[3]``, we are declaring that ``foo`` must be exactly equal
to ``3`` and no other value.

リテラル型は、変数が特定の具体的な値を持つことを示します。 たとえば、あるタイプの ``Literal[3]`` を持つように変数 ``foo`` を定義すると、 ``foo`` は ``3`` と全く同じでなければならず、他の値と同じでなければならないと宣言します。

Given some value ``v`` that is a member of type ``T``, the type
``Literal[v]`` shall be treated as a subtype of ``T``. For example,
``Literal[3]`` is a subtype of ``int``.

``T`` 型のメンバである ``v`` という値が与えられると、 ``Literal[v]`` 型は ``T`` のサブタイプとして扱われるものとします。 例えば、 ``Literal[3]`` は ``int`` のサブタイプです。

All methods from the parent type will be directly inherited by the
literal type. So, if we have some variable ``foo`` of type ``Literal[3]``
it’s safe to do things like ``foo + 5`` since ``foo`` inherits int’s
``__add__`` method. The resulting type of ``foo + 5`` is ``int``.

親型からのすべてのメソッドは、リテラル型によって直接継承されます。 したがって、 ``Literal[3]`` 型の変数 ``foo`` があれば、 ``foo`` はintの ``__add__`` メソッドを継承するので、 ``foo + 5`` のようなことをしても安全です。 結果の ``foo + 5`` は ``int`` です。

This "inheriting" behavior is identical to how we
`handle NewTypes. <newtypes_>`_.

この「継承」動作は、NewTypeを処理する方法と同じです。

Equivalence of two Literals
---------------------------

Two types ``Literal[v1]`` and ``Literal[v2]`` are equivalent when
both of the following conditions are true

次の2つの条件が成り立つとき、2つの型 ``Literal[v1]`` と ``Literal[v2]`` は等価です:

1. ``type(v1) == type(v2)``
2. ``v1 == v2``

For example, ``Literal[20]`` and ``Literal[0x14]`` are equivalent.
However, ``Literal[0]`` and ``Literal[False]`` is *not* equivalent
despite that ``0 == False`` evaluates to 'true' at runtime: ``0``
has type ``int`` and ``False`` has type ``bool``.

例えば、 ``Literal[20]`` と ``Literal[0x14]`` は同等です。しかし、 ``Literal[0]`` と ``Literal[False]`` は実行時に ``0 == False`` が `true' と評価されるにもかかわらず*同等*ではありません： ``0`` 型は ``int`` と ``False`` は ``bool`` 型です。

Shortening unions of literals
-----------------------------

Literals are parameterized with one or more values. When a Literal is
parameterized with more than one value, it's treated as exactly equivalent
to the union of those types. That is, ``Literal[v1, v2, v3]`` is equivalent
to ``Union[Literal[v1], Literal[v2], Literal[v3]]``.

リテラルは1つ以上の値でパラメータ化されています。 リテラルが複数の値でパラメータ化されている場合、それはこれらの型の和集合とまったく同じものとして扱われます。 つまり、 ``Literal[v1、v2、v3]`` は ``Union[Literal[v1], Literal[v2], Literal[v3]]`` と同義です。

This shortcut helps make writing signatures for functions that accept
many different literals more ergonomic — for example, functions like
``open(...)``

このショートカットは、多くの異なるリテラルを受け付ける関数の署名をより人間工学的にするのを助けます - 例えば、 ``open(...)`` のような関数::

   # Note: this is a simplification of the true type signature.
   _PathType = Union[str, bytes, int]

   @overload
   def open(path: _PathType,
            mode: Literal["r", "w", "a", "x", "r+", "w+", "a+", "x+"],
            ) -> IO[Text]: ...
   @overload
   def open(path: _PathType,
            mode: Literal["rb", "wb", "ab", "xb", "r+b", "w+b", "a+b", "x+b"],
            ) -> IO[bytes]: ...

   # Fallback overload for when the user isn't using literal types
   @overload
   def open(path: _PathType, mode: str) -> IO[Any]: ...

The provided values do not all have to be members of the same type.
For example, ``Literal[42, "foo", True]`` is a legal type.

提供された値がすべて同じタイプのメンバーである必要はありません。 たとえば、 ``Literal[42, "foo", True]`` は有効な型です。

However, Literal **must** be parameterized with at least one type.
Types like ``Literal[]`` or ``Literal`` are illegal.

ただし、Literal は少なくとも1つの型でパラメータ化する必要があります。 ``Literal[]`` や ``Literal`` のような型は違法です。

Legal and illegal parameterizations
===================================

This section describes what exactly constitutes a legal ``Literal[...]`` type:
what values may and may not be used as parameters.

このセクションでは、正当な ``Literal[...]`` タイプを正確に構成するものについて説明します。パラメータとして使用できる値と使用できない値を説明します。

In short, a ``Literal[...]`` type may be parameterized by one or more literal
expressions, and nothing else.

一言で言えば、 ``Literal[...]`` 型は1つ以上のリテラル式でパラメータ化できますが、それ以外は何もできません。

Legal parameters for ``Literal`` at type check time
---------------------------------------------------

``Literal`` may be parameterized with literal ints, byte and unicode strings,
bools, Enum values and ``None``. So for example, all of
the following would be legal

``Literal`` は文字通りのint、byteとunicodeの文字列、bool、Enumの値と ``None`` でパラメータ化できます。 したがって、たとえば、以下のすべてが合法となります::

   Literal[26]
   Literal[0x1A]  # Exactly equivalent to Literal[26]
   Literal[-4]
   Literal["hello world"]
   Literal[b"hello world"]
   Literal[u"hello world"]
   Literal[True]
   Literal[Color.RED]  # Assuming Color is some enum
   Literal[None]

**Note:** Since the type ``None`` is inhabited by just a single
value, the types ``None`` and ``Literal[None]`` are exactly equivalent.
Type checkers may simplify ``Literal[None]`` into just ``None``.

** Note:** ``None`` 型はただ1つの値によって占められているので、 ``None`` 型と ``Literal[None]`` 型は完全に同等です。 型チェッカーは ``Literal [None]`` を単に ``None`` に単純化するかもしれません。

``Literal`` may also be parameterized by other literal types, or type aliases
to other literal types. For example, the following is legal

``Literal`` は他のリテラル型、または他のリテラル型への型エイリアスによってもパラメータ化されます。 例えば、以下は合法です::

    ReadOnlyMode         = Literal["r", "r+"]
    WriteAndTruncateMode = Literal["w", "w+", "wt", "w+t"]
    WriteNoTruncateMode  = Literal["r+", "r+t"]
    AppendMode           = Literal["a", "a+", "at", "a+t"]

    AllModes = Literal[ReadOnlyMode, WriteAndTruncateMode,
                       WriteNoTruncateMode, AppendMode]

This feature is again intended to help make using and reusing literal types
more ergonomic.

この機能も、リテラル型の使用と再利用をより人間工学的なものにするためのものです。

**Note:** As a consequence of the above rules, type checkers are also expected
to support types that look like the following

**Note:** 上記の規則の結果として、型チェッカーは次のような型をサポートすることも期待されています。::

    Literal[Literal[Literal[1, 2, 3], "foo"], 5, None]

This should be exactly equivalent to the following type

これは次の型と正確に等価であるべきです::

    Literal[1, 2, 3, "foo", 5, None]

...and also to the following type

...そして次のようなタイプにも::

    Optional[Literal[1, 2, 3, "foo", 5]]

**Note:** String literal types like ``Literal["foo"]`` should subtype either
bytes or unicode in the same way regular string literals do at runtime.

**Note:** ``Literal["foo"]`` のような文字列リテラル型は、実行時に通常の文字列リテラルが行うのと同じ方法で、バイトかユニコードのどちらかをサブタイプするべきです。

For example, in Python 3, the type ``Literal["foo"]`` is equivalent to
``Literal[u"foo"]``, since ``"foo"`` is equivalent to ``u"foo"`` in Python 3.

たとえば、Python 3では、 ``Literal["foo"]`` 型は ``Literal[u"foo"]`` と等価です。なぜならPython3では ``"foo"`` は ``u "foo"`` と等価だからです。

Similarly, in Python 2, the type ``Literal["foo"]`` is equivalent to
``Literal[b"foo"]`` -- unless the file includes a
``from __future__ import unicode_literals`` import, in which case it would be
equivalent to ``Literal[u"foo"]``.

同様に、Python 2では、 ``Literal["foo"]`` 型は ``Literal[b"foo"]`` と同等です - ファイルが ``from __future__ import unicode_literals`` インポートを含まない限り、 その場合、それは ``Literal[u"foo"]`` と同等になります。

Illegal parameters for ``Literal`` at type check time
-----------------------------------------------------

The following parameters are intentionally disallowed by design

以下のパラメータは意図的に設計上禁止されています:

- Arbitrary expressions like ``Literal[3 + 4]`` or
  ``Literal["foo".replace("o", "b")]``.

  ``Literal[3 + 4]`` や ``Literal["foo".replace("o", "b")]`` のような任意の表現。

  - Rationale: Literal types are meant to be a
    minimal extension to the PEP 484 typing ecosystem and requiring type
    checkers to interpret potentially expressions inside types adds too
    much complexity. Also see `Rejected or out-of-scope ideas`_.

    理論的根拠：リテラル型は、PEP 484タイピングエコシステムに対する最小限の拡張であることを意図しており、型チェッカーが型の中の潜在的な式を解釈することを余儀なくさせると、非常に複雑になります。 「拒否されたアイデアや範囲外のアイデア」も参照してください。

  - As a consequence, complex numbers like ``Literal[4 + 3j]`` and
    ``Literal[-4 + 2j]`` are also prohibited. For consistency, literals like
    ``Literal[4j]`` that contain just a single complex number are also
    prohibited.

    結果として、 ``Literal[4 + 3j]`` や ``Literal[-4 + 2j]`` のような複素数も禁止されています。 一貫性のために、 ``Literal[4j]`` のように単一の複素数しか含まないリテラルも禁止されています。

  - The only exception to this rule is the unary ``-`` (minus) for ints: types
    like ``Literal[-5]`` are *accepted*.

    この規則の唯一の例外は整数のための単項の ``-`` （マイナス）です。 ``Literal[-5]`` のような型は *受け入れられます* 。

-  Tuples containing valid literal types like ``Literal[(1, "foo", "bar")]``.
   The user could always express this type as
   ``Tuple[Literal[1], Literal["foo"], Literal["bar"]]`` instead. Also,
   tuples are likely to be confused with the ``Literal[1, 2, 3]``
   shortcut.

   ``Literal[(1, "foo", "bar")]`` のような有効なリテラル型を含むタプル。 ユーザは常にこの型を ``Tuple[Literal[1], Literal["foo"], Literal["bar"]]`` のように表現することができます。また、タプルは ``Literal[1, 2, 3]`` ショートカットと混同される可能性があります。

-  Mutable literal data structures like dict literals, list literals, or
   set literals: literals are always implicitly final and immutable. So,
   ``Literal[{"a": "b", "c": "d"}]`` is illegal.

   辞書リテラル、リストリテラル、セットリテラルなどの可変リテラルデータ構造体：リテラルは常に暗黙的に最終的で不変です。 したがって、 ``Literal[{"a": "b", "c": "d"}]`` は違法です。

-  Any other types: for example, ``Literal[Path]``, or
   ``Literal[some_object_instance]`` are illegal. This includes typevars: if
   ``T`` is a typevar,  ``Literal[T]`` is not allowed. Typevars can vary over
   only types, never over values.

   それ以外の型：例えば ``Literal[Path]`` や ``Literal[some_object_instance]``は違法です。これはtypevarsを含みます: ``T`` がtypevarの場合、 ``Literal[T]`` は許されません。 型変数は型だけで変化し、値では変化しません。

The following are provisionally disallowed for simplicity. We can consider
allowing them in future extensions of this PEP.

以下は簡潔にするために暫定的に許可されていません。 このPEPの将来の拡張でそれらを許可することを考慮することができます。

-  Floats: e.g. ``Literal[3.14]``. Representing Literals of infinity or NaN
   in a clean way is tricky; real-world APIs are unlikely to vary their
   behavior based on a float parameter.

   フロート： ``リテラル[3.14]`` 。 無限大またはNaNのリテラルをクリーンな方法で表現するのは難しいです。 実際のAPIは、floatパラメータに基づいて動作が変わる可能性は低いです。
  
-  Any: e.g. ``Literal[Any]``. ``Any`` is a type, and ``Literal[...]`` is
   meant to contain values only. It is also unclear what ``Literal[Any]``
   would actually semantically mean.

   どれでも ``リテラル[任意]``。 ``Any`` は型であり、 ``Literal[...]`` は値のみを含むことを意味しています。 また、 ``Literal[Any]`` が実際に意味的に何を意味するのかも明確ではありません。

Parameters at runtime
---------------------

Although the set of parameters ``Literal[...]`` may contain at type check time
is very small, the actual implementation of ``typing.Literal`` will not perform
any checks at runtime. For example

型チェック時に ``Literal[...]`` に含まれるかもしれないパラメータのセットは非常に小さいですが、 ``typing.Literal`` の実際の実装は実行時にチェックを行いません。 例えば::

   def my_function(x: Literal[1 + 2]) -> int:
       return x * 3

   x: Literal = 3
   y: Literal[my_function] = my_function

The type checker should reject this program: all three uses of
``Literal`` are *invalid* according to this spec. However, Python itself
should execute this program with no errors.

型チェッカーはこのプログラムを拒否するべきです： ``Literal`` の3つの使い方はすべてこの仕様書によれば *無効* です。 ただし、Python自体がこのプログラムをエラーなく実行するはずです。

This is partly to help us preserve flexibility in case we want to expand the
scope of what ``Literal`` can be used for in the future, and partly because
it is not possible to detect all illegal parameters at runtime to begin with.
For example, it is impossible to distinguish between ``Literal[1 + 2]`` and
``Literal[3]`` at runtime.

これは、将来的に ``Literal`` が使える範囲を広げたい場合や、最初から実行時にすべての不正なパラメータを検出することが不可能なために、柔軟性を維持するのに役立ちます。 例えば、実行時に ``Literal[1 + 2]`` と ``Literal[3]`` を区別することは不可能です。

Literals, enums, and forward references
---------------------------------------

One potential ambiguity is between literal strings and forward
references to literal enum members. For example, suppose we have the
type ``Literal["Color.RED"]``. Does this literal type
contain a string literal or a forward reference to some ``Color.RED``
enum member?

1つの潜在的なあいまいさは、リテラル文字列とリテラルenumメンバーへの前方参照との間にあります。 たとえば、 ``Literal["Color.RED"]`` という型があるとします。 このリテラル型は文字列リテラルか、あるいは ``Color.RED`` 列挙型メンバーへの前方参照を含んでいますか？

In cases like these, we always assume the user meant to construct a
literal string. If the user wants a forward reference, they must wrap
the entire literal type in a string -- e.g. ``"Literal[Color.RED]"``.

このような場合、ユーザは常にリテラル文字列を構築しようとしていると想定します。 もしユーザが前方参照を望んでいるなら、彼らは文字列でリテラル型全体を包まなければなりません - 例えば。 ``"Literal[Color.RED]"`` 。

Type inference
==============

This section describes a few rules regarding type inference and
literals, along with some examples.

この節では、型推論とリテラルに関するいくつかの規則をいくつかの例とともに説明します。

Backwards compatibility
-----------------------

When type checkers add support for Literal, it's important they do so
in a way that maximizes backwards-compatibility. Type checkers should
ensure that code that used to type check continues to do so after support
for Literal is added on a best-effort basis.

型チェッカーがリテラルのサポートを追加するとき、後方互換性を最大にするようにそれらがそうすることが重要です。 型チェッカーは、リテラルのサポートがベストエフォートベースで追加された後も、型チェックに使用されていたコードが引き続き実行されるようにする必要があります。

This is particularly important when performing type inference. For
example, given the statement ``x = "blue"``, should the inferred
type of ``x`` be ``str`` or ``Literal["blue"]``?

これは型推論を実行するときに特に重要です。 たとえば、ステートメント ``x = "blue"`` を考えた場合、推論されたタイプ ``x`` は ``str`` または ``Literal["blue"]`` のどちらにすべきでしょうか。

One naive strategy would be to always assume expressions are intended
to be Literal types. So, ``x`` would always have an inferred type of
``Literal["blue"]`` in the example above. This naive strategy is almost
certainly too disruptive -- it would cause programs like the following
to start failing when they previously did not

1つの単純な戦略は、式が常にリテラル型であることを意図していると想定することです。 そのため、上記の例では ``x`` は常に推論された ``Literal["blue"]`` 型を持つことになります。 この単純な戦略はほとんど確実に破壊的すぎます - 以前はそうでなかったのに次のようなプログラムが失敗し始めるでしょう::

    # If a type checker infers 'var' has type Literal[3]
    # and my_list has type List[Literal[3]]...
    var = 3
    my_list = [var]

    # ...this call would be a type-error.
    my_list.append(4)

Another example of when this strategy would fail is when setting fields
in objects

この方法が失敗する場合のもう1つの例は、オブジェクトにフィールドを設定するときです。::

    class MyObject:
        def __init__(self) -> None:
            # If a type checker infers MyObject.field has type Literal[3]...
            self.field = 3

    m = MyObject()

    # ...this assignment would no longer type check
    m.field = 4

An alternative strategy that *does* maintain compatibility in every case would
be to always assume expressions are *not* Literal types unless they are
explicitly annotated otherwise. A type checker using this strategy would
always infer that ``x`` is of type ``str`` in the first example above.

他の方法で明示的に注釈が付けられていない限り、すべての場合において互換性を維持する別の方法は、式が常にリテラル型ではないと想定することです。 この戦略を使った型チェッカーは、上の最初の例では ``x`` は ``str`` 型であると常に推論します。

This is not the only viable strategy: type checkers should feel free to experiment
with more sophisticated inference techniques. This PEP does not mandate any
particular strategy; it only emphasizes the importance of backwards compatibility.

これが唯一の実行可能な戦略ではありません。型チェッカーはもっと洗練された推論技術を試して自由に感じるはずです。 このPEPは特定の戦略を強制するものではありません。 それは後方互換性の重要性を強調するだけです。

Using non-Literals in Literal contexts
--------------------------------------

Literal types follow the existing rules regarding subtyping with no additional
special-casing. For example, programs like the following are type safe

リテラル型は、特別な大文字小文字の区別なしに、サブタイプに関する既存の規則に従います。 たとえば、次のようなプログラムはタイプセーフです。::

   def expects_str(x: str) -> None: ...
   var: Literal["foo"] = "foo"

   # Legal: Literal["foo"] is a subtype of str
   expects_str(var)

This also means non-Literal expressions in general should not automatically
be cast to Literal. For example

これはまた、リテラル以外の式を一般に自動的にリテラルにキャストしないことを意味します。 例えば::

   def expects_literal(x: Literal["foo"]) -> None: ...

   def runner(my_str: str) -> None:
       # ILLEGAL: str is not a subclass of Literal["foo"]
       expects_literal(my_str)

**Note:** If the user wants their API to support accepting both literals
*and* the original type -- perhaps for legacy purposes -- they should
implement a fallback overload. See `Interactions with overloads`_.

**Note:** ユーザーが自分のAPIにリテラル *と* 元の型の両方を受け入れることをサポートしたい場合 - おそらくレガシーの目的で - それらはフォールバックオーバーロードを実装する必要があります。 `Interactions with overloads`_ を参照してください。

Interactions with other types and features
==========================================

This section discusses how Literal types interact with other existing types.

この節では、リテラル型が他の既存の型とどのように相互作用するかについて説明します。

Intelligent indexing of structured data
---------------------------------------

Literals can be used to "intelligently index" into structured types like
tuples, NamedTuple, and classes. (Note: this is not an exhaustive list).

リテラルは、タプル、NamedTuple、クラスなどの構造化タイプに「インテリジェントにインデックスを付ける」ために使用できます。 （注：これは完全なリストではありません）。

For example, type checkers should infer the correct value type when
indexing into a tuple using an int key that corresponds a valid index

たとえば、型チェッカーは、有効なインデックスに対応するintキーを使用してタプルにインデックスを付けるときに正しい値の型を推測する必要があります。::

   a: Literal[0] = 0
   b: Literal[5] = 5

   some_tuple: Tuple[int, str, List[bool]] = (3, "abc", [True, False])
   reveal_type(some_tuple[a])   # Revealed type is 'int'
   some_tuple[b]                # Error: 5 is not a valid index into the tuple

We expect similar behavior when using functions like getattr

getattrのような関数を使うときも同じような振る舞いを期待します::

   class Test:
       def __init__(self, param: int) -> None:
           self.myfield = param

       def mymethod(self, val: int) -> str: ...

   a: Literal["myfield"]  = "myfield"
   b: Literal["mymethod"] = "mymethod"
   c: Literal["blah"]     = "blah"

   t = Test()
   reveal_type(getattr(t, a))  # Revealed type is 'int'
   reveal_type(getattr(t, b))  # Revealed type is 'Callable[[int], str]'
   getattr(t, c)               # Error: No attribute named 'blah' in Test

**Note:** See `Interactions with Final`_ for a proposal on how we can
express the variable declarations above in a more compact manner.

**Note: **上 記の変数宣言をもっとコンパクトに表現する方法についての提案は `Interactions with Final`_ を参照してください。

Interactions with overloads
---------------------------

Literal types and overloads do not need to interact in  a special
way: the existing rules work fine.

リテラル型とオーバーロードは特別な方法で対話する必要はありません。既存の規則はうまく機能します。

However, one important use case type checkers must take care to
support is the ability to use a *fallback* when the user is not using literal
types. For example, consider ``open``

ただし、サポートするように注意しなければならない重要なユースケース型チェッカーの1つは、ユーザーがリテラル型を使用していないときに *fallback* を使用する機能です。 例えば、 ``open`` を考えてください::

   _PathType = Union[str, bytes, int]

   @overload
   def open(path: _PathType,
            mode: Literal["r", "w", "a", "x", "r+", "w+", "a+", "x+"],
            ) -> IO[Text]: ...
   @overload
   def open(path: _PathType,
            mode: Literal["rb", "wb", "ab", "xb", "r+b", "w+b", "a+b", "x+b"],
            ) -> IO[bytes]: ...

   # Fallback overload for when the user isn't using literal types
   @overload
   def open(path: _PathType, mode: str) -> IO[Any]: ...

If we were to change the signature of ``open`` to use just the first two overloads,
we would break any code that does not pass in a literal string expression.
For example, code like this would be broken

最初の2つのオーバーロードだけを使用するように ``open`` のシグネチャを変更すると、リテラル文字列式を渡さないコードはすべて壊れます。 例えば、このようなコードは壊れているでしょう::

   mode: str = pick_file_mode(...)
   with open(path, mode) as f:
       # f should continue to be of type IO[Any] here

A little more broadly: we propose adding a policy to typeshed that
mandates that whenever we add literal types to some existing API, we also
always include a fallback overload to maintain backwards-compatibility.

もう少し広く言えば、既存のAPIにリテラル型を追加するときはいつでも、後方互換性を維持するために常にフォールバックオーバーロードを含めることを強制するポリシーをtypeshedに追加することを提案します。

Interactions with generics
--------------------------

Types like ``Literal[3]`` are meant to be just plain old subclasses of
``int``. This means you can use types like ``Literal[3]`` anywhere
you could use normal types, such as with generics.

``Literal[3]`` のような型は、単に ``int`` の単なる古いサブクラスであることを意味します。 これは、総称のように通常の型が使えるところならどこでも ``Literal[3]`` のような型を使えることを意味します。

This means that it is legal to parameterize generic functions or
classes using Literal types

これは、リテラル型を使用してジェネリック関数またはクラスをパラメータ化することは正当であることを意味します。::

   A = TypeVar('A', bound=int)
   B = TypeVar('B', bound=int)
   C = TypeVar('C', bound=int)

   # A simplified definition for Matrix[row, column]
   class Matrix(Generic[A, B]):
       def __add__(self, other: Matrix[A, B]) -> Matrix[A, B]: ...
       def __matmul__(self, other: Matrix[B, C]) -> Matrix[A, C]: ...
       def transpose(self) -> Matrix[B, A]: ...

   foo: Matrix[Literal[2], Literal[3]] = Matrix(...)
   bar: Matrix[Literal[3], Literal[7]] = Matrix(...)

   baz = foo @ bar
   reveal_type(baz)  # Revealed type is 'Matrix[Literal[2], Literal[7]]'

Similarly, it is legal to construct TypeVars with value restrictions
or bounds involving Literal types

同様に、値の制限またはリテラル型を含む範囲でTypeVarsを構築することは合法です。::

   T = TypeVar('T', Literal["a"], Literal["b"], Literal["c"])
   S = TypeVar('S', bound=Literal["foo"])

...although it is unclear when it would ever be useful to construct a
TypeVar with a Literal upper bound. For example, the ``S`` TypeVar in
the above example is essentially pointless: we can get equivalent behavior
by using ``S = Literal["foo"]`` instead.

...リテラルの上限を使ってTypeVarを構築するのがいつ有用であるかは明確ではありませんが。 たとえば、上の例の ``S``  TypeVarは本質的に無意味です。代わりに ``S = Literal["foo"]`` を使っても同等の動作が得られます。

**Note:** Literal types and generics deliberately interact in only very
basic and limited ways. In particular, libraries that want to type check
code containing an heavy amount of numeric or numpy-style manipulation will
almost certainly likely find Literal types as proposed in this PEP to be
insufficient for their needs.

**Note:** リテラル型と総称は、非常に基本的で限られた方法でのみ意図的に対話しています。 特に、大量の数値操作や派手な操作を含むチェックコードを入力したいライブラリでは、このPEPで提案されているリテラル型は、そのニーズに対して不十分であるとほぼ確実に考えられます。

We considered several different proposals for fixing this, but ultimately
decided to defer the problem of integer generics to a later date. See
`Rejected or out-of-scope ideas`_ for more details.

これを修正するためのいくつかの異なる提案を検討しましたが、最終的に整数ジェネリックの問題を後日に延期することにしました。 詳細は `Rejected or out-of-scope ideas`_ をご覧ください。

Interactions with enums and exhaustiveness checks
-------------------------------------------------

Type checkers should be capable of performing exhaustiveness checks when
working Literal types that have a closed number of variants, such as
enums. For example, the type checker should be capable of inferring that
the final ``else`` statement must be of type ``str``, since all three
values of the ``Status`` enum have already been exhausted

列挙型のように、バリアントの数が限られているリテラル型を処理するときは、型チェッカーは徹底的なチェックを実行できなければなりません。 例えば、 `` Status`` enumの3つの値はすべて使い尽くされているので、型チェッカーは最後の `` else``文が `` str``型でなければならないことを推論できなければなりません。::

    class Status(Enum):
        SUCCESS = 0
        INVALID_DATA = 1
        FATAL_ERROR = 2

    def parse_status(s: Union[str, Status]) -> None:
        if s is Status.SUCCESS:
            print("Success!")
        elif s is Status.INVALID_DATA:
            print("The given data is invalid because...")
        elif s is Status.FATAL_ERROR:
            print("Unexpected fatal error...")
        else:
            # 's' must be of type 'str' since all other options are exhausted
            print("Got custom status: " + s)

The interaction described above is not new: it's already
`already codified within PEP 484 <pep-484-enums_>`_. However, many type
checkers (such as mypy) do not yet implement this due to the expected
complexity of the implementation work.

上で説明した相互作用は新しいものではありません。それはすでに `already codified within PEP 484 <pep-484-enums_>`_ の中で成文化されています。 しかし、実装作業の複雑さが予想されるため、多くの型チェッカー（mypyなど）はまだこれを実装していません。

Some of this complexity will be alleviated once Literal types are introduced:
rather than entirely special-casing enums, we can instead treat them as being
approximately equivalent to the union of their values and take advantage of any
existing logic regarding unions, exhaustibility, type narrowing, reachability,
and so forth the type checker might have already implemented.

リテラル型が導入されれば、この複雑さの一部は緩和されます。完全に特殊なケースの列挙型ではなく、それらを値の和集合とほぼ同等であるとみなし、和集合、網羅性、型絞り込みに関する既存のロジックを利用できます。 、到達可能性など、型チェッカーはすでに実装されている可能性があります。

So here, the ``Status`` enum could be treated as being approximately equivalent
to ``Literal[Status.SUCCESS, Status.INVALID_DATA, Status.FATAL_ERROR]``
and the type of ``s`` narrowed accordingly.

そのため、ここでは、 ``Status`` 列挙子は ``Literal[Status.SUCCESS, Status.INVALID_DATA, Status.FATAL_ERROR]`` とほぼ同等であるとして扱うことができ、それに応じて ``s`` の型も狭くなりました。

Interactions with narrowing
---------------------------

Type checkers may optionally perform additional analysis for both enum and
non-enum Literal types beyond what is described in the section above.

型チェッカーは、列挙型と非列挙リテラル型の両方に対して、上記のセクションで説明したもの以外に追加の分析をオプションで実行できます。

For example, it may be useful to perform narrowing based on things like
containment or equality checks

たとえば、封じ込めや等価性チェックなどに基づいて絞り込みを実行すると便利です。::

   def parse_status(status: str) -> None:
       if status in ("MALFORMED", "ABORTED"):
           # Type checker could narrow 'status' to type
           # Literal["MALFORMED", "ABORTED"] here.
           return expects_bad_status(status)

       # Similarly, type checker could narrow 'status' to Literal["PENDING"]
       if status == "PENDING":
           expects_pending_status(status)

It may also be useful to perform narrowing taking into account expressions
involving Literal bools. For example, we can combine ``Literal[True]``,
``Literal[False]``, and overloads to construct "custom type guards"

リテラルブールを含む式を考慮して絞り込みを実行すると便利な場合もあります。 例えば、 ``Literal[True]``、 ``Literal[False]`` 、そしてオーバーロードを組み合わせて「カスタムタイプのガード」を構築することができます。::

   @overload
   def is_int_like(x: Union[int, List[int]]) -> Literal[True]: ...
   @overload
   def is_int_like(x: object) -> bool: ...
   def is_int_like(x): ...

   vector: List[int] = [1, 2, 3]
   if is_int_like(vector):
       vector.append(3)
   else:
       vector.append("bad")   # This branch is inferred to be unreachable

   scalar: Union[int, str]
   if is_int_like(scalar):
       scalar += 3      # Type checks: type of 'scalar' is narrowed to 'int'
   else:
       scalar += "foo"  # Type checks: type of 'scalar' is narrowed to 'str'
    
Interactions with Final
-----------------------

`PEP 591 <pep-591_>`_ proposes adding a "Final" qualifier to the typing
ecosystem. This qualifier can be used to declare that some variable or
attribute cannot be reassigned

`PEP 591 <pep-591_>`_ はタイピングエコシステムに "Final"修飾子を追加することを提案します。 この修飾子は、何らかの変数または属性を再割り当てできないことを宣言するために使用できます。::

    foo: Final = 3
    foo = 4           # Error: 'foo' is declared to be Final

Note that in the example above, we know that ``foo`` will always be equal to
exactly ``3``. A type checker can use this information to deduce that ``foo``
is valid to use in any context that expects a ``Literal[3]``

上の例では、 ``foo`` は常に ``3`` と等しくなることがわかっています。 型チェッカーはこの情報を使って ``foo`` が ``Literal[3]`` を期待する文脈で使うのに有効であると推論することができます。::

    def expects_three(x: Literal[3]) -> None: ...

    expects_three(foo)  # Type checks, since 'foo' is Final and equal to 3

The ``Final`` qualifier serves as a shorthand for declaring that a variable
is *effectively Literal*.

``Final`` 修飾子は変数が *事実上リテラル* であることを宣言するための省略形として働きます。

If both this PEP and PEP 591 are accepted, type checkers are expected to
support this shortcut. Specifically, given a variable or attribute assignment
of the form ``var: Final = value`` where ``value`` is a valid parameter for
``Literal[...]``, type checkers should understand that ``var`` may be used in
any context that expects a ``Literal[value]``.

このPEPとPEP 591の両方が承認された場合、タイプチェッカーはこのショートカットをサポートすると予想されます。 具体的には、 ``var:Final = value`` という形式の変数または属性の代入を考えると、 ``value`` は ``Literal[...]`` の有効なパラメータであるため、型チェッカーは ``var'' を理解するべきです。 ``Literal[value]`` を期待する文脈では ``var`` を使用することができます。

Type checkers are not obligated to understand any other uses of Final. For
example, whether or not the following program type checks is left unspecified

型チェッカーは、Finalの他の用途を理解する義務を負いません。 たとえば、次のプログラムタイプチェックが指定されていないかどうか::

    # Note: The assignment does not exactly match the form 'var: Final = value'.
    bar1: Final[int] = 3
    expects_three(bar1)  # May or may not be accepted by type checkers

    # Note: "Literal[1 + 2]" is not a legal type.
    bar2: Final = 1 + 2
    expects_three(bar2)  # May or may not be accepted by type checkers

Rejected or out-of-scope ideas
==============================

This section outlines some potential features that are explicitly out-of-scope.

この節では、明示的に範囲外となる可能性のある機能について概説します。

True dependent types/integer generics
-------------------------------------

This proposal is essentially describing adding a very simplified
dependent type system to the PEP 484 ecosystem. One obvious extension
would be to implement a full-fledged dependent type system that lets users
predicate types based on their values in arbitrary ways. That would
let us write signatures like the below

この提案は本質的にPEP 484エコシステムに非常に単純化された依存型システムを追加することを説明しています。 明白な拡張の1つは、ユーザーが自分の値に基づいて任意の方法で型を予測できる、本格的な依存型システムを実装することです。 それは以下のような署名を書くことを可能にするでしょう::

   # A vector has length 'n', containing elements of type 'T'
   class Vector(Generic[N, T]): ...

   # The type checker will statically verify our function genuinely does
   # construct a vector that is equal in length to "len(vec1) + len(vec2)"
   # and will throw an error if it does not.
   def concat(vec1: Vector[A, T], vec2: Vector[B, T]) -> Vector[A + B, T]:
       # ...snip...

At the very least, it would be useful to add some form of integer generics.

少なくとも、何らかの形の整数総称を追加すると便利です。

Although such a type system would certainly be useful, it’s out of scope
for this PEP: it would require a far more substantial amount of implementation
work, discussion, and research to complete compared to the current proposal.

そのようなタイプシステムは確かに有用ですが、それはこのPEPの範囲外です：それは現在の提案と比較して完了するためにはるかに多くの量の実装作業、議論と研究を必要とするでしょう。

It's entirely possible we'll circle back and revisit this topic in the future:
we very likely will need some form of dependent typing along with other
extensions like variadic generics to support popular libraries like numpy.

将来このトピックを遡って再検討する可能性は十分にあります。ナンプのような人気のあるライブラリをサポートするためには、可変長ジェネリックのような他の拡張と共に何らかの形の依存型付けが必要になるでしょう。

This PEP should be seen as a stepping stone towards this goal,
rather then an attempt at providing a comprehensive solution.

このPEPは、包括的な解決策を提供するという試みではなく、この目標に向けての足がかりと見なすべきです。

Adding more concise syntax
--------------------------

One objection to this PEP is that having to explicitly write ``Literal[...]``
feels verbose. For example, instead of writing

このPEPに対する1つの異議は、明示的に ``Literal[...]`` を書かなければならないことが冗長に感じるということです。 たとえば、書く代わりに::

   def foobar(arg1: Literal[1], arg2: Literal[True]) -> None:
       pass

...it would be nice to instead write

...代わりに書くといいでしょう::

   def foobar(arg1: 1, arg2: True) -> None:
       pass

Unfortunately, these abbreviations simply will not work with the
existing implementation of ``typing`` at runtime. For example, the
following snippet crashes when run using Python 3.7

残念ながら、これらの省略形は実行時に既存の ``typing`` の実装ではうまくいきません。 たとえば、Python 3.7を使用して実行すると、次のコードがクラッシュします。::

   from typing import Tuple

   # Supposed to accept tuple containing the literals 1 and 2
   def foo(x: Tuple[1, 2]) -> None:
       pass

Running this yields the following exception

これを実行すると、次の例外が発生します。::

   TypeError: Tuple[t0, t1, ...]: each t must be a type. Got 1.

We don’t want users to have to memorize exactly when it’s ok to elide
``Literal``, so we require ``Literal`` to always be present.

``Literal`` を排除しても大丈夫なときにユーザーが正確に暗記しなければならないのではないので、 ``Literal`` が常に存在することを要求します。

A little more broadly, we feel overhauling the syntax of types in
Python is not within the scope of this PEP: it would be best to have
that discussion in a separate PEP, instead of attaching it to this one.
So, this PEP deliberately does not try and innovate Python's type syntax.

もう少し広く言えば、Pythonの型の構文を見直すことはこのPEPの範囲内ではないと考えています。この議論を別のPEPで行うのではなく、このPEPで議論することをお勧めします。 したがって、このPEPは故意にPythonの型構文を試したり革新したりすることはしません。

Backporting the ``Literal`` type
================================

Once this PEP is accepted, the ``Literal`` type will need to be backported for
Python versions that come bundled with older versions of the ``typing`` module.
We plan to do this by adding ``Literal`` to the ``typing_extensions`` 3rd party
module, which contains a variety of other backported types.

このPEPが受け入れられたら、古いバージョンの `` typing``モジュールにバンドルされているPythonバージョンのために `` Literal``型をバックポートする必要があります。 他のさまざまなバックポートされた型を含む `` typing_extensions``サードパーティモジュールに `` Literal``を追加することでこれを行う予定です。

Implementation
==============

The mypy type checker currently has implemented a large subset of the behavior
described in this spec, with the exception of enum Literals and some of the
more complex narrowing interactions described above.

mypy型チェッカーは現在、enum Literalsと上記で説明したより複雑な絞り込み相互作用のいくつかを除いて、この仕様で説明されている動作の大部分を実装しています。

Related work
============

This proposal was written based on the discussion that took place in the
following threads

この提案は、以下のスレッドで行われた議論に基づいて書かれました。:

-  `Check that literals belong to/are excluded from a set of values <typing-discussion_>`_

  `リテラルが値の集合に属している/から除外されていることを確認する<typing-discussion_>` _

-  `Simple dependent types <mypy-discussion_>`_

  `単純な依存型<mypy-discussion_>` _

-  `Typing for multi-dimensional arrays <arrays-discussion_>`_

  `多次元配列の型付け<arrays-discussion_>` _

The overall design of this proposal also ended up converging into
something similar to how
`literal types are handled in TypeScript <typescript-literal-types_>`_.

この提案の全体的な設計もまた、 `リテラル型がTypeScript <typescript-literal-types_>` _でどのように扱われるかに似たものに収束することになりました。

.. _typing-discussion: https://github.com/python/typing/issues/478

.. _mypy-discussion: https://github.com/python/mypy/issues/3062

.. _arrays-discussion: https://github.com/python/typing/issues/513

.. _typescript-literal-types: https://www.typescriptlang.org/docs/handbook/advanced-types.html#string-literal_types

.. _typescript-index-types: https://www.typescriptlang.org/docs/handbook/advanced-types.html#index-types

.. _newtypes: https://www.python.org/dev/peps/pep-0484/#newtype-helper-function

.. _pep-484-enums: https://www.python.org/dev/peps/pep-0484/#support-for-singleton-types-in-unions

.. _pep-591: https://www.python.org/dev/peps/pep-0591/

Acknowledgements
================

Thanks to Mark Mendoza, Ran Benita, Rebecca Chen, and the other members of
typing-sig for their comments on this PEP.

Additional thanks to the various participants in the mypy and typing issue
trackers, who helped provide a lot of the motivation and reasoning behind
this PEP.


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

