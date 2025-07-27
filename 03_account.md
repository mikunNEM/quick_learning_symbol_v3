# 3.アカウント

アカウントは秘密鍵に紐づく情報が記録されたデータ構造体です。アカウントと関連づいた秘密鍵を使って署名することでのみブロックチェーンのデータを更新することができます。  

## 3.1 アカウント生成

アカウントには秘密鍵と公開鍵をセットにしたキーペア、アドレスなどの情報が含まれています。まずはランダムにアカウントを作成して、それらの情報を確認してみましょう。  

### 新規生成

#### v3

```js
alice = facade.createAccount(sdkCore.PrivateKey.random());
console.log(alice);
```

###### 出力例

```js
> SymbolAccount
  > address: Address
      bytes: Uint8Array(24)
  > keyPair: KeyPair
    > publicKey: PublicKey
        bytes: Uint8Array(32)
    > privateKey: PrivateKey
        bytes: Uint8Array(32)
  > publicKey: PublicKey
      bytes: Uint8Array(32)
  > _facade: SymbolFacade
      network: Network
      transactionFactory: TransactionFactory
```

<details><summary> SymbolAccount クラス実装前 (symbol-sdk v3.2.1 まで)</summary>

v3.2.1 までは `SymbolAccount` クラスが無いため、秘密鍵・公開鍵の鍵セットとアドレスを別々に作成する必要ありました。
v3.2.2 以降も別々に作成することはできますが、データを別々に管理する手間やリスクを考慮し、以降は `SymbolAccount` クラスを利用してコードを記述します。

```js
aliceKey = new sdkSymbol.KeyPair(sdkCore.PrivateKey.random());
console.log(aliceKey);
aliceAddress = facade.network.publicKeyToAddress(aliceKey.publicKey);
console.log(aliceAddress);
```

###### 出力例

```js
> KeyPair {_privateKey: PrivateKey, _keyPair: {…}}
    _keyPair: {publicKey: Uint8Array(32), privateKey: Uint8Array(32)}
    _privateKey: PrivateKey {bytes: Uint8Array(32)}
> Address {bytes: Uint8Array(24)}
```

</details>

### 秘密鍵と公開鍵の導出

#### v3

```js
console.log(alice.keyPair.privateKey.toString());
console.log(alice.publicKey.toString());
```

```
> 1E9139CC1580B4AED6A1FE110085281D4982ED0D89CE07F3380EB83069B1****
> D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2
```

#### 注意事項
秘密鍵を紛失するとそのアカウントに紐づけられたデータを操作することが出来なくなります。また、他人は知らないという秘密鍵の性質を利用してデータ操作の署名を行うので、秘密鍵を他人に教えてはいけません。組織のなかで秘密鍵を譲り受けて運用を続けるといった行為も控えましょう。
一般的なWebサービスでは「アカウントID」に対してパスワードが割り振られるため、パスワードの変更が可能ですが、ブロックチェーンではパスワードにあたる秘密鍵に対して一意に決まるID(アドレス)が割り振られるため、アカウントに紐づく秘密鍵を変更するということはできません。  


### アドレスの導出

#### v3

```js
aliceRawAddress = alice.address.toString();
console.log(aliceRawAddress);
```

```js
> TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ
```

これらがブロックチェーンを操作するための最も基本的な情報となります。また、秘密鍵からアカウントを生成したり、公開鍵やアドレスのみを扱うクラスの生成方法も確認しておきましょう。  

### 秘密鍵からアカウント生成

#### v3

```js
alice = facade.createAccount(new sdkCore.PrivateKey("1E9139CC1580B4AED6A1FE110085281D4982ED0D89CE07F3380EB83069B1****"));
```

### 公開鍵クラスの生成

#### v3

```js
alicePublicAccount = facade.createPublicAccount(new sdkCore.PublicKey("D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2"));
console.log(alicePublicAccount);
```

###### 出力例
```js
> SymbolPublicAccount
  > address: Address
      bytes: Uint8Array(24)
  > publicKey: PublicKey
      bytes: Uint8Array(32)
  > _facade: SymbolFacade
      network: Network
      transactionFactory: TransactionFactory
```

### アドレスクラスの生成

#### v3

```js
aliceAddress = new sdkSymbol.Address("TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ");
console.log(aliceAddress);
console.log(aliceAddress.toString());
```
###### 出力例
```js
> Address {bytes: Uint8Array(24)}
> TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ
```

## 3.2 アカウントへの送信

アカウントを作成しただけでは、ブロックチェーンにデータを送信することはできません。  
パブリックブロックチェーンはリソースを有効活用するためにデータ送信時に手数料を要求します。  
Symbolブロックチェーンでは、この手数料をXYMという共通トークンで支払うことになります。  
アカウントを生成したら、この後の章から説明するトランザクションを実行するために必要な手数料を送信しておきます。  

