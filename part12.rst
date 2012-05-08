Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 12)
========================================================================

In the last installment of this series, we have emptied the
``HttpKernel`` クラスを継承する ``Simplex\\Framework`` クラス by extending the ``HttpKernel`` class from
the eponymous component。この空のクラスを見てみると、フロントコントローラからコードの一部を移動させたいと思うかもしれません。::

    <?php

    // example.com/src/Simplex/Framework.php

    namespace Simplex;

    use Symfony\Component\HttpKernel\HttpKernel;
    use Symfony\Component\Routing;
    use Symfony\Component\HttpKernel;
    use Symfony\Component\EventDispatcher\EventDispatcher;

    class Framework extends HttpKernel
    {
        public function __construct($routes)
        {
            $context = new Routing\RequestContext();
            $matcher = new Routing\Matcher\UrlMatcher($routes, $context);
            $resolver = new HttpKernel\Controller\ControllerResolver();

            $dispatcher = new EventDispatcher();
            $dispatcher->addSubscriber(new HttpKernel\EventListener\RouterListener($matcher));
            $dispatcher->addSubscriber(new HttpKernel\EventListener\ResponseListener('UTF-8'));

            parent::__construct($dispatcher, $resolver);
        }
    }

フロントコントローラのコードは簡潔になります。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $framework = new Simplex\Framework($routes);

    $framework->handle($request)->send();

簡潔なフロントコントローラを用意することで、単独のアプリケーションに対して複数のフロントコントローラを用意することができます。なぜこれが便利なのでしょうか？たとえば開発環境と運用環境で異なりコンフィギュレーションを用意することができます。開発環境において、デバッグを楽にするために、エラー報告機能を有効にしてブラウザにエラーを表示させるとよいでしょう。::

    ini_set('display_errors', 1);
    error_reporting(-1);

... しかし運用環境において同じコンフィギュレーションは望んでいないでしょう。2つの異なるフロントコントローラを用意することで、それぞれに対して微妙に異なるコンフィギュレーションを用意する機会がもたらされます。

ですので、コードをフロントコントローラからフレームワークに移動させると、我々のフレームワークのコンフィギュレーションはより調整しやすくなりますが、同時に、たくさんの問題も導入されます。

* ディスパッチャが Framework クラスの外側で利用できないのでカスタムリスナーをもはや登録できなくなります (かんたんな次善策は ``Framework::getEventDispatcher()`` メソッドの追加です);

* 以前あった柔軟性が失われました; ``UrlMatcher`` の実装もしくは ``ControllerResolver`` の実装を変更することができません;

* 以前のポイントと関連して、内部オブジェクトのモックをつくることができないのでフレームワークをかんたんにテストできなくなりました;

* 文字集合を ``ResponseListener`` に渡される文字集合の値を変更できなくなりました (次善策はコンストラクタの引数として渡すことです)。

Dependency Injection を使っていたので以前のコードは同じ問題を禁止しませんでした; all dependencies of our objects were injected into their
constructors (for instance, the event dispatchers were injected into the
framework so that we had total control of its creation and configuration).

Does it mean that we have to make a choice between flexibility, customization,
ease of testing and not to copy and paste the same code into each application
front controller? As you might expect, there is a solution. We can solve all
these issues and some more by using the Symfony2 dependency injection
container:

.. code-block:: json

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*",
            "symfony/http-kernel": "2.1.*",
            "symfony/event-dispatcher": "2.1.*",
            "symfony/dependency-injection": "2.1.*"
        },
        "autoload": {
            "psr-0": { "Simplex": "src/", "Calendar": "src/" }
        }
    }

Create a new file to host the dependency injection container configuration::

    <?php

    // example.com/src/container.php

    use Symfony\Component\DependencyInjection;
    use Symfony\Component\DependencyInjection\Reference;

    $sc = new DependencyInjection\ContainerBuilder();
    $sc->register('context', 'Symfony\Component\Routing\RequestContext');
    $sc->register('matcher', 'Symfony\Component\Routing\Matcher\UrlMatcher')
        ->setArguments(array($routes, new Reference('context')))
    ;
    $sc->register('resolver', 'Symfony\Component\HttpKernel\Controller\ControllerResolver');

    $sc->register('listener.router', 'Symfony\Component\HttpKernel\EventListener\RouterListener')
        ->setArguments(array(new Reference('matcher')))
    ;
    $sc->register('listener.response', 'Symfony\Component\HttpKernel\EventListener\ResponseListener')
        ->setArguments(array('UTF-8'))
    ;
    $sc->register('listener.exception', 'Symfony\Component\HttpKernel\EventListener\ExceptionListener')
        ->setArguments(array('Calendar\\Controller\\ErrorController::exceptionAction'))
    ;
    $sc->register('dispatcher', 'Symfony\Component\EventDispatcher\EventDispatcher')
        ->addMethodCall('addSubscriber', array(new Reference('listener.router')))
        ->addMethodCall('addSubscriber', array(new Reference('listener.response')))
        ->addMethodCall('addSubscriber', array(new Reference('listener.exception')))
    ;
    $sc->register('framework', 'Simplex\Framework')
        ->setArguments(array(new Reference('dispatcher'), new Reference('resolver')))
    ;

    return $sc;

