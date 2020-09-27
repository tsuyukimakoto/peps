PEP: 582
Title: Python local packages directory
Version: $Revision$
Last-Modified: $Date$
Author: Kushal Das <mail@kushaldas.in>, Steve Dower <steve.dower@python.org>,
        Donald Stufft <donald@stufft.io>, Nick Coghlan <ncoghlan@gmail.com>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 16-May-2018
Python-Version: 3.8


Abstract
========

This PEP proposes to add to Python a mechanism to automatically recognize a
``__pypackages__`` directory and prefer importing packages installed in this
location over user or global site-packages. This will avoid the steps to create,
activate or deactivate "virtual environments". Python will use the
``__pypackages__`` from the base directory of the script when present.

この PEP は、 ``__pypackages__`` ディレクトリを自動的に認識し、ユーザやグローバルなサイトパッケージよりもこの場所にインストールされたパッケージを優先的にインポートする仕組みを Python に追加することを提案します。これにより、"仮想環境 "の作成、有効化、無効化のステップを回避することができます。Python はスクリプトのベースディレクトリにある ``__pypackages__`` があればそれを使用します。

Motivation
==========

Python virtual environments have become an essential part of development and
teaching workflow in the community, but at the same time, they create a barrier
to entry for many. The following are a few of the issues people run into while
being introduced to Python (or programming for the first time).

Python の仮想環境はコミュニティでの開発や教育のワークフローに欠かせないものとなっていますが、同時に多くの人にとって参入障壁となっています。以下は、Pythonを導入する際（または初めてプログラミングをする際）に遭遇する問題のいくつかです。

- How virtual environments work is a lot of information for anyone new. It takes
  a lot of extra time and effort to explain them.
  
  仮想環境がどのように機能するのかは、初めての人にとっては多くの情報が必要です。それらを説明するには余計な時間と労力がかかります。

- Different platforms and shell environments require different sets of commands
  to activate the virtual environments. Any workshop or teaching environment with
  people coming with different operating systems installed on their laptops create a
  lot of confusion among the participants.

  異なるプラットフォームやシェル環境では、仮想環境をアクティブにするために異なるコマンドセットを必要とします。ラップトップに異なるオペレーティングシステムをインストールした人たちが集まるワークショップや教育環境では、参加者の間で多くの混乱が生じます。

- Virtual environments need to be activated on each opened terminal. If someone
  creates/opens a new terminal, that by default does not get the same environment
  as in a previous terminal with virtual environment activated.

  仮想環境は開いた端末ごとに有効にする必要があります。誰かが新しい端末を作成したり開いたりした場合、デフォルトでは仮想環境が有効化されている以前の端末と同じ環境にはなりません。


Specification
=============

When the Python binary is executed, it attempts to determine its prefix (as
stored in ``sys.prefix``), which is then used to find the standard library and
other key files, and by the ``site`` module to determine the location of the
``site-package`` directories.  Currently the prefix is found -- assuming
``PYTHONHOME`` is not set -- by first walking up the filesystem tree looking for
a marker file (``os.py``) that signifies the presence of the standard library,
and if none is found, falling back to the build-time prefix hard coded in the
binary. The result of this process is the contents of ``sys.path`` - a list of
locations that the Python import system will search for modules.

Pythonのバイナリが実行されると、そのプレフィックス（``sys.prefix```に格納されているもの）を決定しようとし、標準ライブラリや他のキーファイルを見つけるために使われます。 現在のところ、 ``PYTHONHOME`` が設定されていないと仮定して、まずファイルシステムツリーを歩いて標準ライブラリの存在を示すマーカーファイル(``os.py```)を探し、見つからなければバイナリにハードコードされたビルド時のプレフィックスにフォールバックしてプレフィックスを見つけます。この処理の結果は ``sys.path`` の内容となり、Pythonのインポートシステムがモジュールを検索する場所のリストとなります。

This PEP proposes to add a new step in this process. If a ``__pypackages__``
directory is found in the current working directory, then it will be included in
``sys.path`` after the current working directory and just before the system
site-packages. This way, if the Python executable starts in the given project
directory, it will automatically find all the dependencies inside of
``__pypackages__``.

