# WebStream
WebStreamはMVCアーキテクチャをベースとしたWebアプリケーションフレームワークです。  
さらにS(Service)層を追加した4層構造のアーキテクチャとなっています。

## WebStreamのアーキテクチャ
WebStreamはMVCを拡張したアーキテクチャを採用しており、Serviceレイヤを追加しています。  
MVCはFat Controller/Fat Model問題を引き起こしやすいアーキテクチャであるため、ビジネスロジックはServiceに定義します。  
また、View内でビジネスロジックを書く場合はHelperを利用し、Viewはレンダリングに専念させます。


## [Controller](#controller)
Contollerではクライアントからのリクエストを受け付け、ServiceまたはModelを呼び出します。  
Controllerの処理が完了したらViewを呼び出します。Viewへパラメータを渡す場合、Serviceにセットします。  
原則的にControllerにビジネスロジックを記述してはなりません。  
`app/controllers`に`WebStream\Core\CoreController`クラスを継承したクラスを定義します。

### Controllerクラスの定義
Controllerクラスは`\WebStream\Core\CoreController`クラスを継承します。  
ControllerクラスからはServiceクラスまたはModelクラスを参照できます。またViewテンプレートを呼び出して描画できます。

#### Serviceクラス、Modelクラス呼び出し
Serviceクラス、Modelクラスは以下のように呼び出します。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class BlogController extends CoreController
{
    public funciton execute()
    {
        // $this->{ページ名}->(Service|Modelクラスのメソッド)
        $this->Blog->entry();
    }
}
```

Controllerクラス内の`$this->{ページ名}`オブジェクトにはServiceクラスまたはModelクラスのインスタンスが格納されています。  
Serviceクラスを定義している場合はServiceクラスインスタンスが格納されます。このときのページ名はアッパーキャメルケースで指定します。  
Serviceクラスを定義せずModelクラスのみ定義した場合はModelクラスインスタンスが格納されます。Serviceクラスに特段のビジネスロジックを記述する必要がなく、DBからのデータを取り出したいだけの場合など、Controllerクラスから直接Modelクラスにアクセスすることができます。  
Controllerクラスでは[アノテーション](#annotaion)を使ってメソッドやプロパティを操作できます。

#### Viewテンプレート呼び出し
HTMLを描画するにはControllerからViewテンプレートを呼び出します。Viewテンプレート呼び出しは[アノテーション](#annotaion)を利用します。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class BlogController extends CoreController
{
    /**
     * @Template("index.tmpl")
     */
    public funciton execute()
    {
        // $this->{ページ名}->(Service|Modelクラスのメソッド)
        $this->Blog->entry();
    }
}
```

