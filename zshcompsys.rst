
.. _name:

名前
====

zshcompsys - zsh 補完システム

.. _description:

解説
====

これは compsys と呼ばれる `新しい` 補完システムのためのシェルコードについて
記述します。これは `zshcompwid(1)` に記述されている機能に基づいた、シェル
関数で記述されています。

機能は補完が開始された時点の文脈
多くの補完が既に提供されています。\
このため、ユーザは以下の 初期化_ の節に記述されているシステムの初期化以外の\
詳細を何も知らなくても、多くのタスクをこなすことができます。

補完の実行を決める文脈は次のとおりです:

* 引数またはオプションの位置: これは補完が要求される、コマンドライン上での\
  位置を指します。例えば 'rmdir の最初の引数は、ディレクトリの名前が補完' \
  されます;

* シェルの構文の要素を表す特別なコンテキスト。
  例えば 'コマンド位置' や '配列の添字' など。


我々が説明すべき、文脈の完全な仕様には他の要素も含まれています。

コマンド名とコンテキストのほか、補完システムはあと二つの概念を採用しています。
スタイルとタグです。これはユーザがシステムの動作を設定する方法を提供します。

タグには二つの役割を果たしています。マッチの分類システムとして機能します。
一般的には、ユーザが区別する必要がある対象の分類を示します。
例えば、lsコマンドの引数を補完するとき、ユーザにとってはディレクトリよりも\
ファイルが先に補完される方が好ましいかも知れません。
そして、これらはどちらもタグなのです。これらはコンテキスト指定の最も右端の\
要素として表示もされます。

スタイルは出力フォーマットだけではなく、どのコンプリータが（どんな順番で）\
使われるか、どのタグが評価されるかなどのように、補完システムの様々な操作を\
変更します。スタイルは引数を受け取ることができ、zshmodules(1)で説明されている\
zstyleコマンドを使って操作します。

まとめると、タグは何の補完オブジェクトか、スタイルはどうやって補完するかを\
表します。実行のあらゆるポイントで、補完システムは現在のコンテキストに対して\
どのスタイルやタグが定義されているかをチェックし、振る舞いを変更するために\
使います。タグやその他の要素がスタイルの動作にどのように影響するかを決定する\
コンテキスト処理のきちんとした説明は、以下の 補完システム構成 で説明します。

補完が要求されると、ディスパッチャ関数が呼ばれます; 以下のコントロール関数\
一覧の中にある _main_complete の説明を参照してください。このディスパッチャは\
補完を提供するのにどの関数が呼ばれるべきかを決定し、呼び出します。結果は\
単純補完、エラー訂正、エラー訂正を伴う補完、メニュー選択などの、個別の\
補完戦略を実装したコンプリータ、関数に渡されます。

より一般的には、補完システムに含まれるシェル関数には２つのタイプがあります:

* 'comp' で始まる、直接呼び出されるもの; これは少ししか無い

* '_' で始まる、補完コードに呼び出されるもの。
  補完の動作を実装し、キーストロークにバインドすることもできるこれらの\
  シェル関数は 'ウィジェット' と呼ばれます。
  これらは新しい補完が必要になるに従って増殖します。


.. _initialization:

初期化
======

補完システムが正しくインストールされていれば、あなたは自分の初期化ファイルから\
compinit シェル関数を呼び出すだけで充分です; 次のセクションを参照してください。
ただし compinstall 関数は補完システムのさまざまな特徴を構成するためにユーザが\
実行できます。

通常、compinstallは .zshrc にコードを挿入しますが、それが書き込み不可の場合は\
別の場所に保存してその場所を通知します。.zshrcに挿入された行が実際に実行される\
ことを確認するのはあなた次第です; 例えば、通常 .zshrc を以前のものに戻す場合、\
それらをファイル内の以前の場所に戻す必要があるかも知れません。なので、それらを\
（最初と最後のコメント行を含めた）すべて保持していれば、compinstallを再実行\
してそれらの行を正しい配置・編集することができます。しかし、このセクションに\
コードを追加している場合、'zstyle'コマンドを使った行は正しく処理されるべき\
ですが、compinstallを再実行した時に失われる可能性があります。

新しいコードは次にシェルを起動した時、または.zshrcを手動で実行した時に反映\
されます; すぐに反映させるためのオプションもあります。しかし、compinstallが\
定義を削除した場合、変更を結果を確認するにはシェルを再起動する必要があります。

