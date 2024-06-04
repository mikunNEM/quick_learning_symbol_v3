# 7.メタデータ

アカウント・モザイク・ネームスペースに対してKey-Value形式のデータを登録することができます。  
Valueの最大値は1024バイトです。
本章ではモザイク・ネームスペースの作成アカウントとメタデータの作成アカウントがどちらもAliceであることを前提に説明します。

本章のサンプルスクリプトを実行する前に以下を実行して必要ライブラリを読み込んでおいてください。

#### v2

```js
metaRepo = repo.createMetadataRepository();
mosaicRepo = repo.createMosaicRepository();
metaService = new sym.MetadataTransactionService(metaRepo);
```

#### v3

v3 ではキーを生成するメソッドが無いため、 v2 と同様のキーを生成するためにハッシュライブラリをインポートする必要があります。

```js
sha3_256 = (await import('https://cdn.skypack.dev/@noble/hashes/sha3')).sha3_256;
```

## 7.1 アカウントに登録

アカウントに対して、Key-Value値を登録します。

#### v2

```js
key = sym.KeyGenerator.generateUInt64Key("key_account");
value = "test";

tx = await metaService.createAccountMetadataTransaction(
    undefined,
    networkType,
    alice.address, //メタデータ記録先アドレス
    key,value, //Key-Value値
    alice.address //メタデータ作成者アドレス
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
  sym.Deadline.create(epochAdjustment),
  [tx.toAggregate(alice.publicAccount)],
  networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// ターゲットと作成者アドレスの設定
targetAddress = alice.address;  // メタデータ記録先アドレス
sourceAddress = alice.address;  // メタデータ作成者アドレス

// キーと値の設定
key = sdkSymbol.metadataGenerateKey("key_account");
value = new TextEncoder().encode("test");

// 同じキーのメタデータが登録されているか確認
query = new URLSearchParams({
  "targetAddress": targetAddress.toString(),
  "sourceAddress": sourceAddress.toString(),
  "scopedMetadataKey": key.toString(16).toUpperCase(),
  "metadataType": 0
});
metadataInfo = await fetch(
  new URL('/metadata?' + query.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
// 登録済の場合は差分データを作成する
sizeDelta = value.length;
if (metadataInfo.length > 0) {
  sizeDelta -= metadataInfo[0].metadataEntry.valueSize;
  value = sdkSymbol.metadataUpdateValue(sdkCore.utils.hexToUint8(metadataInfo[0].metadataEntry.value, value));
}

// アカウントメタデータ登録Tx作成
descriptor = new sdkSymbol.descriptors.AccountMetadataTransactionV1Descriptor(  // Txタイプ:アカウントメタデータ登録Tx
  targetAddress,  // ターゲットアドレス
  key,            // キー
  sizeDelta,      // サイズ差分
  value           // 値
);
tx = facade.createEmbeddedTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
);

embeddedTransactions = [
  tx
];

// アグリゲートTx作成
aggregateDescriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV2Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  aggregateDescriptor,  // トランザクション Descriptor 設定
  alice.publicKey,      // 署名者公開鍵
  100,                  // 手数料乗数
  60 * 60 * 2,          // Deadline:有効期限(秒単位)
  0                     // 連署者数
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

<details><summary>symbol-sdk v3.2.1 まで</summary>

v3.2.2 で `metadataGenerateKey()` が実装されたことにより、 v2 と同じメタデータキーを生成することができるようになりました。
v3.2.1 までにおける、 v2 と同じメタデータキーを導出するプログラムは以下となります。

```js
// キー生成(v2 準拠)
key = "key_account";
hasher = sha3_256.create();
hasher.update((new TextEncoder()).encode(key));
digest = hasher.digest();
lower = [...digest.subarray(0, 4)];
lower.reverse();
lowerValue = BigInt("0x" + sdkCore.utils.uint8ToHex(lower));
higher = [...digest.subarray(4, 8)];
higher.reverse();
higherValue = BigInt("0x" + sdkCore.utils.uint8ToHex(higher)) | 0x80000000n;
keyId = lowerValue + higherValue * 0x100000000n;
```

また、 `metadataUpdateValue()` が実装されたことにより、メタデータの差分データ生成が簡単になりました。
v3.2.1 までにおける、メタデータの差分データ生成プログラムは以下となります。

```js
// 同じキーのメタデータが登録されているか確認
// metadataInfo = await fetch(...); の実行

