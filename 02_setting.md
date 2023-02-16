# 2.建立開發環境

這個章節說明如何閱讀本文件。

## 2.1 語言

所有的程式碼都將使用 JavaScript 撰寫。

### SDK-軟體開發套件

symbol-sdk-typescript-javascript v2.0.0  
https://github.com/symbol/symbol-sdk-typescript-javascript

在瀏覽器的開發人員控制台中，將上面的 SDK 作為 Browserify 載入。
https://github.com/xembook/nem2-browserify

##### 注意事項

目前 symbol-sdk 的 alpha 版本是 3.0.0，2.0.3 版本已不再支援。
版本 3 移除了許多 rxjs 依賴的功能，因此建議直接存取 REST API。

### 參考文獻

Symbol SDK 支援 TypeScript 和 JavaScript
https://symbol.github.io/symbol-sdk-typescript-javascript/1.0.3/

Catapult REST端點 (1.0.3)
https://symbol.github.io/symbol-openapi/v1.0.3/

## 2.2 範例原始碼

### 變數宣告

本文件中，我們沒有宣告 const，是希望您在控制台上重複編寫它以驗證其是否正常運作。當開發應用程序時，請通過聲明 const 確保安全性。

### 檢查輸出值

Console.log() 會輸出該變數的內容。依照個人偏好，可以試著使用輸出函數。輸出結果會在 > 下方。在練習範例時，可以不包含這部分的內容。

### 同步和非同步

有些習慣於其他語言的開發者可能會因為不習慣寫非同步處理而感到不安，因此除非有特殊需求，本文中的解釋都是不使用非同步處理的。

### 帳戶

#### Alice

This manual focuses on the Alice account. We will continue to use the Alice account created in chapter 3 in subsequent chapters. Please go on reading this manual with sufficient XYM sent.

#### Bob

Bob is created as an account for transacting with Alice, as required in the chapters. Others, such as Carol, are used in the multisig chapters.

### Fee

In this document, transactions are created with a transaction fee multiplier of 100.

## 2.3 Preparations in advance

From the node list, open the page of any node with e.g. Chrome browser. This manual is based on the assumption of a testnet.

- Testnet
  - https://symbolnodes.org/nodes_testnet/
- Mainnet
  - https://symbolnodes.org/nodes/

Press F12 to open the developer console, and enter the following script.

```js
(script = document.createElement("script")).src =
  "https://xembook.github.io/nem2-browserify/symbol-sdk-pack-2.0.0.js";
document.getElementsByTagName("head")[0].appendChild(script);
```

Then, run the common logic that is used in almost all chapters.

```js
NODE = window.origin; //The URL of the page is shown here.
sym = require("/node_modules/symbol-sdk");
repo = new sym.RepositoryFactoryHttp(NODE);
txRepo = repo.createTransactionRepository();
(async () => {
  networkType = await repo.getNetworkType().toPromise();
  generationHash = await repo.getGenerationHash().toPromise();
  epochAdjustment = await repo.getEpochAdjustment().toPromise();
})();
function clog(signedTx) {
  console.log(NODE + "/transactionStatus/" + signedTx.hash);
  console.log(NODE + "/transactions/confirmed/" + signedTx.hash);
  console.log("https://symbol.fyi/transactions/" + signedTx.hash);
  console.log("https://testnet.symbol.fyi/transactions/" + signedTx.hash);
}
```

You are now ready to go.
If the content of this manual is a little confusing, please refer to the Qiita article.

[Symbol ブロックチェーンのテストネットで送金を体験する](https://qiita.com/nem_takanobu/items/e2b1f0aafe7a2df0fe1b)
