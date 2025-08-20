# 11.制限

アカウントに対する制限とモザイクのグローバル制限についての方法を紹介します。
本章では、既存アカウントの権限を制限してしまうので、使い捨てのアカウントを新規に作成してお試しください。

```js
// 使い捨てアカウントCarolの生成
carol = facade.createAccount(sdkCore.PrivateKey.random());
console.log(carol.address.toString());

// FAUCET URL出力
console.log("https://testnet.symbol.tools/?recipient=" + carol.address.toString() +"&amount=100");
```

## 11.1 アカウント制限

### 指定アドレスからの受信制限・指定アドレスへの送信制限

```js
bob = facade.createAccount(sdkCore.PrivateKey.random());

// 制限設定
f = sdkSymbol.models.AccountRestrictionFlags.ADDRESS.value; // アドレス制限
f += sdkSymbol.models.AccountRestrictionFlags.BLOCK.value; // ブロック
flags = new sdkSymbol.models.AccountRestrictionFlags(f);

// アドレス制限設定Tx作成
descriptor = new sdkSymbol.descriptors.AccountAddressRestrictionTransactionV1Descriptor(  // Txタイプ:アドレス制限設定Tx
  flags,  // アドレス制限フラグ
  [       // 設定アドレス
    bob.address,
  ],
  []      // 解除アドレス
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  carol.publicKey,  // 署名者公開鍵
  100,                // 手数料乗数
  60 * 60 * 2         // Deadline:有効期限(秒単位)
);

// 署名
sig = carol.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);

// アドレス制限設定Txをアナウンス
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

`restrictionFlags` は v2 の `AddressRestrictionFlag` に相当します。
`AddressRestrictionFlag` との対応は以下の通りです。

- AllowIncomingAddress：指定アドレスからのみ受信許可
  - sdkSymbol.models.AccountRestrictionFlags.ADDRESS
- AllowOutgoingAddress：指定アドレス宛のみ送信許可
  - sdkSymbol.models.AccountRestrictionFlags.ADDRESS + sdkSymbol.models.AccountRestrictionFlags.OUTGOING
- BlockIncomingAddress：指定アドレスからの受信受拒否
  - sdkSymbol.models.AccountRestrictionFlags.ADDRESS + sdkSymbol.models.AccountRestrictionFlags.BLOCK
- BlockOutgoingAddress：指定アドレス宛への送信禁止
  - sdkSymbol.models.AccountRestrictionFlags.ADDRESS + sdkSymbol.models.AccountRestrictionFlags.BLOCK + sdkSymbol.models.AccountRestrictionFlags.OUTGOING

### 指定モザイクの受信制限

```js
// 制限設定
f = sdkSymbol.models.AccountRestrictionFlags.MOSAIC_ID.value; // モザイク制限
f += sdkSymbol.models.AccountRestrictionFlags.BLOCK.value;    // ブロック
flags = new sdkSymbol.models.AccountRestrictionFlags(f);

// モザイク制限設定Tx作成
descriptor = new sdkSymbol.descriptors.AccountMosaicRestrictionTransactionV1Descriptor(  // Txタイプ:モザイク制限設定Tx
  flags,  // モザイク制限フラグ
  [       // 設定モザイク
    new sdkSymbol.models.UnresolvedMosaicId(0x72C0212E67A08BCEn),
  ],
  []      // 解除モザイク
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  carol.publicKey,  // 署名者公開鍵
  100,                // 手数料乗数
  60 * 60 * 2         // Deadline:有効期限(秒単位)
);

// 署名
sig = carol.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);

// モザイク制限設定Txをアナウンス
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

アカウント制限と同様、 `restrictionFlags` は v2 の `MosaicRestrictionFlag` に相当します。
`MosaicRestrictionFlag` との対応は以下の通りです。

- AllowMosaic：指定モザイクを含むトランザクションのみ受信許可
  - sdkSymbol.models.AccountRestrictionFlags.MOSAIC_ID