本PEPでは、このプロセスに新しいステップを追加することを提案します。もし ``__pypackages__`` ディレクトリが現在の作業ディレクトリにあった場合、それは ``sys.path`` の中で現在の作業ディレクトリの後、システムサイトパッケージの直前に含まれます。このようにして、Pythonの実行ファイルが与えられたプロジェクトディレクトリで起動した場合、 ``__pypackages__``の中にあるすべての依存関係を自動的に見つけます。

In case of Python scripts, Python will try to find ``__pypackages__`` in the
same directory as the script. If found (along with the current Python version
directory inside), then it will be used, otherwise Python will behave as it does
currently.

Pythonスクリプトの場合、Pythonはスクリプトと同じディレクトリにある ``__pypackages__``` を見つけようとします。もし見つかれば(現在のPythonのバージョンのディレクトリと一緒に)それが使われ、そうでなければPythonは現在と同じように動作します。

If any package management tool finds the same ``__pypackages__`` directory in
the current working directory, it will install any packages there and also
create it if required based on Python version.

パッケージ管理ツールが現在の作業ディレクトリに同じ ``__pypackages__``` ディレクトリを見つけた場合、そこにパッケージをインストールし、必要に応じて Python のバージョンに基づいて作成します。

Projects that use a source management system can include a ``__pypackages__``
directory (empty or with e.g. a file like ``.gitignore``). After doing a fresh
check out the source code, a tool like ``pip`` can be used to install the
required dependencies directly into this directory.

ソース管理システムを使っているプロジェクトでは、 ``__pypackages__``` ディレクトリ (空か、あるいは ``.gitignore`` のようなファイル) を作ることができます。ソースコードを新たにチェックアウトした後、 ``pip``` のようなツールを使って、必要な依存関係をこのディレクトリに直接インストールすることができます。

Example
-------

The following shows an example project directory structure, and different ways
the Python executable and any script will behave.

以下は、プロジェクトのディレクトリ構造の例であり、Python の実行ファイルと任意のスクリプトの動作の異なる方法を示しています。

::

    foo
        __pypackages__
            3.8
                lib
                    bottle
        myscript.py

    /> python foo/myscript.py
    sys.path[0] == 'foo'
    sys.path[1] == 'foo/__pypackages__/3.8/lib'


    cd foo

    foo> /usr/bin/ansible
        #! /usr/bin/env python3
    foo> python /usr/bin/ansible

    foo> python myscript.py

    foo> python
    sys.path[0] == '.'
    sys.path[1] == './__pypackages__/3.8/lib'

    foo> python -m bottle

We have a project directory called ``foo`` and it has a ``__pypackages__``
inside of it. We have ``bottle`` installed in that
``__pypackages__/3.8/lib``, and have a ``myscript.py`` file inside of the
project directory. We have used whatever tool we generally use to install ``bottle``
in that location.

foo``というプロジェクトディレクトリがあり、その中に ``__packages__``` があります。この ``__pypackages__/3.8/lib`` に ``bottle`` をインストールし、プロジェクトディレクトリの中に ``myscript.py`` ファイルを作成しています。この場所に ``bottle`` をインストールするために一般的に使われているツールを使っています。

For invoking a script, Python will try to find a ``__pypackages__`` inside of
the directory that the script resides[1]_, ``/usr/bin``.  The same will happen
in case of the last example, where we are executing ``/usr/bin/ansible`` from
inside of the ``foo`` directory. In both cases, it will **not** use the
``__pypackages__`` in the current working directory.

スクリプトを起動する際，Pythonはスクリプトが存在するディレクトリ[1]_，```/usr/bin```の中から ``__pypackages__`` を見つけようとします． 最後の例では、 ``foo`` ディレクトリの中から ``/usr/bin/ansible`` を実行しています。どちらの場合も，現在の作業ディレクトリにある ``__pypackages__``` は**使用されません**．

Similarly, if we invoke ``myscript.py`` from the first example, it will use the
``__pypackages__`` directory that was in the ``foo`` directory.

