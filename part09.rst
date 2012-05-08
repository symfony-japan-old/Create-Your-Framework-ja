Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 9)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


我々のフレームワークにはよいフレームワークにあるべき主な特徴が欠けています:
*拡張性* です。拡張できるということはリクエストが処理される方法を修正するフレームワークのライフサイクルに手軽にフックできることを意味します。

どんな種類のフックを話しているのでしょうか？たとえば、認証もしくはキャッシュです。柔軟性を保つために、フックはプラグアンドプレイでなければなりません; アプリケーションのために「登録する」ものはあなた独自のニーズしだいの次のものとは異なります。多くのソフトウェアは Drupal や Wordpress のような似たコンセプトをもっています。プログラミング言語によっては、Python の `WSGI`_ や Ruby の `Rack`_ などの標準さえあります。

PHP にはそのような標準はありませんが、フレームワークに任意のふるまいをアタッチすることを可能にするために、よく知られたデザインパターンである *Observer* を使おうとしています; Symfony2 EventDispatcher コンポーネントはこのパターンの軽量バージョンです。

.. code-block:: javascript

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*",
            "symfony/http-kernel": "2.1.*",
            "symfony/event-dispatcher": "2.1.*"
        },
        "autoload": {
            "psr-0": { "Simplex": "src/", "Calendar": "src/" }
        }
    }

どのように動くのでしょうか？ *ディスパッチャ* 、イベントディスパッチャシステムの中心的なオブジェクトは 自身にディスパッチされた *イベント* の *リスナー* に通知します。別の説明をします:
あなたのコードがイベントをディスパッチャにディスパッチし、ディスパッチャはイベントに対して登録されたすべてのリスナーに通知し、それぞれのリスナーはイベントに対してお望みのことを行います。

たとえば、Google
Analytics のコードをすべてのレスポンスに透過的に追加するリスナーをつくってみましょう。

このコードが動くようにするために、Response インスタンスを返す直前にフレームワークはイベントをディスパッチしなければなりません。::

    <?php

    // example.com/src/Simplex/Framework.php

    namespace Simplex;

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Matcher\UrlMatcherInterface;
    use Symfony\Component\Routing\Exception\ResourceNotFoundException;
    use Symfony\Component\HttpKernel\Controller\ControllerResolverInterface;
    use Symfony\Component\EventDispatcher\EventDispatcher;

    class Framework
    {
        protected $matcher;
        protected $resolver;
        protected $dispatcher;

        public function __construct(EventDispatcher $dispatcher, UrlMatcherInterface $matcher, ControllerResolverInterface $resolver)
        {
            $this->matcher = $matcher;
            $this->resolver = $resolver;
            $this->dispatcher = $dispatcher;
        }

        public function handle(Request $request)
        {
            $this->matcher->getContext()->fromRequest($request);

            try {
                $request->attributes->add($this->matcher->match($request->getPathInfo()));

                $controller = $this->resolver->getController($request);
                $arguments = $this->resolver->getArguments($request, $controller);

                $response = call_user_func_array($controller, $arguments);
            } catch (ResourceNotFoundException $e) {
                $response = new Response('Not Found', 404);
            } catch (\Exception $e) {
                $response = new Response('An error occurred', 500);
            }

            // response イベントをディスパッチします
            $this->dispatcher->dispatch('response', new ResponseEvent($response, $request));

            return $response;
        }
    }

フレームワークが Request を処理する度に、 ``ResponseEvent`` イベントがディスパッチされます。::

    <?php

    // example.com/src/Simplex/ResponseEvent.php

    namespace Simplex;

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\EventDispatcher\Event;

    class ResponseEvent extends Event
    {
        private $request;
        private $response;

        public function __construct(Response $response, Request $request)
        {
            $this->response = $response;
            $this->request = $request;
        }

        public function getResponse()
        {
            return $this->response;
        }

        public function getRequest()
        {
            return $this->request;
        }
    }

最後のステップはフロントコントローラの中でディスパッチャオブジェクトの生成と ``response`` イベントに対するリスナーの登録です。::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    // ...

    use Symfony\Component\EventDispatcher\EventDispatcher;

    $dispatcher = new EventDispatcher();
    $dispatcher->addListener('response', function (Simplex\ResponseEvent $event) {
        $response = $event->getResponse();

        if ($response->isRedirection()
            || ($response->headers->has('Content-Type') && false === strpos($response->headers->get('Content-Type'), 'html'))
            || 'html' !== $event->getRequest()->getRequestFormat()
        ) {
            return;
        }

        $response->setContent($response->getContent().'GA CODE');
    });

    $framework = new Simplex\Framework($dispatcher, $matcher, $resolver);
    $response = $framework->handle($request);

    $response->send();

.. note::

    リスナーはたんなる概念の証明で body タグの直前に Google
    Analytics のコードを追加すべきです。

ご覧のとおり、 ``addListener()`` は有効な PHP コールバックを名前つきイベント (``response``) に関連づけます; イベントの名前は ``dispatch()`` の呼び出しで使われたものと同じでなければなりません。

リスナーにおいて、レスポンスがリダイレクトではなく、リクエストされたフォーマットが HTML であり、レスポンスの Content-Type が HTML である場合にかぎり、Google Analytics コードを追加します (これらの条件はコードから Request と Response のデータを操作する作業の負担を和らげてくれることの実証になります)。

