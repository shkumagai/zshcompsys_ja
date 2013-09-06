
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

\-redirect- コンテキストの場合、追加情報は '-redirect,op,command'の形式に\
なっており、opの箇所はリダイレクト演算子、commandは行に対するコマンドの名前に\
なります。行に対するコマンドがまだ無い場合は、commnadフィールドは空になります。

\-value- コンテキストの場合、形式は '-value-,name,command' となり、nameは\
パラメータの名前になります。例えば、'assoc=(key <TAB>)'のように、連想配列の\
要素の場合、nameは'name-key'に展開されます。'make CFLAGS=' の後に補完をする\
ような、ある特殊なコンテキストではcommand部分は、コマンドの名前\
（ここではmake）を与えます。それ以外の場合は空です。

提供されている関数は'-default-'で要素を徐々に置き換えることによって、補完を\
生成しようとするので、特定の補完を完全に定義する必要はありません。
例えば、'foo=<TAB>'の後に補完をする時、_value は '-value-,foo'（コマンド部が\
空であることに注意してください）、'-value-,foo,-default-'、\
'-value-,-default-,-default-' というnamesを、この順番で、このコンテキストを\
処理する関数が見つかるまで試行します。

例えば::

    compdef '_files -g "*.log"' '-redirect-,2>,-default-'

は、特定のハンドラが定義されていない任意のコマンドに対して、'2> <TAB>' の\
後に '\*.log' に一致するファイル名を補完します。

また::

    compdef _foo -value-,-default-,-default-

は、_fooが特定の関数が定義されていないパラメータの値の補完を提供することを\
示しています。これは通常、_value関数そのもので処理されます。

スタイル（後述）を参照すると、同じ参照ルールが使用されています。例えば::

    zstyle ':completion:*:*:-redirect-,2>,*:*' file-patterns '*.log'

は '2> <TAB>' の後に '\*.log' にマッチするファイル名を補完させるための、\
もう一つの方法です。


.. _functions:

関数
----

以下の関数はcompinitによって定義され、直接呼び出すことができます。

| compdef [ -ane ] function names...  [ -[pP] ] patterns... [ -N names... ] ]
| compdef -d names...
| compdef -k [ -an ] function sytle key-sequences...
| compdef -K [ -an ] function name style key-sequences ...

    1つ目の形式は、前述の#compdefタグのように、与えられたコンテキストで\
    補完を行うために呼び出す関数を定義します。

    またすべての引数は'cmd=service'の形式を取ることができます。前述のように\
    ここではserviceが既に、#compdefファイルの中で'cmd1=servie'という行に\
    よって定義されている必要があります。cmdの引数はserviceと同じ方法で\
    保管されます。

    また関数の引数には、ほぼすべてのシェルコードを含む文字列を指定できます。
    文字列に等号が含まれている場合は、上記が優先されます。オプション-eは、\
    最初の引数に等号が含まれている場合でも、シェルコードとして評価すべきで\
    あることを指定するために使用します。文字列は、補完候補を生成するために\
    組み込みのevalコマンドを使って実行されます。これにより新しい補完関数を\
    定義する必要を回避できます。例えば、コマンドfooの引数として'.h'で終わる\
    ファイル名を補完する方法です::

        compdef '_files -g "*.h"' foo

    オプション-nは、既にコマンドやコンテキストに対して定義されている任意の\
    補完が上書きされるのを防ぎます。

    オプション-dは、コマンドやリストされたコンテキストに対して定義されて\
    いる任意の補完を削除します。

    #compdefタグで説明されているように、namesは-p、-Pおよび-Nオプションを\
    含みます。引数リストへの影響は同じで、最初に試行するパターン、最後に\
    試行するパターンおよび通常のコマンドまたはコンテキストの間で定義を\
    切り替えます。

    パラメータ $_compskipはpatternコンテキストに定義された任意の関数によって\
    設定できます。部分文字列'patterns'を含む値が設定されると、いずれの\
    パターン関数も呼び出されません; 部分文字列'all'を含む値が設定されている\
    場合は、他の関数も呼び出されません。

    \-kを持つ形式はそれぞれのキー・シーケンスに対して呼び出される関数と\
    同じ名前のウィジェットを定義します; これは #compdef -k tag のような\
    ものです。この関数は必要に応じて補完候補を生成する必要があり、そうでない\
    場合はスタイル引数として与えられている名前のウィジェットと同じように\
    振る舞います。これに使えるウィジェットは以下のとおり: complete-word, \
    delete-char-or-list, expand-or-complete, expand-or-complete-prefix, \
    list-choices, menu-complete, menu-expand-or-complete, および\
    reverse-menu-completeで、zsh/complistモジュールが読み込まれている場合は\
    menu-selectも同様です。オプション-nは、既に未定義キーを除いて何かに\
    バインドされているキーに対してバインドされることを防ぎます。

    \-Kを持つ形式も似ていて、同じ関数に複数のウィジェットを定義します。
    これらはそれぞれ name、style、key-sequence の3つの引数を必要とし、後の\
    2つは-kと同様で、最初の引数はアンダースコアで始まる固有のウィジェット名\
    でなければなりません。

    オプション-aは、どこで適用されても autoload -U function と同様に\
    関数を自動読み込み可能にします。