### フォーセットから送信

テストネットではフォーセット（蛇口）サービスから検証用のXYMを入手することができます。  
メインネットの場合は取引所などでXYMを購入するか、投げ銭サービス(QUEST)などを利用して寄付を募りましょう。  

テストネット
- FAUCET(蛇口)
  - https://testnet.symbol.tools/

メインネット
- QUEST
  - https://quest-bc.com/



### エクスプローラーで確認

フォーセットから作成したアカウントへ送信が成功したらエクスプローラーで確認してみましょう。

- テストネット
  - https://testnet.symbol.fyi/
- メインネット
  - https://symbol.fyi/

## 3.3 アカウント情報の確認

ノードに保存されているアカウント情報を取得します。

### 所有モザイク一覧の取得

#### v3

```js
accountInfo = await fetch(
  new URL('/accounts/' + aliceAddress.toString(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.account;
});
console.log(accountInfo);
```

###### 出力例

```js
> {version: 1, address: '986F4982FE77894ABC3EBFDC16DFD4A5C2C7BC05BFD44ECE', addressHeight: '52812', publicKey: '0000000000000000000000000000000000000000000000000000000000000000', publicKeyHeight: '0', …}
    address: "986F4982FE77894ABC3EBFDC16DFD4A5C2C7BC05BFD44ECE"
    publicKey: "0000000000000000000000000000000000000000000000000000000000000000"
  > mosaics: Array(1)
      0:
        amount: "3510000000"
        id: "72C0212E67A08BCE"
```

#### publicKey
クライアント側で作成しただけで、ブロックチェーンでまだ利用されていないアカウント情報は記録されていません。宛先として指定されて受信することで初めてアカウント情報が記録され、署名したトランザクションを送信することで公開鍵の情報が記録されます。そのため、publicKeyは現在`00000...`表記となっています。

#### UInt64 ※v2 のみ

JavaScriptでは大きすぎる数値はあふれてしまうため、idやamountはUInt64というsdkの独自フォーマットで管理されています。文字列に変換する場合は toString()、数値に変換する場合は compact()、16進数にする場合は toHex() で変換してください。


#### v3

v3 では UInt64 は定義されておらず、大きすぎる数値を表現するために JavaScript の `BigInt` が使用されています。
以降の章で登場するため、ここで構文を紹介します。

```js
12345n;           // 数値の後ろに n を付ける
BigInt(12345);    // BigInt(数値)
0x12345n;         // 16進数も同様
BigInt(0x12345);
```

#### 表示桁数の調整

所有するトークンの量は誤差の発生を防ぐため、整数値で扱います。トークンの定義から可分性を取得することができるので、その値を使って正確な所有量を表示してみます。  


#### v3

```js
mosaicAmount = accountInfo.mosaics[0].amount;
mosaicInfo = await fetch(
  new URL('/mosaics/' + accountInfo.mosaics[0].id, NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.mosaic;
});

divisibility = mosaicInfo.divisibility; //可分性
if(divisibility > 0){
  displayAmount = mosaicAmount.slice(0,mosaicAmount.length-divisibility)  
  + "." + mosaicAmount.slice(-divisibility);
}else{
  displayAmount = mosaicAmount;
}
console.log(displayAmount);
```

## 3.4 現場で使えるヒント
### 暗号化と署名

アカウントとして生成した秘密鍵や公開鍵は、そのまま従来の暗号化や電子署名として活用することができます。信頼性に問題点があるアプリケーションを使用する必要がある場合も、個人間（エンドツーエンド）でデータの秘匿性・正当性を検証することができます。  

#### 事前準備：対話のためのBobアカウントを生成

#### v3

```js
bob = facade.createAccount(sdkCore.PrivateKey.random());
```

#### 暗号化

Aliceの秘密鍵・Bobの公開鍵で暗号化し、Aliceの公開鍵・Bobの秘密鍵で復号します（AES-GCM形式）。


#### v3

```js
message = 'Hello Symbol!';
encryptedMessage = alice.messageEncoder().encode(bob.publicKey, new TextEncoder().encode(message));
console.log(Buffer.from(encryptedMessage).toString("hex").toUpperCase());
```

```js
> 0167AF68C3E7EFBD7048F6E9140FAA14256B64DD19FD0708EDCF17758A81FCC00084D869D6F1434A77AF
```

#### 復号化


#### v3

```js
decryptMessageData = bob.messageEncoder().tryDecode(alice.publicKey, Uint8Array.from(Buffer.from("0167AF68C3E7EFBD7048F6E9140FAA14256B64DD19FD0708EDCF17758A81FCC00084D869D6F1434A77AF", "hex"))); // 暗号化時のデータに置き換えてください
console.log(decryptMessageData);
if (decryptMessageData.isDecoded) {
  decryptMessage = new TextDecoder().decode(decryptMessageData.message);
  console.log(decryptMessage);
} else {
  console.log("decrypt failed!");
}
```

