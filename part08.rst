Web フレームワークをつくろう - Symfony2 コンポーネントの上に (パート 8)
=======================================================================

.. note::

    この記事は Symfony2 コンポーネントでフレームワークをつくる方法を説明した連載記事の一部です: `1`_, `2`_, `3`_, `4`_, `5`_, `6`_, `7`_, `8`_, `9`_, `10`_, `11`_, `12`_


用心深い読者の方に昨日つくったフレームワークにささいだが重大なバグがあることを指摘しました。フレームワークをつくるとき、フレームワークのふるまいが想定どおりなのか確認しなければなりません。そうでなければ、それにもとづいたすべてのアプリケーションは同じバグをさらすことになります。よいニュースはバグを修正すれば、無数のアプリケーションも修正したことになります。

今日のミッションは `PHPUnit`_ を使ってすでに作成したフレームワークのユニットテストを書くことです。
``example.com/phpunit.xml.dist`` の中で PHPUnit の設定ファイルを作ります。

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>

    <phpunit backupGlobals="false"
             backupStaticAttributes="false"
             colors="true"
             convertErrorsToExceptions="true"
             convertNoticesToExceptions="true"
             convertWarningsToExceptions="true"
             processIsolation="false"
             stopOnFailure="false"
             syntaxCheck="false"
             bootstrap="vendor/.composer/autoload.php"
    >
        <testsuites>
            <testsuite name="Test Suite">
                <directory>./tests</directory>
            </testsuite>
        </testsuites>
    </phpunit>

このコンフィギュレーションはたいていの PHPUnit の道理に適ったデフォルト設定を定義します; さらに興味深いことに、オートローダはテストをブートストラップするために使われ、テストは ``example.com/tests/`` ディレクトリに保存されます。

では、「not found」のリソースに対するテストを書いてみましょう。テストを書くときにすべての依存オブジェクトの生成を避けるためと、本当に必要なことだけをユニットテストするために、 `テストダブル`_ を使います。テストダブルは具象クラスの代わりにインターフェイスに依存するときに作るのがかんたんです。さいわいなことに、Symfony2 は URL マッチャとコントローラリゾルバのようなコアオブジェクトに対するインターフェイスを提供します。これらを利用するためにフレームワークを修正します。::

    <?php

    // example.com/src/Simplex/Framework.php

    namespace Simplex;

    // ...

    use Symfony\Component\Routing\Matcher\UrlMatcherInterface;
    use Symfony\Component\HttpKernel\Controller\ControllerResolverInterface;

    class Framework
    {
        protected $matcher;
        protected $resolver;

        public function __construct(UrlMatcherInterface $matcher, ControllerResolverInterface $resolver)
        {
            $this->matcher = $matcher;
            $this->resolver = $resolver;
        }

        // ...
    }

最初のテストを書く準備はできています。::

    <?php

    // example.com/tests/Simplex/Tests/FrameworkTest.php

    namespace Simplex\Tests;

    use Simplex\Framework;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\Routing\Exception\ResourceNotFoundException;

    class FrameworkTest extends \PHPUnit_Framework_TestCase
    {
        public function testNotFoundHandling()
        {
            $framework = $this->getFrameworkForException(new ResourceNotFoundException());

            $response = $framework->handle(new Request());

            $this->assertEquals(404, $response->getStatusCode());
        }

        protected function getFrameworkForException($exception)
        {
            $matcher = $this->getMock('Symfony\Component\Routing\Matcher\UrlMatcherInterface');
            $matcher
                ->expects($this->once())
                ->method('match')
                ->will($this->throwException($exception))
            ;
            $resolver = $this->getMock('Symfony\Component\HttpKernel\Controller\ControllerResolverInterface');

            return new Framework($matcher, $resolver);
        }
    }

このテストはどのルートにもマッチしないリクエストをシミュレートします。そういうものとして、
``match()`` メソッドは``ResourceNotFoundException`` 例外を返し、我々のフレームワークがこの例外を 404 レスポンスに変換することをテストしています。

このテストの実行はシンプルで ``example.com`` ディレクトリから ``phpunit`` を実行するだけです。

.. code-block:: bash

    $ phpunit

.. note::

    この連載の目的からはずれるので、コードがどのように動くのかくわしくは説明しませんが、
    何が行われているのかわからなければ、
    PHPUnit の `テストダブル`_ のドキュメントを読むことをおすすめします。

テストを実行した後で、緑のバーが見えます。そうでなければ、テストもしくはフレームワークのコードにバグがあります！

コントローラに投げられる例外のユニットテストを追加することはかんたんです。::

    public function testErrorHandling()
    {
        $framework = $this->getFrameworkForException(new \RuntimeException());

        $response = $framework->handle(new Request());

        $this->assertEquals(500, $response->getStatusCode());
    }

最後に、適切な Response が実際にあるときのテストを書いてみましょう。::

    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Controller\ControllerResolver;

    public function testControllerResponse()
    {
        $matcher = $this->getMock('Symfony\Component\Routing\Matcher\UrlMatcherInterface');
        $matcher
            ->expects($this->once())
            ->method('match')
            ->will($this->returnValue(array(
                '_route' => 'foo',
                'name' => 'Fabien',
                '_controller' => function ($name) {
                    return new Response('Hello '.$name);
                }
            )))
        ;
        $resolver = new ControllerResolver();

        $framework = new Framework($matcher, $resolver);

        $response = $framework->handle(new Request());

        $this->assertEquals(200, $response->getStatusCode());
        $this->assertContains('Hello Fabien', $response->getContent());
    }

このテストにおいて、マッチしてシンプルなコントローラを返すルートをシミュレートします。レスポンスステータスが 200 でコンテンツがコントローラにセットしたものと同じであることをチェックします。

すべてのあり得るユースケースをカバーしたことをチェックするために、PHPUnit のテストカバレッジを実行します (最初に `XDebug`_ を有効にする必要があります)。

.. code-block:: bash

    $ phpunit --coverage-html=cov/

``example.com/cov/src_Simplex_Framework.php.html`` をブラウザで開き、 Framework クラスに対するすべての行が緑色であることをチェックします (このことはテストが実行されたときにこれらが訪問済みであることを意味します)。

これまで書いたシンプルなオブジェクト指向のコードのおかげで、フレームワークのあり得るすべてのユースケースをカバーするユニットテストを書くことができました; テストダブルは我々が Symfony2 のコードではなく我々のコードを本当にテストしていることを保証しました。

我々が書いたコートに(再び)確信を得ることができたので、次に我々のフレームワークに追加したいバッチの機能を安心して考えることができます。

.. _`PHPUnit`:      http://www.phpunit.de/manual/current/ja/index.html
.. _`テストダブル`: http://www.phpunit.de/manual/current/ja/test-doubles.html
.. _`XDebug`:       http://xdebug.org/
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