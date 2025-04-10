# バックエンド ベースコントローラー

## 概要

このプロジェクトのバックエンドAPIは、すべてのAPIエンドポイントに共通の機能を提供する階層的なコントローラー構造に基づいて構築されています。ベースコントローラーシステムは、特殊な機能のための様々なトレイトを持つ階層的な継承構造で構成されています。

## コントローラー階層

```
Laravel BaseController
     ↑
StubController (ベースコントローラー)
     ↑
ApiController
     ↑
特定のAPIコントローラー
```

## 全体処理フロー図

以下の図はベースコントローラーを使用した全体的なリクエスト処理フローを示しています：

```mermaid
flowchart TD
    A[クライアントリクエスト] --> B[ルーター]
    B --> C[ミドルウェア処理]
    C --> D[コントローラーメソッド実行]
    
    D --> E{リクエスト検証}
    E -->|失敗| F[例外発生]
    F --> G[例外ハンドラー]
    G --> H[エラーレスポンス生成]
    
    E -->|成功| I[ページネーションパラメータ処理]
    I --> J[データアクセス層]
    J --> K[ビジネスロジック実行]
    
    K -->|例外発生| R[try-catchブロック]
    R --> S{例外タイプ判別}
    S -->|ErrUsr| T[ユーザーエラー処理]
    S -->|ErrPrg| U[システムエラー処理]
    S -->|WarnUsr| V[警告処理]
    
    T --> H
    U --> H
    V --> H
    
    K --> L{処理結果}
    L -->|成功| M[buildOK レスポンス]
    L -->|警告| N[buildWarn レスポンス]
    L -->|エラー| O[buildNG レスポンス]
    
    M --> P[JSONレスポンス変換]
    N --> P
    O --> P
    H --> P
    
    P --> Q[クライアントへ応答]
    
    subgraph "LimitOffsetAware"
    I
    end
    
    subgraph "Assertions"
    E
    F
    end
    
    subgraph "ExceptionHandler"
    R
    S
    T
    U
    V
    end
    
    subgraph "JsonResponseFactory"
    M
    N
    O
    H
    P
    end
```

この図は、リクエストがルーターを通過し、ミドルウェア処理を経て、StubControllerから継承された機能を利用してAPIリクエストを処理する全体的な流れを示しています。各ステップでは、ベースコントローラーに組み込まれたトレイトの機能が活用されます。

## 主要コンポーネント

### 1. StubController

`app/Contracts/Http/Controllers/StubController.php`に位置し、すべてのAPIコントローラーが継承する主要なベースコントローラーです。LaravelのBaseControllerを拡張し、複数の有用なトレイトを組み込んでいます。

```php
namespace W3\Contracts\Http\Controllers;

use Carbon\Carbon;
use Illuminate\Foundation\Bus\DispatchesJobs;
use Illuminate\Foundation\Validation\ValidatesRequests;
use Illuminate\Routing\Controller as BaseController;
use W3\Helpers\Assertions;
use W3\Helpers\JsonResponseFactory;
use W3\Helpers\ExceptionHandler;
use W3\Models\Element;
use W3\Models\Unit;
use W3\Models\User;

class StubController extends BaseController
{
    use Assertions, DispatchesJobs, ValidatesRequests, LimitOffsetAware, JsonResponseFactory, ExceptionHandler;

    protected $selectLimitMax = 10000;
}
```

### 2. ApiController

`app/Http/Controllers/API/ApiController.php`に位置し、StubControllerを拡張し、すべてのAPI固有のコントローラーのベースとなるレイヤーです。

```php
namespace W3\Http\Controllers\API;

use W3\Contracts\Http\Controllers\StubController;

class ApiController extends StubController
{
    public function __construct()
    {
    }
}
```

## 主要トレイト

### 1. JsonResponseFactory

`app/Helpers/JsonResponseFactory.php`に位置し、このトレイトはAPI応答を構築するための標準化されたメソッドを提供します：

- **buildOK($data, $count)** - 成功応答を作成
- **buildNG($errorCode, $messages, $data, $statusCode)** - エラー応答を作成
- **buildWarn($errorCode, $messages, $data, $statusCode)** - 警告応答を作成
- **buildNGValidation($messages)** - バリデーションエラー応答を作成

応答フォーマット：
```json
// 標準フォーマット
{
  "status": "OK|NG|WARN",
  "errorCode": 0,
  "messages": [],
  "data": {} // 応答データ
}

// 簡易フォーマット（r_format=briefがリクエストされた場合）
{} // データオブジェクト直接
```

### 2. LimitOffsetAware

`app/Contracts/Http/Controllers/LimitOffsetAware.php`に位置し、このトレイトはページネーション機能を処理します：

