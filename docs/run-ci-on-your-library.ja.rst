How to run CI on your library for competitive programming (Japanese)
====================================================================

あなたが競技プログラマなら、おそらく自分用の競技プログラミング用のライブラリを内製していることでしょう。
そのようなライブラリには「そのライブラリを使った提出が既存の問題で AC することを確認する」という形の手動でのテストが行われるのが通常です。

しかしこれはとても面倒な作業です。
そして、単に忘れてしまったり、「修正は 1 行だけなので提出してテストしなくてもいいかな」などと言い訳して省略されてしまったりします。
そのようなライブラリにはきっとバグが潜んでいることでしょう。

ではどうすればよいでしょうか？
もちろんこれは自動化によって解決できます。


Test Script
-----------

online-judge-tools には、ライブラリのテストに利用できる機能があります。
それらを組み合わせればテストの自動化が可能です。

例えば以下のようなシェルスクリプトを書くことができます。
まず ``.test.cpp`` という拡張子を持つファイルを作り、その中で ``#define PROBLEM "http://judge.u-aizu.ac.jp/onlinejudge/description.jsp?id=DSL_2_A&lang=jp"`` のような形で問題を指定しておきます (例: `union_find_tree.test.cpp <https://github.com/kmyk/competitive-programming-library/blob/d4e35b5afe641bffb18cc2d6404fa1a67765b5ba/data_structure/union_find_tree.test.cpp>`_)。
するとこのスクリプトは、そのような拡張子のファイルを探し、自動でコードをコンパイルし、システムテストの入出力を自動で取得し、テストをしてくれます。
ローカルでの実行なのでサーバには比較的やさしいです。

.. code-block:: bash

   #!/bin/bash
   set -e
   oj --version

   CXX=${CXX:-g++}
   CXXFLAGS="${CXXFLAGS:--std=c++14 -O2 -Wall -g}"
   ulimit -s unlimited  # make the stack size unlimited

   # list files to test
   for file in $(find . -name \*.test.cpp) ; do

       # get the URL for verification
       url="$(sed -e 's/^# *define \+PROBLEM \+"\(https\?:\/\/.*\)"/\1/ ; t ; d' "$file")"
       if [[ -z ${url} ]] ; then
           continue
       fi

       dir=cache/$(echo -n "$url" | md5sum | sed 's/ .*//')
       mkdir -p ${dir}

       # download sample cases
       if [[ ! -e ${dir}/test ]] ; then
           sleep 2
           oj download --system "$url" -d ${dir}/test
       fi

       # run test
       $CXX $CXXFLAGS -I . -o ${dir}/a.out "$file"
       oj test --tle 10 --c ${dir}/a.out -d ${dir}/test
   done


このスクリプトはあくまで一例であり、「Python にも対応させたい」「差分だけテストしたい」などの要求がある場合は各々で拡張してください。
そのような拡張の一例として `online-judge-verify-helper <https://github.com/kmyk/online-judge-verify-helper>`_ があり、これを利用することもできます。


Continuous Integration
----------------------

「実行すると自動でテストしてくれるスクリプトがある」というだけではまだ不足です。
なぜならそのスクリプト自体は人間が手動で実行しなければならないためです。
この問題を解決するものは Continuous Integration (CI) と呼ばれます。

CI とは、自動で継続的にテストを行う仕組みや運用のことです。
GitHub のリポジトリにコミットが追加されるたびに自動でテストが実行されるようにしておく、というのはその例です。

もしあなたがライブラリを GitHub 上で管理しているのなら、 CI の導入は簡単です。
CI を実行してくれるサービスのひとつである Travis CI を利用しましょう。
Travis CI のページ https://travis-ci.org/ から登録してライブラリの GitHub レポジトリとの連携を有効化し、前節のシェルスクリプトを ``test.sh`` という名前で保存しているとき、以下の内容の ``.travis.yml`` というファイルを作れば終了です。

.. code-block:: yaml

   language: cpp
   compiler:
       - clang
       - gcc
   dist: xenial
   addons:
       apt:
           packages:
               - python3
               - python3-pip
               - python3-setuptools

   before_install:
       - pip3 install -U setuptools
       - pip3 install -U online-judge-tools=='7.*'
   script:
       - bash test.sh


自動で実行されたテスト結果は Travis CI 上のページ (例: https://travis-ci.org/kmyk/competitive-programming-library) などから見ることができます。
``https://img.shields.io/travis/USER/REPO/master.svg`` の形の URL から |badge| のようなバッジを生成できるので、これを README に貼っておくのもよいでしょう。
このバッチはテストの成功失敗に応じて色が勝手に変化します。

.. |badge| image:: https://img.shields.io/travis/kmyk/competitive-programming-library/master.svg
   :target: https://travis-ci.org/kmyk/competitive-programming-library

(注意: この節は GitHub Actions が public release される前に書かれました。現在では Travis CI でなく GitHub Actions を使ってみてもよいかもしれません。)


Examples
--------

onlinejudge-tools がライブラリの verify に使われている例として次のふたつを挙げておきます。現在はどちらも `online-judge-verify-helper <https://github.com/kmyk/online-judge-verify-helper>`_ を利用しています。

- https://github.com/kmyk/competitive-programming-library
- https://github.com/beet-aizu/library

他にも CI を回している競プロライブラリはあり、例えば以下が知られています。

- https://github.com/asi1024/competitive-library
- https://github.com/blue-jam/ProconLibrary
