Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 11)
========================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


我々のフレームワークをすぐに使いたいのであれば、カスタムエラーメッセージのサポートを追加しなければならないでしょう。404エラーと500エラーのサポートがありますが、レスポンスがフレームワークの中でハードコードされています。これらをカスタマイズできるようにすることはとてもかんたんです: 新しいイベントをディスパッチし、リスニングするようにします。これを行うことはリスナーが通常のコントローラを呼び出さなければならないということです。しかしエラーコントローラが例外を投げたら？無限ループに陥ります。もっとかんたんな方法があるのでしょうか？

``HttpKernel`` クラスに入ります。同じ問題を何度も解決したり、車輪の再発明を繰り返す代わりに、 ``HttpKernel``
クラスは一般的で、拡張性があり、
``HttpKernelInterface`` の柔軟性のある実装です。

このクラスはこれまで書いてきたフレームワーククラスとよく似ています: これはリクエスト処理のあいだのある戦略的なポイントにおいてイベントをディスパッチし、リクエストをディスパッチするコントローラを選ぶコントローラリゾルバを使い、さらによいことに、これはエッジケースを考慮し、問題が起きたときにすばらしいフィードバックを提供してくれます。

新しいフレームワークのコードは次のようになります。::

    <?php

    // example.com/src/Simplex/Framework.php

    namespace Simplex;

    use Symfony\Component\HttpKernel\HttpKernel;

    class Framework extends HttpKernel
    {
    }

新しいフロントコントローラです。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing;
    use Symfony\Component\HttpKernel;
    use Symfony\Component\EventDispatcher\EventDispatcher;

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $context = new Routing\RequestContext();
    $matcher = new Routing\Matcher\UrlMatcher($routes, $context);
    $resolver = new HttpKernel\Controller\ControllerResolver();

    $dispatcher = new EventDispatcher();
    $dispatcher->addSubscriber(new HttpKernel\EventListener\RouterListener($matcher));

    $framework = new Simplex\Framework($dispatcher, $resolver);

    $response = $framework->handle($request);
    $response->send();

``RouterListener`` はフレームワークにある同じロジックの実装です: これはやってくるリクエストのマッチングを行い、ルートパラメータをリクエスト属性に投入します。

コードはこれまでよりもはるかに簡潔で驚くべきほど強力になりました。たとえば、エラー管理機能を調整できるようにするためには組み込みの ``ExceptionListener`` を使います。::

    $errorHandler = function (HttpKernel\Exception\FlattenException $exception) {
        $msg = 'Something went wrong! ('.$exception->getMessage().')';

        return new Response($msg, $exception->getStatusCode());
    });
    $dispatcher->addSubscriber(new HttpKernel\EventListener\ExceptionListener($errorHandler));

``ExceptionListener`` は例外の操作と表示を楽にするために ``Exception`` のインスタンスを投げる代わりに ``FlattenException`` のインスタンスを提供します。これは任意のコントローラを例外ハンドラとして引数にとるので、 Closure を使う代わりに ErrorController クラスをつくることができます。::

    $listener = new HttpKernel\EventListener\ExceptionListener('Calendar\\Controller\\ErrorController::exceptionAction');
    $dispatcher->addSubscriber($listener);

エラーコントローラは次のようになります。::

    <?php

    // example.com/src/Calendar/Controller/ErrorController.php

    namespace Calendar\Controller;

    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Exception\FlattenException;

    class ErrorController
    {
        public function exceptionAction(FlattenException $exception)
        {
            $msg = 'Something went wrong! ('.$exception->getMessage().')';

            return new Response($msg, $exception->getStatusCode());
        }
    }

ほら！労力もなしにクリーンでカスタマイズ可能なエラー管理機能が手に入りました。そしてもちろん、コントローラが例外を投げる場合、HttpKernel はそれをきちんと処理します。

