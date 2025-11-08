# 4.トランザクション
ブロックチェーン上のデータ更新はトランザクションをネットワークにアナウンスすることによって行います。

## 4.1 トランザクションのライフサイクル

トランザクションを作成してから、改ざんが困難なデータとなるまでを順に説明します。

- トランザクション作成
  - ブロックチェーンが受理できるフォーマットでトランザクションを作成します。
- 署名
  - アカウントの秘密鍵でトランザクションを署名します。
- アナウンス
  - 任意のノードに署名済みトランザクションを通知します。
- 未承認トランザクション
  - ノードに受理されたトランザクションは、未承認トランザクションとして全ノードに伝播します
    - トランザクションに設定した最大手数料が、各ノード毎に設定されている最低手数料を満たさない場合はそのノードへは伝播しません。
- 承認済みトランザクション
  - 約30秒に1度ごとに生成されるブロックに未承認トランザクションが取り込まれると、承認済みトランザクションとなります。
- ロールバック
  - ノード間の合意に達することができずロールバックされたブロックに含まれていたトランザクションは、未承認トランザクションに差し戻されます。
    - 有効期限切れや、キャッシュからあふれたトランザクションは切り捨てられます。
- ファイナライズ
  - 投票ノードによるファイナライズプロセスによりブロックが確定するとトランザクションはロールバック不可なデータとして扱うことができます。

### ブロックとは

ブロックは約30秒ごとに生成され、高い手数料を支払ったトランザクションから優先に取り込まれ、ブロック単位で他のノードと同期します。
同期に失敗するとロールバックして、ネットワークが全体で合意が取れるまでこの作業を繰り返します。

## 4.2 トランザクション作成

まずは最も基本的な転送トランザクションを作成してみます。

### Bobへの転送トランザクション

送信先のBobアドレスを作成しておきます。

```js
bob = facade.createAccount(sdkCore.PrivateKey.random());
console.log(bob.address.toString());
```

```js
> TDWBA6L3CZ6VTZAZPAISL3RWM5VKMHM6J6IM3LY
```

トランザクションを作成します。

v3 では Descriptor によりトランザクションのタイプや必要な情報を指定します。

```js
messageData = "\0Hello, Symbol!"; // 平文メッセージ

// トランザクション Descriptor 設定
descriptor = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  bob.address,      // 受取アドレス
  [
    //// 1XYM送金
    // new sdkSymbol.descriptors.UnresolvedMosaicDescriptor(
    //   new sdkSymbol.models.UnresolvedMosaicId(0x72C0212E67A08BCEn),
    //   new sdkSymbol.models.Amount(1000000n)
    // )
  ],
  messageData       // メッセージ
);

tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);
console.log(tx);
```

各設定項目について説明します。

#### 有効期限
最大6時間まで指定可能です。

`facade.createTransactionFromTypedDescriptor()` の第4引数に秒単位で指定します。

```js
tx = facade.createTransactionFromTypedDescriptor(
  ,           // Descriptor
  ,           // 署名者公開鍵
  ,           // 手数料乗数
  60 * 60 * 6 // Deadline:有効期限(秒単位)
);
```

#### メッセージ
トランザクションに最大1023バイトのメッセージを添付することができます。
バイナリデータであってもrawdataとして送信することが可能です。

##### 空メッセージ

```js
messageData = new Uint8Array();
```

##### 平文メッセージ

v3 では、平文メッセージの場合は文字列そのままで指定することができます。

```js
messageData = 'Hello, Symbol!';
```

先頭にメッセージタイプ `0x00` を付加する場合は以下のように指定します。

```js
messageData = '\0Hello, Symbol!';
```

エクスプローラーやデスクトップウォレットはこのフラグで判別してメッセージを表示しています。

##### 暗号文メッセージ

EncryptedMessageを使用すると、「指定したメッセージが暗号化されています」という意味のフラグ（目印）がつきます。 エクスプローラーやウォレットはそのフラグを参考にメッセージを無用にデコードしなかったり、非表示にしたりなどの処理を行います。 このメソッドが暗号化をするわけではありません。

`MessageEncoder` を使用して暗号化すると、自動で暗号文メッセージを表すメッセージタイプ `0x01` が付加されます。

```js
messageData = alice.messageEncoder().encode(bob.publicKey, new TextEncoder().encode('Hello Symbol!'));
```

##### 生データ

v3 では先頭に生データを表すメッセージタイプ `0xFF` を付加します。

