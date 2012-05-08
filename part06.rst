Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 6)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


我々のフレームワークはすでにとても堅牢であるとお考えかもしれません。そしてはそれはおそらく正しいです。それでも改善する方法を見てみましょう。

今すぐ、我々のコードは手続き型のコードを使いますが、コントローラは任意の PHP コールバックになることを覚えておいてください。コントローラを適切なクラスに変換しましょう。::

    class LeapYearController
    {
        public function indexAction($request)
        {
            if (is_leap_year($request->attributes->get('year'))) {
                return new Response('Yep, this is a leap year!');
            }

            return new Response('Nope, this is not a leap year.');
        }
    }

それに応じて、ルートの定義を更新します。::

    $routes->add('leap_year', new Routing\Route('/is_leap_year/{year}', array(
        'year' => null,
        '_controller' => array(new LeapYearController(), 'indexAction'),
    )));

コードの移動はとても直感的で複数のページを作るとすぐに効果がありますが、望ましくない副作用に気づくことになります。リクエストされた URL が ``leap_year`` ルートにマッチしない場合でも ``LeapYearController`` クラスのインスタンスが *常に* 生成されることです。これは1つの主な理由からわるいことです: パフォーマンスに目を向けると、すべてのルートに対してすべてのコントローラのインスタンスを生成しなければなりません。マッチしたルートと関係のあるコントローラのインスタンスだけが生成されるようにコントローラが遅延ロードされると好都合でしょう。

この問題を解決するために HttpKernel
コンポーネントをインストールして使いましょう。::

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*",
            "symfony/http-kernel": "2.1.*"
        }
    }

HttpKernel コンポーネントはたくさんの興味深いフィーチャが備わっていますが、いますぐ必要なのは *コントローラリゾルバ* です。コントローラリゾルバは Request オブジェクトにもとづいて実行するコントローラとそれに渡す引数を決める方法を知っています。すべてのコントローラリゾルバは次のインターフェイスを実装します。::

    namespace Symfony\Component\HttpKernel\Controller;

    interface ControllerResolverInterface
    {
        function getController(Request $request);

        function getArguments(Request $request, $controller);
    }

``getController()`` メソッドは以前定義したものと同じ慣習に頼っています: ``_controller`` リクエスト属性は Request に関連するコントローラを含んでいなければなりません。PHP 組み込みのコールバックに加えて、
``getController()`` は「class::method」のように、クラスの名前と2つのコロンの後に続く有効なコールバックとしてのメソッドの名前で構成される文字列もサポートします。::

    $routes->add('leap_year', new Routing\Route('/is_leap_year/{year}', array(
        'year' => null,
        '_controller' => 'LeapYearController::indexAction',
    )));

このコードが機能するようにするため、HttpKernel からコントローラリゾルバを使うようにフレームワークのコードを修正します。::

    use Symfony\Component\HttpKernel;

    $resolver = new HttpKernel\Controller\ControllerResolver();

    $controller = $resolver->getController($request);
    $arguments = $resolver->getArguments($request, $controller);

    $response = call_user_func_array($controller, $arguments);

.. note::

    おまけとして、コントローラリゾルバはあなたの代わりにエラーを適切に管理します。
    たとえば、Route に対して ``_controller`` 属性を定義することを忘れたときなどです。

では、コントローラの引数がどのように推測されるのか見てみましょう。 ``getArguments()``
は PHP ネイティブの `リフレクション`_ を利用することでどの引数を渡すかどうか決めるためにコントローラのシグネチャをイントロスペクトします。

``indexAction()`` メソッドは Request オブジェクトを引数として必要とします。
```getArguments()`` はタイプヒントが適切であればそれをインジェクトするタイミングを知っています。::

    public function indexAction(Request $request)

    // won't work
    public function indexAction($request)

より興味深いことに、 ``getArguments()`` は任意の Request
属性もインジェクトできます; 引数は対応する属性と同じ名前をもつことが必要なだけです。::

    public function indexAction($year)

Request と属性を同時にインジェクトすることもできます (マッチングは引数の名前もしくはタイプヒントによって行われるので、引数の順序は関係ありません)::

    public function indexAction(Request $request, $year)

    public function indexAction($year, Request $request)

最後に、Request のオプション属性にマッチする引数のデフォルト値を定義することもできます。::

    public function indexAction($year = 2012)

コントローラに対して ``$year`` リクエスト属性をインジェクトしましょう。::

    class LeapYearController
    {
        public function indexAction($year)
        {
            if (is_leap_year($year)) {
                return new Response('Yep, this is a leap year!');
            }

            return new Response('Nope, this is not a leap year.');
        }
    }

コントローラリゾルバはコール可能なコントローラと引数のバリデーションも考慮します。問題があれば、問題を説明するすばらしいメッセージつきの例外を投げします (コントローラクラスが存在しない、メソッドが定義されていない、属性にマッチする引数が存在しない、など)。

.. note::

    デフォルトのコントローラリゾルバの大いなる柔軟性によって、
    誰かがなぜ別のものを作りたいのか疑問に思うかもしれません (そうでなければなぜインターフェイスが存在するのでしょう
    )。2つの例を挙げます: Symfony2 において ``getController()`` は
    `サービスとしてのコントローラ`_ をサポートするよう強化されました;
    `FrameworkExtraBundle`_ において ``getArguments()`` はパラメータコンバータをサポートするように強化され、
    リクエスト属性はオブジェクトに自動変換されます。

新しいバージョンのフレームワークで締めくくりましょう。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing;
    use Symfony\Component\HttpKernel;

    function render_template($request)
    {
        extract($request->attributes->all());
        ob_start();
        include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

        return new Response(ob_get_clean());
    }

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $context = new Routing\RequestContext();
    $context->fromRequest($request);
    $matcher = new Routing\Matcher\UrlMatcher($routes, $context);
    $resolver = new HttpKernel\Controller\ControllerResolver();

    try {
        $request->attributes->add($matcher->match($request->getPathInfo()));

        $controller = $resolver->getController($request);
        $arguments = $resolver->getArguments($request, $controller);

        $response = call_user_func_array($controller, $arguments);
    } catch (Routing\Exception\ResourceNotFoundException $e) {
        $response = new Response('Not Found', 404);
    } catch (Exception $e) {
        $response = new Response('An error occurred', 500);
    }

    $response->send();

より深く考えてみましょう: 我々のフレームワークはより強固で柔軟になり、まだ40行未満のコードです。

.. _`リフレクション`:              http://php.net/reflection
.. _`FrameworkExtraBundle`:    http://symfony.com/doc/current/bundles/SensioFrameworkExtraBundle/annotations/converters.html
.. _`サービスとしてのコントローラ`: http://symfony.com/doc/current/cookbook/controller/service.html
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


.. 2012/05/02 masakielastic d0ff8bc245d198bd8eadece0a2f62b9ecd6ae6ab