関数compdefは、新しいコマンドと既存の補完関数を関連付けるために使えます。
例えば、::

    compdef _pids foo

は、コマンドfooに対して、プロセスIDを補完する_pids関数を使います。

後述する_gnu_generic関数は、'--help'オプションを解釈するコマンドのオプション\
を補完するために使うことができるので、こちらも気に留めておいてください。


.. _completion system configuration:

補完システム構成
================

このセクションでは、補完システムがどのように機能するかの簡単な概要と、ユーザ\
がどのように設定するかや、いつ、どのようにマッチが生成されるかの詳細について\
解説します。

.. _overview:

概要
----

コマンドラインのどこかを補完しようとした時、補完システムは最初のコンテキスト\
が動作します。これは（例えば'grep'や'zsh'のような）コマンドワードや、（引数\
としてシェルオプションを取るzshの'-o'オプションのように）カレントワードへの\
オプションは引数があるかといった、いくつかの事柄を考慮します。

このコンテキスト情報は、コロンで区切られた複数のフィールドから構成される\
文字列に凝縮され、ドキュメントの後半では単に'コンテキスト'と呼ばれます。
これはスタイルや、補完システムを構成するために使用可能なコンテキスト依存の\
オプションを参照するために使われます。検索に使われるコンテキストは、補完\
システムへの同一の呼び出しの間であっても異なる場合があります。

コンテキスト文字列は常にフィールドの固定したセットと、コロン区切りで、先頭に\
コロンを持つ形 :completion:function:completer:command:argument:tag で構成\
されます。これらは次のような意味があります:

* リテラル文字列 completion はこのスタイルが補完システムに使われることを\
  表します。これは例えば、zleウィジェットとZFTP関数によって使用される\
  コンテキストを区別します。

* function は、通常の補完システムではなく、名前付きウィジェットから呼び\
  だされた場合の関数です。これは通常空白になっていて、predict-onのような\
  特殊なウィジェットやディストリビューションのウィジェットディレクトリに\
  含まれる様々な関数によってその関数の名前を設定されますが、しばしば省略\
  されます。

* completer は現在アクティブの、先頭のアンダースコアがなく、他のアンダー\
  スコアをハイフンに置き換えた関数の名前です。'completer'は補完が行われる\
  方法の全体的な制御です; 'complete'が最もシンプルですが、他のコンプリータは\
  誤り訂正や、以降のコンプリータの振る舞いを変更するなどのような、関連した\
  タスクを実行するために存在します。
  詳細については、'Control Functions' のセクションを参照してください。