compinstallを実行するには、それがfpathパラメータで示されるディレクトリに\
存在することを確かめる必要があります。これはあなたのスタートアップファイルが\
適切なディレクトリをfpathから削除しないように適切に構成されている限り、既に\
そうなっているはずです。そうすれば自動的に読み込まれるでしょう。
（ ``autoload -U compinstall`` が推奨です）
インストレーションが情報の入力を求めるプロンプトが表示されている間は、いつでも\
中断できますし、あたなの.zshrcは全く変更されません; 変更されたものは、終了時に\
あなたが具体的に確認を求められた場所へ置かれます。


.. _use of compinit:

compinit の使用
---------------

このセクションでは、直接呼び出された場合に現在のセッションの補完を初期化する\
compinitの使い方について説明します; もしcompinstallを実行しているなら、\
あなたの.zshrcから自動的に呼び出されています。

システムを初期化するには、compinit関数がfpathパラメータで示されるディレクトリ\
に存在し、自動的に読み込まれる（ ``autoload -U compinit`` が推奨）必要があり、\
それから ``compinit`` と単純に実行します。これは、いくつかのユーティリティ\
関数を定義し、必要なすべてのシェル関数が自動読み込みされるように手配し、\
新しいシステムを使用するよう、補完を行うすべてのウィジェットを再定義します。
あなたがもしzsh/compinitモジュールの一つであるmenu-selectウィジェットを使って\
いるなら、ウィジェットも再定義できるようにそのモジュールがcompinitよりも前に\
読み込まれていることを確認してみるとよいでしょう。補完スタイル（後述）が\
デフォルトで補完だけでなく展開も実行されるように設定されていて、TABキーが\
expand-or-completeにバインドされている場合、compinitはそれをcomplete-wordに\
バインドし直します; これは展開の正しい形式を使うために必要なことです。

元の補完コマンドを使う必要がある場合は、例えば ``.expand-or-complete`` の\
ようにウィジェット名の前に ``.`` を置くことによって古いウィジェットにキーを\
バインドできます。

compinitの実行速度を上げるため、以降の呼び出し時に読み込まれる設定のダンプを\
生成させることができます; これはデフォルトですが、-Dオプションを付けて\
compinitを呼び出すことでオフにできます。ダンプされたファイルはスタートアップ\
ファイルと同じディレクトリ（$ZDOTDIR または $HOME）にある .zcompdump です; \
別の方法として ``compinit -d dumpfile`` として明示的にファイル名を渡すことも\
できます。次回のcompinitの呼び出しでは、完全な初期化を実行するかわりに\
ダンプされたファイルを読み込みます。

補完ファイルの数が変更された場合、compinitはこれを認識して新しいダンプ\
ファイルを生成します。しかし、関数の名前が変更されたり、最初の行の #compdef\
関数（後述）の引数が変更された場合は、compinitが次に実行された時に再作成\
できるように、手動でダンプファイルを削除するのが最も簡単です。
新しい関数があるかどうかを確認するために行われるチェックは、-Cオプションを\
与えることで省略できます。この場合、ダンプファイルが既に存在していない場合に\
のみ作成されます。

ダンプは、実際は別の関数compdumpによって行われていますが、設定を変更（例えば、\
compdefを使用する）して、新しい設定をダンプしたい場合にのみ、これを自分自身で\
実行する必要があります。
古いダンプファイルの名前はこの目的のために記憶されます。

パラメータ_compdirが設定されている場合、compinitはそこを補完スクリプトが\
存在するディレクトリとして使用します; これはまだ関数の検索パスに含まれて\
いない場合にのみ必要です。

セキュリティ上の理由から、compinitは補完システムがrootやカレントユーザが\
所有してしないファイル、または誰でも、ないしはグループ書き込み可能もしくは\
rootやカレントユーザの所有ではないディレクトリにあるファイルを使う場合も\
チェックを行います。
そのようなファイルやディレクトリが検出された場合、compinitはそれらを実際に\
使う必要があるかどうか尋ねてきます。これらの確認を回避し、検出されたすべての\
ファイルを使えるようにするには、-uオプションを使用し、compinitが安全でない\
ファイルやディレクトリを無視するようにするには、-iオプションを使用します。
-Cオプションが指定されている場合には、このセキュリティチェックは完全に\
スキップされます。

セキュリティチェックは関数compauditを実行することで、いつでも再施行できます。
これはcompinitで使用されるチェックと同じですが、直接実行された場合、fpathへの\
変更は関数のローカルに作られるので、これらは持続しません。チェックする対象の\
ディレクトリは引数として渡すことができます; 何も指定されていない場合、\
compauditは補完システムディレクトリを検出して、必要に応じて欠けているものを\
追加するためにfpathと_compdirを使います。現時点でfpathに指定されている\
ディレクトリに厳密にチェックを矯正するには、compauditまたはcompinitを呼び出す\
前に_compdirに空文字列を設定します。

