# 2. 環境構築（v3版）

## 2.1 使用言語

JavaScript を使用します。  

### SDK
- **symbol-sdk v3 系**  
  https://github.com/symbol/symbol-sdk  

アルファ版としてリリースされており、rxjs 依存の多くの機能が削除されています。  
そのため、**REST API への直接アクセス**が推奨されます。  

### リファレンス
- [Symbol SDK for TypeScript and JavaScript](https://symbol.github.io/symbol-sdk-typescript-javascript/1.0.3/)  
- [Catapult REST Endpoints (1.0.3)](https://symbol.github.io/symbol-openapi/v1.0.3/)  

---

## 2.2 サンプルソースコード

### 変数宣言
検証しやすいように `const` は使用しません。  
実際のアプリ開発ではセキュリティ確保のため `const` を利用してください。  

### 出力値確認
変数の内容確認には `console.log()` を使用します。  

### 同期・非同期
特に問題が無い限り同期処理で解説します。  

### アカウント
- **Alice**：メインのアカウント  
- **Bob**：送受信用のアカウント（その他、必要に応じて Carol なども利用）  

### 手数料
手数料乗数は **100** でトランザクションを作成します。  

---

## 2.3 事前準備

- **テストネット**  
  https://symbolnodes.org/nodes_testnet/  

- **メインネット**  
  https://symbolnodes.org/nodes/  

Chrome などでノードページを開き、F12 キーで開発者コンソールを開いて以下を入力します。  

### SDK 読み込み（v3）

```js
const SDK_VERSION = "3.2.3";
const sdk = await import(`https://www.unpkg.com/symbol-sdk@${SDK_VERSION}/dist/bundle.web.js`);
const sdkCore = sdk.core;
const sdkSymbol = sdk.symbol;

// Buffer を読み込む
(script = document.createElement('script')).src = 'https://bundle.run/buffer@6.0.3';
document.getElementsByTagName('head')[0].appendChild(script);

const NODE = window.origin; // 現在開いているページのURL
const Buffer = buffer.Buffer;
```

続いて、ほぼすべての章で利用する共通ロジック部分を実行しておきます。

```js
// ネットワーク情報
fetch(new URL('/node/info', NODE), { method: 'GET', headers: { 'Content-Type': 'application/json' } })
  .then((res) => res.json())
  .then((json) => {
    networkType = json.networkIdentifier;
    generationHash = json.networkGenerationHashSeed;
  });

// プロパティ情報
fetch(new URL('/network/properties', NODE), { method: 'GET', headers: { 'Content-Type': 'application/json' } })
  .then((res) => res.json())
  .then((json) => {
    e = json.network.epochAdjustment;
    epochAdjustment = Number(e.substring(0, e.length - 1));
    identifier = json.network.identifier;            // v3 only
    facade = new sdkSymbol.SymbolFacade(identifier); // v3 only
  });

// トランザクション確認用
function clog(signedTx){
  hash = facade.hashTransaction(signedTx).toString();
  console.log(NODE + "/transactionStatus/" + hash);
  console.log(NODE + "/transactions/confirmed/" + hash);
  console.log("https://symbol.fyi/transactions/" + hash);
  console.log("https://testnet.symbol.fyi/transactions/" + hash);
}
```

これで準備完了です。