// 登録済の場合は差分データを作成する
value = new TextEncoder().encode("test");
sizeDelta = value.length;
if (metadataInfo.length > 0) {
  sizeDelta -= metadataInfo[0].metadataEntry.valueSize;
  originData = sdkCore.utils.hexToUint8(metadataInfo[0].metadataEntry.value);
  diffData = new Uint8Array(Math.max(originData.length, value.length));
  for (idx = 0; idx < diffData.length; idx++) {
    diffData[idx] = (originData[idx] == undefined ? 0 : originData[idx]) ^ (value[idx] == undefined ? 0 : value[idx]);
  }
  value = diffData;
}
```

</details>

メタデータの登録には記録先アカウントが承諾を示す署名が必要です。
また、記録先アカウントと記録者アカウントが同一でもアグリゲートトランザクションにする必要があります。

異なるアカウントのメタデータに登録する場合は署名時に
signTransactionWithCosignatoriesを使用します。

#### v2

```js
tx = await metaService.createAccountMetadataTransaction(
    undefined,
    networkType,
    bob.address, //メタデータ記録先アドレス
    key,value, //Key-Value値
    alice.address //メタデータ作成者アドレス
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
  sym.Deadline.create(epochAdjustment),
  [tx.toAggregate(alice.publicAccount)],
  networkType,[]
).setMaxFeeForAggregate(100, 1); // 第二引数に連署者の数:1

signedTx = aggregateTx.signTransactionWithCosignatories(
  alice,[bob],generationHash,// 第二引数に連署者
);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// ターゲットと作成者アドレスの設定
targetAddress = bob.address;    // メタデータ記録先アドレス
sourceAddress = alice.address;  // メタデータ作成者アドレス

// キーと値の設定
key = sdkSymbol.metadataGenerateKey("key_account");
value = new TextEncoder().encode("test");

// 同じキーのメタデータが登録されているか確認
query = new URLSearchParams({
  "targetAddress": targetAddress.toString(),
  "sourceAddress": sourceAddress.toString(),
  "scopedMetadataKey": key.toString(16).toUpperCase(),
  "metadataType": 0
});
metadataInfo = await fetch(
  new URL('/metadata?' + query.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
// 登録済の場合は差分データを作成する
sizeDelta = value.length;
if (metadataInfo.length > 0) {
  sizeDelta -= metadataInfo[0].metadataEntry.valueSize;
  value = sdkSymbol.metadataUpdateValue(sdkCore.utils.hexToUint8(metadataInfo[0].metadataEntry.value, value));
}

// アカウントメタデータ登録Tx作成
descriptor = new sdkSymbol.descriptors.AccountMetadataTransactionV1Descriptor(  // Txタイプ:アカウントメタデータ登録Tx
  targetAddress,  // ターゲットアドレス
  key,            // キー
  sizeDelta,      // サイズ差分
  value           // 値
);
tx = facade.createEmbeddedTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
);

embeddedTransactions = [
  tx
];

// アグリゲートTx作成
aggregateDescriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV2Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  aggregateDescriptor,  // トランザクション Descriptor 設定
  alice.publicKey,      // 署名者公開鍵
  100,                  // 手数料乗数
  60 * 60 * 2,          // Deadline:有効期限(秒単位)
  1                     // 連署者数
);

// 作成者による署名
sig = alice.signTransaction(aggregateTx);
jsonPayload = facade.transactionFactory.static.attachSignature(aggregateTx, sig);

// 記録先アカウントによる連署
coSig = bob.cosignTransaction(aggregateTx, false);
aggregateTx.cosignatures.push(coSig);

// アナウンス
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({"payload": sdkCore.utils.uint8ToHex(aggregateTx.serialize())}),
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

bobの秘密鍵が分からない場合はこの後の章で説明する
アグリゲートボンデッドトランザクション、あるいはオフライン署名を使用する必要があります。

## 7.2 モザイクに登録

ターゲットとなるモザイクに対して、Key値・ソースアカウントの複合キーでValue値を登録します。
登録・更新にはモザイクを作成したアカウントの署名が必要です。

#### v2