関数bashcompinitはbashのプログラマブル補完システムとの互換性を提供します。
実行するとcompgenとcompleteという、同じ名前を持つbash組み込みコマンドに\
対応する関数を定義します。
それ以降は、bash用に書かれた補完仕様や関数を使えます。


.. _autoloaded files:

自動的に読み込まれるファイル
----------------------------

補完で使用される、自動で読み込まれる関数の規則は、アンダースコアから始まる\
ことです; 既に述べたように、fpath/FPATHパラメータはそれらが格納されている\
ディレクトリを含める必要があります。
zshが正しくシステムにインストールされている場合、fpath/FPATHは自動的に\
標準の関数のために必要なディレクトリを含んでいます。

インストールが完全でない場合、compinitが検索パス内でアンダースコアで始まる\
ファイルを充分な数（20未満）見つけられないと、検索パスに _compdir を追加して\
さらに探そうとします。
対象のディレクトリにBaseという名前のサブディレクトリがある場合、すべての\
サブディレクトリがパスに追加されます。さらにBaseサブディレクトリが\
Coreという名前のサブディレクトリを持つ場合、compinitはそのサブディレクトリの\
すべてのサブディレクトリもパスに追加します: これは関数がzshのソース配布物に\
含まれる関数のフォーマットと同じであることを許容します。

compinitが実行されるとき、compinitはfpath/FPATHを介してアクセス可能なそれらの\
ファイルを検索し、それぞれの最初の行を読み取ります。この行には以下に示す\
タグのうちの1つを含める必要があります。最初の行が、これらのタグのいずれか1つで\
始まらないファイルは補完システムのパーツと見なされず特別扱いはされません。

タグは:

#compdef names... [ -[pP] patterns... [ -N names... ] ]
    ファイルが自動的にロードされ、namesが補完される時にファイルに定義された\
    関数が呼び出されます。これらはそれぞれ引数が補完されるべきコマンドの\
    名前か、以下で説明する-context-形式の中の、数ある特別なコンテキストの\
    ひとつ、のいずれかです。

    それぞれのnameは`cmd=service'の形式でもかまいません。cmdコマンドを\
    補完するとき、関数は通常そのコマンド（または特別なコンテキスト）\
    サービスが、代わりに補完されたかのように振る舞います。
    これは多くの異なる補完ができる関数の振る舞いを変更する\
    方法を提供します。
    これは関数が呼び出される際に、$serviceパラメータを設定することによって\
    実装されています; 関数はこれを望むように解釈することも選択できますし、\
    単純な関数であればこれを無視します。

    #compdefの行にオプションの1つ-pまたは-Pが含まれている場合、続く単語を\
    パターンであるとみなします。補完がパターンのうちの1つにマッチする\
    コマンドまたはコンテキストに対して試行されるときに、関数が呼ばれます。
    オプション-pおよび-Pはそれぞれ他の補完の前や後に試行されるパターンを\
    指定するために使います。
    従って、-Pはデフォルトアクションを指定するために使えます。

    オプション-Nは、-pや-P以降のリストに使用し、残りの単語にはもうパターンを\
    定義しないことを指定します。
    3つのオプションで必要な回数だけ切り替えることができます。


#compdef -k style key-sequences...
    このオプションは、もしあれば、組み込みのウィジェットスタイルのように\
    振る舞うウィジェットを作成し、指定されたキー・シーケンスにバインドします。
    スタイルは補完を行う組み込みウィジェット、すなわち complete-word, \
    delete-char-or-list, expand-or-complete, expand-or-complete-prefix, \
    list-choices, menu-complete, menu-expand-or-complete, \
    またはreverse-menu-completeのいずれかでなければいけません。
    zsh/complistモジュール（zshmodules(1)を参照）が読み込まれていれば\
    menu-selectウィジェットも利用可能です。

    キー配列のいずれかをタイプすると、ファイル内の関数はマッチを生成するために\
    呼び出されます。既にキーシーケンスが存在する（つまり、未定義ではない\
    なにかにバインドされていた）場合、キーは再バインドされないことに\
    注意してください。
    作成されたウィジェットはファイルと同じ名前を持ち、通常どおりバインド\
    キーを使って任意の他のキーにバインドできます。


#compdef -K widget-name style key-sequences ...
    これは、各ウィジェット名-スタイルのペアに与えられるキー・シーケンスが\
    1つだけであることを除いて、-kオプションと同様です。しかし3つの引数全体の\
    セットは、引数の異なるセットを使って繰り返すことができます。
    ウィジェット名はそれぞれのセットで個別でなければらないことに、特に\
    注意してください。'_'で始まらない場合、これが追加されます。
    ウィジェット名は既存のウィジェット名と衝突してはいけません: 関数の名前に\
    基づいた名前が最も便利です。例えば、 ::

      #compdef -K _foo_complete complete-word "^X^C" \
        _foo_list list-choices "^X^D"

    （全部で一行です）これは"^X^C"に補完のための _foo_complete をバインドし、\
    "^X^D"に一覧のウィジェット _foo_list をバインドするよう定義しています。


#autoload [ options ]
    #autoloadのタグが付いた関数は自動読み込みのためにマークされていますが、\
    それ以外は特別扱いされません。一般的には、これらは補完関数のいずれかの\
    中で呼び出されるべきものです。提供されるオプションはすべて組み込みの\
    autoloadに渡されます; 一般的な用途として、関数がすぐに読み込まれるように\
    強制するため +X を使います。-Uおよび-zフラグは、常に暗黙的に追加される\
    ことに注意してください。

#はタグ名の一部であり、後ろに空白を含めることはできません。#compdefタグは\
後述のcompdef関数を使います; 主な違いは関数名が暗黙的に供給されることです。

補完関数の定義に使える特別なコンテキストは、次のとおりです:

\-array-value-
    配列代入の右辺 ('foo=(...)')

\-brace-parameter-
    中括弧の中のパラメータ展開の名前 ('${...}')

\-assign-parameter-
    代入中のパラメータの名前、つまり'='の左辺

\-command-
    コマンドの位置のワード

\-condition-
    条件式の中のワード ('[[...]]')

\-default-
    他の補完が定義されていない、任意のワード

\-equal-
    等号から始まるワード

\-first-
    これは他の補完関数の前に試行されます。呼び出された関数は、いくつかの\
    値のうちいずれかを _compskip に設定します:
    all: それ以上の補完は行われません。
    部分文字列のパターンを含む文字列: パターン補完関数は呼び出されません
    defaultを含む文字列: '-default-' コンテキストの関数は呼び出されませんが、\
    関数が定義されることはないでしょう

\-math-
    '((...))'のような、数学的コンテキストの内部

\-parameter-
    パラメータ展開の名前 ('$...')

\-redirect-
    リダイレクト演算子の後のワード

\-subscript-
    パラメータの添字の中

\-tilde-
    先頭のチルダ ('~') の後。但し単語の最初のスラッシュの前。

\-value-
    代入式の右辺

デフォルトの実装は、これらのコンテキスト毎に提供されています。ほとんどの場合、\
-context- というコンテキストは対応する _context 関数によって実装されています。\
例えば、コンテキスト'-tilda'と関数'_tilda'です。

\-redirect- コンテキストと -value- コンテキストは、追加のコンテキスト固有の\
情報を受け付けます。（内部的には、これは関数 _dispatch を呼び出すコンテキスト\
ごとの関数によって処理されます。）
追加の情報は、カンマで区切って付与されます。



hogehgoe.

As an example:


.. _functions:

関数
----

::

   compdef [ -ane ] function names...  [ -[pP] ] patterns... [ -N names... ] ]
   compdef -d names...
   compdef -k [ -an ] function sytle key-sequences...
   compdef -K [ -an ] function name style key-sequences ...


.. _completion system configuration:

補完システム構成
================

.. _overview:

概要
----

.. _standard tags:

標準タグ
--------

Some of the following are only used when looking up particular styles
and do not refer to a  type of match.

:accounts:
    used to look up the users-hosts style.

:all-expanstions:
    used by the ``_expand`` completer when adding the single string
    containing all possible exansions.

:all-files:
    for the names of all files (as distinct from a particular subset,
    see the globbed-files tag).

:arguments:
    for arguments to a command

:arrays:
    for names of array parameters

:assosiation-keys:
    for keys of assosiative arrays; used when completing inside a
    subscript to a parameter of this type

:bookmarks:
    when completing bookmarks (e.g. for URLs and the zftp function
    suite)

:builtin:
    for names of builtin commands

:characters:
    for single characters in arguments of commands such as stty.
    Also used when completing character classes after an opening
    bracket

:colormapids:
    for X colormap ids

:colors:
    for color names

:commands:
    for names of external commands. Also used by complex commands
    such as cvs when completing names subcommands.

:contexts:
    for contexts in arguments to the zstyle builtin command

:corrections:
    used by the ``_approximate`` and ``_correct`` completers for possible
    corrections

:cursors:
    for cursor names used by X programs

:default:
    used in some contexts to provide a way of supplying a default
    when more specific tags are also valid. Note that this tag is
    used when only the function field of the context name is set

:descriptions:
    used when looking up the value of the format style to generate
    descriptions for tyles of matches

:devices:
    for names of device special files

:directories:
    for names of directories -- local-directories is used instead
    wheh completing arguments of cd and related builtin commands
    when the cdpath array is set

:directory-stack:
    for entries in the directory stack

:displays:
    for X display names

:domains:
    for network domains

:expansions:
    used by the ``_expand`` completer for individual words (as opposed
    to the complete set of expansions) resulting from the expansion
    of a word on the command line

:extensions:
    for X server extensions

:file-descriptors:
    for numbers of open file descriptors

:files:
    the generic file-matching tag used by functions completing filenames

:fonts:
    for X font names

:fstypes:
    for file system types (e.g. for the mount command)

:functions:
    names of functions -- normally shell functions, although certain
    commands mey understand other kinds of function

:globbed-files:
    for filenames when the name has been generated by pattern matching

:groups:
    for names of user group

:history-words:
    for words from the history

:hosts:
    for hostnames

:indexes:
    for array indexes

:jobs:
    for jobs (as listed by the `jobs` builtin)

:interfaces:
    for network interfaces

:keymaps:
    for names of zsh keymaps

:keysyms:
    for names of X keysyms

:libraries:
    for names of system libraries

:limits:
    for system limits

:local-directories:
    for names of directories that are subdirectories of the current
    working directory when completing arguments of cd and related
    builtin commands (compare path-directories0 -- when the cdpath
    array is unset, directories is used instead

:manuals:
    for names of manual pages

:mailboxes:
    for e-mail folders

:maps:
    for map names (e.g. NIS maps)

:messages:
    used to look up the format style for messages

:modifiers:
    for names of X modifiers

:modules:
    for modules (e.g. zsh modules)

:my-accounts:
    used to look up the users-hosts style

:named-directories:
    for named directories (you wouldn't have guessed that, would you?)

:names:
    for all kinds of names

:newsgroups:
    for USENET groups

:nicknames:
    for nicknames of NIS maps

:options:
    for command options

:original:
    used by the ``_approximate``, ``_correct`` and ``_expand`` completers when
    offering the original string as a match

:other-accounts:
    used to look up the users-hosts style

:other-files:
    for the names of any non-directory files. This is used instead
    of all-files when the list-dirs-first style is in effect.

:packages:
    for packages (e.g. rpm or installed Debian packages)

:parameters:
    for names of parameters

:path-directories:
    for names of directories found by searching the cdpath array
    when completing arguments of cd and related builtin commands
    (compare local-directories)

:paths:
    used to look up the values of the expand, ambiguous and
    special-dirs style

:pods:
    for perl pods (documentation files)

:ports:
    for communication ports

:prefixes:
    for prefixes (like those of a URL)

:printers:
    for print queue names

:processes:
    for process identifiers

:process-names:
    used to look up the command style when generating the names of
    processes for killall

:sequences:
    for sequences (e.g. mh sequences)

:sessions:
    for sessions in  the zftp function suite

:signals:
    for signal names

:strings:
    for strings (e.g. the replacement strings for the cd builtin
    command)

:styles:
    for styles used by the zstyle builtin command

:suffixes:
    for filename extensions

:tags:
    for tags (e.g. rpm tags)

:targets:
    for makefile targets

:time-zones:
    for time zones (e.g. when setting the TZ parameter)

:types:
    for types of whatever (e.g. address types for the xhost command)

:urls:
    used to look up the urls and local styles when completing URLs

:users:
    for usernames

:values:
    for one of a set of values in certain lists

:variant:
    used by ``_pick_variant`` to look up the command to run when determining
    what program is installed for a particular command name.

:visuals:
    for X visuals

:warnings:
    used to look up the format style for warnings

:widgets:
    for zsh widget names

:windows:
    for IDs of X windows

:zsh-options:
    for shell options


.. _standard styles:

標準スタイル
------------

:accept-exact:
    hoge

:accept-exact-dirs:
    hoge

:add-space:
    hoge

:ambiguous:
    hoge

:assign-list:
    hoge

:auto-description:
    hoge

:avoid-completer:
    hoge

:cache-path:
    hoge

:cache-policy:
    hoge

:call-command:
    hoge

:command:
    hoge

:command-path:
    hoge

:commands:
    hoge

:complete:
    hoge

:complete-options:
    hoge

:completer:
    hoge

:condition:
    hoge

:delimiters:
    hoge

:disabled:
    hoge

:domains:
    hoge

:environ:
    hoge

:expand:
    hoge

:fake:
    hoge

:fake-always:
    hoge

:fake-files:
    hoge

:fake-parameters:
    hoge

:fake-list:
    hoge

:file-patterns:
    hoge

:file-sort:
    hoge

:filter:
    hoge

:force-list:
    hoge

:format:
    hoge

:glob:
    hoge

:global:
    hoge

:group-name:
    hoge

:group-order:
    hoge

:groups:
    hoge

:hidden:
    hoge

:hosts:
    hoge

:hosts-ports:
    hoge

:ignore-line:
    hoge

:ignore-parents:
    hoge

:extra-verbose:
    hoge

:ignore-patterns:
    hoge

:insert:
    hoge

:insert-ids:
    hoge

:insert-tab:
    hoge

:insert-unambiguous:
    hoge

:keep-prefix:
    hoge

:last-prompt:
    hoge

:known-hosts-files:
    hoge

:list:
    hoge

:list-colors:
    hoge

:list-dirs-first:
    hoge

:list-groups:
    hoge

:list-packed:
    hoge

:list-prompt:
    hoge

:list-rows-first:
    hoge

:list-suffixes:
    hoge

:list-separator:
    hoge

:local:
    hoge

:mail-directory:
    hoge

:match-original:
    hoge

:matcher:
    hoge

:matcher-list:
    hoge

:max-errors:
    hoge

:max-matches-width:
    hoge

:menu:
    hoge

:muttrc:
    hoge

:numbers:
    hoge

:old-list:
    hoge

:old-matches:
    hoge

:old-menu:
    hoge

:original:
    hoge

:packageset:
    hoge

:path:
    hoge

:path-completion:
    hoge

:pine-directory:
    hoge

:ports:
    hoge

:prefix-hidden:
    hoge

:prefix-needed:
    hoge

:preserve-prefix:
    hoge

:range:
    hoge

:regular:
    hoge

:rehash:
    hoge

:remote-access:
    hoge

:remote-all-dups:
    hoge

:select-prompt:
    hoge

:select-scroll:
    hoge

:separate-sections:
    hoge

:show-completer:
    hoge

:single-ifnored:
    hoge

:sort:
    hoge

:special-dirs:
    hoge

:squeeze-slashes:
    hoge

:stop:
    hoge

:strip-comments:
    hoge

:subst-globs-only:
    hoge

:substitute:
    hoge

:suffix:
    hoge

:tag-order:
    hoge

:urls:
    hoge

:use-cache:
    hoge

:use-compctl:
    hoge

:use-ip:
    hoge

:users:
    hoge

:users-hosts:
    hoge

:users-hosts-ports:
    hoge

:verbose:
    hoge

:word:
    hoge


.. _control functions:

操作関数
========

:_all_matches:
    hoge

:_appriximate:
    hoge

:_complete:
    hoge

:_correct:
    hoge

:_expand:
    hoge

:_expand_alias:
    hoge

:_history:
    hgoe

:_ignored:
    hoge

:_list:
    hoge

:_match:
    hoge

:_menu:
    hoge

:_oldlist:
    hoge

:_prefix:
    hoge

:_user_expand:
    hoge

    :$hash:
        fuga

    :_func:
        moga


.. _bindable commands:

バインド可能なコマンド
======================

:_bash_completions:
    hoge

:_correct_filename (^XC):
    hoge

:_correct_word (^Xc):
    hoge

:_expand_alias (^Xa):
    hoge

:_expand_word (^Xe):
    hoge

:_generic:
    hoge

:_history_complete_word (\\e/):
    hoge

:_most_recent_file (^Xm):
    hoge

:_next_tags (^Xn):
    hoge

:_read_comp (^X^R):
    hoge

:_complete_debug (^X?):
    hoge

:_complete_help (^Xh):
    hoge

:_complete_help_generic:
    hoge

:_complete_tag (^Xt):
    hoge


.. _utility functions:

ユーティリティ関数
==================

:_all_labels [ -x ] [ -12VJ ] tag name descr [ command args ... ]:
    hoge

:_alternative [ -O name ] [ -C name ] spec ...:
    hoge

:_arguments [ -nswWACRS ] [ -O name ] [ -M matchspec ] [ \: ] spec ...:

    この関数は UNIX 標準のオプションや引数の規約に従って、コマンドの引数に
    完全な指定を与えるために使います。後ろに続く書式は、オプションや引数の
    セットをそれぞれ指定します; 曖昧さを避けるため、これらと _arguments
    自身を、単一のコロンで分離できます。 _arguments 自身のオプションは、
    区切らなければなりません。つまり -sw ではなく -s や -w のようにします。

    オプション -n を使うと、_arguments は NORMARGパラメータを $words 配列 の
    中の最初の通常引数の位置、つまり最後のオプションの後ろの位置にセットします。
    引数がない場合、NORMARG は -1 にセットされます。オプション -n を渡す場合は、
    呼び出し元で 'integer NORMARG' を宣言する必要があり、もし宣言しないと
    パラメータは使われません。

    | n:message:action
    | n::message:action

    .. epigraph::

       これは n 番目の通常引数を表します。 ``message`` は生成された一致候補の
       上に表示され、 ``action`` はこの位置に補完できるものを示します(以下を
       見てください)。 ``message`` の前にコロンが二つある場合、引数は
       オプショナルです。もし ``message`` が空白しか含まない場合、 ``action``
       自身がの説明文を追加しない限り一致候補の上には何も表示されません。

    | :message:action
    | ::message:action

    .. epigraph::

       似ていますが、ある番号の次の引数を表します。この形式で全ての引数が正しい
       順番で指定された場合、番号は不要です。

    | \*:message:action
    | \*::message:action
    | \*:::message:action

    .. epigraph::

       これは引数(通常、オプションではない引数で、 ``-`` や ``+`` で始まらない
       ) が上記の２つの形式のどちらにも提供されなかった場に補完する方法を
       表します。どの順番の引数でも、この形式で補完することができます。

       ``message`` の前にコロンが２つの場合、特殊配列 `words` と特殊パラメータ
       ``CURRENT`` は、 ``action`` が実行または評価される時に、通常引数のみを
       参照するように編集されます。 ``message`` の前にコロンが３つの場合は、
       この説明で触れられている通常引数のみを参照するように編集されます。

    | optspec
    | optspec:...

    .. epigraph::

       これはオプションを表します。コロンは一つ以上のオプション引数に対する
       処理を表します; もし書かれていない場合は、そのオプションは引数を
       取りません。

       初期値では、オプションは複数文字による名前で、１オプションにつき
       １語です。 ``-s`` オプションを渡すと、１文字オプションが可能になり、
       １語で一つ以上のオプションが使えますが、 ``'--prefix'`` のように
       ハイフン２つで始まる語を完全なオプション名とも見做します。
       これは標準 GNU オプションに適しています。

       ``-s`` と ``-w`` の組み合わせでは、たとえ一つ以上のオプションが引数を
       とる場合でも、１文字オプションを一つの語に結合することができます。
       たとえば、 ``-a`` が引数をとる場合に、 ``-s`` を渡さない場合の ``-ab``
       は単一 (扱っていない) のオプションと解釈されます; ``-s`` を渡した場合
       の ``-ab`` は、オプション ``b`` と解釈されます; ``-s`` と ``-w`` の
       両方を渡した場合は、オプション ``-a`` と、引数を持つオプション ``-b``
       になります。

       オプション ``-W`` は、さらに一つ先の立ち位置を取ります: 同じ語による
       引数の後でも、一文字オプションの補完が可能です。しかし、この時点で
       オプションが本当に補完されるかどうかは実行された ``action`` に
       依ります。細かく制御するには ``action`` の一部に ``_guard`` のような
       ユーティリティ関数を使います。

       以下の形式は、オプションが引数を持つか否かに関わらず、初期化時の
       ``optspec`` に使えます。

       | \*optspec

       .. epigraph::

          この ``optspec`` は以下に示す ``optspec`` のうちにいずれか、です。
          以下の ``optspec`` の繰り返しも表します。対応するオプションが既に
          コマンドラインのカーソルよりも左側に存在すれば、再度表示される
          ことはありません。

       | \-optname
       | \+optname

       .. epigraph::

          ``optspec`` の中でもっともシンプルな形式は、 ``-foo`` のように
          マイナスまたはプラス記号から始まるものです。オプションの最初の
          引数は (もしあれば) オプションの直ぐ後に、独立した語として続けて
          渡さなければいけません。

          ``-optname`` と ``+optname`` の両方を有効として指定するために、
          ``-+optname`` や ``+-optname`` のどちらも使えます。

          以降のすべての形式で、先頭の ``-`` はこのように ``+`` と
          置き換えたり、組み合わせることができます。

       | \-optname\-

       .. epigraph::

          (このオプションの) 最初の引数はオプション名と同じ語の後ろに直接
          来なければいけません。例えば、 ``'-foo-:...'`` は ``-fooarg`` と
          なるようなオプションと引数の補完を指定します。

       | \-optname\+

       .. epigraph::

          (このオプションの) 最初の引数はオプション名の直後に一つの語として
          現れるか、オプションの後に独立した語として現れます。たとえば、
          ``'-foo+:...'`` は ``-fooarg`` と ``'-foo arg'`` のどちらとも
          オプションと引数の補完を指定します。

       | \-optname\=

       .. epigraph::

          (このオプションの) 引数は、 ``'-foo=arg'`` や ``'-foo arg'`` の
          ように次の語として現れるか、等号で区切った一つの語として現れます。

       | \-optname\=\-

       .. epigraph::

          オプションの引数は、必ず等号で区切られた一つの語として現れ、
          次の語として引数が与えられることはありません。

       | optspec[explanation]

       .. epigraph::

          ``'-q[query operation]'`` のように、ブラケットで囲むことで、前述の
          ``optspec`` のいずれにも説明文字列を追加することができます。

          冗長なスタイルは、補完リストの中で説明文字列がオプションと共に
          表示されるかどうかを決定するために使われます。

          ブラケットで括られた説明文字列は与えられていないが、自動記述
          スタイルが設定されており、一つの引数のみがこの ``optspec`` で
          記述されている場合、 スタイルの値は、出現する '%d' のシーケンスを
          いずれも ``optspec`` に続く最初の ``optarg`` のメッセージで
          置き換えて表示されます; 下記を参照してください。

    リテラルの ``+`` や ``=`` の付くオプションは可能ですが、これらの文字は
    例えば '-\+' のように引用符で囲む必要があります。

    ``optspec`` に続く ``optarg`` はそれぞれ以下のいずれが一つの形式を取る
    必要があります:

    | \:message\:action
    | \:\:message\:action

    .. epigraph::

       オプションの引数; ``message`` と ``action`` は通常の引数として
       扱われます。最初の形式では、引数はmandatoryで、二つ目の形式では
       引数はオプショナルです。

       このグループは複数の引数を取るオプションには繰り返して使えます。
       言い換えれば、 \:message1\:action1\:message2\:action2 は
       引数を２つ取る、と言うことを表します。

    | \:\*pattern\:message:action
    | \:\*pattern\:\:message:action
    | \:\*pattern\:\:\:message:action

    .. epigraph::

       これは複数の引数を表します。この形式では、複数の引数を取る
       オプションには最後の ``optarg`` だけを指定することができます。
       ``pattern`` が空の場合 (例えば、 \:\*\:)、入力行に残っている
       全ての語は ``action`` で記述されたように補完されます;
       そうでない場合、入力行の最後までで ``pattern`` にマッチする
       すべての語は ``action`` を使って補完されます。

       コロンが複数ある場合は、通常の引数に対する '\*\:...' の形式と
       同様に扱われます: メッセージの前にコロンが２つある場合、
       ``words`` 特殊配列と ``CURRENT`` 特殊パラメータは、 ``action`` が
       実行ないしは評価されている間はオプションの後ろの語のみを
       参照するように変更されます。コロンが３つある場合、この解説で
       触れている語のみを参照するように変更されます。

``optname``, ``message`` や ``action`` に含まれるコロンは、いずれも
'\:' のようにバックスラッシュを付けます。



hoge


:_cache_invalid cache_identifier:
    hoge

:_call_function return name [ args ... ]:
    hoge

:_call_program tag string:
    hoge

:_combination [ -s pattern ] tag style spec ... field opts ...:
    hoge

:_describe [ -oO | -t tag ] descr name1 [ name2 ] opts ... -- ...:
    hoge

:_description [ -x ] [ -12VJ ] tag name descr [ spec ... ]:
    hoge

:_dispatch context string ...:
    hoge

:_files:
    hoge

:_gnu_generic:
    hoge

:_guard [ options ] pattern descr:
    hoge

:_message [ -r12 ] [ -VJ group ] descr:
:_message -e [ tag ] descr:
    hoge

:_multi_parts sep array:
    hoge

:_next_label [ -x ] [ -12VJ ] tag name descr [ options ... ]:
    hoge

:_normal:
    hoge

:_options:
    hoge

:_options_set and _options_unset:
    hoge

:_parameters:
    hoge

:_path_files:
    hoge

:_pick_variant [ -b builtin-label ] [ -c command ] [ -r name ]
  label=pattern ... label [ args ... ]:
    hoge

:_regex_arguments name spec ...:
    hoge

:_regex_words tag description spec ...:
    hoge

:_requested [ -x ] [ -12VJ ] tag [ name descr [ command args ...] ]:
    hoge

:_retrieve_cache cache_identifier:
    hoge

:_sep_parts:
    hoge

:_setup tag [ group ]:
    hoge

:_store_cache cache_identifier params ...:
    hoge

:_tags [ [ -C name ] tags ... ]:
    hoge

:_values [ -O name ] [ -s sep ] [ -S sep ] [ -wC ] desc spec ...:
    hoge

:_wanted [ -x ] [ -C name ] [ -12VJ ] tag name descr command args ...:
    hoge


.. _completion directories:

ディレクトリの補完
==================

hoge

:Base:
    hoge

:Zsh:
    hoge

:Unix:
    hoge

:X, AIX, BSD, ...:
    hoge


.. END
