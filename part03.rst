Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 3)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


これまで、ページが1つしかなかったので、我々のアプリケーションは割り切ったものでした。少しおもしろくするために、頭をやわらかくして、Goodbye を伝える別のページを追加しましょう。::

    <?php

    // framework/bye.php

    require_once __DIR__.'/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();

    $response = new Response('Goodbye!');
    $response->send();

ご覧の通り、大半のコードは最初のページに対して書いたものとまったく同じです。すべてのページのあいだで共有できる共通のコードを抽出しましょう。コードのシェアリングは我々の最初の「実際の」フレームワークをつくるためのよいプランに見えます！

PHP 流のリファクタリングはおそらくはファイルのインクルードです。::

    <?php

    // framework/init.php

    require_once __DIR__.'/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();
    $response = new Response();

実際の例を見てみましょう。::

    <?php

    // framework/index.php

    require_once __DIR__.'/init.php';

    $input = $request->get('name', 'World');

    $response->setContent(sprintf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8')));
    $response->send();

そして「Goodbye」を表示するページです。::

    <?php

    // framework/bye.php

    require_once __DIR__.'/init.php';

    $response->setContent('Goodbye!');
    $response->send();

共有コードの大半を中心的な場所に実際に移動させましたが、あまりよい抽象化には見ませんよね？最初に、すべてのページにおいて ``send()`` メソッドがすでに用意されていますが、我々のページはテンプレートとは異なるので、このコードをテストすることはまだできません。

さらに、新しいページを追加することは URL を通じてエンドユーザーに公開される名前をもつ PHP スクリプトを新たに作る必要があるということです
(``http://example.com/bye.php``): PHP
スクリプトの名前とクライアント URL のあいだに直接のマッピングが存在しています。これはリクエストのディスパッチが Web サーバーによって直接行われるからです。このディスパッチを我々のコードに移すことはよりよい柔軟性のためによいことでしょう。これはすべてのクライアントのリクエストを単独の PHP スクリプトにルーティングさせることでかんたんに実現できます。

.. tip::

    エンドユーザーに単独の PHP スクリプトを公開するやりかたは
    「 `フロントコントローラf`_ 」と呼ばれるデザインパターンです。

このようなスクリプトは次のようになります。::

    <?php

    // framework/front.php

    require_once __DIR__.'/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();
    $response = new Response();

    $map = array(
        '/hello' => __DIR__.'/hello.php',
        '/bye'   => __DIR__.'/bye.php',
    );

    $path = $request->getPathInfo();
    if (isset($map[$path])) {
        require $map[$path];
    } else {
        $response->setStatusCode(404);
        $response->setContent('Not Found');
    }

    $response->send();

そして新しい ``hello.php`` スクリプトの例です。::

    <?php

    // framework/hello.php

    $input = $request->get('name', 'World');
    $response->setContent(sprintf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8')));

``front.php`` スクリプトにおいて、 ``$map`` は URL のパスを対応する PHP スクリプトのパスに関連づけます。

おまけとして、URL マップの中で定義されていないパスをクライアントが問い合わせると、カスタマイズされた 404 ページが返されます。Web サイトを思いどおりにできます。

ページにアクセスするには、 ``front.php`` スクリプトを使わなければなりません。

* ``http://example.com/front.php/hello?name=Fabien``

* ``http://example.com/front.php/bye``

``/hello`` と ``/bye`` の両方はページの *パス* です。

.. tip::

    Apache もしくは nginx のような Web サーバーはやってくる URL を書き換え、フロントコントローラのスクリプトを取り除くので、
    Web サイトのユーザーはずっと見やすい ``http://example.com/hello?name=Fabien`` を入力できるようになります。

これは、サブディレクトリを含む ((必要な場合のみ -- 上記のティップをご覧ください)) フロントコントローラスクリプトの名前を取り除くことによって Request オブジェクトのパスを返す ``Request::getPathInfo()`` メソッドを使うことで実現されます。

.. tip::

    コードをテストするために Web サーバーをセットアップする必要はありません。
    代わりに、 ``$request = Request::create('/hello?name=Fabien');`` のような ``$request = Request::createFromGlobals();`` に置き換えます。引数はシミュレートしたい URL のパスです。

これで Web サーバーはすべてのページに対して同じスクリプト (``front.php``) にアクセスするので、ほかのすべての PHP ファイルを Web サイトのルートディレクトリの外側に移動させることで我々のコードをよりセキュアなものにできます。

.. code-block:: text

    example.com
    ├── composer.json
    │   src
    │   ├── autoload.php
    │   └── pages
    │       ├── hello.php
    │       └── bye.php
    ├── vendor
    └── web
        └── front.php

これで、 ``web/`` に指し示す Web サーバーのルートディレクトリの設定を行い、ほかのすべてのファイルはクライアントからアクセスできなくなります。

.. note::

    新しい構造が機能するためには、さまざまな PHP ファイルのパスを調整しなければなりません。
    変更の作業は読者の練習課題として残しておきます。

最後の取り組みはそれぞれのページで繰り返される ``setContent()`` の呼び出しです。コンテンツを echo で出力し ``setContent()`` をフロントコントローラスクリプトを直接呼び出すだけですべてのページを「テンプレート」に変換できます。::

    <?php

    // example.com/web/front.php

    // ...

    $path = $request->getPathInfo();
    if (isset($map[$path])) {
        ob_start();
        include $map[$path];
        $response->setContent(ob_get_clean());
    } else {
        $response->setStatusCode(404);
        $response->setContent('Not Found');
    }

    // ...

そして ``hello.php`` スクリプトはテンプレートに変換できます。::

    <!-- example.com/src/pages/hello.php -->

    <?php $name = $request->get('name', 'World') ?>

    Hello <?php echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>

We have our framework for today::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../src/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();
    $response = new Response();

    $map = array(
        '/hello' => __DIR__.'/../src/pages/hello.php',
        '/bye'   => __DIR__.'/../src/pages/bye.php',
    );

    $path = $request->getPathInfo();
    if (isset($map[$path])) {
        ob_start();
        include $map[$path];
        $response->setContent(ob_get_clean());
    } else {
        $response->setStatusCode(404);
        $response->setContent('Not Found');
    }

    $response->send();

新しいページを追加する作業は2つのステップになります。エントリをマップに追加し、 ``src/pages/`` の中で PHP テンプレートを作ります。テンプレートから、
``$request`` 変数を通じて Request のデータを取得し、 ``$response``
変数を通じて Response ヘッダーを調整します。

.. note::

    ここで止めるのであれば、URL マップを設定ファイルに抽出することであなたのフレームワークを強化できるでしょう。

.. _`フロントコントローラ`: http://symfony.com/doc/current/book/from_flat_php_to_symfony2.html#a-front-controller-to-the-rescue
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


.. 2012/05/04 username d0ff8bc245d198bd8eadece0a2f62b9ecd6ae6ab
