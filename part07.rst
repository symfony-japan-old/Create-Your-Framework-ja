Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 7)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


今のところ我々のフレームワークの1つの欠点は新しい Web サイトをつくるたびに ``front.php`` でコードをコピー＆ペーストする必要があることです。40行のコードは多くありませんが、このコードを適切なクラスに包むことができれば都合がよいです。It would bring us better *reusability* and easier testing to name just
a few benefits.

コードをよく見てみると、 ``front.php`` には1つの入力である
Request と1つの出力である Response があります。我々のフレームワークは次のシンプルな原則にしたがいます: ロジックは Request に関連する Response をつくることに専念します。

Symfony2 コンポーネントは PHP 5.3 を必須とするので、我々のフレームワークのために独自の名前空間をつくりましょう: ``Simplex`` 

リクエストハンドリングのロジックを ``Simplex\\Framework`` クラスに移動させます。::

    <?php

    // example.com/src/Simplex/Framework.php

    namespace Simplex;

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Matcher\UrlMatcher;
    use Symfony\Component\Routing\Exception\ResourceNotFoundException;
    use Symfony\Component\HttpKernel\Controller\ControllerResolver;

    class Framework
    {
        protected $matcher;
        protected $resolver;

        public function __construct(UrlMatcher $matcher, ControllerResolver $resolver)
        {
            $this->matcher = $matcher;
            $this->resolver = $resolver;
        }

        public function handle(Request $request)
        {
            $this->matcher->getContext()->fromRequest($request);

            try {
                $request->attributes->add($this->matcher->match($request->getPathInfo()));

                $controller = $this->resolver->getController($request);
                $arguments = $this->resolver->getArguments($request, $controller);

                return call_user_func_array($controller, $arguments);
            } catch (ResourceNotFoundException $e) {
                return new Response('Not Found', 404);
            } catch (\Exception $e) {
                return new Response('An error occurred', 500);
            }
        }
    }

そしてそれに応じて ``example.com/web/front.php`` をアップデートします。::

    <?php

    // example.com/web/front.php

    // ...

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $context = new Routing\RequestContext();
    $matcher = new Routing\Matcher\UrlMatcher($routes, $context);
    $resolver = new HttpKernel\Controller\ControllerResolver();

    $framework = new Simplex\Framework($matcher, $resolver);
    $response = $framework->handle($request);

    $response->send();

リファクタリングを仕上げるために、 ``example.com/src/app.php`` からルートの定義以外のすべてをまた別の名前空間: ``Calendar`` に移動させましょう。

``Simplex`` と ``Calendar`` 名前空間のもとで定義されたクラスがオートロードされるようにするため、 ``composer.json`` ファイルをアップデートします。

.. code-block:: javascript

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*",
            "symfony/http-kernel": "2.1.*"
        },
        "autoload": {
            "psr-0": { "Simplex": "src/", "Calendar": "src/" }
        }
    }

.. note::

    オートローダをアップデートするため、 ``php composer.phar update`` を実行します。

コントローラを ``Calendar\\Controller\\LeapYearController`` に移動させます。::

    <?php

    // example.com/src/Calendar/Controller/LeapYearController.php

    namespace Calendar\Controller;

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Calendar\Model\LeapYear;

    class LeapYearController
    {
        public function indexAction(Request $request, $year)
        {
            $leapyear = new LeapYear();
            if ($leapyear->isLeapYear($year)) {
                return new Response('Yep, this is a leap year!');
            }

            return new Response('Nope, this is not a leap year.');
        }
    }

そして ``is_leap_year()`` 関数を独自のクラスに移動させます。::

    <?php

    // example.com/src/Calendar/Model/LeapYear.php

    namespace Calendar\Model;

    class LeapYear
    {
        public function isLeapYear($year = null)
        {
            if (null === $year) {
                $year = date('Y');
            }

            return 0 == $year % 400 || (0 == $year % 4 && 0 != $year % 100);
        }
    }

それにしたがって ``example.com/src/app.php`` ファイルをアップデートすることをお忘れなく。::

    $routes->add('leap_year', new Routing\Route('/is_leap_year/{year}', array(
        'year' => null,
        '_controller' => 'Calendar\\Controller\\LeapYearController::indexAction',
    )));

まとめると、新しいファイルのレイアウトは次のようになります。

.. code-block:: text

    example.com
    ├── composer.json
    │   src
    │   ├── app.php
    │   └── Simplex
    │       └── Framework.php
    │   └── Calendar
    │       └── Controller
    │       │   └── LeapYearController.php
    │       └── Model
    │           └── LeapYear.php
    ├── vendor
    └── web
        └── front.php

これでおしまいです！我々のフレームワークには4つの異なるレイヤーが用意され、それぞれに明確なゴールがあります。

* ``web/front.php``: フロントコントローラ; クライアントにインターフェイスを公開する
  唯一の PHP コード (Request を取得し Response を返す) で 
  フレームワークとアプリケーションを初期化するボイラーテンプレートコードを提供します;

* ``src/Simplex``: やってくる Request の処理を抽象化する再利用可能なフレームワーク (ところで、これはコントローラ/テンプレートをかんたんにテストできるようにします -- あとでくわしく説明します);

* ``src/Calendar``: アプリケーション固有のコード (コントローラとモデル);

* ``src/app.php``: アプリケーションのコンフィギュレーション/フレームワークのカスタマイズ内容。

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


.. 2012/05/05 masakielastic d0ff8bc245d198bd8eadece0a2f62b9ecd6ae6ab