```js
messageData = new Uint8Array([0xFF,...(new TextEncoder('utf-8')).encode('Hello, Symbol!')]);
```

#### 最大手数料

ネットワーク手数料については、常に少し多めに払っておけば問題はないのですが、最低限の知識は持っておく必要があります。
アカウントはトランザクションを作成するときに、ここまでは手数料として払ってもいいという最大手数料を指定します。
一方で、ノードはその時々で最も高い手数料となるトランザクションのみブロックにまとめて収穫しようとします。
つまり、多く払ってもいいというトランザクションが他に多く存在すると承認されるまでの時間が長くなります。
逆に、より少なく払いたいというトランザクションが多く存在し、その総額が大きい場合は、設定した最大額に満たない手数料額で送信が実現します。

トランザクションサイズ x feeMultiprilerというもので決定されます。
176バイトだった場合 maxFee を100で設定すると 17600μXYM = 0.0176XYMを手数料として支払うことを許容します。
feeMultiprier = 100として指定する方法とmaxFee = 17600 として指定する方法があります。

##### feeMultiprier = 100として指定する方法

`facade.createTransactionFromTypedDescriptor()` の第3引数で指定します。

```js
tx = facade.createTransactionFromTypedDescriptor(
  ,     // Descriptor
  ,     // 署名者公開鍵
  100,  // 手数料乗数
        // Deadline:有効期限(秒単位)
);
```

##### maxFee = 17600 として指定する方法

```js
tx = facade.createTransactionFromTypedDescriptor(
  // 省略
);
tx.fee = new sdkSymbol.models.Amount(BigInt(17600)); //手数料
```

本書では以後、feeMultiprier = 100として指定する方法で統一して説明します。

## 4.3 署名とアナウンス

作成したトランザクションを秘密鍵で署名して、任意のノードを通じてアナウンスします。

### 署名

```js
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
```
###### 出力例
```js
> '{"payload": "AF0000000000000041EE9F8B3EB4D54F069B1CD47A79656DCE4C85B486D1735DF054B91838ECF6E06B6F371BB986E676A5F5BF091A5DEF5230BC6E6112F7D2104BE24923355B890869A31A837EB7DE323F08CA52495A57BA0A95B52D1BB54CEA9A94C12A87B1CADB00000000019854415C44000000000000426B07250500000098A8D76FEF8382274D472EE377F2FF3393E5B62C08B4329D0F00000000000000FF48656C6C6F2C2053796D626F6C21"}'
```

### アナウンス

```js
res = await fetch(
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
console.log(res);
```
```js
> {message: 'packet 9 was pushed to the network via /transactions'}
```

上記のスクリプトのように `packet n was pushed to the network` というレスポンスがあれば、トランザクションはノードに受理されたことになります。
これはトランザクションのフォーマット等に異常が無かった程度の意味しかありません。
Symbolではノードの応答速度を極限に高めるため、トランザクションの内容を検証するまえに受信結果の応答を返し接続を切断します。
レスポンス値はこの情報を受け取ったにすぎません。フォーマットに異常があった場合は以下のようなメッセージ応答があります。

##### アナウンスに失敗した場合の応答例

```js
> {code: 'InvalidArgument', message: 'payload has an invalid format'}
```

## 4.4 確認

### ステータスの確認

ノードに受理されたトランザクションのステータスを確認

```js
body = {
  "hashes": facade.hashTransaction(tx).toString()
};
transactionStatus = await fetch(
  new URL('/transactionStatus', NODE),
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
if (transactionStatus.length <= 0) {
  console.error("not exist tx.");
} else {
  console.log(transactionStatus[0]);
}
```
###### 出力例
```js
> {group: 'confirmed', code: 'Success', hash: '02DC42E10B3E51B49AA9CD1074481C4B0764D8DA4BE7F33F83DC1DC9ED84C79D', deadline: '22164976452', height: '636865'}
```

承認されると ` group: "confirmed"`となっています。

受理されたものの、エラーが発生していた場合は以下のような出力となります。トランザクションを書き直して再度アナウンスしてみてください。

```js
> TransactionStatus
    group: "failed"
    code: "Failure_Core_Insufficient_Balance"
    deadline: Deadline {adjustedValue: 11990156766}
    hash: "A82507C6C46DF444E36AC94391EA2D0D7DD1A218948DED465A7A4F9D1B53CA0E"
    height: undefined
```