これまでのところうまくいっていますが、同じイベントに別のリスナーを追加します。Response の ``Content-Length`` がまだセットされていない場合にこれをセットすることを考えてみましょう。

    $dispatcher->addListener('response', function (Simplex\ResponseEvent $event) {
        $response = $event->getResponse();
        $headers = $response->headers;

        if (!$headers->has('Content-Length') && !$headers->has('Transfer-Encoding')) {
            $headers->set('Content-Length', strlen($response->getContent()));
        }
    });

以前に登録したリスナーの前か後にこのコードのピースを追加するかによって、 ``Content-Length`` ヘッダーに対して間違ったもしくは正しい値を得ることになります。ときには、リスナーの順番は重要ですが、デフォルトでは、すべてのリスナーは同じ優先順位の ``0`` で登録されます。リスナーをより早く実行するようディスパッチャに伝えるには、優先順位を正の数に変更します; 負の数は優先順位の低いリスナーに使うことができます。ここでは ``Content-Length`` リスナーを最後に実行させたいので、優先順位を ``-255`` に変更します。::

    $dispatcher->addListener('response', function (Simplex\ResponseEvent $event) {
        $response = $event->getResponse();
        $headers = $response->headers;

        if (!$headers->has('Content-Length') && !$headers->has('Transfer-Encoding')) {
            $headers->set('Content-Length', strlen($response->getContent()));
        }
    }, -255);

.. tip::

    フレームワークをつくるとき、優先順位を考え (たとえばreserve some numbers
    内部のリスナーに対するいくつかの値を反転させます) それらをドキュメントに徹底的に記載します。

ちょっとコードをリファクタリングして Google リスナーを独自のクラスに移動させましょう。::

    <?php

    // example.com/src/Simplex/GoogleListener.php

    namespace Simplex;

    class GoogleListener
    {
        public function onResponse(ResponseEvent $event)
        {
            $response = $event->getResponse();

            if ($response->isRedirection()
                || ($response->headers->has('Content-Type') && false === strpos($response->headers->get('Content-Type'), 'html'))
                || 'html' !== $event->getRequest()->getRequestFormat()
            ) {
                return;
            }

            $response->setContent($response->getContent().'GA CODE');
        }
    }

ほかのリスナーと同じことを行います。::

    <?php

    // example.com/src/Simplex/ContentLengthListener.php

    namespace Simplex;

    class ContentLengthListener
    {
        public function onResponse(ResponseEvent $event)
        {
            $response = $event->getResponse();
            $headers = $response->headers;

            if (!$headers->has('Content-Length') && !$headers->has('Transfer-Encoding')) {
                $headers->set('Content-Length', strlen($response->getContent()));
            }
        }
    }

フロントコントローラは次のようになります。::

    $dispatcher = new EventDispatcher();
    $dispatcher->addListener('response', array(new Simplex\ContentLengthListener(), 'onResponse'), -255);
    $dispatcher->addListener('response', array(new Simplex\GoogleListener(), 'onResponse'));

コードがクラスにすばらしく包まれていますが、それでも少し問題が残っています: 優先順位の知識がリスナー自身ではなくフロントコントローラに「ハードコード」されています。それぞれのアプリケーションに対して、適切な優先順位を設定することを覚えておかなければなりません。さらに、リスナーのメソッド名はここにも公開され、このことは我々のリスナーをリファクタリングすることはこれらのリスナーに依存しているすべてのアプリケーションを変更することを意味します。もちろん、ソリューションがあります。リスナーの代わりにサブスクライバを使います。::

    $dispatcher = new EventDispatcher();
    $dispatcher->addSubscriber(new Simplex\ContentLengthListener());
    $dispatcher->addSubscriber(new Simplex\GoogleListener());

サブスクライバは興味のあるすべてのイベントを知っており、 ``getSubscribedEvents()`` メソッドを通じてこの情報をディスパッチャに渡します。新しいバージョンの ``GoogleListener`` を見てみましょう。::

    <?php

    // example.com/src/Simplex/GoogleListener.php

    namespace Simplex;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;

    class GoogleListener implements EventSubscriberInterface
    {
        // ...

        public static function getSubscribedEvents()
        {
            return array('response' => 'onResponse');
        }
    }

そして新しいバージョンの ``ContentLengthListener`` です。::

    <?php

    // example.com/src/Simplex/ContentLengthListener.php

    namespace Simplex;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;

    class ContentLengthListener implements EventSubscriberInterface
    {
        // ...

        public static function getSubscribedEvents()
        {
            return array('response' => array('onResponse', -255));
        }
    }

.. tip::

    単独のサブスクライバは必要な数だけリスナーをホストできます。

フレームワークを本当に柔軟なものにするためには、さらにイベントを追加することをためらわないでください; そしてそのまま使えるようにするには、さらにリスナーを追加してください。繰り返しますが、この連載は一般的なフレームワーク作成に関するものではなく、あなたのニーズに合わせてテーラーメードされるものです。満足したら止めて、そこからコードをさらに進化させましょう。

.. _`WSGI`: http://www.python.org/dev/peps/pep-0333/#middleware-components-that-play-both-sides
.. _`Rack`: http://rack.rubyforge.org/
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


.. 2012/05/06 masakielastic d0ff8bc245d198bd8eadece0a2f62b9ecd6ae6ab