* command または特別な -context- は、単に#compdefタグやcompdef関数の後ろに\
  表示されるものです。サブコマンドを持つコマンドの補完関数は、通常この\
  フィールドをコマンド名の後ろにマイナス記号とサブコマンド名が続く名前に\
  変更します。例えば、cvsコマンドの補完関数は、addサブコマンドに引数を\
  補完する際、このフィールドにcvs-addを設定します。

* argument は、補完しようとしているコマンドラインまたはオプション引数を\
  示します。コマンド引数の場合、これは一般的にはargument-nの形式をとり、\
  nには引数の数が入りますし、オプションの引数の場合は、option-opt-nの\
  形式で、nはオプションの引数optの数が入ります。しかしこれはコマンド\
  ラインが標準UNIXスタイルのオプションや引数で解析されている場合のみの\
  ため、ほとんどの補完ではこれを設定していません。

* tag です。既に説明したように、タグはあるコンテキストにおいて補完関数が\
  生成できるマッチの種類を区別するために使用されます。いずれの補完関数も\
  任意に好みのタグ名を使えますが、より一般的なものの一覧を以下に示します。

コンテキストは、:completion:と必要に応じて関数の要素が追加された、メイン\
エントリポイントで始まり、関数が実行されるにつれて徐々に組み立てられてます。
completerはその後コンプリータ要素が追加されます。コンテキスト補完は、\
コマンドと引数のオプションを追加します。最後に、補完の種類が分かっている\
場合にはタグが追加されます。例えば、コンテキスト名 ::

    :completion::complete:dvips:option-o-1:files

は、通常の補完がコマンドdvipsの-oオプションの第一引数として試行されたこと\
を表し、::

    dvips -o ...

補完関数はファイル名を生成します。

通常、補完は補完関数によって指定された順番で、すべての利用可能なタグに\
対して試行されます。ですがこれは tag-order スタイルを使って変更できます。
それ以降、補完は所定の順序で与えられたタグのリストに制限されます。

バインド可能コマンド _complete_help は、ある特定の時点での補完に使える\
すべてのコンテキストとタグを表示します。これで tag-order やその他のスタイル\
のための情報を見つけることができます。後述する `バインド可能コマンド`_ の\
セクションに記載されています。

スタイルは一致するものがどのように生成されるか、といったものを決定します。
シェルのオプションと同様ですが、はるかに多くのことを制御します。スタイルは\
値として任意の数の文字列を持つことができます。これらはzsh組み込みコマンドに\
よって定義されています（see zshmodules(1)）。

スタイルを参照する際、補完システムはタグを含めた完全なコンテキスト名を使い\
ます。スタイルの値を参照することは、すなわち2つの要素で構成されます: \
パターンとしてマッチするコンテキストと、正確に与えられたスタイル名そのもの\
です。

多くの補完関数は、単純かつ冗長なマッチを生成したり、使用すべき形式を\
決めるために冗長なスタイルを使うことができます。このような関数がすべて\
冗長な形式を使うようにするには、 ::

    zstyle ':completion:*' verbose yes

をスタートアップファイル（恐らく、.zshrc）に記述します。これはコンテキストが\
より具体的な定義を持っていない限り、補完システム内のすべてのコンテキストで\
verboseスタイルにyesを設定します。スタイルが補完システムの外で何らかの意味を\
持っている場合には、'\*'のようなコンテキストを渡すのは避けるのがベストです。

このような汎用スタイルの多くは、compinstall関数を使って簡単に構成できます。

冗長なスタイルの使用の、より具体的な例は、組み込みの kill コマンドの補完\
です。スタイルが設定されている場合、組み込み関数は完全なジョブ文字列と\
プロセスコマンドラインをリストします; そうでない場合は、単にジョブ番号と\
PIDを表示します。スタイルをoffにするにはこうするだけです::

    zstyle ':completion:*:*:kill:*' verbose no

