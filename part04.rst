Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 4)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


今日のトピックを始める前に、テンプレートをもっと読みやすくするために、現在の我々のフレームワークを少しリファクタリングしましょう。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../src/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();

    $map = array(
        '/hello' => 'hello',
        '/bye'   => 'bye',
    );

    $path = $request->getPathInfo();
    if (isset($map[$path])) {
        ob_start();
        extract($request->query->all(), EXTR_SKIP);
        include sprintf(__DIR__.'/../src/pages/%s.php', $map[$path]);
        $response = new Response(ob_get_clean());
    } else {
        $response = new Response('Not Found', 404);
    }

    $response->send();

リクエストクエリパラメータを抽出したので、``hello.php``
テンプレートを次のように簡略化します。::

    <!-- example.com/src/pages/hello.php -->

    Hello <?php echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>

これで、新しい機能を追加するのによい状態にあります。

Web サイトのとても重要な側面は URL の形式です。URL マップのおかげで、関連するレスポンスを生成するコードから URL を分離しましたが、まだじゅうぶんに柔軟ではありません。たとえば、クエリ文字列に依存する代わりにデータを URL に直接埋め込むコトができるようにするために動的なパスをサポートしたいとします。:

    # 変更前
    /hello?name=Fabien

    # 変更後
    /hello/Fabien

このフィーチャをサポートするために、Symfony2 の Routing コンポーネントを使います。
いつものように、これを ``composer.json`` に追加しインストールするために ``php composer.phar
update`` を実行します。

.. code-block:: json

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*"
        }
    }

これから、我々独自の ``autoload.php`` の代わりに生成された Composer のオートローダーを使います。 ``autoload.php`` ファイルを取り除き、 ``front.php`` への参照を置き換えます。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    // ...

URL マップに対して配列の代わりに、Routing コンポーネントは ``RouteCollection`` インスタンスに依存します。::

    use Symfony\Component\Routing\RouteCollection;

    $routes = new RouteCollection();

``/hello/SOMETHING`` URL を記述するルートを追加し、シンプルな ``/bye`` の URL に対する別のルートも追加しましょう。::

    use Symfony\Component\Routing\Route;

    $routes->add('hello', new Route('/hello/{name}', array('name' => 'World')));
    $routes->add('bye', new Route('/bye'));

コレクションのそれぞれのエントリは名前 (``hello``) と ``Route``
のインスタンスによって定義されます。これはルートのパターン (``/hello/{name}``) とルートの属性に対するデフォルト値の配列 (``array('name' => 'World')``) によって定義されます。

.. note::

    URL 生成、属性のルーティング要件、HTTP
    メソッドの強化、YAML もしくは XML ファイルのローダー、PHP へのダンパーもしくは
    性能の向上のための Apache の書き換えルールなどのたくさんのフィーチャをもっと学びたいのであれば 公式の `ドキュメント`_ をご覧ください。

``RouteCollection`` インスタンスに保存された情報にもとづいて、
``UrlMatcher`` インスタンスは URL パスのマッチングを行います。::

    use Symfony\Component\Routing\RequestContext;
    use Symfony\Component\Routing\Matcher\UrlMatcher;

    $context = new RequestContext();
    $context->fromRequest($request);
    $matcher = new UrlMatcher($routes, $context);

    $attributes = $matcher->match($request->getPathInfo());

``match()`` メソッドはリクエストのパスを引数にとり属性の配列を返します
(マッチ済みのルートは特別な
``_route`` 属性のもとに自動的に保存されることにご注意ください)::

    print_r($matcher->match('/bye'));
    array (
      '_route' => 'bye',
    );

    print_r($matcher->match('/hello/Fabien'));
    array (
      'name' => 'Fabien',
      '_route' => 'hello',
    );

    print_r($matcher->match('/hello'));
    array (
      'name' => 'World',
      '_route' => 'hello',
    );

.. note::

    我々の例でリクエストのコンテクストを厳密に必要としないとしても、メソッドの要件を強化するために実際の世界のアプリケーションで使われています。

マッチするルートがないときに URL マッチャは例外を投げます。::

    $matcher->match('/not-found');

    // throws a Symfony\Component\Routing\Exception\ResourceNotFoundException

このことを念頭において、フレームワークの新しいバージョンを書きましょう。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing;

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $context = new Routing\RequestContext();
    $context->fromRequest($request);
    $matcher = new Routing\Matcher\UrlMatcher($routes, $context);

    try {
        extract($matcher->match($request->getPathInfo()), EXTR_SKIP);
        ob_start();
        include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

        $response = new Response(ob_get_clean());
    } catch (Routing\Exception\ResourceNotFoundException $e) {
        $response = new Response('Not Found', 404);
    } catch (Exception $e) {
        $response = new Response('An error occurred', 500);
    }

    $response->send();

コードの中に少し新しい内容が含まれています。::

* ルートの名前はテンプレートの名前に使われます;

* ``500`` エラーは正しく管理されます;

* リクエストの属性はテンプレートをシンプルに保つために抽出されました::

      <!-- example.com/src/pages/hello.php -->

      Hello <?php echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>

* ルートのコンフィギュレーションは独自のファイルに移動しました:

  .. code-block:: php

      <?php

      // example.com/src/app.php

      use Symfony\Component\Routing;

      $routes = new Routing\RouteCollection();
      $routes->add('hello', new Routing\Route('/hello/{name}', array('name' => 'World')));
      $routes->add('bye', new Routing\Route('/bye'));

      return $routes;

  これでコンフィギュレーション (``app.php`` の中のアプリケーションに固有なすべての内容) とフレームワーク (``front.php`` の中の我々のアプリケーションに動力を提供する一般的なコード) のあいだが明確に分離されました。

30行以下のコードによって、以前よりも強力で柔軟な新しいフレームワークが手に入りました。エンジョイしましょう！

Routing コンポーネントを使うことで1つの追加された大きな恩恵があります: Route の定義にもとづいて URL を生成する機能です。URL マッチ と URL
の生成を使うことで、URL パターンを変更してもほかのところに影響がありません。ジェネレータの使い方を知りたいですか？きわめてかんたんです。::

    use Symfony\Component\Routing;

    $generator = new Routing\Generator\UrlGenerator($routes, $context);

    echo $generator->generate('hello', array('name' => 'Fabien'));
    // /hello/Fabien を出力します

コードがやっていることは自明です; そしてコンテクストのおかげで、絶対 URL を生成できます。::

    echo $generator->generate('hello', array('name' => 'Fabien'), true);
    // http://example.com/somewhere/hello/Fabien のようなものを出力します

.. tip::

    パフォーマンスが心配ですか？ルートの定義にもとづき、
    デフォルトの ``UrlMatcher`` を置き換える高度に最適化された URL マッチャクラスを作ります。::

        $dumper = new Routing\Matcher\Dumper\PhpMatcherDumper($routes);

        echo $dumper->dump();

    さらにパフォーマンスを改善したいですか？ルートを Apache の書き換えルールのセットとして吐き出させます。::

        $dumper = new Routing\Matcher\Dumper\ApacheMatcherDumper($routes);

        echo $dumper->dump();

.. _`ドキュメント`: http://symfony.com/doc/current/components/routing.html
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
