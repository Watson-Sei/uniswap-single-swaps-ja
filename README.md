# Single Swaps （シングルスワップ）

スワップは、Uniswapプロトコルで最も一般的なインタラクションです。次の例では、作成した2つの関数を使用するシングルパスのスワップ契約を実装する方法を示しています。

- `swapExactInputSingle`
- `swapExactOutputSingle`

`swapExactInputSingle`関数は、あるトークンの固定量を別のトークンの最大量と交換する、正確な入力スワップを実行するための関数です。この関数は、[ISwapRouter](https://docs.uniswap.org/protocol/reference/periphery/interfaces/ISwapRouter) インターフェースの `ExactInputSingleParams`構造体と `exactInputSingle`関数を使用します。

`swapExactOutputSingle`関数は、あるトークンの最小量を別のトークンの固定量と交換する、正確な出力スワップを実行するための関数です。この関数は、`ExactOutputSingleParams`構造体と[ISwapRouter](https://docs.uniswap.org/protocol/reference/periphery/interfaces/ISwapRouter)インターフェースの`exactOutputSingle`関数を使用します。

簡略化のため、この例ではトークンコントラクトのアドレスをハードコードしていますが、以下に説明するように、契約を変更して、トランザクションごとにプールやトークンを変更することも可能です。

スマートコントラクトから取引を行う場合、最も注意しなければならないのは、外部の価格ソースへのアクセスが必要なことです。これがないと、取引の際にかなりの損失を被る可能性があります。

**注**：スワップ例は本番環境に対応したコードではなく、学習を目的とした単純な方法で実装されています。

## コントラクトの作成

コントラクトのコンパイルに使用されたsolidityのバージョンを宣言し、スワップを実行する際に使用される機能である、任意のネストされた配列や構造体をcalldataでエンコードおよびデコードできるようにするための`abicoder v2`を宣言します。

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity =0.7.6;
pragma abicoder v2;
```

npmパッケージのインストールから、関連する2つのコントラクトをインポートします。

```solidity
import '@uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol';
import '@uniswap/v3-periphery/contracts/libraries/TransferHelper.sol';
```

`SwapExamples`というコントラクトを作成し、`ISwapRouter`型の不変のパブリック変数`swapRouter`を宣言します。これにより、`ISwapRouter`インターフェイスの関数を呼び出すことができます。

```solidity
contract SwapExamples {
    // 今回のスワップの例では、
    // `exactInput`、`exactInputSingle`、`exactOutput`、`exactOutputSingle`を使用する際の設計上の注意点を詳しく説明します。
    // これらの例では、スワップルーターを継承するのではなく、コンストラクタの引数として渡していることに注意してください。
    // より高度なコントラクト例では、スワップルーターを安全に継承する方法を詳しく説明します。
    // この例では、シングルパスのスワップにはDAI/WETH9を、マルチパスのスワップにはDAI/USDC/WETH9を入れ替えています。

    ISwapRouter public immutable swapRouter;
```

例題のトークンコントラクトのアドレスとプールの料金階層をハードコードします。本番環境では、入力パラメータを使用し、入力をメモリ変数に渡して、コントラクトがトランザクションごとにやり取りするプールとトークンを変更できるようにすることが考えられますが、概念的にシンプルにするために、ここではハードコーディングしています。

```solidity
    address public constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address public constant WETH9 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    // 今回の例では、プール料金を0.3％に設定します。
    uint24 public constant poolFee = 3000;

    constructor(ISwapRouter _swapRouter) {
        swapRouter = _swapRouter;
    }
```

## 正確なインプットスワップ

呼び出し側は、スワップを実行するために呼び出し側のアドレスのアカウントからトークンを引き出すコントラクトを `承認(approve)` しなければなりません。私たちのコントラクトはコントラクトそのものであり、呼び出し元（私たち）の延長ではないため、呼び出し先アドレス（私たち）から引き出された後に私たちのコントラクトが所有することになるトークンを使用するために、Uniswapプロトコルのルーターコントラクトを承認しなければならないことを覚えておいてください。

そして、Daiの`amount`を呼び出し元のアドレスから私たちのコントラクトに転送し、`amount`を2つ目の`approve`に渡される値として使用します。

```solidity
    /// @注意swapExactInputSingleは、固定量のDAIを最大量のWETH9にスワップします。
    /// スワップルーターで `exactInputSingle` を呼び出すことで、DAI/WETH9 0.3%プールを使用して、固定量のDAIと最大量のWETH9を交換します。
    /// @dev この関数が成功するためには、呼び出し元のアドレスがこのコントラクトを承認して、少なくとも`amountIn`相当のDAIを費やす必要があります。
    /// @param amountIn WETH9とスワップされるDAIの正確な量です。
    /// @return amountOut 受信したWETH9の量です。
    function swapExactInputSingle(uint256 amountIn) external returns (uint256 amountOut) {
        // msg.senderはこのコントラクトを承認する必要があります。

        // 指定した量のDAIをこのコントラクトに転送します。
        TransferHelper.safeTransferFrom(DAI, msg.sender, address(this), amountIn);

        // DAIを使うためのルーターを承認する。
        TransferHelper.safeApprove(DAI, address(swapRouter), amountIn);
```

### 入力パラメータの交換

スワップ機能を実行するには、`ExactInputSingleParams`に必要なスワップデータを入力する必要があります。これらのパラメータは、スマートコントラクトのインターフェースに記載されており、[こちら](https://docs.uniswap.org/protocol/reference/periphery/interfaces/ISwapRouter)から閲覧することができます。

パラメータの概要:
- `tokenIn` 受信したトークンのコントラクトアドレス
- `tokenOut` 発信するトークンのコントラクトアドレス
- `fee` スワップを実行する正しいプールコントラクトを決定するために使用される、プールの手数料階層
- `recipient` 発信トークンの宛先アドレス
- `deadline`: スワップが失敗するまでのUnixの時間は、保留中の取引や価格の乱高下を防ぐためのものです。
- `amountOutMinimum`: 0に設定していますが、これは本番環境での重大なリスクです。 実際の展開では、この値はSDKまたはオンチェーン価格オラクルを使用して計算する必要があります-これは、フロントランニングサンドイッチまたは別のタイプの価格操作が原因で取引に異常に悪い価格が発生するのを防ぐのに役立ちます
- `sqrtPriceLimitX96`: ここでは0に設定し、このパラメータを非アクティブにしています。製品版では、この値を使用して、スワップがプールを押し上げる価格の上限を設定することができます。これにより、価格の影響からの保護や、価格に関連するさまざまなメカニズムのロジックの設定に役立ちます。

### 関数を呼び出す

```solidity
        // 本番環境では、Oracleなどのデータソースを使用して、amountOutMinimumのより安全な値を選択してください。
        // また、sqrtPriceLimitx96を0に設定して、正確な入力量をスワップするようにしています。
        ISwapRouter.ExactInputSingleParams memory params =
            ISwapRouter.ExactInputSingleParams({
                tokenIn: DAI,
                tokenOut: WETH9,
                fee: poolFee,
                recipient: msg.sender,
                deadline: block.timestamp,
                amountIn: amountIn,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

        // `exactInputSingle`の呼び出しでは、スワップを実行します。
        amountOut = swapRouter.exactInputSingle(params);
    }
```

## 正確な出力スワップ

正確な出力は、入力トークンの可能な限り最小の量と出力トークンの固定量を交換します。これはあまり一般的ではないスワップスタイルですが、様々な状況で役立ちます。

この例では、スワップを想定してインバウンド資産を送金しているため、スワップ実行後にインバウンドトークンの一部が残る可能性があり、そのためにスワップ終了時に呼び出しアドレスに返金しています。

### 関数を呼び出す

```solidity
/// @注意 swapExactOutputSingle は、可能な限り少ない量の DAI を一定量の WETH と交換します。
/// @dev この機能が成功するためには、呼び出し元のアドレスがこのコントラクトを承認してDAIを消費する必要があります。入力DAIの量は可変であるため 呼び出し元のアドレスは、多少のばらつきを想定して、少し高めの金額を承認する必要があります。
/// @param amountOut スワップから受け取る正確なWETH9の量。
/// @param amountInMaximum 指定した量のWETH9を受け取るために消費するDAIの量です。
/// @return amountIn スワップで実際に使われたDAIの量です。
function swapExactOutputSingle(uint256 amountOut, uint256 amountInMaximum) external returns (uint256 amountIn) {
// 指定された金額のDAIをこのコントラクトに転送します。.
TransferHelper.safeTransferFrom(DAI, msg.sender, address(this), amountInMaximum);

        // 指定された`amountInMaximum`のDAIを使うためにルータを承認する。
        // 本番では、より良いスワップを実現するために、オラクルや他のデータソースに基づいて、消費する最大量を選択する必要があります。
        TransferHelper.safeApprove(DAI, address(swapRouter), amountInMaximum);

        ISwapRouter.ExactOutputSingleParams memory params =
            ISwapRouter.ExactOutputSingleParams({
                tokenIn: DAI,
                tokenOut: WETH9,
                fee: poolFee,
                recipient: msg.sender,
                deadline: block.timestamp,
                amountOut: amountOut,
                amountInMaximum: amountInMaximum,
                sqrtPriceLimitX96: 0
            });

        // 希望するamountOutを受け取るために必要なamountInを返すスワップを実行します。
        amountIn = swapRouter.exactOutputSingle(params);

        // 正確な出力交換の場合は、AmountInMaximumがすべて使われていない可能性があります。
        // 実際に使用した金額（amountIn）が指定した最大金額よりも少ない場合は、msg.senderに返金し、swapRouterに0を使わせることを承認しなければなりません。
        if (amountIn < amountInMaximum) {
            TransferHelper.safeApprove(DAI, address(swapRouter), 0);
            TransferHelper.safeTransfer(DAI, msg.sender, amountInMaximum - amountIn);
        }
    }
```

## 完全なシングルスワップコントラクト

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity =0.7.6;
pragma abicoder v2;

import '@uniswap/v3-periphery/contracts/libraries/TransferHelper.sol';
import '@uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol';

contract SwapExamples {
    // For the scope of these swap examples,
    // we will detail the design considerations when using
    // `exactInput`, `exactInputSingle`, `exactOutput`, and  `exactOutputSingle`.

    // It should be noted that for the sake of these examples, we purposefully pass in the swap router instead of inherit the swap router for simplicity.
    // More advanced example contracts will detail how to inherit the swap router safely.

    ISwapRouter public immutable swapRouter;

    // This example swaps DAI/WETH9 for single path swaps and DAI/USDC/WETH9 for multi path swaps.

    address public constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address public constant WETH9 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    // For this example, we will set the pool fee to 0.3%.
    uint24 public constant poolFee = 3000;

    constructor(ISwapRouter _swapRouter) {
        swapRouter = _swapRouter;
    }

    /// @notice swapExactInputSingle swaps a fixed amount of DAI for a maximum possible amount of WETH9
    /// using the DAI/WETH9 0.3% pool by calling `exactInputSingle` in the swap router.
    /// @dev The calling address must approve this contract to spend at least `amountIn` worth of its DAI for this function to succeed.
    /// @param amountIn The exact amount of DAI that will be swapped for WETH9.
    /// @return amountOut The amount of WETH9 received.
    function swapExactInputSingle(uint256 amountIn) external returns (uint256 amountOut) {
        // msg.sender must approve this contract

        // Transfer the specified amount of DAI to this contract.
        TransferHelper.safeTransferFrom(DAI, msg.sender, address(this), amountIn);

        // Approve the router to spend DAI.
        TransferHelper.safeApprove(DAI, address(swapRouter), amountIn);

        // Naively set amountOutMinimum to 0. In production, use an oracle or other data source to choose a safer value for amountOutMinimum.
        // We also set the sqrtPriceLimitx96 to be 0 to ensure we swap our exact input amount.
        ISwapRouter.ExactInputSingleParams memory params =
            ISwapRouter.ExactInputSingleParams({
                tokenIn: DAI,
                tokenOut: WETH9,
                fee: poolFee,
                recipient: msg.sender,
                deadline: block.timestamp,
                amountIn: amountIn,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

        // The call to `exactInputSingle` executes the swap.
        amountOut = swapRouter.exactInputSingle(params);
    }

    /// @notice swapExactOutputSingle swaps a minimum possible amount of DAI for a fixed amount of WETH.
    /// @dev The calling address must approve this contract to spend its DAI for this function to succeed. As the amount of input DAI is variable,
    /// the calling address will need to approve for a slightly higher amount, anticipating some variance.
    /// @param amountOut The exact amount of WETH9 to receive from the swap.
    /// @param amountInMaximum The amount of DAI we are willing to spend to receive the specified amount of WETH9.
    /// @return amountIn The amount of DAI actually spent in the swap.
    function swapExactOutputSingle(uint256 amountOut, uint256 amountInMaximum) external returns (uint256 amountIn) {
        // Transfer the specified amount of DAI to this contract.
        TransferHelper.safeTransferFrom(DAI, msg.sender, address(this), amountInMaximum);

        // Approve the router to spend the specifed `amountInMaximum` of DAI.
        // In production, you should choose the maximum amount to spend based on oracles or other data sources to acheive a better swap.
        TransferHelper.safeApprove(DAI, address(swapRouter), amountInMaximum);

        ISwapRouter.ExactOutputSingleParams memory params =
            ISwapRouter.ExactOutputSingleParams({
                tokenIn: DAI,
                tokenOut: WETH9,
                fee: poolFee,
                recipient: msg.sender,
                deadline: block.timestamp,
                amountOut: amountOut,
                amountInMaximum: amountInMaximum,
                sqrtPriceLimitX96: 0
            });

        // Executes the swap returning the amountIn needed to spend to receive the desired amountOut.
        amountIn = swapRouter.exactOutputSingle(params);

        // For exact output swaps, the amountInMaximum may not have all been spent.
        // If the actual amount spent (amountIn) is less than the specified maximum amount, we must refund the msg.sender and approve the swapRouter to spend 0.
        if (amountIn < amountInMaximum) {
            TransferHelper.safeApprove(DAI, address(swapRouter), 0);
            TransferHelper.safeTransfer(DAI, msg.sender, amountInMaximum - amountIn);
        }
    }
}
```