更に制御するために、スタイルには'jobs'または'processes'を使えます。jobsに\
対してのみ冗長表示をoffにするには::

    zstyle ':completion:*:*:kill:*:jobs' verbose no

zstyleの-eオプションも補完関数のコードがスタイルの引数として表示されること\
を可能にしますが; これには補完関数内部（zshcompwid(1)を参照）のいくつかの\
理解が必要です。例えば ::

    zstyle -e ':completion:*' hosts 'reply=($myhosts)'

これはホスト名が必要になるたびに変数myhostsが読まれるように、hostsスタイルの\
値を強制します; myhostsの値が動的に変更できる場合、これは便利です。
別の有用な例としては、後述のfile-listスタイルの説明にある例を参照して\
ください。この形式は遅くなることがあり、menuやlist-rows-firstのように\
一般的に試されるスタイルに対しては避けるべきです。

スタイルが定義された順序は問題ではないという点に注意してください。スタイルの\
メカニズムは値のセットを決定するために、あるスタイルに対して最も特定可能な\
一致（the most specific possible match）を用います。より正確にいえば、\
文字列はパターンよりも好ましく（例えば':completion::complete:foo'は\
':completion::complete:'よりも具体的です）、長いパターンはより短いパターン\
よりも好ましいのです。

スタイル名は、タグのそれらと同様に任意であり、補完関数に依存します。
ですが、次の2つのセクションでは最も一般的なタグやスタイルをいくつか\
リストアップします。


.. _standard tags:

標準タグ
--------

以下のうち、いくつかは特定のスタイルを参照する場合にのみ使われるものです。
また、マッチの種類を示すものではありあｍせん。

accounts
    users-hosts スタイルを参照するために使用

all-expanstions
    すべての有効な展開を含む単一の文字列を追加するときに、_expandコンプ\
    リータによって使用

all-files
    すべてのファイルの名前（特定のサブセットとは区別されます。globbed-files\
    タグを参照してください）

arguments
    コマンドの引数

arrays
    配列パラメータの名前

assosiation-keys
    連想配列のキー; この型のパラメータの添え字を補完する際に使用

bookmarks
    ブックマークを補完する場合（例: URLやzftp関数スイートなど向け）

builtin
    組み込みコマンドの名前

characters
    sttyのようなコマンドの引数内の単一文字。
    左ブラケットの後の文字クラスを補完する場合にも使用

colormapids
    X colormapのID

colors
    色名

commands
    外部コマンドの名前。cvsのように複雑なコマンドのサブコマンドを補完する\
    場合にも使用

contexts
    zstyle組み込みコマンドの引数内のコンテキスト

corrections
    有効な誤り訂正のために_approximateと_correctコンプリータが使用

cursors
    Xプログラムが使用するカーソル名

default
    あるコンテキストにおいて、より具体的なタグも有効な時に、デフォルト値を\
    提供するために使用。
    このタグはコンテキスト名のfunctionフィールドが設定されている場合にのみ\
    使用されることに注意

descriptions
    マッチ候補の種類の説明を生成するためのフォーマットスタイルの値を\
    参照する時に使用

devices
    デバイス特殊ファイルの名前

directories
    ディレクトリ名 -- cdpath 配列が設定されている場合に、cdや関連する\
    組み込みコマンドの引数を補完する時は、代わりにlocal-directoriesを使用

directory-stack
    ディレクトリスタックのエントリ

displays
    Xディスプレイ名

domains
    ネットワークドメイン

expansions
    コマンドライン上のワードの展開に起因した（展開の完全なセットではなく）\
    個々のワードの補完のために_expandコンプリータが使用

extensions
    Xサーバ拡張

file-descriptors
    オープンしているファイル記述子の番号

files
    ファイル名を補完する関数によって使用される汎用のfile-matchingタグ

fonts
    Xフォント名

fstypes
    ファイルシステムタイプ（例: mountコマンド向け）