```js
mosaicId = new sym.MosaicId("1275B0B7511D9161");
mosaicInfo = await mosaicRepo.getMosaic(mosaicId).toPromise();

key = sym.KeyGenerator.generateUInt64Key('key_mosaic');
value = 'test';

tx = await metaService.createMosaicMetadataTransaction(
  undefined,
  networkType,
  mosaicInfo.ownerAddress, //モザイク作成者アドレス
  mosaicId,
  key,value, //Key-Value値
  alice.address
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [tx.toAggregate(alice.publicAccount)],
    networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// ターゲットと作成者アドレスの設定
targetMosaic = 0x1275B0B7511D9161n;  // メタデータ記録先モザイク
mosaicInfo = await fetch(
new URL('/mosaics/' + targetMosaic.toString(16).toUpperCase(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
targetAddress = new sdkSymbol.Address(sdkCore.utils.hexToUint8(mosaicInfo.mosaic.ownerAddress));  // モザイク作成者アドレス

// キーと値の設定
key = sdkSymbol.metadataGenerateKey("key_mosaic");
value = new TextEncoder().encode("test");

// 同じキーのメタデータが登録されているか確認
query = new URLSearchParams({
  "targetId": targetMosaic.toString(16).toUpperCase(),
  "sourceAddress": targetAddress.toString(),
  "scopedMetadataKey": key.toString(16).toUpperCase(),
  "metadataType": 1
});
metadataInfo = await fetch(
  new URL('/metadata?' + query.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
// 登録済の場合は差分データを作成する
sizeDelta = value.length;
if (metadataInfo.length > 0) {
  sizeDelta -= metadataInfo[0].metadataEntry.valueSize;
  value = sdkSymbol.metadataUpdateValue(sdkCore.utils.hexToUint8(metadataInfo[0].metadataEntry.value, value));
}

// モザイクメタデータ登録Tx作成
descriptor = new sdkSymbol.descriptors.MosaicMetadataTransactionV1Descriptor(  // Txタイプ:モザイクメタデータ登録Tx
  targetAddress,  // ターゲットアドレス
  key,            // キー
  new sdkSymbol.models.UnresolvedMosaicId(targetMosaic),  // メタデータ記録先モザイク
  sizeDelta,      // サイズ差分
  value           // 値
);
tx = facade.createEmbeddedTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
);

embeddedTransactions = [
  tx
];

// アグリゲートTx作成
aggregateDescriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV2Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  aggregateDescriptor,  // トランザクション Descriptor 設定
  alice.publicKey,      // 署名者公開鍵
  100,                  // 手数料乗数
  60 * 60 * 2,          // Deadline:有効期限(秒単位)
  0                     // 連署者数
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

## 7.3 ネームスペースに登録

ネームスペースに対して、Key-Value値を登録します。
登録・更新にはネームスペースを作成したアカウントの署名が必要です。

#### v2

```js
nsRepo = repo.createNamespaceRepository();
namespaceId = new sym.NamespaceId("xembook");
namespaceInfo = await nsRepo.getNamespace(namespaceId).toPromise();

key = sym.KeyGenerator.generateUInt64Key('key_namespace');
value = 'test';