この処理で`@Template`に指定したテンプレートファイル`index.tmpl`を呼び出します。  
テンプレートファイルは`app/views/(ページ名)/`に保存します。このときのページ名はスネークケースで指定します(詳細は[View](#view)で説明します)。


## [Service](#service)
ServiceクラスではContollerクラスから受け取ったリクエストやデータを使って処理をしたり、View経由でビジネスロジックを実行します。  
メインとなるビジネスロジックはServiceに記述します。データベースへの問い合わせが必要な場合はModelへ問い合わせます。  
また、Serviceでは開発者が個別に定義したクラス(ライブラリ)を利用することができます。Serviceで処理するロジックがない場合などはServiceを定義する必要はありません。  
`app/services`に`WebStream\Core\CoreService`クラスを継承したクラスを定義します。

```php
namespace MyBlog;

use WebStream\Core\CoreService;

class BlogService extends CoreService
{
    public funciton entry()
    {
        // $this->{ページ名}->(Modelクラスのメソッド)
        $this->Blog->getEntryData();
    }
}
```

Serviceクラス内の`$this->{ページ名}`オブジェクトにはModelクラスのインスタンスが格納されています。ModelクラスにアクセスしてDB処理を実行します。  
また、ServiceクラスにはContoller、Service、Model、Helperの各クラスに属さないユーザ定義クラスへのパスが通っています。`app/libraries/`ディレクトリに任意のクラスを定義することでServiceクラスからアクセスできます。例えば、外部APIにアクセスするクラスや、データをバインドするEntityクラスなど特定用途のクラスはlibrariesに定義してください。

Serviceクラスは、Viewから参照されるデータを格納する**ViewModel**の機能を内包しています。  
Viewから参照するデータはServiceクラスのGetterメソッドを経由して参照することになりますが、データ量が多くなる場合、Getterメソッドとそれに使うプロパティが増えていきます。
これらはServiceの機能とは本質的には無関係な定義であるにもかかわらず、コード量として多くなってしまうため、Serviceでは`PropertyProxy`という機能を使い、簡単にViewModelを使えるようにしています。  
Serviceクラスの中で、未定義のプロパティに値をセットすると、PropertyProxyにより、Serviceクラス内で値が保持されます。保持されたデータはViewから参照可能になります。
これにより、Viewから参照するデータをGetterメソッドなしで参照でき、カプセル化した状態で値を保持することができます。

**PropertyProxyを使わない場合**

```php
namespace MyBlog;

use WebStream\Core\CoreService;

// PropertyProxyなしの場合、Getterメソッドが必要
class BlogService extends CoreService
{
    // プロパティはpublicにすべきではない
    private $title;

    public function getTitle()
    {
        // ViewからはpublicなGetterメソッドでなければ参照不可
        return $this->title;
    }

    public funciton entry()
    {
        $this->title = "マイブログ";
    }
}
```

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ja" lang="ja">
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <head>
        <title>%H{$model->getTitle()}</title>
    </head>
    <body>
    </body>
</html>
```

**PropertyProxyを使った場合**


```php
namespace MyBlog;

use WebStream\Core\CoreService;

// PropertyProxy使用の場合、Getterメソッドは不要
class BlogService extends CoreService
{
    public funciton entry()
    {
        $this->title = "マイブログ";
    }
}
```

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ja" lang="ja">
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <head>
        <title>%H{$model->title}</title>
    </head>
    <body>
    </body>
</html>
```

なお、定義済みのプロパティにセットした場合は、PropertyProxyは使われません。  
また、この機能はModelでも有効ですが、Serviceクラスを作る必要のない簡単な処理の場合以外はServiceクラスのViewModel機能を使うことを推奨します。

## [Model](#model)
ModelクラスはControllerクラス、ServiceクラスまたはViewクラスからのリクエストや受け取ったデータを元にデータベースに問い合わせます。  
Serviceクラスが定義されない場合はController、Viewから直接呼び出されます。Modelにはデータベース問い合わせ処理を記述します。  
`app/models`に`WebStream\Core\CoreModel`クラスを継承したクラスを定義します。

```php
namespace MyBlog;

use WebStream\Core\CoreModel;

/**
 * @Database(driver="WebStream\Database\Driver\Mysql", config="config/database.mysql.ini")
 */
class BlogModel extends CoreModel
{
    public funciton getEntryData()
    {
        $sql = "SELECT * FROM T_Blog";
        return $this->select($sql);
    }
}
```

外部変数をパラメータに指定するには`$bind`変数にパラメータをセットします。
`$bind`変数には連想配列でプリペアードステートメントに設定する値を指定します。
データベース接続設定はクラスに[アノテーション](#annotaion)を指定します。

```php
namespace MyBlog;

use WebStream\Core\CoreModel;

/**
 * @Database(driver="WebStream\Database\Driver\Mysql", config="config/database.mysql.ini")
 */
class BlogModel extends CoreModel
{
    public funciton getEntryData()
    {
        $sql = "SELECT * FROM T_Blog WHERE id = :id";
        $bind = ["id" => 10];
        return $this->select($sql, $bind);
    }
}
```

Modelクラスでは以下のメソッドが利用可能です。

#### Modelで利用可能なメソッド一覧
メソッド                                                  |内容
-------------------------------------------------------|----
select(string $sql)<br>select(string $sql, array $bind)|SELECTを実行する。
insert(string $sql, array $bind)                       |INSERTを実行する。
update(string $sql, array $bind)                       |UPDATEを実行する。
delete(string $sql)<br>delete(string $sql, array $bind)|DELETEを実行する。
beginTransation(int $isolationLevel = 2)               |トランザクションを開始する。
commit()                                               |コミットする。
rollback()                                             |ロールバックする。
connect()                                              |DBに接続する。
disconnect()                                           |DBを切断する。

#### クエリファイルによるSQL実行
Modelクラスでは直接SQLをメソッド内に記述する以外に、クエリファイル(XML)を使ってSQLを実行できます。クエリファイルは`query/`に保存します。

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper PUBLIC
  "-//github.com/mapserver2007//DTD Mapper 3.0//EN" "http://localhost/webstream-model-mapper.dtd">
<mapper namespace="MyBlog">
  <select id="getData">
    SELECT
        *
    FROM
        T_Blog
    WHERE
        id = :id
  </select>
</mapper>
```
クエリファイルのDTDは同ディレクトリに配置し、DOCTYPEの値は適宜修正しDTDを指すようにします。  
mapperタグの`namespace`にModelクラスの名前空間を指定します。名前空間が一致すればModelクラスからクエリファイルを呼び出すことができます。  
mapperタグ配下にSQLを記述するタグを記述します。`<select>`、`<insert>`、`<update>`、`<delete>`タグが指定可能です。タグの`id`をModelクラスのメソッドからアクセスするとSQLを実行できます。

```php
namespace MyBlog;

use WebStream\Core\CoreModel;

/**
 * @Database(driver="WebStream\Database\Driver\Mysql", config="config/database.mysql.ini")
 */
class BlogModel extends CoreModel
{
    /**
     * @Query(file="query/myblog.xml")
     */
    public funciton getEntryData()
    {
        $bind = ["id" => 10];
        return $this->getData($bind);
    }
}
```
#### クエリファイルによるSQL実行結果をエンティティクラスにマッピングする
クエリファイルの`<select>`タグの`entity`属性にクラスパスを設定すると、取得結果をマッピングすることができます。  
Modelクラスでは直接SQLをメソッド内に記述する以外に、クエリファイル(XML)を使ってSQLを実行できます。クエリファイルは`query/`に保存します。

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper PUBLIC
  "-//github.com/mapserver2007//DTD Mapper 3.0//EN" "http://localhost/webstream-model-mapper.dtd">
<mapper namespace="MyBlog">
  <select id="getData" entity="\MyBlog\Entity\BlogEntity">
    SELECT
        title,
        description
    FROM
        T_Blog
  </select>
</mapper>
```

```php
namespace MyBlog\Entity;

class BlogEntity
{
    private $title;
    private $description;

    public funciton getTitle()
    {
        return $this->title;
    }

    public funciton getDescription()
    {
        return $this->description;
    }
}
```

テーブルのカラム名とエンティティクラスのフィールド名には対応関係があり、それにしたがってマッピング定義する必要があります。  
エンティティクラスのフィールド名はローワーキャメルケースで定義すると、型を含めてマッピングが実行されます。  
テーブルのカラム名がスネークケースである場合、自動的に名前をローワーキャメルケースに変換した上でマッピングされます。カラム名がエンティティクラスのフィールド名にマッピングできない命名になっている場合、SQLのAS句で別名を付けて対応してください。

#### 取得結果を配列にする
`select`で取得した結果を配列に変換します。

```php
$this->select($sql, $bind)->toArray();
```
```php
$this->getData($bind)->toArray();
```

#### 取得結果をエンティティにする
`select`で取得した結果をエンティティに変換します。

```php
$entityClasspath = "\MyBlog\Entity\BlogEntity";
$this->select($sql, $bind)->toEntity($entityClasspath);
```
```php
$entityClasspath = "\MyBlog\Entity\BlogEntity";
$this->getData($bind)->toEntity($entityClasspath);
```

[アノテーション](#annotaion)を使い、クエリファイルパスを指定します。これによりクエリファイルに記述したSQLが自動的に紐付けられます。

#### トランザクション処理
`$this->beginTransation()`でトランザクションを開始し`$this->commit()`でコミット、`$this->rollback()`でロールバックを実行します。  
ただし、DBMSがトランザクション処理に対応していない場合はトランザクション処理は有効になりません。  
なお、トランザクション処理を明示しない場合、処理が終了後、自動的にコミットを実行します。  
`$this->beginTransation()`はトランザクション分離レベルを引数として指定できます。  
指定可能なトランザクション分離レベルは`Doctrine\DBAL\Connection`に定義されている値となります。  

また、明示的にcommit,rollbackを記述しなくてもトランザクションスコープを定義できる`transactional()`を用意しています。  

```php
$manager = new DatabaseManager($container);
$manager->loadConnection($filepath);
$manager->connect();
$manager->transactional(function ($conn) {
    // 正常終了した場合、自動的にコミット
    // 例外が発生した場合、自動的にロールバック
    $conn->query('INSERT INTO T_WebStream (name) VALUES (:name)', ['name' => 'test'])->insert();
});
```

## [View](#view)
Viewは画面に出力するHTMLなどを描画し、Controllerクラスから呼ばれます。HTML等の描画はWebStream独自のテンプレート機能を利用します。  
ViewからはHelperまたはModel、Serviceを呼び出してビジネスロジックを実行することができます。

ViewテンプレートはWebStream独自のBasicテンプレートとTwigテンプレートが使えます。
Baicテンプレートファイルは`.tmpl`拡張子を、Twigテンプレートは`.twig`を付け、`app/views`にページ名をスネークケースに変換したフォルダを作成し保存します。  
`__cache`、`__public`、`__shared`フォルダを作成すると、それぞれテンプレートキャッシュファイル、静的ファイル、共通テンプレートファイルを使用することができます。  
ViewにはModel/Serviceオブジェクトが渡されるので、Model、Serviceで取得した値やビジネスロジックの実行がViewで可能になります。  
Model/Serviceオブジェクトは`$model`変数に格納されます。また、Helperオブジェクトは`$helper`変数に格納されます。

ContollerクラスからViewテンプレートを呼び出します。`@Template`の仕様は[アノテーション](#annotaion)を参照してください。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

/**
 * テンプレートを呼び出す。
 * @Template("index.tmpl")
 */
class BlogController extends CoreController
{
    public funciton execute()
    {
        $this->Blog->entry();
    }
}
```

`__shared`に保存した共通テンプレートを呼び出すことができます。  
共通点プレートはheaderやfooterなど共通になる部分を定義するときに使用します。  

テンプレートを部品化したい場合、部分テンプレートとして呼び出すことができます。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

/**
 * 基本テンプレートと共通テンプレートを呼び出す。
 * @Template("base.tmpl")
 */
class BlogController extends CoreController
{
    public funciton execute()
    {
        $this->Blog->entry();
    }
}
```

ViewテンプレートにはHTMLを記述しますが、Service/Modelの値などを埋め込むことができます。

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ja" lang="ja">
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <head>
        <title>%H{$model->getTitle()}</title>
    </head>
    <body>
        <div>%H{$model->getContent()}</div>
        %T{parts.tmpl}
    </body>
</html>
```

`$model`にアクセスするとServiceクラスまたはModelクラスにアクセスできます。また、`@Template`の`name`属性に指定した名前は変数としてアクセスできます。

使用するテンプレートエンジンを明示する場合は以下のようにします。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

/**
 * Basicテンプレートを使用する
 * @Template("index.tmpl", engine="basic")
 */
class BlogController extends CoreController
{
    public funciton execute()
    {
        $this->Blog->entry();
    }
}
```

```php
namespace MyBlog;

use WebStream\Core\CoreController;

/**
 * Twigテンプレートを使用する
 * @Template("index.tmpl", engine="twig")
 */
class BlogController extends CoreController
{
    public funciton execute()
    {
        $this->Blog->entry();
    }
}
```

Basicテンプレートにはテンプレートキャッシュ機能とCSRF対策トークン自動挿入機能がつきます。この機能により、CSRF対策が自動的に有効になります。  
Basicテンプレートを指定している場合に`<form>`タグが含まれる場合、CSRF対策トークンが自動挿入されます。  
(ただし、method="get"の場合は有効になりません。)

```html
<head>
<meta charset="utf-8">
<title>CSRF CHECK</title>
</head>
<body>
<form action="/form_register" method="post">
    <input type="button" value="submit">
    <input type="hidden" name="__CSRF_TOKEN__" value="a2891f68edeb487a9140edf8575a8e3382f96c0d">
</form>
</body>
```

フォームの送り先のControllerクラスで`@CsrfProtection`アノテーションを指定すると、CSRFチェック処理が実行されます。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class BlogController extends CoreController
{
    /**
     * @CsrfProtection
     */
    public funciton formRegister()
    {
        // CSRFエラーがあった場合、例外が発生し、ここには到達しない
    }
}
```

CSRF対策トークンの送信はPOSTで通常行いますが、HTTPヘッダ`X-CSRF-Token`にトークン文字列を指定して送信することも可能です。

テンプレートキャッシュ機能は、出力した内容をまるごとキャッシュする機能で、`cacheTime`属性で指定した時間(秒)だけキャッシュします。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class BlogController extends CoreController
{
    /**
     * テンプレートキャッシュを600秒有効にする例
     * @Template("index.tmpl", engine="basic", cacheTime=600)
     */
    public funciton execute()
    {
        $this->Blog->entry();
    }
}
```

Viewテンプレートでは以下の構文が使用可能です。

### [Viewテンプレート構文](#template_keyword)
構文     |説明
---------|----
%P{$hoge}|%P{}で囲ったPHPのコードを実行する。関数を実行する場合などに使用する。<br>ただし、関数の実行結果をreturnしても画面表示されない。また、関数内部で`echo`で標準出力した場合はエスケープされないので注意。<br>実行結果を伴わない関数の実行が必要な場合に使用する。
%H{$hoge}|%H{}で囲った変数を安全な値にエスケープしてHTMLとして表示する。関数を実行する場合も使用可能で、returnで返却された結果をエスケープして画面表示する。<br>ただし、関数内部で`echo`で標準出力した場合はエスケープされないので注意。
%J{$hoge}|%J{}で囲った変数を安全な値にエスケープしてJavaScriptコードとして評価する。関数を実行する場合も使用可能で、returnで返却された結果をエスケープして画面表示する。<br>ただし、関数内部で`echo`で標準出力した場合はエスケープされないので注意。
%X{$hoge}|%X{}で囲った変数を安全な値にエスケープしてXMLとして評価する。関数を実行する場合も使用可能で、returnで返却された結果をエスケープして画面表示する。<br>ただし、関数内部で`echo`で標準出力した場合はエスケープされないので注意。
%T{$template}|%T{}で囲ったテンプレートパスを読み込む。

## [Helper](#helper)
Viewの描画に関するロジックが必要な場合はHelperを呼び出します。  
Helperクラスは`app/helpers`に`WebStream\Core\CoreHelper`クラスを継承したクラスを定義します。

```php
namespace WebStream\Test\TestData\Sample\App\Helper;

use WebStream\Core\CoreHelper;
use WebStream\Core\CoreService;

class TestHelperHelper extends CoreHelper
{
    public function help1()
    {
         return $this->help2($model->getName());
    }
}
```

Helperクラス内ではViewテンプレート内と同様に`$model`オブジェクトからModelクラス、Serviceクラスを呼び出すことができます。  
Helperクラスのメソッドは`$helper`オブジェクトにより呼び出します。

```php
$helper->method();
```

メソッド呼び出しにより、Viewテンプレートで必要なロジックを実行します。  
メソッドの戻り値はViewテンプレートに描画されますが、<a href="#template_keyword">Viewテンプレート構文</a>によりエスケープして出力し、安全な値として出力する必要があります。

```php
%H{$helper->method()}
```

ただし、Helper内で直接echoで出力するとエスケープされないので注意してください。

## [命名規則まとめ](#naming_rule)
各クラスの命名規則、保存場所のまとめは以下のとおりです。

レイヤ      |サンプルクラス名    |保存場所
----------|----------------|------
Controller|SampleController|app/controllers/SampleController.php
Service   |SampleService   |app/services/SampleService.php
Model     |SampleModel     |app/models/SampleModel.php
View      |(任意の名前).tmpl|app/views/sample/(任意の名前).tmpl
Helper    |SampleHelper    |app/helpers/SampleHelper.php

上記レイヤに乗らない、開発者が独自に定義したいクラスを格納する階層を作成することができます。

レイヤ      |サンプルクラス名    |保存場所
----------|----------------|------
独自レイヤ  |(任意の名前)     |app/(任意の場所)/(任意の名前).php

エンティティマッピングで使用するエンティティクラスや外部APIを利用するためのクラスなどはここに定義します。

## [ルーティング定義](#routing)
### routes.yml
ルーティング設定により、URI設計を行うことができます。ルーティングにはmod_rewiteが必要です。  
ルーティング定義は`config/routes.yml`に記述します。

```yml
'/login': 'sample#login'
'/blog/:id': 'blog#entry'
```

キーにURIパス定義、バリューにクラス、アクション定義を記述します。誤った定義が記述された場合、例外が発生します。

### URIパス定義
URIパスは`/path/to`形式で定義します。またURIには変数の設定が可能で、`:value`形式で記述します。例えば、`/blog/:id`と定義し、`/blog/10`にアクセスした場合、Controllerクラスでは以下の方法で値を取得出来ます。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class BlogController extends CoreController
{
    public function execute(array $params)
    {
        $id = $params['id']; // 10
    }
}
```

## [バリデーション定義](#validate)
Controllerクラスのアクションメソッドに`@Validate`アノテーションを記述することでバリデーションを有効にできます。

```php
namespace MyBlog;

use WebStream\Core\CoreController;
use WebSteram\Annotation\Validate;

class BlogController extends CoreController
{
    /**
     * GETリクエストのtestパラメータの指定がない場合、エラーになる例
     * @Validate(key="test", rule="required", method="get")
     */
    public function execute()
    {
    }
}
```

バリデーションルールは以下のものが用意されています。

####バリデーションチェックルール

ルール        |内容
-------------|---------
required     |必須チェック
number       |数値チェック(整数)
min[n]       |最小値チェック(整数)
max[n]       |最大値チェック(整数)
min_length[n]|最小文字数チェック(整数)
max_length[n]|最大文字数チェック(整数)
equal        |文字列一致チェック
length       |文字数一致チェック
range[n..m]  |範囲チェック(整数)
regexp[//]   |正規表現チェック

### カスタムバリデーション
用意されているバリデーションルール以外に、開発者が独自にルールを定義することが出来ます。  
`app`ディレクトリ配下の任意の場所にバリデーションクラスを作成します。  

カスタムバリデーションクラスを定義するには以下の満たす必要があります。  
* 名前空間は`WebStream\Annotation\Attributes\Ext\ValidateRule`にする
* `WebStream\Annotation\Attributes\Ext\ValidateRule\IValidate`を実装し、`isValid`メソッドを定義する
* クラス名とルール名が対応づいている
    * クラス名が`CustomValidate`の場合、ルール名は`custom_validate`

```php
namespace WebStream\Annotation\Attributes\Ext\ValidateRule;

use WebStream\Annotation\Attributes\Ext\ValidateRule\IValidate;

class Page implements IValidate
{
    public function isValid($value, $rule)
    {
        return $value === null || (bool) preg_match('/^[1-9]{1,}[0-9]{0,}$/', $value);
    }
}
```

`WebStream\Annotation\Attributes\Ext\ValidateRule\IValidate`インタフェースを実装し、戻り値が`bool`型の`isValid`メソッドを実装します。  
バリデーションが成功すればtrue、失敗すればfalseを返すようにします。  
クラス名がルール名と紐付いているので、`@Validate`アノテーションに指定します。

```php
namespace MyBlog;

use WebStream\Core\CoreController;
use WebSteram\Annotation\Validate;

class BlogController extends CoreController
{
    /**
     * @Validate(key="p", rule="page", method="get")
     */
    public function execute()
    {
    }
}
```

## [リクエストパラメータ](#request)
GET/POST/PUT/DELETEで送信した値をControllerで取得できます。  
`$this->request`オブジェクトからリクエストパラメータを取得でき、`get`,`post`,`put`,`delete`メソッドにそれぞれアクセスします。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class BlogController extends CoreController
{
    public function execute()
    {
        $getParams = $this->request->get; // GETパラメータすべて取得
        $getParam  = $this->request->get["name"];
    }
}
```

## [セッション](#session)
ログイン処理などを実装するときに、セッション管理を使用しますが、WebStreamでは`$this->session`オブジェクトを使用します。セッション期限を指定するには`restart`メソッドを使用します。

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class LoginController extends CoreController
{
    public function execute()
    {
        $expire = 6000; // 10分
        $path = "/login";
        $domain = ".mydomain.com";
        $getParams = $this->session->restart($expire, $path, $domain);
    }
}
```

セッションがタイムアウトした場合、`SessionTimeoutException`が発生します。

## [キャッシュ](#cache)
キャッシュ機能を使うことができます。使用できるキャッシュライブラリは以下のとおりです。

| クラス        | 出力先                       | 設定方法                                           |
|---------------|------------------------------|----------------------------------------------------|
| Apcu          | APCuキャッシュを使用する     | APCuをインストールする                             |
| Memcached     | memcachedを使用する          | memcachedをインストールする config/cache.ymlに設定 |
| Redis         | redisを使用する              | redisをインストールする config/cache.ymlに設定     |
| TemporaryFile | ファイルキャッシュを使用する | 利用時に出力先を指定する                           |

キャッシュは`CacheDriverFactory`クラスを使うことで簡単に使用できます。  

```php
use WebStream\Module\Container;
use WebStream\Cache\Driver\CacheDriverFactory;

$factory = new CacheDriverFactory();
$config = new Container(false);
$config->cachePrefix = "test_cache";
$cache = $factory->create("WebStream\Cache\Driver\Apcu", $config);
$cache->inject('logger', $this->logger);

$cache->add("key", "value");
$value = $cache->get("key");
```

さらに簡単に使うためのUtilityが用意されているので、それを使うとより簡潔に書くことができます。  

```php
use WebStream\Module\Utility\CacheUtils;

$cache = $this->getCacheDriver("apcu", "test_cache");
$cache->inject('logger', $this->logger);

$cache->add("key", "value");
$value = $cache->get("key");
```

キャッシュライブラリとして使う場合の接続先情報は`config/cache.yml`に設定を書く必要があります。  
`cache.yml`のサンプルは以下のとおりです。

```yml
---
  memcached:
    host: 192.168.1.100
    port: 11211
  redis:
    host: 192.168.1.100
    port: 6379
  temporaryfile:
    path: "/tmp"
```

## [ロギング](#logging)
### ログ設定
`config/log.ini`または`config/log.yml`を配置し、ログ設定を記述します。  
ログ格納ディレクトリには書き込み権限を与えておく必要があります。  

```
; プロジェクトルートからの相対パスまたは絶対パス
path = log/webstream.log
; ログレベル
level = debug
; ログローテート設定(day|month|year)
rotate_cycle = day
; ログローテート設定(サイズ:Byte)
rotate_size = 1024
; ログに表示するアプリケーション名
applicationName = webstream
; ログフォーマット設定
; 設定しない場合はデフォルト設定になる
; %c ログに表示するアプリケーション名
: %d 日付(%Y-%m-%d %H:%M)
; %d{{(指定フォーマット)}} 日付 例：%d{{%Y-%m-%d %H:%M.%f}}
; %l ログレベル(小文字)
; %L ログレベル(大文字)
; 表示幅指定 例：%5L → [INFO ]、[DEBUG]など
;   表示幅指定は%c、%l、%L、%dに適用可
; %m ログメッセージ
format = [%c][%d{%Y-%m-%d %H:%M:%S.%f}][%5L] %m
```

Controller、Service、Model、Helperクラスでロガーを使用できます。  

```php
namespace MyBlog;

use WebStream\Core\CoreController;

class LoginController extends CoreController
{
    public function index()
    {
        $this->logger->debug("logging in controller");
    }
}
```

ログレベルについては、[PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md)に準拠しています。

### ロガーのカスタマイズ
デフォルトではロガーはログファイル出力しますが、Outputterを切り替えることで出力先を変更することができます。  
ロガーインスタンスを新たに生成し、Outputterを差し替えます。

```
$instance = Logger::getInstance();
$instance->setOutputter([
   new BrowserOutputter()
]);
$customLogger = new LoggerAdapter($instance);
```

WebStream標準では以下のOutputterが用意してあります。独自に定義すれば出力先を自由に変えることができます。  

| Outputter種類    | 出力先                     |
|------------------|----------------------------|
| FileOutputter    | 指定したログファイルに出力 |
| BrowserOutputter | ブラウザに出力             |
| ConsoleOutputter | コンソールに出力           |

### ログの遅延書き込み機能
キャッシュ機能が有効になっている場合、ログは一旦キャッシュに保存され、すべての処理終了後にまとめてログファイルに出力されます。  
キャッシュは`APCu`、`Memcached`、`Redis`のいずれかが使用可能です(デフォルトはAPCu)。  
これらのキャッシュライブラリが使用不可の場合はキャッシュせず即時ログ出力を行いますが、パフォーマンス上あらかじめキャッシュライブラリをインストールしておくことを推奨します。

## [ファイル入出力](#io)
PHP標準の`file`、`file_get_contents`、`file_put_contents`などを使うとファイル入出力ができますが、WebStreamでは安全にファイル入出力を行うためのクラスが用意してあります。  

| クラス           | 内容                                                                                                       |
|------------------|------------------------------------------------------------------------------------------------------------|
| FileReader       | ファイル読み込みを行う<br>ファイルはロックしない                                                           |
| FileWriter       | ファイル書き込みを行う<br>書き込み時に排他ロックを取得する<br>ファイルを閉じるまでロックし続ける           |
| SimpleFileWriter | ファイル書き込みを行う<br>書き込み時に排他ロックを取得する<br>書き込む度にファイルを閉じてロックを解放する |

これらのクラスはファイルの読み込みや書き込みになんらかの問題が発生した場合、`IOException`をスローするので、適切に例外処理することができます。  
さらに以下のクラスを使うことで、ファイルオブジェクトを取得することができます。それにより、さらに高度なファイル処理が可能になります。  

| クラス             | 内容                                               |
|--------------------|----------------------------------------------------|
| InputStreamReader  | ファイル読み込みを行う<br>入力ストリームを操作可能 |
| OutputstreamWriter | ファイル書き込みを行う<br>出力ストリームを操作可能 |

## [アノテーション](#annotation)
ControllerとModelではアノテーションを使ってクラスやメソッドを操作することができます。アノテーションを利用することで便利な処理が可能になります。  

アノテーションを適用するクラスは`IAnnotatable`を実装する必要があります。  
例えば`app/libraries/`に独自で定義したクラスには開発者が`IAnnotatable`を実装しないと、アノテーションが有効になりません。  
なお、Controller、Service、Model、Helperの各クラスはデフォルトで実装されているため改めて開発者が実装する必要はありません。

#### すべてのレイヤで使用可能なアノテーション
アノテーション      |説明                                         |サンプル
-----------------|---------------------------------------------|------
@Alias           |指定されたメソッド名で受けてアノテーションが定義されたメソッドへ転送する |@Alias(name="aliasMethod")
@Filter          |メソッドが呼ばれる前または後に任意の処理を実行する    |@Filter(type="before")<br>@Filter(type="after")<br>@Filter(type="before" except="method1")<br>@Filter(type="before" only="method2")<br>@Filter(type="before",only="method1",except="method2")<br>@Filter(type="after",except={"method1","method2"})

#### Controllerで使用可能なアノテーション
アノテーション     |説明                                         |サンプル
-----------------|---------------------------------------------|------
@Header          |リクエスト/レスポンスを制御する                       |@Header(contentType="html")<br>@Header(contentType="xml")<br>@Header(allowMethod="POST")<br>@Header(allowMethod={"GET","POST"})
@Template        |Viewテンプレートを設定する                         |@Template("index.tmpl")<br>@Template("index.tmpl",engine="basic")<br>@Template("index.tmpl",engine="twig",cacheTime="1000")
@ExceptionHandler|例外を補足して別処理を実行する                     |@ExceptionHandler("\Exception")<br>@ExceptionHandler({"\RuntimeException","\LogicException"})
@CsrfProtection  |CSRF対策処理を実行する                          |@CsrfProtection

#### Modelで使用可能なアノテーション
アノテーション |説明                                 |サンプル
-------------|------------------------------------|---------------------------------------------------------------------------------------
@Database    |Modelクラスに対してデータベース設定をする|@Database(driver="WebStream\Database\Driver\Mysql", config="config/database.mysql.ini")
@Query       |読み込むクエリファイルを指定する         |@Query(file="query/blog_query.xml")

### カスタムアノテーション
用意されているアノテーション(デフォルトアノテーション)以外に独自のカスタムアノテーションを定義することができます。

#### 定義方法
`app`ディレクトリ以下の任意の場所にクラスを定義します。定義したクラスは自動的にクラスパスが通ります。  
クラスの定義は以下のルールに従って定義してください。

No |ルール                                      |内容                                                       |必須かどうか
---|-------------------------------------------|-----------------------------------------------------------|-------------------
1  |`@Annotation`、`@Target`をクラスにアノテート  |Doctrine Annotationを使用する                               |必須
2  |`\WebStream\Annotation\Annotation`を継承    |アノテーション処理実行可能なクラスにする                        |必須
3  |`Annotation#onInject`を実装する             |アノテーション初期処理を実行                                   |必須
4  |`IClass`を実装する                          |クラスに対するアノテーションを実行する                          |任意
5  |`IClass#onClassInject`を実装する            |クラスに対するアノテーションを実行したときに呼ばれる              |IClass実装時は必須
6  |`IMethod`を実装する                         |実行するメソッド(アクションメソッド)に対するアノテーションを実行する|任意
7  |`IMethod#onMethodInject`を実装する          |メソッド(アクションメソッド)に対するアノテーションを実行する       |IMethod実装時は必須
8  |`IMethods`を実装する                        |すべてのメソッドに対するアノテーションを実行する                  |任意
9  |`IMethods#onMethodInject`を実装する         |すべてのメソッドに対するアノテーションを実行する                  |IMethods実装時は必須
10 |`IProperty`を実装する                       |すべてのプロパティに対するアノテーションを実行する                |任意
11 |`IProperty#onPropertyInject`を実装する      |すべてのプロパティに対するアノテーションを実行する                |IProperty実装時は必須
12 |`IRead`を実装する                           |アノテーション処理実行後、任意のデータを返却する処理を実行         |任意
13 |`IRead#onInjected`を実装する                |任意のデータを返却する                                        |IRead実装時は必須

デフォルトアノテーションは使用できるレイヤが制限されていますが、カスタムアノテーションはController/Service/Modelレイヤで使用可能です。