functions
    関数名 -- 特定のコマンドは他の種類の関数を理解できるかもしれないが、\
    通常はシェル関数

globbed-files
    パターンマッチによって生成されたファイル名

groups
    ユーザグループ名

history-words
    履歴内のワード

hosts
    ホスト名

indexes
    配列のインデックス

jobs
    （組み込みのjobsによってリストアップされる）ジョブ

interfaces
    ネットワーク・インターフェース

keymaps
    zshキーマップ名

keysyms
    X keysyms名

libraries
    システムライブラリ名

limits
    システムリミット

local-directories
    cdや関連する組み込みコマンド（path-directoriesと比べて下さい）の引数を\
    補完する場合の、現在の作業ディレクトリのサブディレクトリに当たる\
    ディレクトリ名 -- cdpath配列が設定されていない場合は、代わりに\
    directoriesを使用

manuals
    マニュアルページ名

mailboxes
    e-mailフォルダ

maps
    マップ名（例: NISマップ）

messages
    メッセージのフォーマット形式を参照するために使用

modifiers
    X modifier名

modules
    モジュール（例: zshモジュール）

my-accounts
    users-hostsスタイルを参照するために使用

named-directories
    指名されたディレクトリ（予想していなかったのではないでしょうか？）

names
    すべての名前

newsgroups
    USENETグループ

nicknames
    NISマップのニックネーム

options
    コマンドオプション

original
    一致候補として元の文字列を返す場合に、_approximate、_correntおよび\
    _expandが使用

other-accounts
    users-hostsスタイルを参照するために使用

other-files
    ディレクトリではないファイルの名前。これはlist-dirs-firstが有効である\
    場合に、all-filesの代わりに使用される

packages
    パッケージ（例: rpmやインストールされたDebianパッケージ）

parameters
    パラメータ名

path-directories
    cdや関連する組み込みコマンド（local-directoriesと比べてください）の\
    引数を補完する際に、cdpath配列を検索して見つかったディレクトリの名前

paths
    expand、ambiguousおよびspecial-dirsスタイルの値を参照するために使用

pods
    perlのPOD（ドキュメントファイル）

ports
    通信ポート

prefixes
    （URLのそれのような）プレフィックス

printers
    プリンタキュー名

processes
    プロセス識別子

process-names
    killallのためにプロセスの名前を生成する際に、commandスタイルを参照する\
    ために使用

sequences
    シーケンス（例: mhシーケンス）

sessions
    zftp関数スイートのセッション

signals
    シグナル名

strings
    文字列（例: cd組み込みコマンド向けの置換文字列）

styles
    zstyle組み込みコマンドが使用するスタイル

suffixes
    ファイル名拡張子

tags
    タグ（例: rpmタグ）

targets
    makefileターゲット

time-zones
    タイムゾーン（例: TZパラメータを設定する場合など）

types
    何らかのタイプ（例: xhostコマンド用のアドレスタイプなど）

urls
    URLを補完する際に、urlsおよびlocalスタイルを参照するために使用

users
    ユーザ名

values
    特定のリストの値のセットのいずれか

variant
    特定のコマンド名に対して、どんなプログラムがインストールされているかを\
    決定する際に実行するコマンドを参照するために_pick_variantが使用

visuals
    Xヴィジュアル

warnings
    警告のフォーマット形式を参照するために使用

widgets
    zshウィジェット名

windows
    XウィンドウのID

zsh-options
    シェルオプション


.. _standard styles:

標準スタイル
------------

これらのスタイルのうち、いくつかの値はブール値を表すことに注意してください。
文字列'true'、'on'、'yes'および'1'のいずれかを'true'値として使うことが\
でき、文字列'false'、'off'、'no'および'0'のいずれかを'false'値に使えます。
その他の値の振る舞いは、明示的に言及されている場合を除いて未定義です。
スタイルが設定されていない場合のデフォルト値は、trueまたはfalseのいずれか\
です。