```js
> {isDecoded: true, message: Uint8Array(13)}
> "Hello Symbol!"
```

<details><summary>symbol-sdk v3.0.7 での注意点</summary>

注意：
v3.0.7 では復号化データの構造が異なります。
v3.0.8 以降では、結果とメッセージを持つ **オブジェクト** ですが、v3.0.7 では結果とメッセージの **配列** です。
このため、復号化したメッセージへのアクセス方法が異なります。

##### v3.0.7

```js
if (decryptMessageData[0]) {
  decryptMessage = new TextDecoder().decode(decryptMessageData[1]);
  console.log(decryptMessage);
} else {
  console.log("decrypt failed!");
}
```

```js
> [true, Uint8Array(13)]
> "Hello Symbol!"
```

</details>

#### 署名

Aliceの秘密鍵でメッセージを署名し、Aliceの公開鍵と署名でメッセージを検証します。


#### v3

```js
payload = Buffer.from("Hello Symbol!", 'utf-8');
signature = alice.keyPair.sign(payload);
console.log(signature.toString());
```

```
> B8A9BCDE9246BB5780A8DED0F4D5DFC80020BBB7360B863EC1F9C62CAFA8686049F39A9F403CB4E66104754A6AEDEF8F6B4AC79E9416DEEDC176FDD24AFEC60E
```

#### 検証


#### v3

```js
v = new sdkSymbol.Verifier(alice.publicKey);
isVerified = v.verify(Buffer.from("Hello Symbol!", 'utf-8'), signature);
console.log(isVerified);
```

```js
> true
```

ブロックチェーンを使用しない署名は何度も再利用される可能性があることにご注意ください。

### アカウントの保管

アカウントの管理方法について説明しておきます。  
秘密鍵はそのままで保存しないようにしてください。symbol-qr-libraryを利用して秘密鍵をパスフレーズで暗号化して保存する方法を紹介します。  

#### 秘密鍵の暗号化

```js
qr = require("/node_modules/symbol-qr-library");

//パスフレーズでロックされたアカウント生成
signerQR = qr.QRCodeGenerator.createExportAccount(
  alice.privateKey, networkType, generationHash, "パスフレーズ"
);

//QRコード表示
signerQR.toBase64().subscribe(x =>{

  //HTML body上にQRコードを表示する例
  (tag= document.createElement('img')).src = x;
  document.getElementsByTagName('body')[0].appendChild(tag);
});

//アカウントを暗号化したJSONデータとして表示
jsonSignerQR = signerQR.toJSON();
console.log(jsonSignerQR);
```
###### 出力例
```js
> {"v":3,"type":2,"network_id":152,"chain_id":"7FCCD304802016BEBBCD342A332F91FF1F3BB5E902988B352697BE245F48E836","data":{"ciphertext":"e9e2f76cb482fd054bc13b7ca7c9d086E7VxeGS/N8n1WGTc5MwshNMxUiOpSV2CNagtc6dDZ7rVZcnHXrrESS06CtDTLdD7qrNZEZAi166ucDUgk4Yst0P/XJfesCpXRxlzzNgcK8Q=","salt":"54de9318a44cc8990e01baba1bcb92fa111d5bcc0b02ffc6544d2816989dc0e9"}}
```
このjsonSignerQRで出力されるQRコード、あるいはテキストを保存しておけばいつでも秘密鍵を復元することができます。

#### 暗号化された秘密鍵の復号

```js
//保存しておいたテキスト、あるいはQRコードスキャンで得られたテキストをjsonSignerQRに代入
jsonSignerQR = '{"v":3,"type":2,"network_id":152,"chain_id":"7FCCD304802016BEBBCD342A332F91FF1F3BB5E902988B352697BE245F48E836","data":{"ciphertext":"e9e2f76cb482fd054bc13b7ca7c9d086E7VxeGS/N8n1WGTc5MwshNMxUiOpSV2CNagtc6dDZ7rVZcnHXrrESS06CtDTLdD7qrNZEZAi166ucDUgk4Yst0P/XJfesCpXRxlzzNgcK8Q=","salt":"54de9318a44cc8990e01baba1bcb92fa111d5bcc0b02ffc6544d2816989dc0e9"}}';

qr = require("/node_modules/symbol-qr-library");
signerQR = qr.AccountQR.fromJSON(jsonSignerQR,"パスフレーズ");
console.log(signerQR.accountPrivateKey);
```
###### 出力例
```js
> 1E9139CC1580B4AED6A1FE110085281D4982ED0D89CE07F3380EB83069B1****
```

