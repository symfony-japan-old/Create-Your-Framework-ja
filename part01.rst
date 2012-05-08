Web フレームワークをつくろう - Symfony コンポーネントの上に (パート 1)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_

Symfony2 は単体で独立していて、疎結合され、凝縮された PHP コンポーネントの集まりで、Web 開発の共通の問題を解決します。

これらの低水準のコンポーネントに取り組む代わりに、フルスタックフレームワークの Symfony2 をすぐに使い始めることができます。これは先ほどあげた Symfony2 のコンポーネントがもとになっています。もしくはまったくあなた独自のフレームワークを作ることもできます。この連載は後者に関するものです。

.. note::

    フルスタックフレームワークの Symfony2 をふつうに使うだけなら、公式 `ドキュメント`_ を読むとよいでしょう。

なぜあなたは独自のフレームワークを作りたいのでしょうか？
---------------------------------------------------------

まず第一に、なぜ独自のフレームワークを作りたいのでしょうか？周りを見回すと、誰もが車輪を再発明することはよくないことで、既存のフレームワークを選び、独自のフレームワークを作ることは忘れたほうがよいと言うでしょう。大抵の場合、それは正しいことですが、独自のフレームワークを作り始めるのによい理由がいくつか挙げることができます。

* モダンな Web フレームワークの低水準のアーキテクチャおよびとりわけフルスタックフレームである Symfony2 の内部をよりくわしく学ぶため

* 非常に特殊なニーズのために適合したフレームワークを作るため (まずはあなたのニーズが本当に特殊なのか最初にご確認ください)

* 楽しみのためにフレームワークを作ることを試すため (学んで捨てるアプローチ)

* Web 開発の最新のベストプラクティスを一服必要とする古い/既存のアプリケーションをリファクタリングするために

* あなたが実際に自分自身でフレームワークを作ることができることを世の中に示すため (ただし少し労力が必要です)。

Web フレームワークの作成を通じて、一歩ずつ丁寧に優しく導きます。
それぞれのステップにおいて、そのまま使えて、独自のものを作り始めるための完全なフレームワークが手に入ります。シンプルなフレームワークで始めて、時間とともに機能が追加されてゆきます。結局のところ、フルフィーチャのフルスタックの Web フレームワークを手に入れることになります。

そしてもちろん、それぞれの段階で Symfony2 コンポーネントの一部についてくわしく学ぶ機会があります。

.. tip::

    この連載記事をすべて読む時間がない、もしくはすぐにアプリケーションを作り始めたいのであれば、 Symfony コンポーネントをもとにしたマイクロフレームワークである `Silex`_ を見るとよいでしょう。コードはよりスリムで Symfony2 コンポーネントのさまざまな特徴を活かしています。

モダンな Web フレームワークの多くは自分自身を MVC フレームワークと呼んでいます。ここでは MVC を説明しません。Symfony2 コンポーネントを使えば、MVC アーキテクチャに従うフレームワークだけでなく、任意のフレームワークを作ることができるからです。ともかく、MVC セマンティックスを見ると、この連載はフレームワークの Controller に当たる部分の作り方に関するものです。Model と View に関して、本当に個人の好みの問題であり、既存のサードパーティを使っていただくことになります (Doctrine、Propel、もしくは Model のための簡素な PDO。View に関しては PHP もしくは Twig)。

フレームワークを作成するとき、MVC パターンに従うことは正しいゴールではありません。メインとなるゴールは「関心ごとの分離」です。本当に大切すべきであるもののデザインパターンでしかないと筆者は考えております。Symfony2 コンポーネントの基本原則は HTTP の仕様を中心としています。ですので、これから作ろうとしているフレームワークをより正確に説明すると HTTP フレームワークもしくは Request/Response フレームワークと分類されるものです。

始める前に
-----------

フレームワークの作り方を読むことだけではじゅうぶんではありません。手順に従い、これから取り組む例題のコードを実際に入力しなければなりません。そのためには、必要なのは最新バージョンの PHP (5.3.8 もしくはそれ以降のバージョンがじゅうぶんによいものです)、Web サーバー (Apache や Nginx など)、PHP のじゅうぶんな知識とオブジェクト指向の理解です。

準備はできていますか？始めましょう。

ブートストラップ
-----------------

最初のフレームワークを作ることを考える前に、コードをどこに保存し、どのようにクラスの名前をつけ、外部の依存パッケージをどのように参照するかなどについて、約束ごとを話す必要があります。

フレームワークを保存するために、あなたのマシンのどこかにディレクトリを作りましょう。

.. code-block:: sh

    $ mkdir framework
    $ cd framework