これらのスタイルのうち、いくつかはマッチの種類に対応するすべての有効なタグ\
に対してまず試行され、スタイルが定義されていない場合はdefaultタグに対して\
試行されます。このタイプの最も顕著なスタイルには、list-packedやlast-prompt\
のような補完リストを制御するmenu、list-colorsそしてstylesがあります。
defaultタグに対して試行されると::

    zstyle ':completion:*:default' menu ...

に似た形で、defaultタグを使うスタイルが正常に定義できるよう、コンテキスト\
のfunctionフィールドにだけ値が設定されます。

accept-exact
    これは現在のコンテキストに対して妥当なタグに加えて、defaultタグに\
    対しても試行されます。これが'true'に設定され、トライアルマッチが\
    コマンドライン上の文字列と同じである場合、（たとえ、それ以外の場合に\
    は曖昧だと考えられるとしても）一致候補はすぐに受け入れられます。

    パス名を補完（使用されるタグは'paths'）すると、このスタイルはブール値に\
    加えて任意の数のパターンを値として受け入れます。これらのパターンの\
    いずれかにマッチするパス名は、コマンドラインがいくつか部分的に入力された\
    パスの構成要素が含まれていて、受け入れられたディレクトリの下にファイルが\
    存在しなかったとしても受け入れられます。

    このスタイルはチルダやパラメータ展開で始まるワードが展開されるべきかを\
    _expandコンプリータが判断するためにも使われます。例えば、パラメータ\
    'foo'と'foobar'がある場合、accept-exactが'true'に設定されていれば、\
    文字列'$foo'だけが展開されます; そうでない場合、補完システムは\$fooを
    \$foobarに展開することができます。スタイルの値が'continue'に設定されて\
    いる場合、_expandは展開をマッチとして追加し、補完システムは継続すること\
    ができます。。

accept-exact-dirs
    hoge

add-space
    hoge

ambiguous
    hoge

assign-list
    hoge

auto-description
    hoge

avoid-completer
    hoge

cache-path
    hoge

cache-policy
    hoge

call-command
    hoge

command
    hoge

command-path
    hoge

commands
    hoge

complete
    hoge

complete-options
    hoge

completer
    hoge

condition
    hoge

delimiters
    hoge

disabled
    hoge

domains
    hoge

environ
    hoge

expand
    hoge

fake
    hoge

fake-always
    hoge

fake-files
    hoge

fake-parameters
    hoge

fake-list
    hoge

file-patterns
    hoge

file-sort
    hoge

filter
    hoge

force-list
    hoge

format
    hoge

glob
    hoge

global
    hoge

group-name
    hoge

group-order
    hoge

groups
    hoge

hidden
    hoge

hosts
    hoge

hosts-ports
    hoge

ignore-line
    hoge

ignore-parents
    hoge

extra-verbose
    hoge

ignore-patterns
    hoge

insert
    hoge

insert-ids
    hoge

insert-tab
    hoge

insert-unambiguous
    hoge

keep-prefix
    hoge

last-prompt
    hoge

known-hosts-files
    hoge

list
    hoge

list-colors
    hoge

list-dirs-first
    hoge

list-groups
    hoge

list-packed
    hoge

list-prompt
    hoge

list-rows-first
    hoge

list-suffixes
    hoge

list-separator
    hoge

local
    hoge

mail-directory
    hoge

match-original
    hoge

matcher
    hoge

matcher-list
    hoge

max-errors
    hoge

max-matches-width
    hoge

menu
    hoge

muttrc
    hoge

numbers
    hoge

old-list
    hoge

old-matches
    hoge

old-menu
    hoge

original
    hoge

packageset
    hoge

path
    hoge

path-completion
    hoge

pine-directory
    hoge

ports
    hoge

prefix-hidden
    hoge

prefix-needed
    hoge

preserve-prefix
    hoge

range
    hoge

regular
    hoge

rehash
    hoge

remote-access
    hoge