- BlockMosaic：指定モザイクを含むトランザクションを受信拒否
  - sdkSymbol.models.AccountRestrictionFlags.MOSAIC_ID + sdkSymbol.models.AccountRestrictionFlags.BLOCK

モザイク送信の制限機能はありません。
また、後述するモザイクのふるまいを制限するグローバルモザイク制限と混同しないようにご注意ください。

### 指定トランザクションの送信制限

```js
// 制限設定
f = sdkSymbol.models.AccountRestrictionFlags.TRANSACTION_TYPE.value;  // トランザクション制限
f += sdkSymbol.models.AccountRestrictionFlags.OUTGOING.value;         // 送信
flags = new sdkSymbol.models.AccountRestrictionFlags(f);

// トランザクション制限設定Tx作成
descriptor = new sdkSymbol.descriptors.AccountOperationRestrictionTransactionV1Descriptor(  // Txタイプ:トランザクション制限設定Tx
  flags,  // トランザクション制限フラグ
  [       // 設定トランザクション
    sdkSymbol.models.TransactionType.ACCOUNT_OPERATION_RESTRICTION.value,
  ],
  []      // 解除トランザクション
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  carol.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名
sig = carol.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);

// トランザクション制限設定Txをアナウンス
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

アカウント制限やモザイク制限と同様、 `restrictionFlags` は v2 の `OperationRestrictionFlag` に相当します。
`OperationRestrictionFlag` との対応は以下の通りです。

- AllowOutgoingTransactionType：指定トランザクションの送信のみ許可
  - sdkSymbol.models.AccountRestrictionFlags.TRANSACTION_TYPE + sdkSymbol.models.AccountRestrictionFlags.OUTGOING
- BlockOutgoingTransactionType：指定トランザクションの送信を禁止
  - sdkSymbol.models.AccountRestrictionFlags.TRANSACTION_TYPE + sdkSymbol.models.AccountRestrictionFlags.OUTGOING + sdkSymbol.models.AccountRestrictionFlags.BLOCK

TransactionTypeについては以下の通りです。
```js
{16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS'}
```

トランザクション受信の制限機能はありません。

##### 注意事項
17232: 'ACCOUNT_OPERATION_RESTRICTION' の制限は許可されていません。
つまり、AllowOutgoingTransactionTypeを指定する場合は、ACCOUNT_OPERATION_RESTRICTIONを必ず含める必要があり、
BlockOutgoingTransactionTypeを指定する場合は、ACCOUNT_OPERATION_RESTRICTIONを含めることはできません。

### 確認

設定した制限情報を確認します

```js
res = await fetch(
  new URL('/restrictions/account/' + carol.address.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.accountRestrictions;
});
console.log(res);
```

###### 出力例

```js
> {version: 1, address: '98A3670EF79748312687154D5B92AFED6B7C5E5A7560A423', restrictions: Array(3)}
    address: "98A3670EF79748312687154D5B92AFED6B7C5E5A7560A423"
  > restrictions: Array(3)
    > 0:
        restrictionFlags: 32769
      > values: Array(1)
          0: "98676BDA0CC03FFA5897BA9706005D119F0C240F71F587B9"
          length: 1
    > 1: 
        restrictionFlags: 32770
      > values: Array(1)
          0: "72C0212E67A08BCE"
          length: 1
    > 2: 
        restrictionFlags: 16388
      > values: Array(1)
          0: 17232
          length: 1
      length: 3
    version: 1
```

## 11.2 グローバルモザイク制限

グローバルモザイク制限はモザイクに対して送信可能な条件を設定します。  
その後、各アカウントに対してグローバルモザイク制限専用の数値メタデータを付与します。  
送信アカウント・受信アカウントの両方が条件を満たした場合のみ、該当モザイクを送信することができます。  

最初に必要ライブラリの設定を行います。
併せて、前項の操作で制限がかかってしまっているため、必要に応じて新しい使い捨てのアカウントを作成してください。

```js
// 必要ライブラリの設定
sha3_256 = (await import('https://cdn.skypack.dev/@noble/hashes/sha3')).sha3_256;
```

### グローバル制限機能つきモザイクの作成
restrictableをtrueにしてCarolでモザイクを作成します。

```js
// モザイクフラグ設定
f = sdkSymbol.models.MosaicFlags.NONE.value;
f += sdkSymbol.models.MosaicFlags.SUPPLY_MUTABLE.value; // 供給量変更の可否
f += sdkSymbol.models.MosaicFlags.TRANSFERABLE.value;   // 第三者への譲渡可否
f += sdkSymbol.models.MosaicFlags.RESTRICTABLE.value;   // グローバル制限設定の可否
f += sdkSymbol.models.MosaicFlags.REVOKABLE.value;      // 発行者からの還収可否
flags = new sdkSymbol.models.MosaicFlags(f);

// ナンス設定
array = new Uint8Array(sdkSymbol.models.MosaicNonce.SIZE);
crypto.getRandomValues(array);
nonce = sdkSymbol.models.MosaicNonce.deserialize(array);

// モザイク定義
mosaicDefDescriptor = new sdkSymbol.descriptors.MosaicDefinitionTransactionV1Descriptor(  // Txタイプ:モザイク定義Tx
  new sdkSymbol.models.MosaicId(sdkSymbol.generateMosaicId(carol.address, nonce)),
  new sdkSymbol.models.BlockDuration(0n), // duration:有効期限
  nonce,  // ナンス
  flags,  // モザイクフラグ
  0       // divisibility:可分性
);
mosaicDefTx = facade.createEmbeddedTransactionFromTypedDescriptor(
  mosaicDefDescriptor,  // トランザクション Descriptor 設定
  carol.publicKey,      // 署名者公開鍵
);

// モザイク変更
mosaicChangeDescriptor = new sdkSymbol.descriptors.MosaicSupplyChangeTransactionV1Descriptor( // Txタイプ:モザイク変更Tx
  new sdkSymbol.models.UnresolvedMosaicId(mosaicDefTx.id.value),
  new sdkSymbol.models.Amount(1000000n),              // 数量
  sdkSymbol.models.MosaicSupplyChangeAction.INCREASE  // アクション
);
mosaicChangeTx = facade.createEmbeddedTransactionFromTypedDescriptor(
  mosaicChangeDescriptor, // トランザクション Descriptor 設定
  carol.publicKey,        // 署名者公開鍵
);

// キー生成(v2 準拠)
// メタデータキー生成と同じロジックのため、metadataGenerateKey()を利用する
key = sdkSymbol.metadataGenerateKey("KYC"); // restrictionKey 

// グローバルモザイク制限
mosaicGlobalResDescriptor = new sdkSymbol.descriptors.MosaicGlobalRestrictionTransactionV1Descriptor( // Txタイプ:グローバルモザイク制限Tx
  new sdkSymbol.models.UnresolvedMosaicId(mosaicDefTx.id.value),  // 制限をかけるモザイクID
  new sdkSymbol.models.UnresolvedMosaicId(0n),                    // 参照するモザイクID。制限をかけるモザイクIDと一致する場合は 0 を指定する
  key,  // restriction key 
  0n,   // 現在の restriction value
  1n,   // 新しく設定する restriction value
  sdkSymbol.models.MosaicRestrictionType.NONE,  // 現在の restriction type
  sdkSymbol.models.MosaicRestrictionType.EQ     // 新しく設定する restriction type
);
mosaicGlobalResTx = facade.createEmbeddedTransactionFromTypedDescriptor(
  mosaicGlobalResDescriptor,  // トランザクション Descriptor 設定
  carol.publicKey,            // 署名者公開鍵
);
// 更新する場合は以下も設定する必要あり
//   - mosaicGlobalResTx.previousRestrictionValue
//   - mosaicGlobalResTx.previousRestrictionType

embeddedTransactions = [
  mosaicDefTx,
  mosaicChangeTx,
  mosaicGlobalResTx
];

// アグリゲートTx作成
aggregateDescriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV2Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  aggregateDescriptor,  // トランザクション Descriptor 設定
  carol.publicKey,      // 署名者公開鍵
  100,                  // 手数料乗数
  60 * 60 * 2,          // Deadline:有効期限(秒単位)
  0                     // 連署者数
);