以下のようにResourceNotFoundエラーが発生した場合はトランザクションが受理されていません。
```js
Uncaught Error: {"statusCode":404,"statusMessage":"Unknown Error","body":"{\"code\":\"ResourceNotFound\",\"message\":\"no resource exists with id '18AEBC9866CD1C15270F18738D577CB1BD4B2DF3EFB28F270B528E3FE583F42D'\"}"}
```

考えられる可能性としては、トランザクションで指定した最大手数料が、ノードで設定された最低手数料に満たない場合や、
アグリゲートトランザクションとしてアナウンスすることが求められているトランザクションを単体のトランザクションでアナウンスした場合に発生するようです。

### 承認確認

トランザクションがブロックに承認されるまでに30秒程度かかります。

#### エクスプローラーで確認
facade.hashTransaction(tx).toString() で取得できるハッシュ値を使ってエクスプローラーで検索してみましょう。

```js
console.log(facade.hashTransaction(tx).toString());
```

```js
> "661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747"
```

- メインネット　
  - https://symbol.fyi/transactions/661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747
- テストネット　
  - https://testnet.symbol.fyi/transactions/661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747

#### SDKで確認

```js
txInfo = await fetch(
  new URL('/transactions/confirmed/' + facade.hashTransaction(tx).toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
console.log(txInfo);
```
###### 出力例
```js
> {meta: {…}, transaction: {…}, id: '64B253382F7CE156B01031C1'}
    id: "64B253382F7CE156B01031C1"
  > meta: 
      feeMultiplier: 100
      hash: "02DC42E10B3E51B49AA9CD1074481C4B0764D8DA4BE7F33F83DC1DC9ED84C79D"
      height: "636865"
      index: 0
      merkleComponentHash: "02DC42E10B3E51B49AA9CD1074481C4B0764D8DA4BE7F33F83DC1DC9ED84C79D"
      timestamp: "22157844926"
  > transaction: 
      deadline: "22164976452"
      maxFee: "17500"
      message: "0048656C6C6F2C2053796D626F6C21"
      mosaics: []
      network: 152
      recipientAddress: "98A8D76FEF8382274D472EE377F2FF3393E5B62C08B4329D"
      signature: "29783E718E414D2A2B62821FC53E66DCFEBDC1EF10FFD35F7662263FA7332F01CD47676429A982A85FEC7F2E7983D36F4BCE5C5AB1BB4A591BC54979AF906D0A"
      signerPublicKey: "69A31A837EB7DE323F08CA52495A57BA0A95B52D1BB54CEA9A94C12A87B1CADB"
      size: 175
      type: 16724
      version: 1
```

##### 注意点

トランザクションはブロックで承認されたとしても、ロールバックが発生するとトランザクションの承認が取り消される場合があります。
ブロックが承認された後、数ブロックの承認が進むと、ロールバックの発生する確率は減少していきます。
また、Votingノードの投票で実施されるファイナライズブロックを待つことで、記録されたデータは確実なものとなります。

##### スクリプト例
トランザクションをアナウンスした後は以下のようなスクリプトを流すと、チェーンの状態を把握しやすくて便利です。

```js
hash = facade.hashTransaction(tx).toString();
// ステータスの確認
body = {
  "hashes": hash
};
transactionStatus = await fetch(
  new URL('/transactionStatus', NODE),
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
if (transactionStatus.length <= 0) {
  console.error("not exist tx.");
} else {
  console.log(transactionStatus[0]);
}
console.log(transactionStatus);
// 承認確認
txInfo = await fetch(
  new URL('/transactions/confirmed/' + hash, NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
console.log(txInfo);
```

## 4.5トランザクション履歴

Aliceが送受信したトランザクション履歴を一覧で取得します。



```js
params = new URLSearchParams({
  "address": alice.address.toString(),
  "embedded": true,
});
result = await fetch(
  new URL('/transactions/confirmed?' + params.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});

txes = result.data;
txes.forEach(tx => {
  console.log(tx);
});
```
###### 出力例
```js
// 取得できたTx数分表示されます
> {meta: {…}, transaction: {…}, id: '636EF1EF8EFA5A777403DEDA'}
    id: "636EF1EF8EFA5A777403DEDA"
  > meta: 
      feeMultiplier: 1136363
      hash: "EFDD7A4D419CAC459DA034089DF7F25E416A4782A1A6D9DBEC4ECF9AC3AAF1A9"
      height: "18925"
      index: 0
      merkleComponentHash: "EFDD7A4D419CAC459DA034089DF7F25E416A4782A1A6D9DBEC4ECF9AC3AAF1A9"
      timestamp: "964811467"
  > transaction: 
      deadline: "971971235"
      maxFee: "200000000"
    > mosaics: Array(1)
        0: 
          amount: "100000000"
          id: "72C0212E67A08BCE"
      network: 152
      recipientAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
      signature: "17104B0B8C9CB34C4FC42E448488A6608387841715921160F81C0FD60054CFACAE7200E3F41D911AA914D431197D923D6D98566E711DDC13C4BF0C768AAB3A04"
      signerPublicKey: "81EA7C15E7EC06261C9F654F54EAC4748CFCF00E09A8FE47779ACD14A7602004"
      size: 176
      type: 16724
      version: 1
```

