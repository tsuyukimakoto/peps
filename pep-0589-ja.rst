PEP: 589
Title: TypedDict: Type Hints for Dictionaries with a Fixed Set of Keys
Author: Jukka Lehtosalo <jukka.lehtosalo@iki.fi>
Sponsor: Guido van Rossum <guido@python.org>
BDFL-Delegate: Guido van Rossum <guido@python.org>
Discussions-To: typing-sig@python.org
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 20-Mar-2019
Python-Version: 3.8
Post-History:


Abstract
========

PEP 484 [#PEP-484]_ defines the type ``Dict[K, V]`` for uniform
dictionaries, where each value has the same type, and arbitrary key
values are supported.  It doesn't properly support the common pattern
where the type of a dictionary value depends on the string value of
the key.  This PEP proposes a type constructor ``typing.TypedDict`` to
support the use case where a dictionary object has a specific set of
string keys, each with a value of a specific type.

PEP 484 [＃PEP-484] _統一辞書用に ``Dict[K, V]`` 型を定義しています。それぞれの値は同じ型で、任意のキー値がサポートされています。 辞書値の型がキーの文字列値に依存する一般的なパターンを正しくサポートしていません。 このPEPは辞書オブジェクトがそれぞれ特定の型の値を持つ文字列キーの特定のセットを持つユースケースをサポートするために型コンストラクタ ``typing.TypedDict`` を提案します。

Here is an example where PEP 484 doesn't allow us to annotate
satisfactorily

これはPEP 484が私達が満足に注釈をつけることを可能にしない例です::

    movie = {'name': 'Blade Runner',
             'year': 1982}

This PEP proposes the addition of a new type constructor, called
``TypedDict``, to allow the type of ``movie`` to be represented
precisely

このPEPでは、 ``movie`` の型を正確に表現できるようにするために、 ``TypedDict`` と呼ばれる新しい型コンストラクタを追加することを提案しています。::

    from typing import TypedDict

    class Movie(TypedDict):
        name: str
        year: int

Now a type checker should accept this code

型チェッカーはこのコードを受け入れます::

    movie: Movie = {'name': 'Blade Runner',
                    'year': 1982}


Motivation
==========

Representing an object or structured data using (potentially nested)
dictionaries with string keys (instead of a user-defined class) is a
common pattern in Python programs.  Representing JSON objects is
perhaps the canonical use case, and this is popular enough that Python
ships with a JSON library.  This PEP proposes a way to allow such code
to be type checked more effectively.

（ユーザー定義クラスの代わりに）文字列キーを持つ（潜在的にネストされた）辞書を使用してオブジェクトまたは構造化データを表現することは、Pythonプログラムでは一般的なパターンです。 JSONオブジェクトを表現することは、おそらく標準的なユースケースであり、そしてこれはPythonがJSONライブラリと共に出荷するのに十分人気があります。 このPEPはそのようなコードがより効果的に型チェックされることを可能にする方法を提案します。

More generally, representing pure data objects using only Python
primitive types such as dictionaries, strings and lists has had
certain appeal.  They are are easy to serialize and deserialize even
when not using JSON.  They trivially support various useful operations
with no extra effort, including pretty-printing (through ``str()`` and
the ``pprint`` module), iteration, and equality comparisons.

より一般的には、辞書、文字列、リストなどのPythonプリミティブ型のみを使用して純粋なデータオブジェクトを表現することには、一定の魅力があります。 JSONを使用していなくても、シリアライズおよびデシリアライズが容易です。 きれいな印刷（ ``str()`` と ``pprint`` モジュールを使って）、繰り返し、そして等価性の比較など、特別な労力なしで便利なさまざまな操作を簡単にサポートします。

PEP 484 doesn't properly support the use cases mentioned above.  Let's
consider a dictionary object that has exactly two valid string keys,
``'name'`` with value type ``str``, and ``'year'`` with value type
``int``.  The PEP 484 type ``Dict[str, Any]`` would be suitable, but
it is too lenient, as arbitrary string keys can be used, and arbitrary
values are valid.  Similarly, ``Dict[str, Union[str, int]]`` is too
general, as the value for key ``'name'`` could be an ``int``, and
arbitrary string keys are allowed.  Also, the type of a subscription
expression such as ``d['name']`` (assuming ``d`` to be a dictionary of
this type) would be ``Union[str, int]``, which is too wide.

PEP 484は上記のユースケースを正しくサポートしていません。 値型が ``str`` の ``name`` と、値型が ``int`` の ``year`` の2つの有効な文字列キーを持つ辞書オブジェクトを考えてみましょう。 PEP 484タイプの ``Dict[str, Any]`` は適切でしょうが、任意の文字列キーを使用することができ、任意の値が有効であるため、寛容すぎます。 同様に、 ``Dict[str, Union[str, int]]`` は一般的すぎます。キー ``name`` の値は ``int`` でもよく、任意の文字列キーを使用できます。 また、 ``d['name']``(``d`` がこの型の辞書であると仮定している）のような購読表現の型は ``Union[str, int]`` になり、広すぎます。

Dataclasses are a more recent alternative to solve this use case, but
there is still a lot of existing code that was written before
dataclasses became available, especially in large existing codebases
where type hinting and checking has proven to be helpful.  Unlike
dictionary objects, dataclasses don't directly support JSON
serialization, though there is a third-party package that implements
it [#dataclasses-json]_.

データクラスはこのユースケースを解決するためのより最近の代替手段ですが、データタイプが利用可能になる前に書かれた多くの既存のコード、特に型のヒントとチェックが役立つことがわかっている大規模な既存コードベースがまだあります。 辞書オブジェクトとは異なり、データクラスはJSONシリアライゼーションを直接サポートしませんが、それを実装するサードパーティ製のパッケージがあります [#dataclasses-json]_ 。

Specification
=============

A TypedDict type represents dictionary objects with a specific set of
string keys, and with specific value types for each valid key.  Each
string key can be either required (it must be present) or
non-required (it doesn't need to exist).

TypedDict型は、特定の文字列キーのセットと、有効な各キーの特定の値型を持つ辞書オブジェクトを表します。 各文字列キーは、必須（存在する必要がある）または不要（存在する必要はない）のいずれかです。

This PEP proposes two ways of defining TypedDict types.  The first uses
a class-based syntax.  The second is an alternative
assignment-based syntax that is provided for backwards compatibility,
to allow the feature to be backported to older Python versions.  The
rationale is similar to why PEP 484 supports a comment-based
annotation syntax for Python 2.7: type hinting is particularly useful
for large existing codebases, and these often need to run on older
Python versions.  The two syntax options parallel the syntax variants
supported by ``typing.NamedTuple``.  Other proposed features include
TypedDict inheritance and totality (specifying whether keys are
required or not).

このPEPはTypedDict型を定義する2つの方法を提案します。 1つ目はクラスベースの構文を使います。 2つ目は、後方互換性のために提供されている代替の代入ベースの構文で、この機能を古いPythonバージョンにバックポートできるようにします。 理論的根拠は、PEP 484がPython 2.7用のコメントベースのアノテーション構文をサポートする理由と似ています。型ヒントは、既存の大規模コードベースに特に有用であり、これらは古いPythonバージョンで実行する必要があります。2つの構文オプションは ``typing.NamedTuple`` でサポートされている構文の変形と似ています。 提案されている他の機能には、TypedDictの継承と全体性（キーが必要かどうかを指定）があります。

This PEP also provides a sketch of how a type checker is expected
to support type checking operations involving TypedDict objects.
Similar to PEP 484, this discussion is left somewhat vague on purpose,
to allow experimentation with a wide variety of different type
checking approaches.  In particular, type compatibility should be
based on structural compatibility: a more specific TypedDict type can
be compatible with a smaller (more general) TypedDict type.

このPEPは、TypedDictオブジェクトを含む型チェック操作を型チェッカーがどのようにサポートすることが期待されるかについてのスケッチも提供します。 PEP 484と同様に、この議論は意図的にやや曖昧なままにされて、多種多様な異なる型チェックアプローチでの実験を可能にします。 特に、型の互換性は構造的な互換性に基づいている必要があります。より具体的なTypedDict型は、より小さな（より一般的な）TypedDict型と互換性があります。

Class-based Syntax
------------------

A TypedDict type can be defined using the class definition syntax with
``typing.TypedDict`` as the sole base class

TypedDict型は唯一の基本クラスとして ``typing.TypedDict`` を持つクラス定義構文を使って定義することができます::

    from typing import TypedDict

    class Movie(TypedDict):
        name: str
        year: int

``Movie`` is a TypedDict type with two items: ``'name'`` (with type
``str``) and ``'year'`` (with type ``int``).

``Movie`` は2つの項目を持つTypedDict型です： ``'name'`` (型 ``str``) と ``'year'``(型 ``int``) 。

A type checker should validate that the body of a class-based
TypedDict definition conforms to the following rules

型チェッカーは、クラスベースのTypedDict定義の本体が次の規則に準拠していることを検証する必要があります。:

* The class body should only contain lines with item definitions of the
  form ``key: value_type``, optionally preceded by a docstring.  The
  syntax for item definitions is identical to attribute annotations,
  but there must be no initializer, and the key name actually refers
  to the string value of the key instead of an attribute name.

  クラス本体は ``key:value_type`` という形式の項目定義を持つ行のみを含み、必要に応じてdocstringが先行します。項目定義の構文は属性アノテーションと同じですが、初期化子があってはならず、キー名は実際には属性名ではなくキーのストリング値を参照します。

* Type comments cannot be used with the class-based syntax, for
  consistency with the class-based ``NamedTuple`` syntax.  (Note that
  it would not be sufficient to support type comments for backwards
  compatibility with Python 2.7, since the class definition may have a
  ``total`` keyword argument, as discussed below, and this isn't valid
  syntax in Python 2.7.)  Instead, this PEP provides an alternative,
  assignment-based syntax for backwards compatibility, discussed in
  `Alternative Syntax`_.

  クラスベースの ``NamedTuple`` 構文との一貫性のために、タイプコメントはクラスベースの構文では使用できません。 （後述するように、クラス定義は ``total`` キーワード引数を持つことができるので、Python 2.7との後方互換性のために型コメントをサポートするだけでは十分ではないことに注意してください。これはPython 2.7では無効な構文です。） 代わりに、このPEPは後方互換性のために代替えの代入ベースの構文を提供します。これは `Alternative Syntax`_ で論じられています。

* String literal forward references are valid in the value types.

  文字列リテラル前方参照は値型で有効です。

* Methods are not allowed, since the runtime type of a TypedDict
  object will always be just ``dict`` (it is never a subclass of
  ``dict``).

  TypedDictオブジェクトの実行時型は常に ``dict`` になるので（メソッドは許されません）（ ``dict`` のサブクラスにはなりません）。

* Specifying a metaclass is not allowed.

  メタクラスを指定することはできません。

An empty TypedDict can be created by only including ``pass`` in the
body (if there is a docstring, ``pass`` can be omitted)

本体に ``pass`` を含めるだけで空のTypedDictを作成できます（docstringがある場合は ``pass`` は省略できます）::

    class EmptyDict(TypedDict):
        pass


Using TypedDict Types
---------------------

Here is an example of how the type ``Movie`` can be used

これが `` Movie``型の使い方の例です。::

    movie: Movie = {'name': 'Blade Runner',
                    'year': 1982}

An explicit ``Movie`` type annotation is generally needed, as
otherwise an ordinary dictionary type could be assumed by a type
checker, for backwards compatibility.  When a type checker can infer
that a constructed dictionary object should be a TypedDict, an
explicit annotation can be omitted.  A typical example is a dictionary
object as a function argument.  In this example, a type checker is
expected to infer that the dictionary argument should be understood as
a TypedDict

そうでなければ後方互換性のために普通の辞書型が型チェッカーによって想定されるかもしれないので、明示的な ``Movie`` 型注釈が一般に必要です。 型チェッカーが、構築された辞書オブジェクトがTypedDictであるべきだと推論できる場合、明示的な注釈を省略することができます。 典型的な例は、関数の引数としての辞書オブジェクトです。 この例では、型チェッカーは辞書の引数がTypedDictとして理解されるべきであると推論すると期待されます。::

    def record_movie(movie: Movie) -> None: ...

    record_movie({'name': 'Blade Runner', 'year': 1982})

Another example where a type checker should treat a dictionary display
as a TypedDict is in an assignment to a variable with a previously
declared TypedDict type

型チェッカーが辞書の表示をTypedDictとして扱うべきもう1つの例は、以前に宣言されたTypedDict型を持つ変数への代入です。::

    movie: Movie
    ...
    movie = {'name': 'Blade Runner', 'year': 1982}

Operations on ``movie`` can be checked by a static type checker

``movie`` の操作は静的型チェッカーで確認できます::

    movie['director'] = 'Ridley Scott'  # Error: invalid key 'director'
    movie['year'] = '1982'  # Error: invalid value type ("int" expected)

The code below should be rejected, since ``'title'`` is not a valid
key, and the ``'name'`` key is missing

``'title'`` は有効なキーではなく、 ``'name'`` キーがないため、以下のコードは拒否されるべきです。::

    movie2: Movie = {'title': 'Blade Runner',
                     'year': 1982}

The created TypedDict type object is not a real class object.  Here
are the only uses of the type a type checker is expected to allow

作成されたTypedDict型オブジェクトは、実際のクラスオブジェクトではありません。 これが型チェッカーが許可すると期待される型の唯一の用途です。:

* It can be used in type annotations and in any context where an
  arbitrary type hint is valid, such as in type aliases and as the
  target type of a cast.

  型注釈や、型エイリアスやキャストのターゲット型など、任意の型ヒントが有効なコンテキストで使用できます。

* It can be used as a callable object with keyword arguments
  corresponding to the TypedDict items.  Non-keyword arguments are not
  allowed.  Example

  TypedDict項目に対応するキーワード引数を持つ呼び出し可能オブジェクトとして使用できます。 キーワード以外の引数は許可されません。 例::

      m = Movie(name='Blade Runner', year=1982)

  When called, the TypedDict type object returns an ordinary
  dictionary object at runtime

  呼び出されると、TypedDict型オブジェクトは実行時に通常の辞書オブジェクトを返します。::

      print(type(m))  # <class 'dict'>

* It can be used as a base class, but only when defining a derived
  TypedDict.  This is discussed in more detail below.

  基本クラスとして使用できますが、派生TypedDictを定義する場合に限ります。 これについては後で詳しく説明します。

In particular, TypedDict type objects cannot be used in
``isinstance()`` tests such as ``isinstance(d, Movie)``. The reason is
that there is no existing support for checking types of dictionary
item values, since ``isinstance()`` does not work with many PEP 484
types, including common ones like ``List[str]``.  This would be needed
for cases like this

特に、TypedDict型オブジェクトは、 `` isinstance（d、Movie） ``のような `` isinstance（） ``テストでは使用できません。 その理由は、 `` isinstance（） ``は `` List [str] ``のような一般的なものを含め多くのPEP 484タイプでは動作しないので、辞書アイテムの値のタイプをチェックするための既存のサポートがないからです。 これはこのような場合に必要となるでしょう::

    class Strings(TypedDict):
        items: List[str]

    print(isinstance({'items': [1]}, Strings))    # Should be False
    print(isinstance({'items': ['x']}, Strings))  # Should be True

The above use case is not supported.  This is consistent with how
``isinstance()`` is not supported for ``List[str]``.

上記のユースケースはサポートされていません。 これは、 `` isinstance（） ``が `` List [str] ``に対してどのようにサポートされていないかと一致しています。

Inheritance
-----------

It is possible for a TypedDict type to inherit from one or more
TypedDict types using the class-based syntax.  In this case the
``TypedDict`` base class should not be included.  Example

TypedDict型は、クラスベースの構文を使用して1つ以上のTypedDict型から継承することができます。 この場合、 `` TypedDict``基本クラスは含まれるべきではありません。 例::

    class BookBasedMovie(Movie):
        based_on: str

Now ``BookBasedMovie`` has keys ``name``, ``year``, and ``based_on``.
It is equivalent to this definition, since TypedDict types use
structural compatibility

これで `` BookBasedMovie``は `` name``、 `` year``、そして `` based_on``のキーを持ちます。 TypedDict型は構造的互換性を使用するため、この定義と同じです。::

    class BookBasedMovie(TypedDict):
        name: str
        year: int
        based_on: str

Here is an example of multiple inheritance

これは多重継承の例です::

    class X(TypedDict):
        x: int

    class Y(TypedDict):
        y: str

    class XYZ(X, Y):
        z: bool

The TypedDict ``XYZ`` has three items: ``x`` (type ``int``), ``y``
(type ``str``), and ``z`` (type ``bool``).

TypedDictの ``XYZ`` には3つの要素があります： ``x`` (タイプ ``int``) 、 ``y`` (タイプ ``str``) 、そして ``z`` (タイプ ``bool``)

A TypedDict cannot inherit from both a TypedDict type and a
non-TypedDict base class.

TypedDictは、TypedDict型とTypedDict以外の基本クラスの両方から継承することはできません。

Totality
--------

By default, all keys must be present in a TypedDict.  It is possible
to override this by specifying *totality*.  Here is how to do this
using the class-based syntax

デフォルトでは、すべてのキーはTypedDictに存在しなければなりません。 *totality* を指定することでこれを上書きすることが可能です。 これは、クラスベースの構文を使用してこれを行う方法です。::

    class Movie(TypedDict, total=False):
        name: str
        year: int

This means that a ``Movie`` TypedDict can have any of the keys omitted. Thus
these are valid

これは、 ``Movie`` TypedDict はキーをどれも省略できることを意味します。 したがって、これらは有効です::

    m: Movie = {}
    m2: Movie = {'year': 2015}

A type checker is only expected to support a literal ``False`` or
``True`` as the value of the ``total`` argument.  ``True`` is the
default, and makes all items defined in the class body be required.

型チェッカーは ``total`` 引数の値としてリテラルの ``False`` または ``True`` をサポートすることのみが期待されています。 ``True`` がデフォルトで、クラスボディで定義されているすべての項目を必須にします。

The totality flag only applies to items defined in the body of the
TypedDict definition.  Inherited items won't be affected, and instead
use totality of the TypedDict type where they were defined.  This makes
it possible to have a combination of required and non-required keys in
a single TypedDict type.

全体フラグは、TypedDict定義の本体に定義されている項目にのみ適用されます。 継承されたアイテムは影響を受けず、代わりに定義された場所でTypedDict型の全体を使用します。 これにより、単一のTypedDict型に必須キーと不要キーを組み合わせることができます。

Alternative Syntax
------------------

This PEP also proposes an alternative syntax that can be backported to
older Python versions such as 3.5 and 2.7 that don't support the
variable definition syntax introduced in PEP 526 [#PEP-526].  It
resembles the traditional syntax for defining named tuples

このPEPは、PEP 526 [＃PEP-526]で導入された変数定義構文をサポートしていない3.5や2.7などの古いPythonバージョンにバックポートできる代替構文も提案します。 これは、名前付きタプルを定義するための従来の構文に似ています::

    Movie = TypedDict('Movie', {'name': str, 'year': int})

It is also possible to specify totality using the alternative syntax

代替構文を使用して全体を指定することも可能です。::

    Movie = TypedDict('Movie',
                      {'name': str, 'year': int},
                      total=False)

The semantics are equivalent to the class-based syntax.  This syntax
doesn't support inheritance, however, and there is no way to
have both required and non-required fields in a single type.  The
motivation for this is keeping the backwards compatible syntax as
simple as possible while covering the most common use cases.

セマンティクスはクラスベースの構文と同等です。 ただし、この構文は継承をサポートしていないため、必須項目と不要項目の両方を1つの型に含めることはできません。 この動機は、最も一般的なユースケースをカバーしながら、後方互換性のある構文を可能な限り単純に保つことです。

A type checker is only expected to accept a dictionary display expression
as the second argument to ``TypedDict``.  In particular, a variable that
refers to a dictionary object does not need to be supported, to simplify
implementation.

型チェッカーは ``TypedDict`` の2番目の引数として辞書の表示式を受け付けることのみが期待されています。 特に、ディクショナリオブジェクトを参照する変数は、実装を簡単にするためにサポートされる必要はありません。

Type Consistency
----------------

Informally speaking, *type consistency* is a generalization of the
is-subtype-of relation to support the ``Any`` type.  It is defined
more formally in PEP 483 [#PEP-483]_).  This section introduces the
new, non-trivial rules needed to support type consistency for
TypedDict types.

非公式に言えば、*型の一貫性* は ``Any`` 型をサポートするためのis-subtype-of関係の一般化です。 これはPEP 483 [＃PEP-483] _）でより正式に定義されています。 このセクションでは、TypedDict型の型の一貫性をサポートするために必要な、重要な新しい規則を紹介します。

First, any TypedDict type is consistent with ``Mapping[str, object]``.
Second, a TypedDict type ``A`` is consistent with TypedDict ``B`` if
``A`` is structurally compatible with ``B``.  This is true if and only
if both of these conditions are satisfied:

まず、どんなTypedDict型も ``Mapping[str, object]`` と一致しています。 次に、 ``A`` が ``B`` と構造的に互換性がある場合、TypedDict型 ``A`` はTypedDict ``B`` と一致します。 これは、これらの条件の両方が満たされる場合に限り、当てはまります。

* For each key in ``B``, ``A`` has the corresponding key and the
  corresponding value type in ``A`` is consistent with the value type
  in ``B``.  For each key in ``B``, the value type in ``B`` is also
  consistent with the corresponding value type in ``A``.

  ``B`` の各キーに対して、 ``A`` は対応するキーを持ち、 ``A`` の対応する値型は ``B`` の値型と一致しています。 ``B`` の各キーについて、 ``B`` の値型も ``A`` の対応する値型と一致しています。

* For each required key in ``B``, the corresponding key is required
  in ``A``.  For each non-required key in ``B``, the corresponding key
  is not required in ``A``.

  ``B`` 内のそれぞれの必須キーに対して、対応するキーは ``A`` 内で必須です。 ``B`` のそれぞれの必須ではないキーに対して、対応するキーは ``A`` では必須ではありません。

Discussion:

* Value types behave invariantly, since TypedDict objects are mutable.
  This is similar to mutable container types such as ``List`` and
  ``Dict``.  Example where this is relevant

  TypedDictオブジェクトは可変であるため、値型は不変的に動作します。 これは ``List`` や ``Dict`` のような可変コンテナ型に似ています。 これが関係する例::

      class A(TypedDict):
          x: Optional[int]

      class B(TypedDict):
          x: int

      def f(a: A) -> None:
          a['x'] = None

      b: B = {'x': 0}
      f(b)  # Type check error: 'B' not compatible with 'A'
      b['x'] + 1  # Runtime error: None + 1

* A TypedDict type with required keys is not consistent with a
  TypedDict type with non-required keys, since the latter allows keys
  to be deleted.  Example where this is relevant

  必須のキーを持つTypedDict型は、必須ではないキーを持つTypedDict型とは一致しません。後者はキーを削除できるからです。 これが関係する例::

      class A(TypedDict, total=False):
          x: int

      class B(TypedDict):
          x: int

      def f(a: A) -> None:
          del a['x']

      b: B = {'x': 0}
      f(b)  # Type check error: 'B' not compatible with 'A'
      b['x'] + 1  # Runtime KeyError: 'x'

* A TypedDict type ``A`` with no key ``'x'`` is not consistent with a
  TypedDict type with a non-required key ``'x'``, since at runtime
  the key ``'x'`` could be present and have an incompatible type
  (which may not be visible through ``A`` due to structural subtyping).
  Example

  キー ``'x'`` を持たないTypedDict型 ``A`` は、実行時にはキー ```x'`` を持つので、必須ではないキー ``'x'`` を持つTypedDict型と矛盾します。 存在する可能性があり、互換性のない型を持つ可能性があります（構造的なサブタイプのために ``A`` を通して見えないかもしれません）。 例::

      class A(TypedDict, total=False):
          x: int
          y: int

      class B(TypedDict, total=False):
          x: int

      class C(TypedDict, total=False):
          x: int
          y: str

       def f(a: A) -> None:
           a[y] = 1

       def g(b: B) -> None:
           f(b)  # Type check error: 'B' incompatible with 'A'

       c: C = {'x': 0, 'y': 'foo'}
       g(c)
       c['y'] + 'bar'  # Runtime error: int + str

* A TypedDict isn't consistent with any ``Dict[...]`` type, since
  dictionary types allow destructive operations, including
  ``clear()``.  They also allow arbitrary keys to be set, which
  would compromise type safety.  Example

  TypedDictはDict[...]型と矛盾しません。辞書型はclear()を含む破壊的な操作を許すからです。 それらはまた、任意のキーを設定することを可能にし、それは型の安全性を危うくします。 例::

      class A(TypedDict):
          x: int

      class B(A):
          y: str

      def f(d: Dict[str, int]) -> None:
          d['y'] = 0

      def g(a: A) -> None:
          f(a)  # Type check error: 'A' incompatible with Dict[str, int]

      b: B = {'x': 0, 'y': 'foo'}
      g(b)
      b['y'] + 'bar'  # Runtime error: int + str

* A TypedDict with all ``int`` values is not consistent with
  ``Mapping[str, int]``, since there may be additional non-``int``
  values not visible through the type, due to structural subtyping.
  These can be accessed using the ``values()`` and ``items()``
  methods in ``Mapping``, for example.  Example

  すべての ``int`` 値を持つTypedDictは、 ``Mapping[str, int]`` と矛盾しません。構造的サブタイプのせいで、型を通して見えない追加の ``int`` 以外の値があるかもしれないからです。 例えば ``Mapping`` の ``values()`` と ``items()`` メソッドを使ってアクセスできます。 例::

      class A(TypedDict):
          x: int

      class B(TypedDict):
          x: int
          y: str

      def sum_values(m: Mapping[str, int]) -> int:
          n = 0
          for v in m.values():
              n += v  # Runtime error
          return n

      def f(a: A) -> None:
          sum_values(a)  # Error: 'A' incompatible with Mapping[str, int]

      b: B = {'x': 0, 'y': 'foo'}
      f(b)


Supported and Unsupported Operations
------------------------------------

Type checkers should support restricted forms of most ``dict``
operations on TypedDict objects.  The guiding principle is that
operations not involving ``Any`` types should be rejected by type
checkers if they may violate runtime type safety.  Here are some of
the most important type safety violations to prevent

型チェッカーはTypedDictオブジェクトに対するほとんどの `` dict``操作の制限された形式をサポートするべきです。 指針となる原則は、 `` Any``型を含まない操作はランタイム型の安全性に違反する可能性がある場合、型チェッカーによって拒否されるべきであるということです。 これを防ぐために最も重要なタイプの安全性違反のいくつかはここにあります:

1. A required key is missing.

  必要なキーがない場合

2. A value has an invalid type.

  バリューが不正な方の場合

3. A key that is not defined in the TypedDict type is added.

  TypedDict型に定義されていないキーが追加された場合

A key that is not a literal should generally be rejected, since its
value is unknown during type checking, and thus can cause some of
the above violations.

リテラルではないキーは、型チェック中にその値が未知であり、したがって上記の違反のいくつかを引き起こす可能性があるため、一般に拒否されるべきです。

The use of a key that is not known to exist should be reported as an
error, even if this wouldn't necessarily generate a runtime type
error.  These are often mistakes, and these may insert values with an
invalid type if structural subtyping hides the types of certain items.
For example, ``d['x'] = 1`` should generate a type check error if
``'x'`` is not a valid key for ``d`` (which is assumed to be a
TypedDict type).

存在することが知られていないキーの使用は、たとえこれが必ずしもランタイム型エラーを生成しないとしても、エラーとして報告されるべきです。 これらはしばしば間違いであり、構造的サブタイプが特定の項目のタイプを隠す場合、これらは無効なタイプの値を挿入する可能性があります。 たとえば、 ``d'' の有効なキーではない場合、 ``d['x'] = 1`` は型チェックエラーを生成します（TypedDict型と見なされます）。 。

Extra keys included in TypedDict object construction should also be
caught.  In this example, the ``director`` key is not defined in
``Movie`` and is expected to generate an error from a type checker

TypedDictオブジェクト構築に含まれる追加のキーもキャッチする必要があります。 この例では、 ``director`` キーは ``Movie`` で定義されていないので型チェッカーからエラーが発生すると予想されます::

    m: Movie = dict(
        name='Alien',
        year=1979,
        director='Ridley Scott')  # error: Unexpected key 'director'

Type checkers should reject the following operations on TypedDict
objects as unsafe, even though they are valid for normal dictionaries

型チェッカーは、TypedDictオブジェクトに対する次の操作は、通常の辞書に対して有効であっても安全ではないとして拒否する必要があります。:

* Operations with arbitrary ``str`` keys (instead of string literals
  or other expressions with known string values) should be rejected.
  This involves both destructive operations such as setting an item
  and read-only operations such as subscription expressions.

  （文字列リテラルや既知の文字列値を持つ他の式の代わりに）任意の ``str`` キーを使った操作は拒否されるべきです。 これには、アイテムの設定などの破壊的な操作と、サブスクリプション式などの読み取り専用の操作の両方が含まれます。

* ``clear()`` is not safe since it could remove required keys, some of
  which may not be directly visible because of structural
  subtyping.  ``popitem()`` is similarly unsafe, even if all known
  keys are not required (``total=False``).

  clear() は必要なキーを削除する可能性があるので安全ではありませんが、構造的なサブタイプのために直接表示されないものもあります。 たとえすべての既知のキーが必要でなくても (``total = False``) 、 ``popitem()``は同様に安全ではありません。

* ``del obj['key']`` should be rejected unless ``'key'`` is a
  non-required key.

  ``'key'`` が必須ではないキーでない限り、 ``del obj['key']`` は拒否されるべきです。

Type checkers may allow reading an item using ``d['x']`` even if
the key ``'x'`` is not required, instead of requiring the use of
``d.get('x')`` or an explicit ``'x' in d`` check.  The rationale is
that tracking the existence of keys is difficult to implement in full
generality, and that disallowing this could require many changes to
existing code.

型チェッカーは、 ``d.get('x')`` の使用を要求する代わりに、キー ``'x'`` が必要でなくても ``d['x']`` を使用して項目を読むことを許可するかもしれません またはd内の明示的な 'x' チェック。 論理的根拠は、キーの存在を追跡することを完全に一般化して実装することは困難であり、これを許可しないことは既存のコードへの多くの変更を必要とするかもしれないということです。

The exact type checking rules are up to each type checker to decide.
In some cases potentially unsafe operations may be accepted if the
alternative is to generate false positive errors for idiomatic code.

正確な型チェック規則は、決定する各型チェッカー次第です。 場合によっては、慣用的なコードに対して誤った肯定的なエラーを生成することである場合、潜在的に危険な操作が受け入れられることがあります。

Backwards Compatibility
=======================

To retain backwards compatibility, type checkers should not infer a
TypedDict type unless it is sufficiently clear that this is desired by
the programmer.  When unsure, an ordinary dictionary type should be
inferred.  Otherwise existing code that type checks without errors may
start generating errors once TypedDict support is added to the type
checker, since TypedDict types are more restrictive than dictionary
types.  In particular, they aren't subtypes of dictionary types.

下位互換性を維持するために、型チェッカーは、プログラマがこれを望んでいることが明確でない限り、TypedDict型を推論してはいけません。 よくわからない場合は、通常の辞書型を推論してください。 それ以外の場合、TypedDict型は辞書型よりも制限が厳しいため、TypedDictサポートが型チェッカーに追加されると、エラーなしで型チェックを行う既存のコードでエラーが発生し始める可能性があります。 特に、それらは辞書型のサブタイプではありません。

Reference Implementation
========================

The mypy [#mypy]_ type checker supports TypedDict types. A reference
implementation of the runtime component is provided in the
``mypy_extensions`` [#mypy_extensions]_ module.

mypy [#mypy]_ 型チェッカーはTypedDict型をサポートします。 ランタイムコンポーネントのリファレンス実装は ``mypy_extensions`` [#mypy_extensions]_ モジュールにあります。

Rejected Alternatives
=====================

Several proposed ideas were rejected.  The current set of features
seem to cover a lot of ground, and it was not not clear which of the
proposed extensions would be more than marginally useful.  This PEP
defines a baseline feature that can be potentially extended later.

いくつかの提案されたアイデアは拒否されました。 現在の一連の機能は多くの分野をカバーしているように見えますが、提案されている拡張機能のうちどれがごくわずかしか有用でないかは明らかではありませんでした。 このPEPは、将来拡張される可能性があるベースライン機能を定義します。

These are rejected on principle, as incompatible with the spirit of
this proposal

この提案の精神と両立しないため、これらは原則として拒絶されています。:

* TypedDict isn't extensible, and it addresses only a specific use
  case.  TypedDict objects are regular dictionaries at runtime, and
  TypedDict cannot be used with other dictionary-like or mapping-like
  classes, including subclasses of ``dict``.  There is no way to add
  methods to TypedDict types.  The motivation here is simplicity.

  TypedDictは拡張性がありません、そしてそれは特定のユースケースだけを扱います。 TypedDictオブジェクトは実行時には通常の辞書であり、TypedDictは `` dict``のサブクラスを含む他の辞書風クラスやマッピング風クラスと一緒に使うことはできません。 TypedDict型にメソッドを追加する方法はありません。 ここでの動機は単純さです。

* TypedDict type definitions could plausibly used to perform runtime
  type checking of dictionaries.  For example, they could be used to
  validate that a JSON object conforms to the schema specified by a
  TypedDict type.  This PEP doesn't include such functionality, since
  the focus of this proposal is static type checking only, and other
  existing types do not support this, as discussed in `Class-based
  syntax`_.  Such functionality can be provided by a third-party
  library using the ``typing_inspect`` [#typing_inspect]_ third-party
  module, for example.

  TypedDict型定義は、辞書の実行時型チェックを実行するためにもっともらしく使用される可能性があります。 たとえば、JSONオブジェクトがTypedDict型で指定されたスキーマに準拠していることを検証するために使用できます。 この提案の焦点は静的型チェックのみであり、他の既存の型はこれをサポートしていないので、このPEPはそのような機能を含みません、  `Class-based
  syntax`_ で議論されています。 そのような機能は、例えば ``typing_inspect`` [#typing_inspect]_ サードパーティモジュールを使ってサードパーティライブラリによって提供されることができます。

* TypedDict types can't be used in ``isinstance()`` or ``issubclass()``
  checks.  The reasoning is similar to why runtime type checks aren't
  supported in general.

  TypedDict types can't be used in ``isinstance()`` or ``issubclass()`` checks. The reasoning is similar to why runtime type checks aren't supported in general.

These features were left out from this PEP, but they are potential
extensions to be added in the future

これらの機能はこのPEPから除外されましたが、将来追加される可能性のある拡張です。:

* TypedDict doesn't support providing a *default value type* for keys
  that are not explicitly defined.  This would allow arbitrary keys to
  be used with a TypedDict object, and only explicitly enumerated keys
  would receive special treatment compared to a normal, uniform
  dictionary type.

  TypedDictは、明示的に定義されていないキーに対して *default value type* を提供することをサポートしていません。 これにより、TypedDictオブジェクトで任意のキーを使用することができ、明示的に列挙されたキーだけが通常の統一された辞書型と比較して特別な扱いを受けます。

* There is no way to individually specify whether each key is required
  or not.  No proposed syntax was clear enough.

  各キーが必要かどうかを個別に指定する方法はありません。 明確な構文は提案されていません。

* TypedDict can't be used for specifying the type of a ``**kwargs``
  argument.  This would allow restricting the allowed keyword
  arguments and their types.  According to PEP 484, using a TypedDict
  type as the type of ``**kwargs`` means that the TypedDict is valid
  as the *value* of arbitrary keyword arguments, but it doesn't
  restrict which keyword arguments should be allowed.  The syntax
  ``**kwargs: Expand[T]`` has been proposed for this [#expand]_.

  TypedDictは ``**kwargs`` 引数の型を指定するのには使えません。 これにより、許可されるキーワード引数とその種類を制限できます。 PEP 484によれば、TypedDict型を ``**kwargs`` の型として使用することはTypedDictが任意のキーワード引数の *value* として有効であることを意味しますが、どのキーワード引数が許可されるべきかを制限しません。 構文 [**kwargs: Expand[T]] がこの [#expand]_ のために提案されました。


Acknowledgements
================

David Foster contributed the initial implementation of TypedDict types
to mypy.  Improvements to the implementation have been contributed by
at least the author (Jukka Lehtosalo), Ivan Levkivskyi, Gareth T,
Michael Lee, Dominik Miedzinski, Roy Williams and Max Moroz.


References
==========

.. [#PEP-484] PEP 484, Type Hints, van Rossum, Lehtosalo, Langa
   (http://www.python.org/dev/peps/pep-0484)

.. [#dataclasses-json] Dataclasses JSON
   (https://github.com/lidatong/dataclasses-json)

.. [#PEP-526] PEP 526, Syntax for Variable Annotations, Gonzalez,
   House, Levkivskyi, Roach, van Rossum
   (http://www.python.org/dev/peps/pep-0484)

.. [#PEP-483] PEP 483, The Theory of Type Hints, van Rossum, Levkivskyi
   (http://www.python.org/dev/peps/pep-0483)

.. [#mypy] http://www.mypy-lang.org/

.. [#mypy_extensions] https://github.com/python/mypy_extensions

.. [#typing_inspect] https://github.com/ilevkivskyi/typing_inspect

.. [#expand] https://github.com/python/mypy/issues/4441


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
