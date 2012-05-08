Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 5)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


賢明な読者は特定の「コード」 (テンプレート) が実行される方法をフレームワークがハードコードしていることに気づいています。これまでつくったようなシンプルなページに関して、問題はありませんが、さらにロジックを追加したい場合、ロジックをテンプレート自身に置くことを強制させられます。これはとりわけ関心の分離をまだ覚えているのであれば、おそらくはよい考えではありません。

新しいレイヤーであるコントローラを追加することでテンプレートのコードをロジックから分離しましょう: *コントローラのミッションはクライアントの Request によって運ばれた情報にもとづいて Response を生成することです。*

次のようにフレームワークのテンプレートレンダリングの部分を変更します。::

    <?php

    // example.com/web/front.php

    // ...

    try {
        $request->attributes->add($matcher->match($request->getPathInfo()));
        $response = call_user_func('render_template', $request);
    } catch (Routing\Exception\ResourceNotFoundException $e) {
        $response = new Response('Not Found', 404);
    } catch (Exception $e) {
        $response = new Response('An error occurred', 500);
    }

レンダリングは外部関数 (ここでは ``render_template()``
) によって行われるので、URL から抽出した属性を渡す必要があります。これらを ``render_template()`` に追加の引数として渡すことができますが、代わりに、*属性* と呼ばれる ``Request`` クラスの別のフィーチャを使いましょう: Request 属性によって HTTP Request データに直接は関係のない情報を添付できます。

これでロジックが指定されていないときにテンプレートをレンダリングする一般的なコントローラである ``render_template()`` 関数をつくることができます。以前と同じテンプレートを保つために、テンプレートがレンダリングされる前にリクエスト属性が抽出されます。::

    function render_template($request)
    {
        extract($request->attributes->all(), EXTR_SKIP);
        ob_start();
        include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

        return new Response(ob_get_clean());
    }

``render_template`` は PHP ``call_user_func()``
関数への引数として使われるので、任意の有効な PHP `コールバック`_ に置き換えることができます。これによって、あなたが選んだ関数、匿名関数、もしくはクラスのメソッドをコントローラとして使うことができます。

関数として、それぞれのルートに対して、 ``_controller`` ルート属性を通じて関連するコントローラの調整が行われます。::

    $routes->add('hello', new Routing\Route('/hello/{name}', array(
        'name' => 'World',
        '_controller' => 'render_template',
    )));

    try {
        $request->attributes->add($matcher->match($request->getPathInfo()));
        $response = call_user_func($request->attributes->get('_controller'), $request);
    } catch (Routing\Exception\ResourceNotFoundException $e) {
        $response = new Response('Not Found', 404);
    } catch (Exception $e) {
        $response = new Response('An error occurred', 500);
    }

ルートはコントローラと関連づけることが可能で、もちろん、コントローラの範囲ではテンプレートをレンダリングするのにまだ ``render_template()`` を使うことができます。::

    $routes->add('hello', new Routing\Route('/hello/{name}', array(
        'name' => 'World',
        '_controller' => function ($request) {
            return render_template($request);
        }
    )));

これはより柔軟性があり Response オブジェクトを後で変更できるのでテンプレートに追加の引数を渡すことができます。::

    $routes->add('hello', new Routing\Route('/hello/{name}', array(
        'name' => 'World',
        '_controller' => function ($request) {
            // $foo はテンプレートの中で利用できるようになります
            $request->attributes->set('foo', 'bar');

            $response = render_template($request);

            // ヘッダーを変更します
            $response->headers->set('Content-Type', 'text/plain');

            return $response;
        }
    )));

フレームワークのアップデートされ改善されたバージョンは次のようになります。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing;

    function render_template($request)
    {
        extract($request->attributes->all(), EXTR_SKIP);
        ob_start();
        include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

        return new Response(ob_get_clean());
    }

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $context = new Routing\RequestContext();
    $context->fromRequest($request);
    $matcher = new Routing\Matcher\UrlMatcher($routes, $context);

    try {
        $request->attributes->add($matcher->match($request->getPathInfo()));
        $response = call_user_func($request->attributes->get('_controller'), $request);
    } catch (Routing\Exception\ResourceNotFoundException $e) {
        $response = new Response('Not Found', 404);
    } catch (Exception $e) {
        $response = new Response('An error occurred', 500);
    }

    $response->send();

新しいフレームワークの誕生を祝うため、シンプルなロジックを必要とする真新しいアプリケーションをつくりましょう。我々のアプリケーションには任意の年がうるう年かどうかを伝える1つのページが用意されています。 ``/is_leap_year`` を呼び出すとき、現在の年に対する回答を得られますが、``/is_leap_year/2009`` のように年を指定することもできます。一般的には、フレームワークを修正する必要はなく、新しい ``app.php`` ファイルをつくるだけですみます。::

    <?php

    // example.com/src/app.php

    use Symfony\Component\Routing;
    use Symfony\Component\HttpFoundation\Response;

    function is_leap_year($year = null) {
        if (null === $year) {
            $year = date('Y');
        }

        return 0 == $year % 400 || (0 == $year % 4 && 0 != $year % 100);
    }

    $routes = new Routing\RouteCollection();
    $routes->add('leap_year', new Routing\Route('/is_leap_year/{year}', array(
        'year' => null,
        '_controller' => function ($request) {
            if (is_leap_year($request->attributes->get('year'))) {
                return new Response('Yep, this is a leap year!');
            }

            return new Response('Nope, this is not a leap year.');
        }
    )));

    return $routes;

``is_leap_year()`` 関数は渡された年がうるう年であれば ``true`` を返し、そうでなければ ``false`` を返します。年が null であれば、現在の年がテストされます。コントローラはシンプルです: リクエスト属性から年を取得し、これを `is_leap_year()`` 関数に渡し、戻り値にしたがって、新しい Response オブジェクトをつくります。

いつものように、ここで止めてフレームワークをそのまま使うことができます; おそらくあなたに必要なことは `websites`_ とその他のファンシーな1ページの `Web サイト`_ のようなシンプルな Web サイトを作ることです。

.. _`コールバック`: http://php.net/callback#language.types.callback
.. _`Web サイト`:  http://kottke.org/08/02/single-serving-sites
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


.. 2012/05/05 username d0ff8bc245d198bd8eadece0a2f62b9ecd6ae6ab