TransactionTypeは以下の通りです。

```js
{0: 'RESERVED', 16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS', 17232: 'ACCOUNT_OPERATION_RESTRICTION'}
```

MessageTypeは以下の通りです。

```js
{0: 'PlainMessage', 1: 'EncryptedMessage', 254: 'PersistentHarvestingDelegationMessage', -1: 'RawMessage'}
```


## 4.5.1 Txペイロード作成時のメッセージの差異

v3 ではメッセージはバイナリデータで表されます。
平文メッセージはバイナリデータに変換され、暗号化メッセージや生データのようなバイナリデータはそのままのバイナリデータとなります。

```js
message = 'Hello Symbol!';
plainMessage = new Uint8Array(new TextEncoder('utf-8').encode('\0Hello, Symbol!'));
console.log(plainMessage);
encryptedMessage = alice.messageEncoder().encode(bob.publicKey, new TextEncoder().encode(message));
console.log(encryptedMessage);
rawMessage = new Uint8Array([0xFF, 0x10, 0x20, 0x30]);
console.log(rawMessage);
```
###### 出力例
```js
> Uint8Array(15) [...]
    0: 0    // メッセージタイプ 0x00 (平文メッセージ)
    1: 72   // 'H'
    2: 101  // 'e'
    3: 108  // 'l'
    // 以下省略
> Uint8Array(42) [...]
    0: 1    // メッセージタイプ 0x01 (暗号化メッセージ)
    1: 163
    2: 55
    3: 105
    // 以下省略
> Uint8Array(4) [...]
    0: 255  // メッセージタイプ 0xFF (生データ)
    1: 16   // 0x10
    2: 32   // 0x20
    3: 48   // 0x30
```

署名時にTxのペイロードを作成する際には、メッセージデータを16進数文字列に変換します。

### 標準の方法でメッセージを読み込む (この部分は、v3から学び始めた方は読み飛ばしてもらって大丈夫です)

実際に異なるSDKバージョンでTxを読み込んでみます。
まずはそれぞれのバージョンでTxを作成し、ブロックチェーン上にアナウンスします。

#### v2

```js
// 暗号化メッセージの作成
encryptedMessage = alice.encryptMessage("Hello Symbol!", bob.publicAccount);

// Tx 作成
tx = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment), //Deadline:有効期限
    bob.address, 
    [],
    encryptedMessage, //メッセージ
    networkType //テストネット・メインネット区分
).setMaxFee(100); //手数料

// 署名とアナウンス
signedTx = alice.sign(tx,generationHash);
res = await txRepo.announce(signedTx).toPromise();
// Txハッシュの表示
console.log(signedTx.hash);
```
```js
> DE663D99BC9E2EEC408E255055CC4DA18CCEEEEF57CE97E607B2C47E9C725085
```

#### v3

```js
// 暗号化メッセージの作成
encryptedMessage = alice.messageEncoder().encode(bob.publicKey, new TextEncoder().encode("Hello Symbol!"));

// トランザクション Descriptor 設定
descriptor = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  bob.address,      // 受取アドレス
  [],               // 送信モザイク
  messageData       // メッセージ
);

// Tx 作成
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
res = await fetch(
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
// Txハッシュの表示
console.log(facade.hashTransaction(tx).toString());
```
```js
> AB6146ADBC96DAF58741F98FB2BACE7D96AAEBFA20416458AE7EF4253FB40ECF
```

次に、Txを作成したバージョンと異なるバージョンのSDKでTxを読み込み、メッセージを表示してみます。

#### v2