// 署名とアナウンス
sig = carol.signTransaction(aggregateTx);
jsonPayload = facade.transactionFactory.static.attachSignature(aggregateTx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

MosaicRestrictionTypeについては以下の通りです。

```js
{0: 'NONE', 1: 'EQ', 2: 'NE', 3: 'LT', 4: 'LE', 5: 'GT', 6: 'GE'}
```

| 演算子  | 略称  | 英語  |
|---|---|---|
| =  | EQ  | equal to  |
| !=  | NE  | not equal to  |
| <  | LT  | less than  |
| <=  | LE  | less than or equal to  |
| >  | GT  | greater than  |
| <=  | GE  | greater than or equal to  |

### アカウントへのモザイク制限適用

Carol,Bobに対してグローバル制限モザイクに対しての適格情報を追加します。  
送信・受信についてかかる制限なので、すでに所有しているモザイクについての制限はありません。  
送信を成功させるためには、送信者・受信者双方が条件をクリアしている必要があります。  
モザイク作成者の秘密鍵があればどのアカウントに対しても承諾の署名を必要とせずに制限をつけることができます。  

```js
// Carolに適用
carolMosaicAddressResDescriptor = new sdkSymbol.descriptors.MosaicAddressRestrictionTransactionV1Descriptor( // Txタイプ:モザイク制限適用Tx
  new sdkSymbol.models.UnresolvedMosaicId(mosaicDefTx.id.value),  // 適用するモザイクID
  key,                  // restriction key 
  0xFFFFFFFFFFFFFFFFn,  // 現在の restriction value (制限がかけられていない場合は 0xFFFFFFFFFFFFFFFF)
  1n,                   // 新しく設定する restriction value
  carol.address
);
carolMosaicAddressResTx = facade.createTransactionFromTypedDescriptor(
  carolMosaicAddressResDescriptor,  // トランザクション Descriptor 設定
  carol.publicKey,                  // 署名者公開鍵
  100,                              // 手数料乗数
  60 * 60 * 2                       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
carolSig = carol.signTransaction(carolMosaicAddressResTx);
carolJsonPayload = facade.transactionFactory.static.attachSignature(carolMosaicAddressResTx, carolSig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: carolJsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});

bob = facade.createAccount(sdkCore.PrivateKey.random());

// Bobに適用
bobMosaicAddressResDescriptor = new sdkSymbol.descriptors.MosaicAddressRestrictionTransactionV1Descriptor( // Txタイプ:モザイク制限適用Tx
  new sdkSymbol.models.UnresolvedMosaicId(mosaicDefTx.id.value),  // 適用するモザイクID
  key,                  // restriction key 
  0xFFFFFFFFFFFFFFFFn,  // 現在の restriction value (制限がかけられていない場合は 0xFFFFFFFFFFFFFFFF)
  1n,                   // 新しく設定する restriction value
  bob.address
);
bobMosaicAddressResTx = facade.createTransactionFromTypedDescriptor(
  bobMosaicAddressResDescriptor,  // トランザクション Descriptor 設定
  carol.publicKey,                // 署名者公開鍵
  100,                            // 手数料乗数
  60 * 60 * 2                     // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
bobSig = carol.signTransaction(bobMosaicAddressResTx);
bobJsonPayload = facade.transactionFactory.static.attachSignature(bobMosaicAddressResTx, bobSig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: bobJsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

### 制限状態確認

ノードに問い合わせて制限状態を確認します。

```js
query = new URLSearchParams({
  "mosaicId": mosaicDefTx.id.toString().substr(2),
});
res = await fetch(
  new URL('/restrictions/mosaic?' + query.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
console.log(res);
```

###### 出力例
```js
> (3) [{…}, {…}, {…}]
  > 0: 
      id: "64BD2ADC6FFE587B6D446083"
    > mosaicRestrictionEntry: 
        compositeHash: "5A697731C9F459E1693B0ECA633FD19E050975A787F8F0E3528662FA187DDC50"
        entryType: 1
        mosaicId: "69594C1564469E28"
      > restrictions: Array(1)
        > 0: 
            key: "9300605567124626807"
          > restriction: 
              referenceMosaicId: "0000000000000000"
              restrictionType: 1
              restrictionValue: "1"
          length: 1
        version: 1
  > 1: 
      id: "64BD3B4C6FFE587B6D4497E5"
    > mosaicRestrictionEntry: 
        compositeHash: "986A5008C27D92E0BEC10D36694BDD7D6514035D115E8CA4224CAAF8F789BAF2"
        entryType: 0
        mosaicId: "69594C1564469E28"
      > restrictions: Array(1)
        > 0: 
            key: "9300605567124626807"
            value: "1"
          length: 1
        targetAddress: "98A3670EF79748312687154D5B92AFED6B7C5E5A7560A423"
        version: 1
  > 2: 
    ...
    length: 3
```

### 送信確認

実際にモザイクを送信してみて、制限状態を確認します。

```js
// 成功（CarolからBobに送信）
trDescriptor = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  bob.address,      // 受取アドレス
  [
    // グローバルモザイク制限をかけたモザイクID
    new sdkSymbol.descriptors.UnresolvedMosaicDescriptor(
      new sdkSymbol.models.UnresolvedMosaicId(mosaicDefTx.id.value),
      new sdkSymbol.models.Amount(1n)
    )
  ],
  new Uint8Array()  // メッセージ
);
trTx = facade.createTransactionFromTypedDescriptor(
  trDescriptor,     // トランザクション Descriptor 設定
  carol.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);
// 署名とアナウンス
sig = carol.signTransaction(trTx);
jsonPayload = facade.transactionFactory.static.attachSignature(trTx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});

// 失敗（CarolからDaveに送信）
dave = facade.createAccount(sdkCore.PrivateKey.random());
// Tx作成
trDescriptor = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  dave.address,     // 受取アドレス
  [
    // グローバルモザイク制限をかけたモザイクID
    new sdkSymbol.descriptors.UnresolvedMosaicDescriptor(
      new sdkSymbol.models.UnresolvedMosaicId(mosaicDefTx.id.value),
      new sdkSymbol.models.Amount(1n)
    )
  ],
  new Uint8Array()  // メッセージ
);
trTx = facade.createTransactionFromTypedDescriptor(
  trDescriptor,     // トランザクション Descriptor 設定
  carol.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = carol.signTransaction(trTx);
jsonPayload = facade.transactionFactory.static.attachSignature(trTx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});