同様に、最初の例で ``myscript.py`` を起動すると、 ``foo`` ディレクトリにあった ``__pypackages__`` ディレクトリが使われます。

If we go inside of the ``foo`` directory and start the Python executable (the
interpreter), it will find the ``__pypackages__`` directory inside of the
current working directory and use it in the ``sys.path``. The same happens if we
try to use the ``-m`` and use a module. In our example, ``bottle`` module will
be found inside of the ``__pypackages__`` directory.

もし ``foo`` ディレクトリの中に入って Python の実行ファイル（インタプリタ）を起動すると、現在の作業ディレクトリの中にある ``__pypackages__`` ディレクトリを見つけて ``sys.path`` の中でそれを使います。同じことが ``-m`` を使ってモジュールを使おうとした場合にも起こります。この例では、 ``bottle`` モジュールは ``__pypackages__`` ディレクトリの中にあります。

The above two examples are only cases where ``__pypackages__`` from current
working directory is used.

上の2つの例は、現在の作業ディレクトリにある ``__pypackages__``` を使用した場合のみです。

In another example scenario, a trainer of a Python class can say "Today we are
going to learn how to use Twisted! To start, please checkout our example
project, go to that directory, and then run ``python3 -m pip install twisted``."

別の例のシナリオでは、Pythonクラスのトレーナーが「今日はTwistedの使い方を学びます！」と言うことができます。まずはサンプルプロジェクトをチェックアウトして、そのディレクトリに移動してから ``python3 -m pip install twisted``` を実行してください。

That will install Twisted into a directory separate from ``python3``. There's no
need to discuss virtual environments, global versus user installs, etc. as the
install will be local by default. The trainer can then just keep telling them to
use ``python3`` without any activation step, etc.

これにより、Twistedは``python3```とは別のディレクトリにインストールされます。デフォルトではローカルにインストールされるので、仮想環境やグローバルとユーザのインストールなどについて議論する必要はありません。これでトレーナーは、アクティベーションなどのステップを踏まずに ``python3``を使うように指示し続けることができます。

.. [1]_: In the case of symlinks, it is the directory where the actual script
   resides, not the symlink pointing to the script


Security Considerations
=======================

While executing a Python script, it will not consider the ``__pypackages__`` in
the current directory, instead if there is a ``__pypackages__`` directory in the
same path of the script, that will be used.

Python スクリプトを実行している間は、カレントディレクトリにある ``__pypackages__``` を考慮せず、代わりにスクリプトの同じパスに ``__pypackages__`` ディレクトリがあれば、それが使われます。

For example, if we execute ``python /usr/share/myproject/fancy.py`` from the
``/tmp`` directory and  if there is a ``__pypackages__`` directory inside of
``/usr/share/myproject/`` directory, it will be used. Any potential
``__pypackages__`` directory in ``/tmp`` will be ignored.

例えば、 ``python /usr/share/myproject/fancy.py`` を ``/tmp`` ディレクトリから実行し、 ``/usr/share/myproject/` ディレクトリの中に ``__pypackages__`` ディレクトリがあれば、それを利用します。もし ``/tmp`` ディレクトリの中に ``__pypackages__`` ディレクトリがあれば、それを利用します。

Backwards Compatibility
=======================

This does not affect any older version of Python implementation.

これは、Pythonの実装の古いバージョンには影響しません。


Impact on other Python implementations
--------------------------------------

Other Python implementations will need to replicate the new behavior of the
interpreter bootstrap, including locating the ``__pypackages__`` directory and
adding it the ``sys.path`` just before site packages, if it is present.

他のPythonの実装では、 ``__pypackages__`` ディレクトリを探して、サイトパッケージの直前に ``sys.path`` を追加するなど、インタプリタブートストラップの新しい動作を再現する必要があります。

Reference Implementation
========================

`Here <https://github.com/kushaldas/cpython/tree/pypackages>`_ is a PoC
implementation (in the ``pypackages`` branch).


Rejected Ideas
==============

``__pylocal__`` and ``python_modules``.


Copyright
=========

This document has been placed in the public domain.


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 80
   coding: utf-8
   End:
