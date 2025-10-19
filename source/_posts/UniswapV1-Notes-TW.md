---
title: 深入解析 Uniswap V1：開啟 AMM 時代的 DeFi 基石
date: 2025-10-19 16:21:05
tags:
  - Uniswap
  - AMM
  - Solidity
  - ERC20
  - Liquidity Pool
categories:
  - DeFi
  - Blockchain
  - Smart Contracts
---

![uniswap](UniswapV1-Notes-TW/uniswap.jpg)

#### 前言：告別訂單簿 (Order Book)

在區塊鏈的世界裡，去中心化交易所 (DEX) 一直是個難題。大多數傳統交易所，無論中心化與否，都依賴「訂單簿」(Order Book) 來媒合買賣雙方。但這種方式在鏈上 (On-chain) 執行不僅速度慢，而且 Gas 消耗極高。

Uniswap V1 的誕生徹底改變了這個賽局。它不使用訂單簿，而是引入了「自動做市商」(Automated Market Maker, AMM) 機制，並持有一系列的流動性儲備（Liquidity Reserves）。所有交易都直接針對這些儲備（資金池）來執行，使其效率極高。

這一切的核心，就是那個簡潔而優雅的公式：**$x * y = k$**（恆定乘積做市商機制）。

#### Uniswap V1 的核心架構

##### 1. Factory 合約：為萬物創建交易所

Uniswap V1 的架構中有一個關鍵角色：工廠合約 (`uniswap_factory.vy`) 。這個合約同時扮演了「工廠」(Factory) 和「註冊處」(Registry) 的角色。

它的核心功能是 `createExchange()` 。任何人都可以為一個尚未有交易所的 ERC20 代幣，呼叫這個函數來部署一個全新的、獨立的「交易合約」(Exchange Contract) 。

這個工廠合約會記錄所有已創建的代幣和交易所地址，並提供 `getExchange()`（用代幣地址查交易所地址）和 `getToken()`（用交易所地址查代幣地址）這樣的查詢功能。

**需要注意的是：** Factory 合約在創建交易所時，並不會對代幣本身進行任何檢查（例如驗證其是否合法或有價值）。它只確保每個 ERC20 代幣只能有一個對應的交易所合約。因此，用戶在使用時必須自行審核，只與自己信任的代幣進行互動。

##### 2. Gas 效率：無可比擬的優勢

Uniswap V1 的設計極其注重 Gas 效率。根據白皮書的基準測試：

- ETH 到 ERC20 的交易，Gas 消耗比 Bancor 少了近 10 倍。
- ERC20 到 ERC20 的交易，效率也高於 0x。
- 相較於 EtherDelta 或 IDEX 這些鏈上訂單簿交易所，Gas 費用也明顯更低。

#### 交易機制：x\*y=k 如何運作？

在 Uniswap V1 中，每個交易合約都只綁定「一種」ERC20 代幣，並同時持有 ETH 和該 ERC20 代幣的流動性池。

匯率（價格）不是由訂單決定的，而是由池中兩種資產的「相對規模」決定的。
機制會始終維持 $eth\_pool * token\_pool = k$ 這個不變量 (Invariant) 。這個 $k$ 值只會在流動性被增加或移除時才會改變。

##### 1. ETH $\leftrightarrow$ ERC20 交易

**以 ETH 換 OMG 為例（ETH $\rightarrow$ Token）：**

假設一個資金池最開始由流動性提供者 (LP) 存入了 10 ETH 和 500 OMG。

1. **初始狀態：**

   - `ETH_pool` = 10
   - `OMG_pool` = 500
   - `Invariant (k)` = $10 * 500 = 5000$

2. **交易發生：**

   - 一位買家發送了 **1 ETH** 到合約
   - 合約會先收取 0.3% 的手續費（_範例中使用 0.25%_），即 0.0025 ETH
   - 剩下的 0.9975 ETH 加入 ETH 池中
   - `ETH_pool` (新) = $10 + 0.9975 = 10.9975$

3. **計算兌換：**

   - 為了維持 $k=5000$，`OMG_pool` 的新數量必須是：
   - `OMG_pool` (新) = $k / ETH\_pool(\text{新}) = 5000 / 10.9975 = 454.65$ OMG
   - 買家收到的 OMG = `OMG_pool` (舊) - `OMG_pool` (新) = $500 - 454.65 = 45.35$ OMG

4. **手續費去向：**

   - 交易完成後，之前收取的 0.0025 ETH 手續費會被「加回」流動性池中
   - `ETH_pool` (最終) = $10.9975 + 0.0025 = 11$
   - `OMG_pool` (最終) = 454.65
   - **新的 Invariant (k)** = $11 * 454.65 = 5001.15$

你會發現 $k$ 值略微增加了！這就是 LP (流動性提供者) 的利潤來源。

**反向的 Token $\rightarrow$ ETH 交易** (`tokenToEthSwap`) 也是同理，代幣池增加，ETH 池按比例減少，價格隨之變動。

##### 2. ERC20 $\leftrightarrow$ ERC20 交易

Uniswap V1 不支援 ERC20 之間的「直接」兌換。但由於 ETH 是所有 ERC20 代幣的通用交易對，因此可以把 ETH 當作中介。

例如，要將 OMG 換成 KNC：

1. 買家在 **OMG 交易所**上呼叫 `tokenToTokenSwap()`
2. OMG 交易所會將買家的 OMG 兌換成 ETH
3. OMG 交易所**並不會**把 ETH 還給買家，而是拿著這筆 ETH，去呼叫 **KNC 交易所**的 `ethToTokenTransfer()` 函數
4. KNC 交易所收到 ETH，將其兌換成 KNC，然後將 KNC 直接發送給「原始買家」的地址

這整個過程在同一筆交易中完成，用戶體驗上就像是直接用 OMG 換到了 KNC。

#### 流動性提供者 (LP) 的世界

Uniswap 的資金池並非憑空而來，它們依賴「流動性提供者」(Liquidity Providers, LP) 提供資金。

##### 1. 增加與移除流動性

- **增加流動性 (Add Liquidity)：**
  - 任何人都可以呼叫 `addLiquidity()`
  - 提供流動性時，必須存入「等值」的 ETH 和 ERC20 Token
  - 第一個 LP 負責設定池子的初始匯率。如果匯率設定不合理，套利者會立刻介入，使價格回歸市場平衡。
- **流動性代幣 (Liquidity Tokens)：**
  - 當 LP 存入資金時，合約會鑄造 (Mint) 一種「流動性代幣」(LP Token) 給他們。
  - LP Token 本身也是一種 ERC20 代幣，它代表了該 LP 在總儲備中所佔的份額。
- **移除流動性 (Removing Liquidity)：**
  - LP 可以隨時銷毀 (Burn) 他們的 LP Token。
  - 合約會根據他們銷毀的 LP Token 比例，將池中相應份額的 ETH 和 ERC20 Token 退還給他們。

##### 2. LP 的收益：交易手續費

LP 為什麼要提供流動性？因為他們可以分享交易手續費。

- **ETH $\leftrightarrow$ ERC20 交易：** 支付 0.3% 的手續費（ETH 支付 ETH，ERC20 支付 ERC20）
- **ERC20 $\leftrightarrow$ ERC20 交易：** 總共會被收兩次費用（一次在輸入交易所，一次在輸出交易所），總費用約為 0.5991%

這些手續費會**立即存入流動性儲備中** 。由於總儲備增加了，但 LP Token 總量不變，這意味著每單位 LP Token 的價值都增加了。這就是 LP 的利潤。

##### 3. LP 的風險：無常損失 (Impermanent Loss)

- **無常損失 (IL) **：指的是當你存入資金池的代幣，其價格比率相較於存入時發生變化，所導致的價值損失。
- **舉例：**
  - 你存入 10 ETH 和 1000 TokenA (此時 1 ETH = 100 TokenA)
  - 此時 $k = 10 * 1000 = 10000$
  - 一段時間後，市場價格變為 1 ETH = 200 TokenA
  - 套利者會介入，直到池中的比例也變為 1:200。此時池中會變為 7.071 ETH 和 1414.21 TokenA (因為 $x*y=10000$ 且 $x=200y$)
  - **如果你現在贖回：** 你會得到 7.071 ETH 和 1414.21 TokenA。以 TokenA 計價，總價值為 $(7.071 * 200) + 1414.21 = 2828.41$ TokenA
  - **如果你當初沒存入 (HODL)：** 你會擁有 10 ETH 和 1000 TokenA。以 TokenA 計價，總價值為 $(10 * 200) + 1000 = 3000$ TokenA
  - **損失：** $3000 - 2828.41 = 171.59$ TokenA，這就是無常損失

LP 獲得的交易手續費，本質上就是為了補償他們承擔這種價格波動（無常損失）的風險。

#### 結語

Uniswap V1 以極簡的設計、卓越的 Gas 效率和革命性的 AMM 理念，為 DeFi 世界奠定了不可或缺的基石。
它雖然存在一些問題（例如必須以 ETH 為中介、無常損失），但也為後來的 V2 和 V3 版本指明了演進的方向。

---