コーディングスタンダード
~~~~~~~~~~~~~~~~~~~~~~~~~

コーディング標準とここで使われるものがなぜとてもひどいものなのかという論争を誰かが始める前に、一貫性のあるかぎり、問題ではないことを認めましょう。この本に関して、 `Symfony2 のコーディング標準`_ を使っています。

コンポーネントのインストール
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

われわれのフレームワークに必要な Symfony2 のコンポーネントをインストールするために、PHP プロジェクトの依存パッケージマネージャである `Composer`_ を使います。最初に、 ``composer.json`` ファイルに依存パッケージのリストを記載します。

.. code-block:: javascript

    {
        "require": {
            "symfony/class-loader": "2.1.*"
        }
    }

ここでは、Symfony2 のコンポーネントである ClassLoader のバージョン 2.1.0 とそれ以降に依存していることを Composer に伝えます。実際にプロジェクトの依存パッケージをインストールするには、composer のバイナリファイルをダウンロードして実行します。

.. code-block:: sh

    $ wget http://getcomposer.org/composer.phar
    $ # or
    $ curl -O http://getcomposer.org/composer.phar

    $ php composer.phar install

``install`` コマンドを実行した後で、新しい ``vendor/``
ディレクトリに Symfony2 の ClassLoader のコードが入っていることを確認しなければなりません。

.. note::

    Composer が一押しですが、コンポーネントのアーカイブもしくは Git のサブモジュールを利用して直接ダウンロードすることもできます。これはあなた次第です。

命名規約とオートロード
~~~~~~~~~~~~~~~~~~~~~~

われわれのクラスをすべて、 `オートロード`_ しようとしています。オートロードを利用しなければ、クラスが利用できるようになる前にクラスが定義されたファイルを読み込む必要があります。しかし、命名規約によっては、PHP にハードワークをさせることができます。

クラスの名前とオートロードに関する PHP のデファクトスタンダードである `PSR-0`_ に Symfony2 は従います。Symfony2 の ClassLoader コンポーネントはこの PSR-0 標準を実装するオートローダーを提供します。大抵の場合、プロジェクトのすべてのクラスをオートロードするために必要なのは Symfony2 の ClassLoader
だけです。

``autoload.php`` ファイルの中で空のオートローダーを作ります。

.. code-block:: php

    <?php

    // framework/autoload.php

    require_once __DIR__.'/vendor/symfony/class-loader/Symfony/Component/ClassLoader/UniversalClassLoader.php';

    use Symfony\Component\ClassLoader\UniversalClassLoader;

    $loader = new UniversalClassLoader();
    $loader->register();

CLI で ``autoload.php`` を実行できます。これは何も行わず、エラーを投げることもしません。

.. code-block:: sh

    $ php autoload.php

.. tip::

    `ClassLoader`_
    コンポーネントに関するくわしい情報は Symfony の公式サイトで公開されています。

.. note::

    インストールしたすべての依存パッケージのために Composer はオートローダーを自動的に生成します。ClassLoader コンポーネントを利用する代わりに、 ``vendor/.composer/autoload.php`` を require 文でも読むこともができます。

われわれのプロジェクト
-----------------------

ゼロからフレームワークを作る代わりに、一度に1つの抽象化を加えながら、同じ「アプリケーション」を何度も書きます。PHP で考えられるもっともシンプルな Web アプリケーションを始めましょう ::

    <?php

    $input = $_GET['name'];

    printf('Hello %s', $input);

このシリーズの最初の部分はこれでおしまいです。次に、HttpFoundation コンポーネントを導入して何がもたらされるか見ることにします。

.. _`ドキュメント`:              http://symfony.com/doc
.. _`Silex`:                     http://silex.sensiolabs.org/
.. _`オートロード`:                  http://fr.php.net/autoload
.. _`Composer`:                  http://packagist.org/about-composer
.. _`PSR-0`:                     https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md
.. _`Symfony2 のコーディング標準`: http://symfony.com/doc/current/contributing/code/standards.html
.. _`ClassLoader`:               http://symfony.com/doc/current/components/class_loader.html
.. _`1`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part01.html
.. _`2`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part02.html
.. _`3`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part03.html
.. _`4`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part04.html
.. _`5`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part05.html
.. _`6`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part06.html
.. _`7`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part07.html
.. _`8`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part08.html
.. _`9`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part09.html
.. _`10`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part10.html
.. _`11`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part11.html
.. _`12`:    http://docs.symfony.gr.jp/symfony2/create-your-framework/part12.html

.. 2012/01/29 masakielastic db93254dea29d07acf1acd066029e5db0fdf33e6