// ステータス確認URL表示
console.log(NODE + "/transactionStatus/" + facade.hashTransaction(trTx).toString());
```

失敗した場合以下のようなエラーステータスになります。

```js
{"hash":"E3402FB7AE21A6A64838DDD0722420EC67E61206C148A73B0DFD7F8C098062FA","code":"Failure_RestrictionMosaic_Account_Unauthorized","deadline":"12371602742","group":"failed"}
```

## 11.3 現場で使えるヒント

ブロックチェーンの社会実装などを考えたときに、法律や信頼性の見地から
一つの役割のみを持たせたいアカウント、関係ないアカウントを巻き込みたくないと思うことがあります。
そんな場合にアカウント制限とグローバルモザイク制限を使いこなすことで、
モザイクのふるまいを柔軟にコントロールすることができます。

### アカウントバーン

AllowIncomingAddressによって指定アドレスからのみ受信可能にしておいて、  
XYMを全量送信すると、秘密鍵を持っていても自力では操作困難なアカウントを明示的に作成することができます。  
（最小手数料を0に設定したノードによって承認されることもあり、その可能性はゼロではありません）  

### モザイクロック
譲渡不可設定のモザイクを配布し、配布者側のアカウントで受け取り拒否を行うとモザイクをロックさせることができます。

### 所属証明
モザイクの章で所有の証明について説明しました。グローバルモザイク制限を活用することで、
KYCが済んだアカウント間でのみ所有・流通させることが可能なモザイクを作り、所有者のみが所属できる独自経済圏を構築することが可能です。