remote-all-dups
    hoge

select-prompt
    hoge

select-scroll
    hoge

separate-sections
    hoge

show-completer
    hoge

single-ifnored
    hoge

sort
    hoge

special-dirs
    hoge

squeeze-slashes
    hoge

stop
    hoge

strip-comments
    hoge

subst-globs-only
    hoge

substitute
    hoge

suffix
    hoge

tag-order
    hoge

urls
    hoge

use-cache
    hoge

use-compctl
    hoge

use-ip
    hoge

users
    hoge

users-hosts
    hoge

users-hosts-ports
    hoge

verbose
    hoge

word
    hoge


.. _control functions:

操作関数
========

_all_matches
    hoge

_appriximate
    hoge

_complete
    hoge

_correct
    hoge

_expand
    hoge

_expand_alias
    hoge

_history
    hgoe

_ignored
    hoge

_list
    hoge

_match
    hoge

_menu
    hoge

_oldlist
    hoge

_prefix
    hoge

_user_expand
    hoge

    :$hash:
        fuga

    :_func:
        moga


.. _bindable commands:

バインド可能コマンド
====================

_bash_completions
    hoge

_correct_filename (^XC)
    hoge

_correct_word (^Xc)
    hoge

_expand_alias (^Xa)
    hoge

_expand_word (^Xe)
    hoge

_generic
    hoge

_history_complete_word (\\e/)
    hoge

_most_recent_file (^Xm)
    hoge

_next_tags (^Xn)
    hoge

_read_comp (^X^R)
    hoge

_complete_debug (^X?)
    hoge

_complete_help (^Xh)
    hoge

_complete_help_generic
    hoge

_complete_tag (^Xt)
    hoge


.. _utility functions:

ユーティリティ関数
==================

_all_labels [ -x ] [ -12VJ ] tag name descr [ command args ... ]
    hoge

_alternative [ -O name ] [ -C name ] spec ...
    hoge

_arguments [ -nswWACRS ] [ -O name ] [ -M matchspec ] [ \: ] spec ...

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


_cache_invalid cache_identifier
    hoge

_call_function return name [ args ... ]
    hoge

_call_program tag string
    hoge

_combination [ -s pattern ] tag style spec ... field opts ...
    hoge

_describe [ -oO | -t tag ] descr name1 [ name2 ] opts ... -- ...
    hoge

_description [ -x ] [ -12VJ ] tag name descr [ spec ... ]
    hoge

_dispatch context string ...
    hoge

_files
    hoge

_gnu_generic
    hoge

_guard [ options ] pattern descr
    hoge

| _message [ -r12 ] [ -VJ group ] descr
| _message -e [ tag ] descr

    hoge

_multi_parts sep array
    hoge

_next_label [ -x ] [ -12VJ ] tag name descr [ options ... ]
    hoge

_normal
    hoge

_options
    hoge

_options_set and _options_unset
    hoge

_parameters
    hoge

_path_files
    hoge

:_pick_variant [ -b builtin-label ] [ -c command ] [ -r name ]
  label=pattern ... label [ args ... ]:
    hoge

_regex_arguments name spec ...
    hoge

_regex_words tag description spec ...
    hoge

_requested [ -x ] [ -12VJ ] tag [ name descr [ command args ...] ]
    hoge

_retrieve_cache cache_identifier
    hoge

_sep_parts
    hoge

_setup tag [ group ]
    hoge

_store_cache cache_identifier params ...
    hoge

_tags [ [ -C name ] tags ... ]
    hoge

_values [ -O name ] [ -s sep ] [ -S sep ] [ -wC ] desc spec ...
    hoge

_wanted [ -x ] [ -C name ] [ -12VJ ] tag name descr command args ...
    hoge


.. _completion directories:

ディレクトリの補完
==================

hoge

Base
    hoge

Zsh
    hoge

Unix
    hoge

X, AIX, BSD, ...
    hoge


.. END
