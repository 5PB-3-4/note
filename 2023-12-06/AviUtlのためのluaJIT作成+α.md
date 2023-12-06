AviUtlのためのluaJIT作成 +α
===

## 初めに
Aviutl Advent Calendar 6日目です。こちらではお使いになっているAviUtlの`lua51.dll`を皆さんで作ってみよう！というお話です。
luaJITといえば最近までは**2.1.0-beta3** のバージョンが最新であったかと思いますが、今年の8月、6年ぶりに新リリースとなる**2.1.ROLLING**バージョンが登場しました。今回はこのバージョンを作ってみようと思います。ついでに、スクリプト作成者が役立つと思うモジュールを一つビルドし、使ってみたいと思います。

<br><br>

## AviUtl中のLua
AviUtlでは「スクリプト」に使われている言語として**Lua5.1**を利用しています。LuaJITはLua5.1に互換性があるため、AviUtlでも使うことができるわけです。また、一般にLuaJITはLua5.1よりも速く動作することが期待されるため、「AviUtl スクリプト 高速化」といった検索をかけるとlua51.dllをluaJITのものに置き換えるといった内容が多く見られるのだと思います。
さて、2023年12月現在でLuaJITをダウンロードできるリンクはいくつか紹介されています。私も含め、利用している方も多いのではないでしょうか？[こちら](https://scrapbox.io/ePi5131/LuaJIT_(ePi%E3%83%93%E3%83%AB%E3%83%89))とか[こちら](https://github.com/Per-Terra/LuaJIT-Auto-Builds/releases)とか。
ですが、[LuaJIT](https://luajit.org/download.html)公式にはこう書かれています。

<b><i>"Please do not use obsolete versions from older tarballs or zip files. Please remove any outdated links to these downloads — these links will cease to work soon.

Do not use pseudo-releases or tarballs created by third parties. Do not use binaries offered for download by third parties.

Pre-built packages should only be installed via a trusted package manager for your operating system (distro). But you should be aware these often carry old versions that miss important fixes. Before reporting an issue, always try the latest version available from the git repository."</i></b>

まあ、自己責任ではありますけれども、LuaJIT公式としては古いものや第三者のリリースの利用はやめてほしいそうです。となりますと、実際に先に上げたリンク先を勧めるのは少し配慮した方がいいように感じます。では、作り方を紹介して各々がビルドすれば良いのではと思った次第でこの記事を書きました。

<br><br>

## LuaJITの作り方
では実際にLuaJITを作ってみましょう。必要となるのは以下のソフトウェアたちです
- MSYS2 (MINGW32を使います)
- LuaJITのソースコード

> mingw32のパッケージについて
>> MSYS2のライブラリにLuaJITがあり、32bit版のlua51.dllも手に入ります。ただし、mingw32のlua51.dllは**libgcc_s_dw2-1.dll**に依存します。依存というのはこのlua51.dllをAviUtlで使う場合は、aviutl.exeがlibgcc_s_dw2-1.dllを読み込める位置に置いておかないといけないという意味です。また、libgcc_s_dw2-1.dllは**libwinpthread-1.dll**に依存します。そのため、lua51.dllの利用のために余分にdllが必要になります。今回のビルドでは、この2つのdllに依存しないlua51.dllを作成します。

<br>

まず、MSYS2をインストールし、mingw32でビルドできる環境を設定します。インストールコマンドは次の通りです
```shell
# このコマンドは2回くらい行ってください。あとY/Nなどと聞かれたら y といれてEnterキーを押したり、デフォルト=ALL？と聞かれたら何も入れずにEnterキーで大丈夫です。
# すでに環境があるという方はそちらをお使いになっても大丈夫です。
pacman -Syuu

# gccやgitをインストールします
pacman -S base-devel mingw-w64-i686-toolchain git

# gccが入っているかは、mingw32のコンソール画面で以下のコマンドを入力(10以上になっていることを推奨します)
gcc --version
```

<br>

必要な方は、環境変数を設定します。環境変数の設定についてはいろんな方が紹介されているので省略しますが、「環境変数を編集」と検索をかけて出てくるGUIの中で設定したり、exportコマンドなどで行うことができるかと思います。私は、ユーザー環境変数の**Path**の中にこんな感じの文を追加しています。
```
C:\msys64\mingw32\bin\
C:\msys64\mingw32\include\
C:\msys64\mingw32\lib\
```

<br>

さてこれからはmingw32のコンソール画面で行います。まず、luaJItのリポジトリをダウンロードしてきます。releaseの中から2.1.ROLLINGの`Source code(zip)`などから取ってくるか`git`を使って持ってきたりします

```
git clone https://github.com/LuaJIT/LuaJIT.git
```

持ってきたフォルダを解凍して入るといくつかのファイルとフォルダがあるかと思います。
```shell
cd LuaJIT/
ls
```
この中で、`Makefile`を編集します。srcディレクトリ内にも同じ名前のファイルがあり、こちらの方の`Makefile`（以下、srcディレクトリの内外にあるMakefileをそれぞれ内Makefileと外Makefileと区別表記します）も編集するので、注意してください。


では、外MakefileをVSCodeなどのエディタを使って開き、次の行を変更します。

|行 | 変更前 | 変更後 |
|-------|-------|-------|
|33|`export PREFIX= /usr/local`|`export PREFIX= <保存先のフォルダパス>`|

保存先のフォルダパスは`\`を使わず、`\\`や`/`に変換してください。またwindows式の`C:/~~~`だけでなく、MSYS2式のパスである`/home/<ユーザー名>/~~~`も利用可能です。私の場合は、外Makefileと同じ階層にlocalという名前のディレクトリを作り、そこにインストールしました。

```shell
export PREFIX= /home/<ユーザー名>/LuaJIT/local
```

次に内Makefileを編集します。
|行 | 変更前 | 変更後 |
|-------|-------|-------|
|29|`CC= $(DEFAULT_CC)`|`CC= $(DEFAULT_CC) -finput-charset=utf-8 -fexec-charset=cp932 -static`|
|38|`CCOPT= -O2 -fomit-frame-pointer`|`CCOPT= -Ofast -flto=auto -fomit-frame-pointer`|
|49|`CCOPT_x86= -march=i686 -msse -msse2 -mfpmath=sse`|`CCOPT_x86= -mtune=native -march=native -msse2 -mfpmath=sse`|

オプションの詳しい内容は[こちら](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html)をご覧ください。また、-Ofastについては複数のオプションが有効/無効となっていますのでこちらをご確認ください。
```shell
gcc -Ofast -Q --help=optimize
```

さて、カレントディレクトリが外Makefileにある状態(プロンプトに`pwd`と打ち込んで決定すると今の位置が分かります。)で以下のコマンドを実行します。

```shell
mingw32-make install
```
これで先ほど`export`で設定したフォルダ内にdllがインストール...されません（は？）。お目当ての`lua51.dll`は内Makefileと同じ**src**内にできます。お手数ですが、lua51.dllとluajit.exeを先ほどインストールしたフォルダ内の**bin**フォルダに、libluajit-5.1.dll.aファイルをインストールフォルダ内の**lib**フォルダ内に入れてください。

ちなみにこの状況からlua51.dllとluajit.exeがsrc内にない状況でvisual studioに付属しているx86 native command pronpt などを使ってsrc内の`msvcbuild.bat`を実行すると、visual studioビルドによるlua51.dllが作成できます。なお、おま環かもしれませんが、mingw32によるビルド実行後でないとヘッダーファイルの作成がうまいこといかず、lua51.dllを作成できませんでした。
```
ディレクトリ構成
C:\
└── msys64
    └── home
        └── <ユーザー名>
            └── LuaJIT
                ├── Makefile      # これが外Makefile
                ├── src
                |   ├── Makefile  # これが内Makefile
                |   ・・・
                ├── <PREFIX>
                |   ├── bin
                |   |   ├── lua51.dll
                |   |   ├── luajit.exe
                |   |   └── luajit-2.1..exe
                |   ├── include
                |   |   └── luajit-2.1
                |   |       ├── lauxlib.h
                |   |       ├── lua.h
                |   |       ・・・
                |   ├── lib
                │   |   ├── lua
                |   |   |   └─ 5.1
                |   |   |      └─ libluajit-5.1.dll.a
                |   |   └── pkgconfig
                |   └── share

```

Aviutlで利用する場合は、既存のlua51.dllを作成したdllに置き換えてください。

<br><br>

## luasocketの作成
ここからはスクリプト制作者向けに、スクリプトを作成する上で役立つsocketというライブラリを紹介します。luasocketはTCP/UDP通信やネットワークサーバの作成等ができる通信ライブラリですが、このライブラリの機能として時間計測があります。なんだ、luaで時間計測なら`os.clock()`でいいじゃないかと思われますが、測定精度が10ms程度まででそれ以上を測る場合はc++のdllを書いたり、luasocketを利用するのが考えられる手段かと思います（他に何かあれば教えてください。）。c++を書くのが面倒！という方もいらっしゃるかと思うので、luasocketのビルドを行い、より精度のよい計測を行ってみましょう！（精度としてはマイクロ秒以下も計測できます）。

mingw32の環境はluajitをビルドした後の環境をそのまま使います。まず、カレントディレクトリが`/home/<ユーザー名>/`の状態でluasocketのソースコードをgithubから持ってきます。zipを解凍してカレントディレクトリに置くか、以下のコマンドを使います
```shell
git clone https://github.com/lunarmodules/luasocket.git
```

持ってきたフォルダを解凍して入るといくつかのファイルとフォルダがあるかと思います。
```shell
cd luasocket
ls
```

こちらもluajitと同じくsrcディレクトリの内外にMakefileがあり、その両方を編集しないといけないので内Makefileと外Makefileの呼び方をこちらでも使います。まず、外Makefileを次のように編集します
|行 | 変更前 | 変更後 |
|-------|-------|-------|
|12|`PLAT?= linux`|`PLAT?= mingw`|

内Makefileを次のように編集します。
|行 | 変更前 | 変更後 |
|-------|-------|-------|
|17|`PLAT?=linux`|`PLAT?=mingw`|
|69|`LUAINC_mingw_base?=/usr/include`|`LUAINC_mingw_base?=C:/msys64/home/<ユーザー名>/LuaJIT/local/include`|
|70|`LUAINC_mingw?=$(LUAINC_mingw_base)/lua/$(LUAV) $(LUAINC_mingw_base)/lua$(LUAV)`|`LUAINC_mingw?=$(LUAINC_mingw_base)/luajit-2.1`|
|71|`LUALIB_mingw_base?=/usr/bin`|`LUALIB_mingw_base?=C:/msys64/home/<ユーザー名>/LuaJIT/local/bin`|
|72|`LUALIB_mingw?=$(LUALIB_mingw_base)/lua/$(LUAV)/lua$(subst .,,$(LUAV)).dll`|`LUALIB_mingw?=$(LUALIB_mingw_base)/lua$(subst .,,$(LUAV)).dll`|
|73|`LUAPREFIX_mingw?=/usr`|`LUAPREFIX_mingw?=<保存先のフォルダパス>`|
|74|`CDIR_mingw?=lua/$(LUAV)`|`CDIR_mingw?=<保存先のフォルダ名>`|
|75|`LDIR_mingw?=lua/$(LUAV)/lua`|`LDIR_mingw?=<保存先のフォルダ名>`|
|214|`CC_mingw=gcc`|`CC_mingw=gcc -finput-charset=utf-8 -fexec-charset=cp932 -static -mtune=native -march=native -msse2 -mfpmath=sse`|
|217|`CFLAGS_mingw=$(LUAINC:%=-I%) $(DEF) -Wall -O2 -fno-common`|`CFLAGS_mingw=$(LUAINC:%=-I%) $(DEF) -Wall -Ofast -flto=auto -fno-common`|
|218|`LDFLAGS_mingw= $(LUALIB) -shared -Wl,-s -lws2_32 -o`|`LDFLAGS_mingw= $(LUALIB) -Wall -Ofast -flto=auto -shared -Wl,-s -lws2_32 -o`|
|219|`LD_mingw=gcc`|`LD_mingw=$(CC_mingw)`|

たまに、**lua.hがない**といったエラーが出るんですが、その場合は、インストールしたluajitの**include**フォルダ内にあるファイルすべてをluasocketのsrc内にコピーしてください

LUAPREFIX_mingwとCFLAGS_mingwとLDFLAGS_mingwで指定したフォルダ内にインストールされます。具体的にはLUAPREFIX_mingwのフォルダの中に、CFLAGS_mingwで指定したフォルダに**lua**、LDFLAGS_mingwで指定したフォルダに**dll**ファイルがインストールされます。自分はluajitと同様にlocalファイルを作成し、そこにフォルダを設定しました。
```shell
LUAPREFIX_mingw?=C:/msys64/home/<ユーザー名>/luasocket/local
```

カレントディレクトリが外Makefileと同じところにいる状態で、以下のコマンドを実行します。
```shell
mingw32-make
mingw32-make install
```
```
ディレクトリ構成
C:\
└── msys64
    └── home
        └── <ユーザー名>
            └── luasocket
                ├── makefile       # これが外Makefile
                ├── src
                |   ├── makefile   # これが内Makefile
                |   ・・・
                ├── <LUAPREFIX_mingw>
                |   ├── <CDIR_mingw>
                |   |   ├── mine
                |   |   └── socket
                |   |       └── core.dll
                |   └── <LDIR_mingw>
                │       ├── socket
                |       |   ├── ftp.lua
                |       |   ├── headers.lua
                |       |   ├── http.lua
                |       |   ・・・
                |       ├── socket.lua
                |       ├── mine.lua
                |       └── ltn12.lua
                ・・・
```

Aviutlで利用する場合は、CDIR_mingwで指定したフォルダ内にあるsocketフォルダをフォルダごとaviutl.exeのあるフォルダへコピーしてください。スクリプトから参照する場合は.anmファイルのあるフォルダ内にsocketフォルダごとコピーしてください。読み込めているかどうかは次のコードをテキストオブジェクトにコピーしてください。
```Lua
<?
local checker = function()
  local s = require("socket.core")
  debug_print("socket version : "..s._VERSION)
  debug_print("luajit version : "..jit.version)
end
local isLoad,ret = pcall(checker)
if not isLoad then
  debug_print(ret)
end
?>

-- socket version : LuaSocket 3.1.0
-- luajit version : LuaJIT 2.1.ROLLING
```

patch.aulのコンソール画面にluasocketのバージョンが表示されていれば成功です。時間計測の仕方は`os.clock()`と同じで次のように書きます。
```Lua
<?
local isLoad, ret = pcall(require, "socket.core")
if not isLoad then
  debug_print([[failed]])
  debug_print(ret)
  return
end

local tstart=ret.gettime()

-- 行いたい処理をここで書く --

local tend=ret.gettime()
debug_print((tend-tstart).."[sec]")
collectgarbage("collect")
?>

-- 9.4280033111572[sec]
```

<br><br>

## おまけ
ここまでお付き合いいただきありがとうございました。実は、上記のluaJITとluasocketのmakefileの編集をすべて行っているmakefileを同リポジトリで配布中でございます（lua51.dllのコピーなども対策済みです）。この記事のあるリポジトリのluajitディレクトリ内にあるMakefileとsrcディレクトリをluajitのソースコードのディレクトリに、luasocketディレクトリ内にあるmakefileとsrcディレクトリをluasocketのソースコードのディレクトリに上書きコピーしてください。
ただし注意点がいくつかあります。
- luaJItとluasocketのディレクトリ名をすべて`git clone`によって持ってきた時の名前にしています。そのため、zipファイルからmsys2の読み込めるフォルダ(`C:/msys64/home/<ユーザー名>/`)へコピーする際、コピー先のフォルダ名を、Luajit-2.1やluaJIT-2.1.ROLLINGといった名前を**LuaJIT**、luasocket-masterやluasocket-3.1.0といった名前を**luasocket**に変更してください。なお`git clone`を用いる場合は変更しなくて大丈夫です。
- カレントディレクトリをそれぞれの外Makefileに移動した後は、luajitなら `mingw32-make install`、luasocketなら`mingw32-make && mingw32-make install`でビルドができるかと思います。
- windowsの環境変数に`USERNAME`がシステム変数にあることを確認してください。
- 私のmakefileによるインストール構成は以下のようになります。
```
ディレクトリ構成
C:\
└── msys64
    └── home
        └── <各自のPCユーザー名>
            ├── LuaJIT
            |   ├── Makefile      # これが外Makefile
            |   ├── src
            |   |   ├── Makefile  # これが内Makefile
            |   |   ・・・
            |   ├── local
            |   |   ├── bin
            |   |   |   ├── lua51.dll
            |   |   |   ├── luajit.exe
            |   |   |   └── luajit-2.1..exe
            |   |   ├── include
            |   |   |   └── luajit-2.1
            |   |   |       ├── lauxlib.h
            |   |   |       ├── lua.h
            |   |   |       ・・・
            |   |   ├── lib
            |   │   |   ├── lua
            |   |   |   |   └─ 5.1
            |   |   |   |      └─ libluajit-5.1.dll.a
            |   |   |   └── pkgconfig
            |   |   └── share
            |   ・・・
            |
            └── luasocket
                ├── makefile       # これが外Makefile
                ├── src
                |   ├── makefile   # これが内Makefile
                |   ・・・
                ├── local
                |   ├── core
                |   |   ├── mine
                |   |   └── socket
                |   |       └── core.dll
                |   └── lua
                │       ├── socket
                |       |   ├── ftp.lua
                |       |   ├── headers.lua
                |       |   ├── http.lua
                |       |   ・・・
                |       ├── socket.lua
                |       ├── mine.lua
                |       └── ltn12.lua
                ・・・
```


<br><br>

## 余談
- `luaconf.h`や`http.lua`ファイルなどを開発などで利用する場合は内部の値が正しいか、改行コードがLFのままで大丈夫か、文字コードがutf-8で問題ないかなど各自変更をお願いいたします。
- dllの依存についてはgccを導入すれば、lddコマンド等で調べることができます。自分は[外部ソフト](https://github.com/lucasg/Dependencies)を入れて調べています。
- socketによる時間計測を使用したスクリプトの作成例を配布しています。ベンチマーク代わりにでもご利用ください。
- ここまで書いてありますが、私はmakefileを自分で書くことができません（~~え？~~）。なので誤った情報を載せている可能性があります。
- 私の理解不足なのですが、mingw32のgccでビルドしたスクリプトについてluaにおいては`print`と`io.write`、c++については`std::cout`の出力結果の表示ができません。コマンドオプションとして`-finput-charset=utf-8 -fexec-charset=cp932`としても効果はなく、msvcとgccで同じプログラムをビルドし使用していますが、gccだけこの症状が出ています。これについてなにか知っている方、ご一報ください。
- この記事、ビルド、作成物につきまして、何らかの不都合が起きましても責任は負いかねますのでご了承ください。

<br><br>

## 最後に
luasocketの資料なさすぎ...

<br>

## 参照
- https://lunarmodules.github.io/luasocket/installation.html
- https://luajit.org/install.html
- http://arithmeticoverflow.blog.fc2.com/blog-entry-65.html
- https://siuncyclone.hatenablog.com/entry/2018/07/21/194629
- https://gist.github.com/Hamayama/eb4b4824ada3ac71beee0c9bb5fa546d
- http://www.furelo.jp/wordpress/?p=62
- http://verifiedby.me/adiary/0156