- **limit()** - リクエスト制限パラメータを取得（デフォルト：30000）
- **offset()** - リクエストオフセットパラメータを取得（デフォルト：0）
- **htLimit()/htOffset()** - HT固有の制限（異なるデフォルト値）
- **appendWhereBetween()** - クエリに日付範囲フィルタリングを追加
- **fromDate()/toDate()** - 日付範囲変換を処理

### 3. Assertions

`app/Helpers/Assertions.php`に位置し、このトレイトはバリデーションとエラー処理のためのメソッドを提供します：

- **log($level, $code, $messages)** - 異なる重大度レベルでのロギング
- **RU()（ユーザー用要件）** - ユーザー入力を検証し、ErrUsr例外をスロー
- **RP()（プログラマー用要件）** - コード前提条件を検証し、ErrPrg例外をスロー
- **WU()（ユーザー用警告）** - 警告を作成し、WarnUsr例外をスロー
- **isUserException/isProgrammerException/etc.** - 例外タイプの確認

### 4. ExceptionHandler

`app/Helpers/ExceptionHandler.php`に位置し、このトレイトは例外処理を一元的に管理するためのメソッドを提供します：

- **handleException($e, $data = null)** - あらゆる種類の例外を処理し、適切なレスポンスを返す

## エラー処理

コントローラー階層は、カスタム例外タイプを使用します：
- **ErrUsr** - ユーザー向けエラー
- **ErrPrg** - プログラミング/内部エラー
- **WarnUsr** - 警告レベルの問題

## 使用例

### 基本的なコントローラーの実装例

```php
namespace W3\Http\Controllers\API;

class ExampleController extends ApiController
{
    public function index()
    {
        try {
            // LimitOffsetAwareによるページネーションサポート
            $limit = $this->limit();
            $offset = $this->offset();
            
            // データ取得
            $data = SomeModel::skip($offset)->take($limit)->get();
            
            // Assertionsを使用した入力検証
            $this->RU(request()->has('required_field'), 'E001', '必須フィールドが不足しています');
            
            // ビジネスロジックの実行
            $result = $this->processData($data);
            
            // JsonResponseFactoryによる標準化された応答
            return $this->buildOK($result, $result->count());
        } catch (\Exception $e) {
            // ExceptionHandlerを使用した例外処理
            return $this->handleException($e);
        }
    }
    
    private function processData($data)
    {
        // ビジネスロジックの実装
        return $data;
    }
}
```

### エラー処理の高度な使用例

```php
try {
    // ユーザー入力の検証
    $this->RU(request()->has('user_id'), 'E001', 'ユーザーIDが必要です');
    
    // ビジネスルールの検証
    $this->RU($user->hasPermission('edit'), 'E002', '編集権限がありません');
    
    // プログラマー向け前提条件
    $this->RP(isset($config['api_key']), 'P001', 'API設定が不足しています');
    
    // 軽度の警告
    $this->WU($data->count() < 1000, 'W001', 'データ量が大きすぎます。パフォーマンスが低下する可能性があります');
    
    // 処理の実行
    $result = $this->processData($data);
    
    // 成功レスポンスを返す
    return $this->buildOK($result);
} catch (\Exception $e) {
    // 例外処理トレイトを使用して一元的に処理
    return $this->handleException($e);
}
```

## ビジネスロジックからのエラー処理メカニズム

ExceptionHandlerトレイトを使用することで、ビジネスロジック実行中に発生する例外を簡潔かつ一貫した方法で処理できます：

1. **例外処理アプローチ**:
   - APIコントローラーのアクションメソッド内で、すべてのビジネスロジックをtry-catchブロックで囲む
   - catchブロック内で`handleException`メソッドを呼び出す

2. **ExceptionHandlerトレイトの利点**:
   - 例外処理ロジックの集中管理
   - コードの重複を防止
   - 一貫した例外処理パターンの適用
   - コントローラーアクションがシンプルで読みやすくなる

3. **実装方法**:
   - `app/Helpers/ExceptionHandler.php`トレイトを作成
   - StubControllerにトレイトを追加
   - 各コントローラーアクションで以下のパターンを使用

   ```php
   public function someAction()
   {
       try {
           // ビジネスロジックの実行
           return $this->buildOK($result);
       } catch (\Exception $e) {
           // 一元化された例外処理
           return $this->handleException($e);
       }
   }
   ```

この方法により、例外処理がシンプルかつ標準化され、コントローラーアクションがビジネスロジックに集中できるようになります。

## エラーコード体系

エラーコードは以下の規則に従って割り当てられます：

- **Exxx**: ユーザー関連エラー
  - E001-E099: 入力検証エラー
  - E100-E199: 権限エラー
  - E200-E299: ビジネスルールエラー

- **Pxxx**: プログラミング/システムエラー
  - P001-P099: 構成エラー
  - P100-P199: 内部不整合エラー
  - P200-P299: 外部サービスエラー

- **Wxxx**: 警告
  - W001-W099: パフォーマンス警告
  - W100-W199: ビジネスロジック警告 