パート2において、 ``Response::prepare()`` メソッドを説明しました。このメソッドは Response が HTTP の仕様にしたがっていることを保証します。Response をクライアントに送る直前にこれを呼び出すのは常によい考えでしょう; ``ResponseListener`` がやっていることはこれです。::

    $dispatcher->addSubscriber(new HttpKernel\EventListener\ResponseListener('UTF-8'));

この呼び出しもかんたんでした！別のものを見てみましょう: ストリーム化されたレスポンスのサポートをそのまま利用したいですか？ ``StreamedResponseListener`` を購読するだけです。::

    $dispatcher->addSubscriber(new HttpKernel\EventListener\StreamedResponseListener());

そしてコントローラの中では、Response のインスタンスの代わりに ``StreamedResponse`` のインスタンスを返します。

.. tip::

    HttpKernel によってディスパッチされるイベントとそれらのイベントによって
    リクエストのフローをどのように変更できるのかに関しては
    Symfony2 の `内部`_ の章をご覧ください。

では、コントローラにフルの Response オブジェクトの代わりに文字列を返すことをさせるリスナーをつくりましょう。::

    class LeapYearController
    {
        public function indexAction(Request $request, $year)
        {
            $leapyear = new LeapYear();
            if ($leapyear->isLeapYear($year)) {
                return 'Yep, this is a leap year! ';
            }

            return 'Nope, this is not a leap year.';
        }
    }

このフィーチャを実装するために、 ``kernel.view``
イベントにリスニングします。これはコントローラが呼び出された直後に発動します。これの目的は必要な場合にかぎり、コントローラの戻り値を適切な Response インスタンスに変換することです。::

    <?php

    // example.com/src/Simplex/StringResponseListener.php

    namespace Simplex;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;
    use Symfony\Component\HttpKernel\Event\GetResponseForControllerResultEvent;
    use Symfony\Component\HttpFoundation\Response;

    class StringResponseListener implements EventSubscriberInterface
    {
        public function onView(GetResponseForControllerResultEvent $event)
        {
            $response = $event->getControllerResult();

            if (is_string($response)) {
                $event->setResponse(new Response($response));
            }
        }

        public static function getSubscribedEvents()
        {
            return array('kernel.view' => 'onView');
        }
    }

コードはシンプルで ``kernel.view`` イベントはコントローラの戻り値が Response ではない場合にかぎり発動しイベント上でレスポンスを設定するとイベントプロパゲーションが止まります (リスナーはほかのビューリスナーに干渉することはできません)。

フロントコントローラでリスナーを登録することをお忘れなく。::

    $dispatcher->addSubscriber(new Simplex\StringResponseListener());

.. note::

    サブスクライバを登録することを忘れると、HttpKernel はわかりやすいメッセージとともに
    例外を投げます: ``The controller must return a response
    (Nope, this is not a leap year. given).``

この点で、我々のフレームワーク全体のコードは可能なかぎりコンパクトで、既存のライブラリの集まりで構成されています。機能の拡張はイベントリスナー/サブスクライバを登録することで行われます。

うまくいけば、 ``HttpKernelInterface`` を求めることがとても強力であることの理解がより深まります。デフォルトの実装である ``HttpKernel`` は労力なしでそのまま使うことのできるたくさんのクールなフィーチャをもたらします。そして ``HttpKernel`` は Symfony2 と Silex フレームワークを推し進めるコードなので、両方の世界の最高のものをもたらします: カスタムフレームワークで、あなたのニーズにテーラメードされていますが、多くのサイトで動くことが証明された堅牢でよくメンテナンスされた低レイヤーのアーキテクチャおよび セキュリティの問題に関して検査されおよびじゅうぶんにスケールアウトできることが証明されたコードにもとづいています。

.. _`内部`: http://symfony.com/doc/current/book/internals.html#events
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


.. 2012/05/08 masakielastic 60617d75a2d7672f8674d9664f892f5178001f27
