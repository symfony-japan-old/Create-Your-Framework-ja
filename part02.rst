Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 2)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


コードのリファクタリングに入る前に、話を前回に戻し、簡素な (plain-old) PHP アプリケーションをそのままに維持する代わりになぜフレームワークを使いたいのか、もっともシンプルなコードのスニペットに対してでさえ、フレームワークを使うはじめることが本当によい考えであるのはなぜ、コンポーネントの上にフレームワークを作ることがゼロから作るよりもよいのかなどについて見ることにします。

.. note::

    大きなアプリケーションを複数人で取り組むときにフレームワークを使う昔からあきらかな利点は話しません。このトピックに関するよい資料はインターネットにたくさんあります。

昨日書いた「アプリケーション」がじゅうぶんにシンプルなものであっても、少数の問題に悩まされるからです。 ::

    <?php

    // framework/index.php

    $input = $_GET['name'];

    printf('Hello %s', $input);

最初に、 ``name`` クエリパラメータが URL のクエリ文字列の中で与えられていなければ、PHP の警告を得ることになります。ですのでこれを直しましょう。 ::

    <?php

    // framework/index.php

    $input = isset($_GET['name']) ? $_GET['name'] : 'World';

    printf('Hello %s', $input);

それから、 *アプリケーションは安全ではありません* 。本当ですよ。PHP コードのスニペットはシンプルですが、インターネットのセキュリティ問題でもっとも広まっている XSS (Cross-Site Scripting) の脆弱性があります。より安全なコードのバージョンは次のようになります。 ::

    <?php

    $input = isset($_GET['name']) ? $_GET['name'] : 'World';

    header('Content-Type: text/html; charset=utf-8');

    printf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8'));

.. note::

    お気づきかもしれませんが、``htmlspecialchars`` でコードをセキュアにする作業は
    退屈でまちがいやすいです。これが `Twig`_ のようなテンプレートエンジンを使う理由の1つです。
    オートエスケープがデフォルトで有効であり、これはよい考えでしょう 
    (そして明示的なエスケープはシンプルな ``e`` フィルタでより苦痛が減ります)。

自分の目で確かめるために、PHP の warning/notice を避けて、コードをよりセキュアにしたいのであれば、最初に書いたシンプルなコードはもはやシンプルではなくなります。

セキュリティの問題だけでなく、このコードはかんたんにテストすることもできません。テストする内容があまりなくても、可能なかぎり、もっともシンプルな PHP コードのスニペットに対してユニットテストを書くことは筆者のとっては自然ではなく不快に思えます。上記のコードに対する仮の PHPUnit
によるユニットテストは次のとおりです。::

    <?php

    // framework/test.php

    class IndexTest extends \PHPUnit_Framework_TestCase
    {
        public function testHello()
        {
            $_GET['name'] = 'Fabien';

            ob_start();
            include 'index.php';
            $content = ob_get_clean();

            $this->assertEquals('Hello Fabien', $content);
        }
    }

.. note::

    アプリケーションが少し大きくなってきたら、より多くの問題を見つけられるでしょう。
    これらにご興味があれば、 `フラットな PHP から Symfony2 へ`_ の章をご覧ください。

この点において、古いやりかたでコードを書くことを止め、フレームワーク (この文脈では採用するフレームワークを限定しない) を採用するのにセキュリティとテストは本当にとてもよい2つの理由であることに納得していないのであれば、この連載を読むことを止め、以前取り組んでいたコードに戻ることができます。

.. note::

    もちろん、フレームワークを使うことでセキュリティとテスタビリティがもたらされるだけでなく、   
    こころに止めておくべきより大切なことはフレームワークによってよりよいコードを早く書くことができるいうにならなければならないということです。

HttpFoundation コンポーネントで OOP を進める
---------------------------------------------

Web のコードを書くということは HTTP とのやりとりです。ですので、我々の基本原則は `HTTP
の仕様`_　を中心に置きます。

HTTP の仕様はクライアント (たとえばブラウザ) とサーバー (Web サーバー経由でのアプリケーション) のやりとりのしかたを記述しています。クライアントとサーバーのあいだの対話は明確に記述された *メッセージ* 、リクエストとレスポンスによって決められます: *クライアントはサーバーにリクエストを送り、このリクエストをもとにサーバーはレスポンスを返します* 。