```js
// v3 でアナウンスしたTxのハッシュ値を指定してTx取得
txInfo = await txRepo.getTransaction("AB6146ADBC96DAF58741F98FB2BACE7D96AAEBFA20416458AE7EF4253FB40ECF",sym.TransactionGroup.Confirmed).toPromise();

// メッセージを復号化して表示
console.log(bob.decryptMessage(txInfo.message, alice.publicAccount));
```
```js
> Uncaught Error: unrecognized hex char, char1:D, char2:�
```

#### v3

```js
// v2 でアナウンスしたTxのハッシュ値を指定してTx取得
txInfo = await fetch(
  new URL('/transactions/confirmed/DE663D99BC9E2EEC408E255055CC4DA18CCEEEEF57CE97E607B2C47E9C725085', NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});

// メッセージを復号化して表示
console.log(bob.messageEncoder().tryDecode(alice.publicKey, Buffer.from(txInfo.transaction.message, "hex")));
```
```js
> (2) [false, Uint8Array(82)]
    0: false
    1: Uint8Array(82)
```

### v2 で作成したメッセージを v3 で読み込む

v2 で作成したメッセージを v3 で読み込むためには、16進数文字列から戻すための変換を2回行う必要があります。

#### v3

```js
messageV2 = txInfo.transaction.message.substr(2); // メッセージタイプを取り除く
hex1 = Buffer.from(messageV2, "hex");             // 1回目の16進数文字列から戻す変換
hex2 = Buffer.from(hex1.toString(), "hex");       // 2回目の16進数文字列から戻す変換
console.log(bob.messageEncoder().tryDecode(alice.publicKey, new Uint8Array([0x01, ...hex2])));  // メッセージタイプ 0x01 を付けてメッセージを復号する
```

```js
> (2) [true, Uint8Array(13)]
    0: true
    1: Uint8Array(13)
```

なお、v3.0.8 以降であれば、自動判別して復号を行うための関数 `tryDecodeDeprecated()` が用意されています。

#### v3

```js
bob.messageEncoder().tryDecodeDeprecated(alice.publicKey, Buffer.from(txInfo.transaction.message, "hex"));
```

### v2 で読み込めるように v3 でメッセージを作成する

v2 で読み込めるようにするため、 v3 でメッセージを作成する際に16進数文字列への変換を2回行います。

#### v3

```js
// 暗号化メッセージの作成
encrypted = alice.messageEncoder().encode(bob.publicKey, new TextEncoder().encode("Hello Symbol!"));
hex1 = Buffer.from(encrypted).subarray(1).toString("hex").toUpperCase();
encryptedMessage = new Uint8Array([0x01, ...(new TextEncoder().encode(hex1))]);
// Tx作成、アナウンス
```

#### v2
```js
txInfo = await txRepo.getTransaction("B50B7D51AE9401C364799EAC1E0FFE9CB1F4B8F531B03AD39658BD4FB5245A7F",sym.TransactionGroup.Confirmed).toPromise();
console.log(bob.decryptMessage(txInfo.message, alice.publicAccount));
```
```js
> PlainMessage {type: 0, payload: 'Hello Symbol!'}
    payload: "Hello Symbol!"
    type: 0
```

こちらも v3.0.8 以降であれば、 v2 向けに暗号化を行うための関数 `encodeDeprecated()` が用意されています。

#### v3

```js
alice.messageEncoder().encodeDeprecated(bob.publicKey, new TextEncoder().encode("Hello Symbol!"));
```

## 4.6 アグリゲートトランザクション

Symbolでは複数のトランザクションを1ブロックにまとめてアナウンスすることができます。
最大で100件のトランザクションをまとめることができます（連署者が異なる場合は25アカウントまでを連署指定可能）。
以降の章で扱う内容にアグリゲートトランザクションへの理解が必要な機能が含まれますので、
本章ではアグリゲートトランザクションのうち、簡単なものだけを紹介します。
### 起案者の署名だけが必要な場合



