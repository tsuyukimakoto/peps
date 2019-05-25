PEP: 570
Title: Python Positional-Only Parameters
Version: $Revision$
Last-Modified: $Date$
Author: Larry Hastings <larry@hastings.org>,
        Pablo Galindo <pablogsal@gmail.com>,
        Mario Corchero <mariocj89@gmail.com>,
        Eric N. Vander Weele <ericvw@gmail.com>
BDFL-Delegate: Guido van Rossum <guido@python.org>
Discussions-To: https://discuss.python.org/t/pep-570-python-positional-only-parameters/1078
Status: Accepted
Type: Standards Track
Content-Type: text/x-rst
Created: 20-Jan-2018


========
Abstract
========

This PEP proposes to introduce a new syntax, ``/``, for specifying
positional-only parameters in Python function definitions.

このPEPは ``/`` を使って、関数定義の引数を位置引数限定として指定できるようにするプロポーザルです。

Positional-only parameters have no externally-usable name. When a function
accepting positional-only parameters is called, positional arguments are mapped
to these parameters based solely on their order.

位置限定引数は外部利用可能な名前ではありません。一限定引数をとる関数が呼ばれると、引数が順番に仮引数にマッピングされます。

When designing APIs (application programming interfaces), library
authors try to ensure correct and intended usage of an API. Without the ability to
specify which parameters are positional-only, library authors must be careful
when choosing appropriate parameter names. This care must be taken
even for required parameters or when the parameters
have no external semantic meaning for callers of the API.

API（アプリケーション・プログラミング・インターフェース）を設計する時、ライブラリの作者はAPIが意図通りに正しく使われるようにしようとします。どのパラメーターが位置指定のみであるかを指定する機能がないと、適切なパラメーター名を選択するときにライブラリー作成者は注意を払う必要があります。 この注意は、必要なパラメータに対して、またはパラメータがAPIの呼び出し側にとって外部的な意味を持たない場合でも行われる必要があります。

In this PEP, we discuss:

* Python's history and current semantics for positional-only parameters

  位置引数限定のパラメータに対するPythonの歴史と現在のセマンティクス

* the problems encountered by not having them

  それらを持たないことによって遭遇する問題

* how these problems are handled without language-intrinsic support for
  positional-only parameters

  位置固有のパラメーターに対する言語固有のサポートなしに、これらの問題がどのように処理されるか

* the benefits of having positional-only parameters

  位置引数限定のパラメータを持つことの利点

Within context of the motivation, we then:

* discuss why positional-only parameters should be a feature intrinsic to the
  language

  位置引数限定のパラメータが言語固有の機能であるべき理由

* propose the syntax for marking positional-only parameters

  位置引数限定引数をマーキングするための構文を提案する

* present how to teach this new feature

  この新機能を教える方法を提示

* note rejected ideas in further detail

  拒否されたアイデアの詳細

==========
Motivation
==========

--------------------------------------------------------
History of Positional-Only Parameter Semantics in Python
--------------------------------------------------------

Python originally supported positional-only parameters. Early versions of the
language lacked the ability to call functions with arguments bound to parameters
by name. Around Python 1.0, parameter semantics changed to be
positional-or-keyword.  Since then, users have been able to provide arguments
to a function either positionally or by the keyword name specified in the
function's definition.

Pythonはもともと位置限定パラメータをサポートしていました。 この言語の初期のバージョンでは、引数を名前でパラメータにバインドして関数を呼び出すことができませんでした。 Python 1.0の前後で、パラメータの意味はposition-or-keywordに変更されました。 それ以来、ユーザーは、位置的に、または関数の定義で指定されたキーワード名によって、関数に引数を渡すことができました。

In current versions of Python, many CPython "builtin" and standard library
functions only accept positional-only parameters. The resulting semantics can be
easily observed by calling one of these functions using keyword arguments

現在のバージョンのPythonでは、多くのCPythonの "組み込み"関数や標準ライブラリ関数は位置のみのパラメータしか受け付けません。 結果の意味論はキーワード引数を使用してこれらの関数の1つを呼び出すことによって容易に観察することができます::

    >>> help(pow)
    ...
    pow(x, y, z=None, /)
    ...

    >>> pow(x=5, y=3)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: pow() takes no keyword arguments

``pow()`` expresses that its parameters are positional-only via the
``/`` marker. However, this is only a documentation convention; Python
developers cannot use this syntax in code.

``pow()`` は、そのパラメータが位置のみであることを ``/`` マーカーで表現します。 ただし、これは単なるドキュメントの表記法です。 Python開発者はこの構文をコードで使用することはできません。

There are functions with other interesting semantics

他の興味深いセマンティクスを持つ関数があります:

* ``range()``, an overloaded function, accepts an optional parameter to the
  *left* of its required parameter. [#RANGE]_

  オーバーロードされた関数 ``range()`` は、その必須パラメータの *左* にオプションのパラメータを受け入れます。

* ``dict()``, whose mapping/iterator parameter is optional and semantically
  must be positional-only. Any externally visible name for this parameter
  would occlude that name going into the ``**kwarg`` keyword variadic parameter
  dict. [#DICT]_

  ``dict()`` 、その mapping / iterator パラメータはオプションであり、意味的には位置引数限定でなければなりません。 このパラメータの外部から見える名前は、その名前が ``**kwarg`` キーワードの可変引数パラメータdictに入ることを防ぎます。

One can emulate these semantics in Python code by accepting
``(*args, **kwargs)`` and parsing the arguments manually. However, this results
in a disconnect between the function definition and what the function
contractually accepts. The function definition does not match the logic of the
argument handling.

``(*args, **kwargs)`` を受け入れて手動で引数を解析することで、Pythonコードでこれらのセマンティクスをエミュレートできます。 ただし、これにより、関数定義とその関数が契約上認めているものとが切り離されます。 関数定義が引数処理のロジックと一致しません。

Additionally, the ``/`` syntax is used beyond CPython for specifying similar
semantics (i.e., [#numpy-ufuncs]_ [#scipy-gammaln]_); thus, indicating that
these scenarios are not exclusive to CPython and the standard library.

さらに、 ``/`` 構文は、CPythonを超えて同様の意味を指定するために使用されます (すなわち、[#numpy-ufuncs]_ [#scipy-gammaln]_) 。したがって、これらのシナリオはCPythonおよび標準ライブラリ専用のものではないことを示しています。

-------------------------------------------
Problems Without Positional-Only Parameters
-------------------------------------------

Without positional-only parameters, there are challenges for library authors
and users of APIs. The following subsections outline the problems
encountered by each entity.

位置のみのパラメータがなければ、ライブラリ作者とAPIの利用者のための課題があります。 以下の小区分は各実体によって直面された問題について概説します。

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Challenges for Library Authors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

With positional-or-keyword parameters, the mix of calling conventions is not
always desirable. Authors may want to restrict usage of an API by disallowing
calling the API with keyword arguments, which exposes the name of the parameter when
part of the public API. This approach is especially useful for required function
parameters that already have semantic meaning (e.g,
``namedtuple(typenames, field_names, …)`` or when the parameter name has no
true external meaning (e.g., ``arg1``, ``arg2``, …, etc for ``min()``). If a
caller of an API starts using a keyword argument, the library author cannot rename
the parameter because it would be a breaking change.

位置パラメータまたはキーワードパラメータでは、呼び出し規約を組み合わせることが常に望ましいとは限りません。 作者は、パブリックAPIの一部であるときにパラメータの名前を公開するキーワード引数を使用してAPIを呼び出すことを禁止することで、APIの使用を制限したい場合があります。 このアプローチは、すでに意味的な意味を持つ必要な関数パラメータ（例えば ``namedtuple(typenames, field_names, ...)`` や、パラメータ名が本当の意味を持たない場合 (例えば ``arg1``, ``arg2``, …, etc for ``min()`` )に特に便利です。APIの呼び出し側がキーワード引数を使用して起動した場合、ライブラリの作者はパラメータの名前を変更できません。これは重大な変更になるためです。

Positional-only parameters can be emulated by extracting arguments from
``*args`` one by one. However, this approach is error-prone and is not
synonymous with the function definition, as previously mentioned. The usage of
the function is ambiguous and forces users to look at ``help()``, the
associated auto-generated documentation, or source code to understand what
parameters the function contractually accepts.

位置引数限定のパラメータは ``*args`` から引数を一つずつ抽出することでエミュレートできます。 ただし、この方法はエラーが発生しやすく、前述のように関数定義と同義ではありません。 この関数の使い方はあいまいであり、関数が契約上どのパラメータを受け入れるかを理解するためには ``help()`` 、関連する自動生成されたドキュメント、またはソースコードを見ることをユーザに強います。

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Challenges for Users of an API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Users may be surprised when first encountering positional-only notation. This
is expected given that it has only recently been documented
[#document-positional-only]_ and it is not possible to use in Python code. For
these reasons, this notation is currently an outlier that appears only in
CPython APIs developed in C. Documenting the notation and making it possible
to use it in Python code would eliminate this disconnect.

最初に位置引数限定の表記法に遭遇したとき、ユーザーは驚くかもしれません。 これはごく最近になって文書化された [#document-positional-only]_ ばかりでPythonコードでは使用できないので驚くのは想像にかたくありません。これらの理由から、この表記法は現在、Cで開発されたCPython APIにのみ現れる異常値です。表記法を文書化し、それをPythonコードで使用できるようにすると、この切断は解消されます。

Furthermore, the current documentation for positional-only parameters is inconsistent

さらに、位置引数限定の引数に関する現在のドキュメントは矛盾しています:

* Some functions denote optional groups of positional-only parameters by
  enclosing them in nested square brackets. [#BORDER]_

  一部の関数は、ネストされた角括弧で囲むことによって、位置限定引数の引数のオプションのグループを表します。

* Some functions denote optional groups of positional-only parameters by
  presenting multiple prototypes with varying numbers of parameters.
  [#SENDFILE]_

  いくつかの関数は、さまざまな数のパラメータを持つ複数のプロトタイプを提示することによって、位置限定引数のオプションのグループを表します。

* Some functions use *both* of the above approaches. [#RANGE]_ [#ADDCH]_

  いくつかの関数は上記の両方のアプローチを使います。

Another point the current documentation does not distinguish is
whether a function takes positional-only parameters. ``open()`` accepts keyword
arguments; however, ``ord()`` does not — there is no way of telling just by
reading the existing documentation.

現在のドキュメントでは区別されていないもう1つの点は、関数が位置のみのパラメータを受け取るかどうかです。 ``open()`` はキーワード引数を受け付けます。 しかし、 ``ord()`` はそうではありません - 既存のドキュメントを読むだけではわかりません。

--------------------------------------
Benefits of Positional-Only Parameters
--------------------------------------

Positional-only parameters give more control to library authors to better
express the intended usage of an API and allows the API to evolve in a safe,
backward-compatible way. Additionally, it makes the Python language more
consistent with existing documentation and the behavior of various
"builtin" and standard library functions.

位置限定引数は、APIの意図された使用法をより適切に表現するためにライブラリー作者により多くの制御を与え、APIを安全で下位互換性のある方法で進化させることを可能にします。 さらに、これはPython言語を既存の文書およびさまざまな「組み込み」関数と標準ライブラリ関数の動作とより一貫性のあるものにします。

^^^^^^^^^^^^^^^^^^^^^^^^^^
Empowering Library Authors
^^^^^^^^^^^^^^^^^^^^^^^^^^

Library authors would have the flexibility to change the name of
positional-only parameters without breaking callers. This flexibility reduces the
cognitive burden for choosing an appropriate public-facing name for required
parameters or parameters that have no true external semantic meaning.

ライブラリの作者は、呼び出し側を壊すことなく、位置のみのパラメータの名前を変更する柔軟性を持っているでしょう。 この柔軟性により、必要なパラメータ、または実際の外部的な意味を持たないパラメータに適切な一般向けの名前を選択するための認知的な負担が軽減されます。

Positional-only parameters are useful in several situations such as

一限定引数は様々な状況で役立ちます:

* when a function accepts any keyword argument but also can accept a positional one

  関数がキーワード引数を受け付けるが、位置引数も受け付けることができる場合

* when a parameter has no external semantic meaning

  パラメータに外部的な意味がない場合

* when an API's parameters are required and unambiguous

  APIのパラメータが必要で明確である場合

A key
scenario is when a function accepts any keyword argument but can also accepts a
positional one. Prominent examples are ``Formatter.format`` and
``dict.update``. For instance, ``dict.update`` accepts a dictionary
(positionally), an iterable of key/value pairs (positionally), or multiple
keyword arguments. In this scenario, if the dictionary parameter were not
positional-only, the user could not use the name that the function definition
uses for the parameter or, conversely, the function could not distinguish
easily if the argument received is the dictionary/iterable or a keyword
argument for updating the key/value pair.

重要なシナリオは、関数が任意のキーワード引数を受け入れるが、位置引数も受け入れることができる場合です。 有名な例は ``Formatter.format`` と ``dict.update`` です。 例えば、 ``dict.update`` は辞書を（位置的に）、繰り返し可能なキー/値のペア（位置的に）、あるいは複数のキーワード引数を受け入れます。 このシナリオでは、ディクショナリパラメータが位置のみではない場合、ユーザは関数定義がパラメータに使用する名前を使用できないか、または逆に、受け取った引数がディクショナリ/反復可能オブジェクトであるか、あるいはキー/値ペアを更新するためのキーワード引数かを簡単に区別できません。

Another scenario where positional-only parameters are useful is when the
parameter name has no true external semantic meaning. For example, let's say
we want to create a function that converts from one type to another

位置限定引数が便利なもう1つのシナリオは、パラメータ名に外部の意味的な意味がまったくない場合です。 たとえば、ある型から別の型に変換する関数を作成したいとしましょう。::

    def as_my_type(x):
        ...

The name of the parameter provides no intrinsic value and forces the API author
to maintain its name forever since callers might pass ``x`` as a keyword
argument.

呼び出し元がキーワード引数として ``x`` を渡すかもしれないので、パラメータの名前は本質的な値を提供せず、API作成者にその名前を永遠に維持することを強制します。

Additionally, positional-only parameters are useful when an API's parameters
are required and is unambiguous with respect to function. For example

さらに、位置のみのパラメータは、APIのパラメータが必要で、機能に関して明確である場合に役立ちます。 例えば::

    def add_to_queue(item: QueueItem):
        ...

The name of the function makes clear the argument expected. A keyword
argument provides minimal benefit and also limits the future evolution of the
API. Say at a later time we want this function to be able to take multiple
items, while preserving backwards compatibility

関数の名前によって、予想される引数が明確になります。 キーワード引数は最小限の利点を提供し、またAPIの将来の進化を制限します。 後で言うと、後方互換性を保ちながら、この関数が複数の項目を受け取ることができるようにします。::

    def add_to_queue(items: Union[QueueItem, List[QueueItem]]):
        ...

or to take them by using argument lists

または引数リストを使用してそれらを受け取る::

    def add_to_queue(*items: QueueItem):
        ...

the author would be forced to always keep the original parameter name to avoid
potentially breaking callers.

作成者は、潜在的な呼び出し元の呼び出しを回避するために、常に元のパラメータ名を保持するように強制されます。

By being able to specify positional-only parameters, an author can change the
name of the parameters freely or even change them to ``*args``, as seen in the
previous example. There are multiple function definitions in the standard
library which fall into this category. For example, the required parameter to
``collections.defaultdict`` (called *default_factory* in its documentation) can
only be passed positionally. One special case of this situation is the *self*
parameter for class methods: it is undesirable that a caller can bind by
keyword to the name ``self`` when calling the method from the class

位置のみのパラメータを指定できるようにすることで、作者は前の例で見たように、パラメータの名前を自由に変更したり、あるいは ``*args`` に変更することさえできます。 このカテゴリに分類される標準ライブラリには複数の関数定義があります。 たとえば、 ``collections.defaultdict`` に必要なパラメータ（そのドキュメントでは *default_factory* と呼ばれています）は、位置的にしか渡すことができません。 このような状況の特別な場合の1つは、クラスメソッドの *self* パラメータです。クラスからメソッドを呼び出すときに、呼び出し側がキーワードで ``self`` という名前にバインドできることは望ましくありません。::

    io.FileIO.write(self=f, b=b"data")

Indeed, function definitions from the standard library implemented in C usually
take ``self`` as a positional-only parameter

確かに、Cで実装された標準ライブラリの関数定義は通常位置限定引数として ``self`` を取ります::

    >>> help(io.FileIO.write)
    Help on method_descriptor:

    write(self, b, /)
        Write buffer b to file, return number of bytes written.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Improving Language Consistency
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Python language would be more consistent with positional-only
parameters. If the concept is a normal feature of Python rather than a feature
exclusive to extension modules, it would reduce confusion for users
encountering functions with positional-only parameters. Some major
third-party packages are already using the ``/`` notation in their function
definitions [#numpy-ufuncs]_ [#scipy-gammaln]_.

Python言語は位置限定引数とより一貫性があります。 概念が拡張モジュール専用の機能ではなく、Pythonの通常の機能である場合は、位置のみのパラメータを持つ関数に遭遇したユーザーの混乱を減らすでしょう。 いくつかの主要なサードパーティ製パッケージはすでにそれらの関数定義で ``/`` 表記を使っています

Bridging the gap found between "builtin" functions which
specify positional-only parameters and pure Python implementations that lack
the positional syntax would improve consistency. The ``/`` syntax is already exposed
in the existing documentation such as when builtins and interfaces are generated
by the argument clinic.

位置限定引数を指定する "組み込み" 関数と位置指定構文を欠く純粋なPython実装との間にあるギャップを埋めることは一貫性を改善するでしょう。 ``/`` 構文は、組み込み関数やインタフェースが引数clinicによって生成されるときなど、既存の文書ですでに公開されています。

Another essential aspect to consider is PEP 399, which mandates that
pure Python versions of modules in the standard library *must* have the same
interface and semantics that the accelerator modules implemented in C. For
example, if ``collections.defaultdict`` were to have a pure Python
implementation it would need to make use of positional-only parameters to match
the interface of its C counterpart.

考慮すべきもう1つの重要な側面はPEP 399です。これは標準ライブラリのモジュールの純粋なPythonバージョンがCでインプリメントされたアクセラレータモジュールと同じインタフェースとセマンティクスを持たなければならないことを強制します。例えば、 ``collections.defaultdict`` は 純粋なPythonの実装を持つためには、そのCの対応する部分のインターフェースに合わせるために位置のみのパラメータを利用する必要があるでしょう。

=========
Rationale
=========

We propose to introduce positional-only parameters as a new syntax to the
Python language.

Python言語の新しい構文として、位置限定引数を導入することを提案します。

The new syntax will enable library authors to further control how their API
can be called. It will allow designating which parameters must be called as
positional-only, while preventing them from being called as keyword arguments.

新しい構文により、ライブラリ作成者は自分のAPIを呼び出す方法をさらに制御できます。 キーワード引数として呼び出されるのを防ぎながら、どのパラメーターを位置限定引数として呼び出す必要があるかを指定することができます。

Previously, (informational) PEP 457 defined the syntax, but with a much more vague
scope. This PEP takes the original proposal a step further by justifying
the syntax and providing an implementation for the ``/`` syntax in function
definitions.

以前は、（情報）PEP 457が構文を定義しましたが、もっと曖昧な範囲です。 このPEPは、構文を正当化し、関数定義の ``/`` 構文の実装を提供することによって、元の提案をさらに一歩進めます。

-----------
Performance
-----------

In addition to the aforementioned benefits, the parsing and handling of
positional-only arguments is faster. This performance benefit can be
demonstrated in this thread about converting keyword arguments to positional:
[#thread-keyword-to-positional]_. Due to this speedup, there has been a recent
trend towards moving builtins away from keyword arguments: recently,
backwards-incompatible changes were made to disallow keyword arguments to
``bool``, ``float``, ``list``, ``int``, ``tuple``.

前述の利点に加えて、位置のみの引数の解析と処理が高速になりました。 このパフォーマンス上の利点は、このスレッドでキーワード引数を位置指定に変換することについて説明できます。 [#thread-keyword-to-positional]_ 。このスピードアップのため、最近ではキーワード引数からビルトインを移動する傾向があります。最近では、キーワード引数を ``bool`` 、 ``float`` 、 ``list`` にしないように後方互換性のない変更が行われました。 ``int`` 、 ``tuple`` 。

---------------
Maintainability
---------------

Providing a way to specify positional-only parameters in Python will make it
easier to maintain pure Python implementations of C modules. Additionally,
library authors defining functions will have the choice for choosing
positional-only parameters if they determine that passing a keyword argument
provides no additional clarity.

Pythonで位置のみのパラメータを指定する方法を提供することは、Cモジュールの純粋なPython実装を維持することをより簡単にするでしょう。 さらに、関数を定義するライブラリ作成者は、キーワード引数を渡してもそれ以上明確になることがないと判断した場合には、位置限定引数を選択することができます。

This is a well discussed, recurring topic on the Python mailing lists

これはPythonのメーリングリストでよく議論されている繰り返しのトピックです。:

* September 2018: `Anders Hovmöller: [Python-ideas] Positional-only
  parameters
  <https://mail.python.org/pipermail/python-ideas/2018-September/053233.html>`_
* February 2017: `Victor Stinner: [Python-ideas] Positional-only
  parameters
  <https://mail.python.org/pipermail/python-ideas/2017-February/044879.html>`_,
  `discussion continued in March
  <https://mail.python.org/pipermail/python-ideas/2017-March/044956.html>`_
* February 2017: [#python-ideas-decorator-based]_
* March 2012: [#GUIDO]_
* May 2007: `George Sakkis: [Python-ideas] Positional only arguments
  <https://mail.python.org/pipermail/python-ideas/2007-May/000704.html>`_
* May 2006: `Benji York: [Python-Dev] Positional-only Arguments
  <https://mail.python.org/pipermail/python-dev/2006-May/064790.html>`_

----------------
Logical ordering
----------------

Positional-only parameters also have the (minor) benefit of enforcing some
logical order when calling interfaces that make use of them. For example, the
``range`` function takes all its parameters positionally and disallows forms
like

位置限定引数には、それを利用するインタフェースを呼び出すときに論理的な順序を強制するという（マイナーな）利点もあります。 たとえば、 ``range`` 関数はすべてのパラメータを位置的に取り、以下のような形式を許可しません。::

    range(stop=5, start=0, step=2)
    range(stop=5, step=2, start=0)
    range(step=2, start=0, stop=5)
    range(step=2, stop=5, start=0)

at the price of disallowing the use of keyword arguments for the (unique)
intended order

（一意の）意図された順序でのキーワード引数の使用を許可しないという犠牲を払って::

    range(start=0, stop=5, step=2)

-------------------------------------------
Compatibility for Pure Python and C Modules
-------------------------------------------

Another critical motivation for positional-only parameters is PEP 399:
Pure Python/C Accelerator Module Compatibility Requirements. This
PEP states that

位置限定引数のもう1つの重要な動機は、PEP 399：Pure Python / C Acceleratorモジュールの互換性要件です。 このPEPは、:

    This PEP requires that in these instances that the C code must pass the
    test suite used for the pure Python code to act as much as a drop-in
    replacement as reasonably possible

    このPEPでは、これらのインスタンスでは、Cコードが純粋なPythonコードに使用されるテストスイートに合格し、合理的に可能な限りドロップイン置換として機能する必要があります。

If the C code is implemented using the existing capabilities
to implement positional-only parameters using the argument clinic, and related
machinery, it is not possible for the pure Python counterpart to match the
provided interface and requirements. This creates a disparity between the
interfaces of some functions and classes in the CPython standard library and
other Python implementations. For example

引数clinicを使用して位置限定引数を実装する既存の機能、および関連する機構を使用してCコードを実装する場合、純粋なPythonの対応するものが提供されるインタフェースおよび要件を満たすことは不可能です。 これにより、CPython標準ライブラリ内の一部の関数とクラスのインターフェースと他のPython実装との間に格差が生じます。 例えば::

    $ python3 # CPython 3.7.2
    >>> import binascii; binascii.crc32(data=b'data')
    TypeError: crc32() takes no keyword arguments

    $ pypy3 # PyPy 6.0.0
    >>>> import binascii; binascii.crc32(data=b'data')
    2918445923

Other Python implementations can reproduce the CPython APIs manually, but this
goes against the spirit of PEP 399 to avoid duplication of effort by
mandating that all modules added to Python's standard library **must** have a
pure Python implementation with the same interface and semantics.

他のPython実装は手動でCPython APIを再現することができますが、これはPEP 399の精神に反し、Pythonの標準ライブラリに追加されるすべてのモジュールが同じインタフェースと意味を持つ純粋なPython実装を持つことを強制することによって努力の重複を避けるためです 。

-------------------------
Consistency in Subclasses
-------------------------

Another scenario where positional-only parameters provide benefit occurs when a
subclass overrides a method of the base class and changes the name of parameters
that are intended to be positional

位置限定引数が利点を提供するもう1つのシナリオは、サブクラスが基本クラスのメソッドをオーバーライドし、位置指定を目的としているパラメーターの名前を変更した場合に発生します。::

    class Base:
        def meth(self, arg: int) -> str:
            ...

    class Sub(Base):
        def meth(self, other_arg: int) -> str:
            ...

    def func(x: Base):
        x.meth(arg=12)

    func(Sub())  # Runtime error

This situation could be considered a Liskov violation — the subclass cannot be
used in a context when an instance of the base class is expected. Renaming
arguments when overloading methods can happen when the subclass has reasons to
use a different choice for the parameter name that is more appropriate for the
specific domain of the subclass (e.g., when subclassing ``Mapping`` to
implement a DNS lookup cache, the derived class may not want to use the generic
argument names ‘key’ and ‘value’ but rather ‘host’ and ‘address’). Having this
function definition with positional-only parameters can avoid this problem
because users will not be able to call the interface using keyword arguments.
In general, designing for subclassing usually involves anticipating code that
hasn't been written yet and over which the author has no control. Having
measures that can facilitate the evolution of interfaces in a
backwards-compatible would be useful for library authors.

この状況は、リスコフ違反と見なされる可能性があります。基本クラスのインスタンスが予想される場合、サブクラスをコンテキスト内で使用することはできません。 サブクラスがサブクラスの特定のドメインにより適したパラメータ名に異なる選択を使用する理由がある場合（例えばDNSルックアップキャッシュを実装するために ``Mapping`` をサブクラス化するときなど） 派生クラスは一般的な引数名 'key' と 'value' を使いたくないかもしれませんが、むしろ 'host' と 'address' を使います。 位置のみのパラメータでこの関数を定義すると、ユーザがキーワード引数を使用してインタフェースを呼び出すことができなくなるため、この問題を回避できます。 一般的に、サブクラス化のための設計は通常、まだ書かれていないコードや、作者が制御できないコードを予想することを伴います。 後方互換性のあるインターフェースの進化を容易にすることができる手段を持つことは、ライブラリの作者にとって有用でしょう。

-------------
Optimizations
-------------

A final argument in favor of positional-only parameters is that they allow some
new optimizations like the ones already present in the argument clinic due to
the fact that parameters are expected to be passed in strict order. For example, CPython's
internal ``METH_FASTCALL`` calling convention has been recently specialized for
functions with positional-only parameters to eliminate the cost for handling
empty keywords. Similar performance improvements can be applied when creating
the evaluation frame of Python functions thanks to positional-only parameters.

位置限定引数を支持する最後の議論は、パラメータが厳密な順序で渡されることが期待されるという事実のために、引数clinicに既に存在するもののようないくつかの新しい最適化を可能にするということです。 たとえば、CPythonの内部の「METH_FASTCALL」呼び出し規約は、空のキーワードを処理するためのコストを削減するために、位置のみのパラメータを持つ関数に特化しました。 位置限定引数のおかげで、Python関数の評価フレームを作成するときにも、同様のパフォーマンスの向上を適用できます。

=============
Specification
=============

--------------------
Syntax and Semantics
--------------------

From the "ten-thousand foot view", eliding ``*args`` and ``**kwargs`` for
illustration, the grammar for a function definition would look like

説明のために ``*args`` と ``**kwargs`` を省略した「1万フィートビュー」から、関数定義の文法は次のようになります。::

    def name(positional_or_keyword_parameters, *, keyword_only_parameters):

Building on that example, the new syntax for function definitions would look
like

その例を基にすると、関数定義の新しい構文は次のようになります。::

    def name(positional_only_parameters, /, positional_or_keyword_parameters,
             *, keyword_only_parameters):

The following would apply:

* All parameters left of the ``/`` are treated as positional-only.

  ``/`` の左側にあるすべてのパラメータは位置限定として扱われます。

* If ``/`` is not specified in the function definition, that function does not
  accept any positional-only arguments.

  関数定義で ``/`` が指定されていない場合、その関数は位置限定引数を受け入れません

* The logic around optional values for positional-only parameters remains the
  same as for positional-or-keyword parameters.

  位置限定引数のオプション値の周りの論理は、位置指定キーワードまたはキーワードパラメーターの場合と同じです

* Once a positional-only parameter is specified with a default, the
  following positional-only and positional-or-keyword parameters need to have
  defaults as well.

  位置限定引数をデフォルトで指定した後は、それに続く位置限定引数および位置指定またはキーワードのパラメーターにもデフォルト値を設定する必要があります。

* Positional-only parameters which do not have default
  values are *required* positional-only parameters.

  デフォルト値を持たない位置限定引数は、 *必須* 位置限定引数です。

Therefore the following would be valid function definitions

したがって、以下は有効な関数定義です。::

    def name(p1, p2, /, p_or_kw, *, kw):
    def name(p1, p2=None, /, p_or_kw=None, *, kw):
    def name(p1, p2=None, /, *, kw):
    def name(p1, p2=None, /):
    def name(p1, p2, /, p_or_kw):
    def name(p1, p2, /):

Just like today, the following would be valid function definitions

今日と同じように、以下は有効な関数定義です。::

    def name(p_or_kw, *, kw):
    def name(*, kw):

While the following would be invalid

以下は無効になりますが::

    def name(p1, p2=None, /, p_or_kw, *, kw):
    def name(p1=None, p2, /, p_or_kw=None, *, kw):
    def name(p1=None, p2, /):

--------------------------
Full Grammar Specification
--------------------------

A simplified view of the proposed grammar specification is

提案された文法仕様の簡略図は、::

    typedargslist:
      tfpdef ['=' test] (',' tfpdef ['=' test])* ',' '/' [','  # and so on

    varargslist:
      vfpdef ['=' test] (',' vfpdef ['=' test])* ',' '/' [','  # and so on

Based on the reference implementation in this PEP, the new rule for
``typedarglist`` would be

このPEPの参照実装に基づくと、 ``typedarglist`` の新しい規則は次のようになります。::

    typedargslist: (tfpdef ['=' test] (',' tfpdef ['=' test])* ',' '/' [',' [tfpdef ['=' test] (',' tfpdef ['=' test])* [',' [
            '*' [tfpdef] (',' tfpdef ['=' test])* [',' ['**' tfpdef [',']]]
          | '**' tfpdef [',']]]
      | '*' [tfpdef] (',' tfpdef ['=' test])* [',' ['**' tfpdef [',']]]
      | '**' tfpdef [',']] ] )| (
       tfpdef ['=' test] (',' tfpdef ['=' test])* [',' [
            '*' [tfpdef] (',' tfpdef ['=' test])* [',' ['**' tfpdef [',']]]
          | '**' tfpdef [',']]]
     | '*' [tfpdef] (',' tfpdef ['=' test])* [',' ['**' tfpdef [',']]]
     | '**' tfpdef [','])

and for ``varargslist`` would be

そして ``varargslist`` の場合は::

    varargslist: vfpdef ['=' test ](',' vfpdef ['=' test])* ',' '/' [',' [ (vfpdef ['=' test] (',' vfpdef ['=' test])* [',' [
            '*' [vfpdef] (',' vfpdef ['=' test])* [',' ['**' vfpdef [',']]]
          | '**' vfpdef [',']]]
      | '*' [vfpdef] (',' vfpdef ['=' test])* [',' ['**' vfpdef [',']]]
      | '**' vfpdef [',']) ]] | (vfpdef ['=' test] (',' vfpdef ['=' test])* [',' [
            '*' [vfpdef] (',' vfpdef ['=' test])* [',' ['**' vfpdef [',']]]
          | '**' vfpdef [',']]]
      | '*' [vfpdef] (',' vfpdef ['=' test])* [',' ['**' vfpdef [',']]]
      | '**' vfpdef [',']
    )

--------------------
Semantic Corner Case
--------------------

The following is an interesting corollary of the specification.
Consider this function definition

以下は仕様の興味深い推論です。 この関数定義を検討する::

    def foo(name, **kwds):
        return 'name' in kwds

There is no possible call that will make it return ``True``.
For example

Trueを返すような呼び出しはありません。例えば::

    >>> foo(1, **{'name': 2})
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: foo() got multiple values for argument 'name'
    >>>

But using ``/`` we can support this

しかし ``/`` を使うことでこれをサポートできます::

    def foo(name, /, **kwds):
        return 'name' in kwds

Now the above call will return ``True``.

これで上記の呼び出しは ``True`` を返します。

In other words, the names of positional-only parameters can be used in
``**kwds`` without ambiguity.  (As another example, this benefits the
signatures of ``dict()`` and ``dict.update()``.)

言い換えれば、位置のみのパラメータの名前はあいまいさなしに ``**kwds`` で使うことができます。 （別の例として、これは ``dict()`` と ``dict.update()`` のシグネチャに役立ちます。）

----------------------------
Origin of "/" as a Separator
----------------------------

Using ``/`` as a separator was initially proposed by Guido van Rossum
in 2012 [#GUIDO]_ 

区切り文字として ``/`` を使うことは、2012年にGuido van Rossumによって最初に提案されました。:

    Alternative proposal: how about using '/' ? It's kind of the opposite
    of '*' which means "keyword argument", and '/' is not a new character.

    別の提案: '/' を使ってはどうですか？ これは「キーワード引数」を意味する「*」とは反対のことで、「/」は新しい文字ではありません。

=================
How To Teach This
=================

Introducing a dedicated syntax to mark positional-only parameters is closely
analogous to existing keyword-only arguments. Teaching these concepts together
may *simplify* how to teach the possible function definitions a user may encounter or
design.

位置のみのパラメータをマークするための専用の構文を導入することは、既存のキーワードのみの引数とよく似ています。 これらの概念を一緒に教えることは、ユーザが遭遇したり設計したりする可能性のある機能定義を教える方法を *単純化* することができます。

This PEP recommends adding a new subsection to the Python documentation, in the
section `"More on Defining Functions"`_, where the rest of the argument types
are discussed. The following paragraphs serve as a draft for these additions.
They will introduce the notation for both positional-only and
keyword-only parameters. It is not intended to be exhaustive, nor should it be
considered the final version to be incorporated into the documentation.

このPEPでは、Pythonドキュメントに新しいサブセクションを追加することをお勧めします。セクション「その他の引数タイプの定義」セクションで、残りの引数タイプについて説明します。 以下の段落は、これらの追加の草案として役立ちます。 それらは、位置限定引数とキーワードのみのパラメータの両方の表記法を紹介します。 それは徹底的であることを意図されていません、そしてそれはドキュメンテーションに組み込まれるべき最終版とみなされるべきでもありません。

.. _"More on Defining Functions": https://docs.python.org/3.7/tutorial/controlflow.html#more-on-defining-functions

-------------------------------------------------------------------------------

By default, arguments may be passed to a Python function either by position
or explicitly by keyword. For readability and performance, it makes sense to
restrict the way arguments can be passed so that a developer need only look
at the function definition to determine if items are passed by position, by
position or keyword, or by keyword.

デフォルトでは、引数は位置によって、またはキーワードによって明示的にPython関数に渡されます。 読みやすさとパフォーマンスのために、引数が渡される方法を制限することは理にかなっているので、開発者は項目が位置によって、位置またはキーワードによって、またはキーワードによって渡されるかどうかを判断するために関数定義を見るだけです。

A function definition may look like

関数定義は次のようになります。::

   def f(pos1, pos2, /, pos_or_kwd, *, kwd1, kwd2):
         -----------    ----------     ----------
           |             |                  |
           |        Positional or keyword   |
           |                                - Keyword only
            -- Positional only

where ``/`` and ``*`` are optional. If used, these symbols indicate the kind of
parameter by how the arguments may be passed to the function:
positional-only, positional-or-keyword, and keyword-only. Keyword parameters
are also referred to as named parameters.

ここで、 ``/`` と ``*`` はオプションです。 使用する場合、これらの記号は、引数を関数に渡す方法によってパラメーターの種類を示します。位置のみ、位置またはキーワード、およびキーワードのみです。 キーワードパラメータは、名前付きパラメータとも呼ばれます。

-------------------------------
Positional-or-Keyword Arguments
-------------------------------

If ``/`` and ``*`` are not present in the function definition, arguments may
be passed to a function by position or by keyword.

``/`` と ``*`` が関数定義に存在しない場合、引数は位置またはキーワードによって関数に渡されます。

--------------------------
Positional-Only Parameters
--------------------------

Looking at this in a bit more detail, it is possible to mark certain parameters
as *positional-only*. If *positional-only*, the parameters' order matters, and
the parameters cannot be passed by keyword. Positional-only parameters would
be placed before a ``/`` (forward-slash). The ``/`` is used to logically
separate the positional-only parameters from the rest of the parameters.
If there is no ``/`` in the function definition, there are no positional-only
parameters.

もう少し詳しく見てみると、特定のパラメータを*位置のみ*とマークすることができます。 *位置限定*の場合、引数の順序が重要であり、パラメータをキーワードで渡すことはできません。 位置のみのパラメータは ``/`` （スラッシュ）の前に置かれます。 ``/`` は、位置のみのパラメータを他のパラメータから論理的に分離するために使用されます。 関数定義に ``/`` がない場合、位置指定専用のパラメータはありません。

Parameters following the ``/`` may be *positional-or-keyword* or *keyword-only*.

``/``の後に続く引数は *位置またはキーワード* または *キーワードのみ* です。

----------------------
Keyword-Only Arguments
----------------------

To mark parameters as *keyword-only*, indicating the parameters must be passed
by keyword argument, place an ``*`` in the arguments list just before the first
*keyword-only* parameter.

パラメータを *keyword-only* としてマークし、パラメータをkeyword引数で渡す必要があることを示すには、最初の *keyword-only* パラメータの直前にある引数リストの中に ``*`` を置きます。

-----------------
Function Examples
-----------------

Consider the following example function definitions paying close attention to the
markers ``/`` and ``*``

マーカ ``/`` と ``*`` に細心の注意を払って、以下の関数定義例を検討してください。::

   >>> def standard_arg(arg):
   ...     print(arg)
   ...
   >>> def pos_only_arg(arg, /):
   ...     print(arg)
   ...
   >>> def kwd_only_arg(*, arg):
   ...     print(arg)
   ...
   >>> def combined_example(pos_only, /, standard, *, kwd_only):
   ...     print(pos_only, standard, kwd_only)


The first function definition ``standard_arg``, the most familiar form,
places no restrictions on the calling convention and arguments may be
passed by position or keyword

最も一般的な形式である最初の関数定義 ``standard_arg`` は呼び出し規約に制限を設けず、引数は位置またはキーワードで渡すことができます::

   >>> standard_arg(2)
   2

   >>> standard_arg(arg=2)
   2

The second function ``pos_only_arg` is restricted to only use positional
parameters as there is a ``/`` in the function definition

2番目の関数 ``pos_only_arg`` は、関数定義に ``/`` があるので位置パラメータのみを使うように制限されています::

   >>> pos_only_arg(1)
   1

   >>> pos_only_arg(arg=1)
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   TypeError: pos_only_arg() got an unexpected keyword argument 'arg'

The third function ``kwd_only_args`` only allows keyword arguments as indicated
by a ``*`` in the function definition

3番目の関数 ``kwd_only_args`` は関数定義の ``*`` で示されるようにキーワード引数のみを許可します::

   >>> kwd_only_arg(3)
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   TypeError: kwd_only_arg() takes 0 positional arguments but 1 was given

   >>> kwd_only_arg(arg=3)
   3

And the last uses all three calling conventions in the same function
definition

And the last uses all three calling conventions in the same function definition::

   >>> combined_example(1, 2, 3)
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   TypeError: combined_example() takes 2 positional arguments but 3 were given

   >>> combined_example(1, 2, kwd_only=3)
   1 2 3

   >>> combined_example(1, standard=2, kwd_only=3)
   1 2 3

   >>> combined_example(pos_only=1, standard=2, kwd_only=3)
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   TypeError: combined_example() got an unexpected keyword argument 'pos_only'

-----
Recap
-----

The use case will determine which parameters to use in the function definition

ユースケースは、関数定義で使用するパラメータを決定します::

   def f(pos1, pos2, /, pos_or_kwd, *, kwd1, kwd2):

As guidance:

* Use positional-only if names do not matter or have no meaning, and there are
  only a few arguments which will always be passed in the same order.

  名前が重要でも意味も持たず、常に同じ順序で渡される引数がいくつかある場合は、位置のみを使用します。

* Use keyword-only when names have meaning and the function definition is
  more understandable by being explicit with names.

  名前に意味があり、名前で明示的にすることで関数定義がより理解しやすい場合は、キーワードのみを使用してください。

========================
Reference Implementation
========================

An initial implementation that passes the CPython test suite is available for
evaluation [#posonly-impl]_.

CPythonテストスイートに合格する最初の実装は評価のために利用可能です

The benefits of this implementations are speed of handling positional-only
parameters, consistency with the implementation of keyword-only parameters (PEP
3102), and a simpler implementation of all the tools and modules that would be
impacted by this change.

この実装の利点は、位置のみのパラメータの処理速度、キーワードのみのパラメータの実装との一貫性（PEP 3102）、およびこの変更の影響を受けるすべてのツールとモジュールのより簡単な実装です。

==============
Rejected Ideas
==============

----------
Do Nothing
----------

Always an option — the status quo. While this was considered, the
aforementioned benefits are worth the addition to the language.

常にオプション - 現状維持。 これは考慮されましたが、前述の利点は言語に追加する価値があります。

----------
Decorators
----------

It has been suggested on python-ideas [#python-ideas-decorator-based]_ to
provide a decorator written in Python for this feature.

この機能のためにPythonで書かれたデコレータを提供することがpython-ideas [#python-ideas-decorator-based]_ で提案されています。

This approach has the benefit of not polluting function definition with
additional syntax. However, we have decided to reject this idea because

このアプローチには、追加の構文で関数定義を汚染しないという利点があります。 しかしながら、我々はこの考えを棄却することにした。:

* It introduces an asymmetry with how parameter behavior is declared.

  それはパラメータの振る舞いがどのように宣言されているかという非対称性を導入します。

* It makes it difficult for static analyzers and type checkers to
  safely identify positional-only parameters.  They would need to query the AST
  for the list of decorators and identify the correct one by name or with extra
  heuristics, while keyword-only parameters are exposed
  directly in the AST.  In order for tools to correctly identify
  positional-only parameters, they would need to execute the module to access
  any metadata the decorator is setting.

  静的アナライザや型チェッカーが位置のみのパラメータを安全に識別することは困難です。 キーワードのみのパラメータはASTで直接公開されていますが、それらはASTにデコレータのリストを問い合わせ、正しいものを名前または追加のヒューリスティックで識別する必要があります。 ツールが位置のみのパラメータを正しく識別するためには、デコレータが設定しているメタデータにアクセスするためにモジュールを実行する必要があります。

* Any error with the declaration will be reported only at runtime.

  宣言に関するエラーは実行時にのみ報告されます。

* It may be more difficult to identify positional-only parameters in long
  function definitions, as it forces the user to count them to know which is
  the last one that is impacted by the decorator.

  It may be more difficult to identify positional-only parameters in long function definitions, as it forces the user to count them to know which is the last one that is impacted by the decorator.

* The ``/`` syntax has already been introduced for C functions. This
  inconsistency will make it more challenging to implement any tools and
  modules that deal with this syntax — including but not limited to, the
  argument clinic, the inspect module and the ``ast`` module.

  ``/`` 構文はすでにC関数に導入されています。 この矛盾は、引数クリニック、検査モジュール、 ``ast`` モジュールを含むがこれらに限定されない、このシンタックスを扱うあらゆるツールやモジュールを実装することをより困難にするでしょう。

* The decorator implementation would likely impose a runtime performance cost,
  particularly when compared to adding support directly to the interpreter.

  デコレータの実装は、特にインタプリタに直接サポートを追加するのと比較した場合、実行時のパフォーマンスコストがかかる可能性があります。


-------------------
Per-Argument Marker
-------------------

A per-argument marker is another language-intrinsic option. The approach adds
a token to each of the parameters to indicate they are positional-only and
requires those parameters to be placed together. Example

引数ごとのマーカーは、他の言語固有のオプションです。 このアプローチでは、位置指定のみであることを示すために各パラメーターにトークンを追加し、それらのパラメーターをまとめて配置する必要があります。 例::

  def (.arg1, .arg2, arg3):

Note the dot (i.e., ``.``) on ``.arg1`` and ``.arg2``. While this approach
may be easier to read, it has been rejected because ``/`` as an explicit marker
is congruent with ``*`` for keyword-only arguments and is less error-prone.

``.arg1`` と ``.arg2`` 上のドット（つまり ``.`` ）に注意してください。 このアプローチは読みやすいかもしれませんが、明示的なマーカーとしての ``/`` はキーワードのみの引数に対する ``*`` と一致し、エラーが発生しにくいため、拒否されています。

It should be noted that some libraries already use leading underscore
[#leading-underscore]_ to conventionally indicate parameters as positional-only.

注意すべき点は、いくつかのライブラリでは、従来からパラメータを位置のみのものとして示すために、先頭のアンダースコア [#leading-underscore]_ がすでに使用されていることです。

-----------------------------------
Using "__" as a Per-Argument Marker
-----------------------------------

Some libraries and applications (like ``mypy`` or ``jinja``) use names
prepended with a double underscore (i.e., ``__``) as a convention to indicate
positional-only parameters. We have rejected the idea of introducing ``__`` as
a new syntax because:

いくつかのライブラリやアプリケーション（ ``mypy`` や ``jinja`` のように）は名前を使います
位置のみのパラメータを示す規約として、二重下線（つまり、 ``__`` ）を前に付けます。 以下の理由により、新しい構文として ``__`` を導入するという考えを拒否しました。

* It is a backwards-incompatible change.

  それは後方互換性のない変更です。

* It is not symmetric with how the keyword-only parameters are currently
  declared.

  キーワードのみのパラメータが現在宣言されている方法と対称的ではありません。

* Querying the AST for positional-only parameters would require checking the
  normal arguments and inspecting their names, whereas keyword-only parameters
  have a property associated with them (``FunctionDef.args.kwonlyargs``).

  位置限定引数についてASTを問い合わせるには通常の引数を調べてその名前を調べる必要がありますが、キーワードのみのパラメータにはそれに関連したプロパティがあります（ ``FunctionDef.args.kwonlyargs`` ）。

* Every parameter would need to be inspected to know when positional-only
  arguments end.

  すべてのパラメーターは、位置のみの引数がいつ終了するかを知るために検査する必要があります。

* The marker is more verbose, forcing marking every positional-only parameter.

  マーカーはより冗長で、すべての位置のみのパラメーターを強制的にマークします。

* It clashes with other uses of the double underscore prefix like invoking name
  mangling in classes.

  クラス内で名前マングリングを呼び出すなど、他の二重下線プレフィックスの使用と衝突します。

-------------------------------------------------
Group Positional-Only Parameters With Parentheses
-------------------------------------------------

Tuple parameter unpacking is a Python 2 feature which allows the use of a tuple
as a parameter in a function definition. It allows a sequence argument to be
unpacked automatically. An example is

タプルパラメータの展開はPython 2の機能で、関数定義のパラメータとしてタプルを使うことができます。 シーケンス引数を自動的に解凍することができます。 例は::

    def fxn(a, (b, c), d):
        pass

Tuple argument unpacking was removed in Python 3 (PEP 3113). There has been a
proposition to reuse this syntax to implement positional-only parameters. We
have rejected this syntax for indicating positional only parameters for several
reasons

タプル引数の展開はPython 3で削除されました（PEP 3113）。 位置限定引数を実装するためにこの構文を再利用するという提案がありました。 いくつかの理由で、位置のみのパラメータを示すためにこの構文を拒否しました:

* The syntax is asymmetric with respect to how keyword-only parameters are
  declared.

  キーワードのみのパラメータの宣言方法に関して、構文は非対称です。

* Python 2 uses this syntax which could raise confusion regarding the behavior
  of this syntax. This would be surprising to users porting Python 2 codebases
  that were using this feature.

  Python 2はこの構文を使用しているため、この構文の動作に関して混乱を招く可能性があります。 この機能を使用していたPython 2コードベースを移植するユーザーにとっては、これは驚くべきことです。

* This syntax is very similar to tuple literals. This can raise additional
  confusion because it can be confused with a tuple declaration.

  この構文はタプルリテラルと非常によく似ています。 これはタプル宣言と混同される可能性があるため、さらに混乱を招く可能性があります。

------------------------
After Separator Proposal
------------------------

Marking positional-parameters after the ``/`` was another idea considered.
However, we were unable to find an approach which would modify the arguments
after the marker. Otherwise, would force the parameters before the marker to
be positional-only as well. For example

``/`` の後に位置引数をマークすることも考えられていました。 しかし、マーカーの後の引数を変更するようなアプローチを見つけることができませんでした。 それ以外の場合は、マーカーの前のパラメーターも位置のみになります。 例えば::

  def (x, y, /, z):

If we define that ``/`` marks ``z`` as positional-only, it would not be
possible to specify ``x`` and ``y`` as keyword arguments. Finding a way to
work around this limitation would add confusion given that at the moment
keyword arguments cannot be followed by positional arguments. Therefore, ``/``
would make both the preceding and following parameters positional-only.

``/`` が ``z`` を位置限定としてマークすると定義した場合、キーワード引数として ``x`` と ``y`` を指定することは不可能です。現時点ではキーワード引数の後に位置引数を続けることはできないため、この制限を回避する方法を見つけることは混乱を招くでしょう。 したがって、 ``/`` は前後のパラメータの両方を位置のみにします。

======
Thanks
======

Credit for some of the content of this PEP is contained in Larry Hastings’s
PEP 457.

Credit for the use of ``/`` as the separator between positional-only and
positional-or-keyword parameters go to Guido van Rossum, in a proposal from
2012. [#GUIDO]_

Credit for discussion about the simplification of the grammar goes to
Braulio Valdivieso.


.. [#numpy-ufuncs]
   https://docs.scipy.org/doc/numpy/reference/ufuncs.html#available-ufuncs

.. [#scipy-gammaln]
   https://docs.scipy.org/doc/scipy/reference/generated/scipy.special.gammaln.html

.. [#DICT]
    http://docs.python.org/3/library/stdtypes.html#dict

.. [#RANGE]
    http://docs.python.org/3/library/functions.html#func-range

.. [#BORDER]
    http://docs.python.org/3/library/curses.html#curses.window.border

.. [#SENDFILE]
    http://docs.python.org/3/library/os.html#os.sendfile

.. [#ADDCH]
    http://docs.python.org/3/library/curses.html#curses.window.addch

.. [#GUIDO]
   Guido van Rossum, posting to python-ideas, March 2012:
   https://mail.python.org/pipermail/python-ideas/2012-March/014364.html
   and
   https://mail.python.org/pipermail/python-ideas/2012-March/014378.html
   and
   https://mail.python.org/pipermail/python-ideas/2012-March/014417.html

.. [#PEP399]
   https://www.python.org/dev/peps/pep-0399/

.. [#python-ideas-decorator-based]
   https://mail.python.org/pipermail/python-ideas/2017-February/044888.html

.. [#posonly-impl]
   https://github.com/pablogsal/cpython_positional_only

.. [#thread-keyword-to-positional]
   https://mail.python.org/pipermail/python-ideas/2016-January/037874.html

.. [#leading-underscore]
   https://mail.python.org/pipermail/python-ideas/2018-September/053319.html

.. [#document-positional-only]
   https://bugs.python.org/issue21314

=========
Copyright
=========

This document has been placed in the public domain.