tx = await metaService.createNamespaceMetadataTransaction(
    undefined,networkType,
    namespaceInfo.ownerAddress, //ネームスペースの作成者アドレス
    namespaceId,
    key,value, //Key-Value値
    alice.address //メタデータの登録者
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [tx.toAggregate(alice.publicAccount)],
    networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// ターゲットと作成者アドレスの設定
namespaceIds = sdkSymbol.generateNamespacePath("xembook");  // メタデータ記録先ネームスペース
targetNamespace = namespaceIds[namespaceIds.length - 1];
namespaceInfo = await fetch(
  new URL('/namespaces/' + targetNamespace.toString(16).toUpperCase(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
targetAddress = new sdkSymbol.Address(sdkCore.utils.hexToUint8(namespaceInfo.namespace.ownerAddress));  // ネームスペース作成者アドレス

// キーと値の設定
key = sdkSymbol.metadataGenerateKey("key_namespace");
value = new TextEncoder().encode("test");

// 同じキーのメタデータが登録されているか確認
query = new URLSearchParams({
  "targetId": targetNamespace.toString(16).toUpperCase(),
  "sourceAddress": targetAddress.toString(),
  "scopedMetadataKey": key.toString(16).toUpperCase(),
  "metadataType": 2
});
metadataInfo = await fetch(
  new URL('/metadata?' + query.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
// 登録済の場合は差分データを作成する
sizeDelta = value.length;
if (metadataInfo.length > 0) {
  sizeDelta -= metadataInfo[0].metadataEntry.valueSize;
  value = sdkSymbol.metadataUpdateValue(sdkCore.utils.hexToUint8(metadataInfo[0].metadataEntry.value, value));
}

// ネームスペースメタデータ登録Tx作成
descriptor = new sdkSymbol.descriptors.NamespaceMetadataTransactionV1Descriptor(  // Txタイプ:アカウントメタデータ登録Tx
  targetAddress,  // ターゲットアドレス
  key,            // キー
  new sdkSymbol.models.NamespaceId(targetNamespace),   // メタデータ記録先モザイク
  sizeDelta,      // サイズ差分
  value           // 値
);
tx = facade.createEmbeddedTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
);

embeddedTransactions = [
  tx
];

// アグリゲートTx作成
aggregateDescriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV2Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  aggregateDescriptor,  // トランザクション Descriptor 設定
  alice.publicKey,      // 署名者公開鍵
  100,                  // 手数料乗数
  60 * 60 * 2,          // Deadline:有効期限(秒単位)
  0                     // 連署者数
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

## 7.4 確認
登録したメタデータを確認します。

#### v2

```js
res = await metaRepo.search({
  targetAddress:alice.address,
  sourceAddress:alice.address}
).toPromise();
console.log(res);
```
###### 出力例
```js
data: Array(3)
  0: Metadata
    id: "62471DD2BF42F221DFD309D9"
    metadataEntry: MetadataEntry
      compositeHash: "617B0F9208753A1080F93C1CEE1A35ED740603CE7CFC21FBAE3859B7707A9063"
      metadataType: 0
      scopedMetadataKey: UInt64 {lower: 92350423, higher: 2540877595}
      sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetId: undefined
      value: "test"
  1: Metadata
    id: "62471F87BF42F221DFD30CC8"
    metadataEntry: MetadataEntry
      compositeHash: "D9E2019D7BD5BA58245320392A68B51752E35A35DA349B08E141DCE99AC3655A"
      metadataType: 1
      scopedMetadataKey: UInt64 {lower: 1789141730, higher: 3475078673}
      sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetId: MosaicId
      id: Id {lower: 1360892257, higher: 309702839}
      value: "test"
  3: Metadata
    id: "62616372BF42F221DF00A88C"
    metadataEntry: MetadataEntry
      compositeHash: "D8E597C7B491BF7F9990367C1798B5C993E1D893222F6FC199F98915339D92D5"
      metadataType: 2
      scopedMetadataKey: UInt64 {lower: 141807833, higher: 2339015223}
      sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetId: NamespaceId
      id: Id {lower: 646738821, higher: 2754876907}
      value: "test"
```

#### v3

```js
query = new URLSearchParams({
  "targetAddress": alice.address,
  "sourceAddress": alice.address,
});
res = await fetch(
  new URL('/metadata?' + query.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
console.log(res);
```
###### 出力例
```js
> {data: Array(3), pagination: {…}}
  data: Array(3)
  > 0: 
      id: "64B918B76FFE587B6D3625C9"
    > metadataEntry: 
        compositeHash: "D2C53CF2F4601F9BD367C2BB4B15CD250D6E21EF59BE99B9D368C25B8A0CF1E1"
        metadataType: 0
        scopedMetadataKey: "9772B71B058127D7"
        sourceAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
        targetAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
        targetId: "0000000000000000"
        value: "74657374"
        valueSize: 4
        version: 1
  > 1: 
      id: "64B9191C6FFE587B6D36266A"
    > metadataEntry: 
        compositeHash: "6B95280F23224CA0A8D02C586C8E0E5043DC339EA6F3EC9EB3CBEBB12D8C5971"
        metadataType: 1
        scopedMetadataKey: "CF217E116AA422E2"
        sourceAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
        targetAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
        targetId: "3D86CF283C1C547D"
        value: "74657374"
        valueSize: 4
        version: 1
  > 2: 
      id: "64B919576FFE587B6D3626D5"
    > metadataEntry: 
        compositeHash: "C92EF86FA799EF32364164584BCFB66A0A874C70C7A8495FA869A8C2E936B99A"
        metadataType: 2
        scopedMetadataKey: "8B6A8A370873D0D9"
        sourceAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
        targetAddress: "982B2AA2295B5C23528ADDEE7F29F6521944E9F2340428AB"
        targetId: "D9D35A75833F4C53"
        value: "74657374"
        valueSize: 4
        version: 1
```

metadataTypeは以下の通りです。
```js
sym.MetadataType
{0: 'Account', 1: 'Mosaic', 2: 'Namespace'}
```

### 注意事項
メタデータはキー値で素早く情報にアクセスできるというメリットがある一方で更新可能であることに注意しておく必要があります。
更新には、発行者アカウントと登録先アカウントの署名が必要のため、それらのアカウントの管理状態が信用できる場合のみ使用するようにしてください。


## 7.5 現場で使えるヒント

### 有資格証明

モザイクの章で所有証明、ネームスペースの章でドメインリンクの説明をしました。
実社会で信頼性の高いドメインからリンクされたアカウントが発行したメタデータの付与を受けることで
そのドメイン内での有資格情報の所有を証明することができます。

#### DID

分散型アイデンティティと呼ばれます。
エコシステムは発行者、所有者、検証者に分かれ、例えば大学が発行した卒業証書を学生が所有し、
企業は学生から提示された証明書を大学が公表している公開鍵をもとに検証します。
このやりとりにプラットフォームに依存する情報はありません。
メタデータを活用することで、大学は学生の所有するアカウントにメタデータを発行することができ、
企業は大学の公開鍵と学生のモザイク(アカウント)所有証明でメタデータに記載された卒業証明を検証することができます。