このファイルの目的はオブジェクトとそれらの依存オブジェクトの設定を行うことです。このコンフィギュレーションの調整ステップにおいてインスタンスの生成は必要はありません。操作して生成する必要のあるオブジェクトの静止的な記述です。オブジェクトはコンテナからそれらにアクセスするときもしくはコンテナがほかのオブジェクトを生成するためにそれらを必要とするときに生成されます。

たとえば、ルーターリスナーをつくりたい場合、クラスの名前が ``Symfony\Component\HttpKernel\EventListener\RouterListener`` であり、それらのコンストラクタがマッチャオブジェクト (``new Reference('matcher')``) を引数にとることを Symfony に伝えます。ご覧のとおり、それぞれのオブジェクトは名前で参照されます。名前は一意性のある文字列でそれぞれのオブジェクトを特定します。名前によってオブジェクトを取得し、ほかのオブジェクトの定義の中でそれを参照することができます。

.. note::

    デフォルトでは、コンテナからオブジェクトを取得するたびに、    まったく同じ名前のインスタンスが返されます。
    これはコンテナが「グローバル」オブジェクトをマネージするからです。

これでフロントコントローラは一緒にすべてのものを結びつけることだけに専念するようになりました。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;

    $routes = include __DIR__.'/../src/app.php';
    $sc = include __DIR__.'/../src/container.php';

    $request = Request::createFromGlobals();

    $response = $sc->get('framework')->handle($request);

    $response->send();

.. note::

    コンテナの軽量な代替版がほしいのであれば、 `Pimple`_ をお考えください。
    これは PHP 約60行の PHP コードによるシンプルな Dependency Injection コンテナです。

では、フロントコントローラでカスタムリスナーを登録する方法は次のとおりです。::

    $sc->register('listener.string_response', 'Simplex\StringResponseListener');
    $sc->getDefinition('dispatcher')
        ->addMethodCall('addSubscriber', array(new Reference('listener.string_response')))
    ;

オブジェクトを記述することに加えて、Dependency Injection コンテナはパラメータを通じてコンフィギュレーションを調整できます。デバッグモードもしくはそうであるかどうかを定義するものをつくってみましょう。::

    $sc->setParameter('debug', true);

    echo $sc->getParameter('debug');

これらのパラメータはオブジェクト定義を定義するときに使います。文字集合の設定を変更できるようにしましょう。::

    $sc->register('listener.response', 'Symfony\Component\HttpKernel\EventListener\ResponseListener')
        ->setArguments(array('%charset%'))
    ;

これを変更すると、レスポンスリスナーオブジェクトを使って文字集合をセットしなければなりません。::

    $sc->setParameter('charset', 'UTF-8');

ルートは ``$routes`` 変数によって定義されるという慣習の代わりに、再度パラメータを使ってみましょう。::

    $sc->register('matcher', 'Symfony\Component\Routing\Matcher\UrlMatcher')
        ->setArguments(array('%routes%', new Reference('context')))
    ;

そしてフロントコントローラのなかの関連する変更内容です。::

    $sc->setParameter('routes', include __DIR__.'/../src/app.php');

コンテナで対処できることの表面をほとんどスクラッチしませんでした: パラメータとしてのクラス名から、既存のオブジェクト定義のオーバーライド、コンテナをプレーンな PHP クラスにダンプするまでのスコープのサポートなどです。Symfony の Dependency Injection コンテナは本当に強力で 任意の PHP クラスをマネージできます。

あなたのフレームワークに Dependency Injection コンテナは必要ないと私に大声で言うのはやめてください。好きでなければ、使わないでください。これはあなたのフレームワークであり、私のものではありません。
これは (すでに) Symfony2 コンポーネントでフレームワークを作成する最後のパートです。多くのトピックがくわしい内容をカバーしていないことを認識していますが、独自のことを始めることと Symfony2 フレームワークが内部でどのように動くのか理解するためにはじゅうぶんな情報が提供されています。
さらにくわしく学びたいのであれば、Silex マイクロフレームワークのソースコード、とりわけ `Application` クラスを読むことをおすすめします。

楽しんでください！

~~ FIN ~~

*P.S.:* じゅうぶんな興味があれば (この投稿をコメントをください)、I might
write some more articles on specific topics (ルーティングのための設定ファイルを使うこと、HttpKernel デバッギングツールを使うこと、ブラウザをシミュレートするために組み込みのクライアントを使うことなどは筆者が思い浮かべているトピックの一部です)。

.. _`Pimple`:      https://github.com/fabpot/Pimple
.. _`Application`: https://github.com/fabpot/Silex/blob/master/src/Silex/Application.php

.. 2012/05/09 masakielastic c0877802ef38c15b936eca69ae0b7dd4254e783a