PHP において、リクエストはグローバル変数によって表れされ (``$_GET`` 、 ``$_POST`` 、 ``$_FILE`` 、``$_COOKIE`` 、``$_SESSION``...) レスポンスは関数によって生成されます (``echo`` 、 ``header`` 、 ``setcookie`` 、 ...)。

コードの改善に向けた最初のステップはおそらく *オブジェクト指向*
のアプローチを使うことです; これが Symfony2 HttpFoundation コンポーネントのメインゴールです:
*オブジェクト指向* のレイヤーによって PHP デフォルトのグローバル変数と関数を置き換えます。

このコンポーネントを使うには、 ``composer.json`` ファイルを開き、プロジェクトの依存するものとして追加します。

.. code-block:: javascript

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*"
        }
    }

それから、composer の ``update`` コマンドを実行します。

.. code-block:: sh

    $ php composer.phar update

最後に、``autoload.php`` ファイルの一番下の行で、コンポーネントをオートロードするために必要なコードを追加します。::

    <?php

    // framework/autoload.php

    $loader->registerNamespace('Symfony\\Component\\HttpFoundation', __DIR__.'/vendor/symfony/http-foundation');

では ``Request`` と
``Response`` クラスを使ってアプリケーションを書き換えましょう。::

    <?php

    // framework/index.php

    require_once __DIR__.'/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();

    $input = $request->get('name', 'World');

    $response = new Response(sprintf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8')));

    $response->send();

``createFromGlobals()`` メソッドは PHP の現在のグローバル変数をもとに ``Request`` オブジェクトを生成します。

``send()`` メソッドは ``Response`` オブジェクトにクライアントに送り戻します (これは最初に HTTP ヘッダーを出力し、その後にコンテンツが続きます)。

.. tip::

    ``send()`` の呼び出しの前に、Response オブジェクトが HTTP の仕様と互換性があることを保証するため ``prepare()`` メソッド (``$response->prepare($request);``) 呼び出しを追加すべきです。たとえば、``HEAD`` メソッドでページを呼び出すのであれば、 Response オブジェクトのコンテンツは削除されます。

前のコードとの主な違いは HTTP メッセージのトータルなコントロールがあることです。お望みのリクエストをつくり、ちょうどよいときにレスポンスを送ることができます。

.. note::

    ``Content-Type`` ヘッダーを明示的に設定しませんでした。
    Response オブジェクトのデフォルトの文字セットが ``UTF-8`` だからです。

``Request`` クラスの親切でシンプルな API のおかげで、すべてのリクエスト情報を思い通りに操作できます。::

    <?php

    // クエリパラメータを除くリクエストされた URI (たとえば /about)
    $request->getPathInfo();

    // GET と POST 変数をそれぞれ取得します
    $request->query->get('foo');
    $request->request->get('bar', 'bar が存在していない場合のデフォルトの値');

    // SERVER 変数を取得します
    $request->server->get('HTTP_HOST');

    // foo の値で特定された UploadedFile のインスタンスを取得します
    $request->files->get('foo');

    // COOKIE の値を取得します
    $request->cookies->get('PHPSESSID');

    // 小文字で標準化されたキーで HTTP リクエストヘッダーを取得します
    $request->headers->get('host');
    $request->headers->get('content_type');

    $request->getMethod();    // GET, POST, PUT, DELETE, HEAD
    $request->getLanguages(); // クライアントが受け付ける言語の配列

リクエストのシミュレーションを行うこともできます。::

    $request = Request::create('/index.php?name=Fabien');

``Response`` クラスによって、レスポンスをかんたんに調整できます。::

    <?php

    $response = new Response();

    $response->setContent('Hello world!');
    $response->setStatusCode(200);
    $response->headers->set('Content-Type', 'text/html');

    // HTTP キャッシュヘッダーの設定を変更します
    $response->setMaxAge(10);

.. tip::

    Response のデバッグを行うには、これを文字にキャスティングします。これはレスポンスの HTML 表現 (ヘッダーとコンテンツ) を返します。

言い忘れていましたが、これらのクラスは、Symfony のコードのほかのすべてのクラスのように、セキュリティの問題に関して独立した会社によって `検査`_ されました。そして Open-Source のプロジェクトであることは世界中の開発者がコードを見てくれており、潜在的なセキュリティの問題がすでに修正されていることも意味します。
あなたがお手製のフレームワークにプロフェッショナルなセキュリティ検査を最後に依頼したのはいつですか？

クライアントの IP アドレスの取得のようなことはシンプルですが、セキュアではありません。::

    <?php

    if ($myIp == $_SERVER['REMOTE_ADDR']) {
        // クライアントは既知のものなので、 より多くの権限がもたらされます
    }

運用サーバーの前にリバースプロキシを追加するまでに完全に動きます; この点で、開発マシン (プロキシがない) とサーバーの両方で動くようにコードを変更しなければなりません。::

    <?php

    if ($myIp == $_SERVER['HTTP_X_FORWARDED_FOR'] || $myIp == $_SERVER['REMOTE_ADDR']) {
        // クライアントは既知のものなので、より多くの権限がもたらされます
    }

``Request::getClientIp()`` メソッドを使うことで、1日目よりも正しいふるまいがもたらされます (プロキシチェーンがあるケースをカバーします)::

    <?php

    $request = Request::createFromGlobals();

    if ($myIp == $request->getClientIp()) {
        // クライアントは既知のものなので、より多くの権限がもたらされます
    }

新しい恩恵が加わります: デフォルトで *セキュア* であることです。セキュアであるということはどういう意味でしょうか？ ``$_SERVER['HTTP_X_FORWARDED_FOR']`` の値は信用できません。プロキシがないときエンドユーザーによって操作できるからです。ですので、プロキシなしの運用環境でこのコードを使うのであれば、システムを悪用することは造作もないことです。 ``trustProxyData()`` を呼び出すことでこのヘッダーを信用することを明示的に示さなければならないので、 ``getClientIp()`` メソッドには当てはまりません::

    <?php

    Request::trustProxyData();

    if ($myIp == $request->getClientIp(true)) {
        // クライアントは既知のものなので、より多くの権限がもたらされます
    }

ですので、 ``getClientIp()`` メソッドはすべての状況で安全に動きます。プロジェクトのコンフィギュレーションが何であれ、すべてのプロジェクトでこれを使うことが可能で、これは正しくかつ安全に動きます。これがフレームワークを使うことのゴールの1つです。ゼロからフレームワークを書くのであれば、これらすべてのケースを考えなければなりません。すでに動くテクノロジーを使いませんか？

.. note::

    HttpFoundation コンポーネントをくわしい内容を学びたいのであれば、
    Symfony の公式サイトの `API`_ もしくは専用の `ドキュメント`_ を見ることができます。

ともかく、我々の手元には最初のフレームワークがあります。望むのであれば今すぐに止められます。Symfony2 HttpFoundation コンポーネントを使うだけで、コードはより改善され、テストできるようになります。たくさんの日常の問題はすでに解決されているのでコードをより早く書くことができるようにもなります。

当然のことながら、Drupal (次のバージョン8)などのプロジェクトが HttpFoundation コンポーネントを採用しました; コンポーネントがそれらのプロジェクトに役立つのであれば、あなたにも役立つことでしょう。車輪は再発明するのはやめましょう。

もう1つ追加された恩恵を話し忘れるところでした: HttpFoundation
コンポーネントを使うことで、すべてのフレームワークとアプリケーションのあいだの相互運用性をよりよくするはじまりとなります (執筆の時点では `Symfony2`_ 、 `Drupal 8`_ 、 `phpBB 4`_ 、 `Silex`_  、 `Midgard CMS`_ 、 `Zikula`_ ...)。

.. _`Twig`:                     http://twig.sensiolabs.com/
.. _`フラットな PHP から Symfony2 へ`: http://docs.symfony.gr.jp/symfony2/book/from_flat_php_to_symfony2.html
.. _`HTTP の仕様`:       http://tools.ietf.org/wg/httpbis/
.. _`API`:                      http://api.symfony.com/2.0/Symfony/Component/HttpFoundation.html
.. _`ドキュメント`:            http://symfony.com/doc/current/components/http_foundation/introduction.html
.. _`検査`:                  http://symfony.com/blog/symfony2-security-audit
.. _`Symfony2`:                 http://symfony.com/
.. _`Drupal 8`:                 http://drupal.org/
.. _`phpBB 4`:                  http://www.phpbb.com/
.. _`Silex`:                    http://silex.sensiolabs.org/
.. _`Midgard CMS`:              http://www.midgard-project.org/
.. _`Zikula`:                   http://zikula.org/
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

.. 2012/04/26 masakielastic d0ff8bc245d198bd8eadece0a2f62b9ecd6ae6ab

