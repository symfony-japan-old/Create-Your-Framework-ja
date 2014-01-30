Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 10)
========================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


この連載のパート2の結論において、Symfony2 コンポーネントを使うことによる大きな恩恵を話しました: コンポーネントを使うすべてのフレームワークとアプリケーションのあいだの *相互運用性* です。 ``HttpKernelInterface`` を実装するフレームワークをつくることでこのゴールに向かって大きく前進しましょう。::

    namespace Symfony\Component\HttpKernel;

    interface HttpKernelInterface
    {
        /**
         * @return Response A Response instance
         */
        function handle(Request $request, $type = self::MASTER_REQUEST, $catch = true);
    }

``HttpKernelInterface`` は冗談抜きに HttpKernel コンポーネントの中でもっとも重要なピースでしょう。このインターフェイスを実装するフレームワークとアプリケーションはじゅうぶんな相互運用性があります。さらに、たくさんのすばらしいフィーチャがそのままついてきます。

このインターフェイスを実装するようにフレームワークをアップデートします。::

    <?php

    // example.com/src/Framework.php

    // ...

    use Symfony\Component\HttpKernel\HttpKernelInterface;

    class Framework implements HttpKernelInterface
    {
        // ...

        public function handle(Request $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
        {
            // ...
        }
    }

この変更がささいなものであれ、たくさんのものがもたらされます！もっとも印象に残るものの1つ: 透過的な `HTTP キャッシング`_ サポートについて語りましょう。

``HttpCache`` クラスは PHP で書かれたフルフィーチャのリバースプロキシを実装します; ``HttpKernelInterface`` を実装し
``HttpKernelInterface`` インスタンスを包み込みます。::

    // example.com/web/front.php

    $framework = new Simplex\Framework($dispatcher, $matcher, $resolver);
    $framework = new HttpKernel\HttpCache\HttpCache($framework, new HttpKernel\HttpCache\Store(__DIR__.'/../cache'));

    $framework->handle($request)->send();

HTTP キャッシングサポートをフレームワークに追加するために必要なことはこれですべてです。驚きました？

キャッシュ機能の調整は HTTP キャッシュヘッダー経由で行うことが必要です。たとえば、10秒のあいだにレスポンスをキャッシュしたいのであれば、 ``Response::setTtl()`` メソッドを使います。::

    // example.com/src/Calendar/Controller/LeapYearController.php

    public function indexAction(Request $request, $year)
    {
        $leapyear = new LeapYear();
        if ($leapyear->isLeapYear($year)) {
            $response = new Response('Yep, this is a leap year!');
        } else {
            $response = new Response('Nope, this is not a leap year.');
        }

        $response->setTtl(10);

        return $response;
    }

.. tip::

    筆者のように、リクエストをシミュレートすることでコマンドラインからフレームワークを実行している場合 (``Request::create('/is_leap_year/2012')``) 、Response のインスタンスを文字列表現 (``echo $response;``) として吐き出させることでかんたんにデバッグすることできます。文字列表現にはレスポンスのコンテントと同じようにすべてのヘッダーが含まれます。

これが正しく動くのか検証するためには、レスポンスのコンテントにランダムな数値を追加し、10秒ごとに数値だけが変わることをチェックします。::

    $response = new Response('Yep, this is a leap year! '.rand());

.. note::

    プロダクション環境にデプロイするとき、Symfony2 のリバースプロキシを使い続けるか
    (共用ホスティングの用途にすぐれています) もしくはよりよい方法は、
    `Varnish`_ のようなより効率的なリバースプロキシに切り替えることです。

アプリケーションキャッシュをマネージするために HTTP キャッシュヘッダーを使うやりかたはとても強力で、HTTP の仕様の有効期限とバリデーションモデルの両方を使うことができるので、キャッシング戦略をきめ細かく調整できます。これらのコンセプトを快適に感じないのであれば、 `HTTP
キャッシング`_ の章を読んでいただくことをおすすめします。

Response クラスにはたくさんのメソッドが用意されているので、HTTP キャッシュの調整をとてもかんたんにできます。もっとも強力なものの1つは ``setCache()`` でもっともよく使われるキャッシング戦略を1つのシンプルな配列に抽象化します。::

    $date = date_create_from_format('Y-m-d H:i:s', '2005-10-15 10:00:00');

    $response->setCache(array(
        'public'        => true,
        'etag'          => 'abcde',
        'last_modified' => $date,
        'max_age'       => 10,
        's_maxage'      => 10,
    ));

    // 上記のコードは次のコードと同等です
    $response->setPublic();
    $response->setEtag('abcde');
    $response->setLastModified($date);
    $response->setMaxAge(10);
    $response->setSharedMaxAge(10);

バリデーションモデルを使うとき、 ``isNotModified()`` メソッドによってレスポンスの生成を可能なかぎり短縮することでレスポンスの時間をかんたんに短くすることができます。::

    $response->setETag('whatever_you_compute_as_an_etag');

    if ($response->isNotModified($request)) {
        return $response;
    }
    $response->setContent('The computed content of the response');

    return $response;

HTTP キャッシングを使うことはすばらしいですが、ページ全体をキャッシュできないとしたらどうでしょうか？
動的なサイドバー以外の残りのコンテンツをキャッシュしたい場合は？Edge Side Includes (`ESI`_) が助けになります！コンテンツ全体を生成する代わりに、ESI によってページの領域をサブリクエスト呼び出しのコンテンツとしてマークできます。::

    This is the content of your page

    Is 2012 a leap year? <esi:include src="/leapyear/2012" />

    Some other content

HttpCache によってサポートされる ESI タグに関して、 ``ESI`` クラスのインスタンスにそれを渡す必要があります。 ``ESI`` クラスは自動的に ESI タグをパースし、これらを適切なコンテントに変換するためにサブリクエストを作成します。::

    $framework = new HttpKernel\HttpCache\HttpCache(
        $framework,
        new HttpKernel\HttpCache\Store(__DIR__.'/../cache'),
        new HttpKernel\HttpCache\ESI()
    );

.. note::

    ESI を動かすために、Symfony2 の実装のようなリバースプロキシを使う必要があります。
    `Varnish`_ は最良の代替ソフトウェアです。

複雑な HTTP キャッシングストラテジと/またはたくさんの ESI のインクルードタグを使うとき、リソースをキャッシュすべきかどうかを理解するのは困難な状況があり得ます。デバッグを楽にするために、デバッグモードを有効にすることができます。::

    $framework = new HttpCache($framework, new Store(__DIR__.'/../cache'), new ESI(), array('debug' => true));

デバッグモードではキャッシュレイヤーが行ったことをそれぞれのレスポンスに ``X-Symfony-Cache`` ヘッダーを追加されます。

.. code-block:: text

    X-Symfony-Cache:  GET /is_leap_year/2012: stale, invalid, store

    X-Symfony-Cache:  GET /is_leap_year/2012: fresh

HttpCache には多くのフィーチャが備わっており、
RFC5861 で定義された ``stale-while-revalidate`` と ``stale-if-error`` HTTP Cache-Control
エクステンションが用意されています。

単一のインターフェイスに加えて、HttpKernel コンポーネントに組み込まれている多くのフィーチャから恩恵を得ることができます; HTTP キャッシングはそれらの1つでしかありませんが、我々のアプリケーションを羽ばたかせるために大切なフィーチャです！

.. _`HTTP キャッシング`: http://symfony.com/doc/current/book/http_cache.html
.. _`ESI`:          http://en.wikipedia.org/wiki/Edge_Side_Includes
.. _`Varnish`:      https://www.varnish-cache.org/
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