```js
bob = facade.createAccount(sdkCore.PrivateKey.random());
carol = facade.createAccount(sdkCore.PrivateKey.random());

// アグリゲートTxに含めるTxを作成
descriptor1 = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  bob.address,      // 送信先アドレス
  [],               // 送信モザイク
  '\0tx1'             // 平文メッセージ
);
innerTx1 = facade.createEmbeddedTransactionFromTypedDescriptor(
  descriptor1,      // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
);

descriptor2 = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  carol.address,    // 送信先アドレス
  [],               // 送信モザイク
  '\0tx2'             // 平文メッセージ
);
innerTx2 = facade.createEmbeddedTransactionFromTypedDescriptor(
  descriptor2,      // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
);

embeddedTransactions = [
  innerTx1,
  innerTx2
];

// アグリゲートTx作成
descriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV3Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2,      // Deadline:有効期限(秒単位)
  0                 // 連署者数
);

// 署名とアナウンス
sig = alice.signTransaction(aggregateTx);
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

まず、アグリゲートトランザクションに含めるトランザクションを作成します。
このときDeadlineを指定する必要はありません。
リスト化するときに、生成したトランザクションにtoAggregateを追加して送信元アカウントの公開鍵を指定します。
ちなみに送信元アカウントと署名アカウントが **必ずしも一致するとは限りません** 。
後の章での解説で「Bobの送信トランザクションをAliceが署名する」といった事が起こり得るためこのような書き方をします。
これはSymbolブロックチェーンでトランザクションを扱ううえで最も重要な概念になります。
なお、本章で扱うトランザクションは同じAliceですので、アグリゲートボンデッドトランザクションへの署名もAliceを指定します。

### アグリゲートトランザクションにおける最大手数料

アグリゲートトランザクションも通常のトランザクション同様、最大手数料を直接指定する方法とfeeMultiprierで指定する方法があります。
先の例では最大手数料を直接指定する方法を使用しました。ここではfeeMultiprierで指定する方法を紹介します。



`facade.createTransactionFromTypedDescriptor()` の第3引数に feeMultiprier、第5引数に連署者の数を指定します。

```js
tx = facade.createTransactionFromTypedDescriptor(
  ,     // Descriptor
  ,     // 署名者公開鍵
  100,  // 手数料乗数
  ,     // Deadline:有効期限(秒単位)
  1     // 必要な連署者の数を指定
);
```



## 4.7 現場で使えるヒント

### 存在証明

アカウントの章でアカウントによるデータの署名と検証する方法について説明しました。
このデータをトランザクションに載せてブロックチェーンが承認することで、
アカウントがある時刻にあるデータの存在を認知したことを消すことができなくなります。
タイムスタンプの刻印された電子署名を利害関係者間で所有することと同じ意味があると考えることもできます。
（法律的な判断は他の方にお任せします）

ブロックチェーンは、この消せない「アカウントが認知したという事実」の存在をもって送信などのデータ更新を行います。
また、誰もがまだ知らないはずの事実を知っていたことの証明としてブロックチェーンを利用することもできます。
ここでは、その存在が証明されたデータをトランザクションに載せる２つの方法について説明します。

#### デジタルデータのハッシュ値(SHA256)出力方法

ファイルの要約値をブロックチェーンに記録することでそのファイルの存在を証明することができます。

各OSごとのファイルのSHA256でハッシュ値を計算する方法は下記の通りです。
```sh
#Windows
certutil -hashfile WINファイルパス SHA256
#Mac
shasum -a 256 MACファイルパス
#Linux
sha256sum Linuxファイルパス
```

#### 大きなデータの分割

トランザクションのペイロードには1023バイトしか格納できないため、
大きなデータは分割してペイロードに詰め込んでアグリゲートトランザクションにします。

```js
bigdata = 'C00200000000000093B0B985101C1BDD1BC2BF30D72F35E34265B3F381ECA464733E147A4F0A6B9353547E2E08189EF37E50D271BEB5F09B81CE5816BB34A153D2268520AF630A0A0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198414140770200000000002A769FB40000000076B455CFAE2CCDA9C282BF8556D3E9C9C0DE18B0CBE6660ACCF86EB54AC51B33B001000000000000DB000000000000000E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198544198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F22338B000000000000000066653465353435393833444430383935303645394533424446434235313637433046394232384135344536463032413837364535303734423641303337414643414233303344383841303630353343353345354235413835323835443639434132364235343233343032364244444331443133343139464435353438323930334242453038423832304100000000006800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F2233BC089179EBBE01A81400140035383435344434373631364336433635373237396800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F223345ECB996EDDB9BEB1400140035383435344434373631364336433635373237390000000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D5A71EBA9C924EFA146897BE6C9BB3DACEFA26A07D687AC4A83C9B03087640E2D1DDAE952E9DDBC33312E2C8D021B4CC0435852C0756B1EBD983FCE221A981D02';

let payloads = [];
for (let i = 0; i < bigdata.length / 1023; i++) {
    payloads.push(bigdata.substr(i * 1023, 1023));
}
console.log(payloads);
```
