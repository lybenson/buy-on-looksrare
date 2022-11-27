<template>
  <h1 @click="handleBuy">Buy</h1>
</template>
<script setup>
import { ref } from 'vue'
import axios from 'axios'
import { ethers, BigNumber } from 'ethers'
import { WETHAbi, LooksRareExchangeAbi, addressesByNetwork, SupportedChainId, encodeOrderParams } from "@looksrare/sdk";


const collection = '0xC70C411cFDbE542e8208AF52092CA4f56B633977'
const tokenId = '2329'

const handleBuy = async () => {
  let orders = await axios.get(`https://api.looksrare.org/api/v1/orders?isOrderAsk=true&collection=0xC70C411cFDbE542e8208AF52092CA4f56B633977&tokenId=2329&status%5B%5D=VALID&sort=PRICE_ASC`)

  orders = orders.data.data
  
  let order
  if (orders && orders.length > 0) {
    order = orders[0]
  }

  if (order) {
    executeBuy(order)
  } else {
    // order not exist
  }
}

const executeBuy = async makerOrder => {
  await window.ethereum.send("eth_requestAccounts");
  const provider = new ethers.providers.Web3Provider(window.ethereum);
  const signer = provider.getSigner()
  const accounts = await provider.listAccounts()
  const account = accounts[0] // 买方

  const chainId = SupportedChainId.MAINNET
  const addresses = addressesByNetwork[chainId]

  console.log(addresses)

  const { encodedParams } = encodeOrderParams(makerOrder.params)
  console.log(encodedParams)

  makerOrder.collection = makerOrder.collectionAddress
  makerOrder.currency = makerOrder.currencyAddress
  makerOrder.params = encodedParams;
  
  const takerOrder = {
    isOrderAsk: false,
    taker: account,
    price: makerOrder.price,
    tokenId: makerOrder.tokenId,
    minPercentageToAsk: 10000,
    params: encodedParams
  }
  // 0.00388379ETH
  console.log(takerOrder)
  console.log(makerOrder)
  console.log(await signer.getAddress());

  const exchangeInterface = new ethers.utils.Interface(LooksRareExchangeAbi)
  const exchangeContract = new ethers.Contract(addresses.EXCHANGE, exchangeInterface, signer)
  console.log(exchangeContract);

  const wethInterface = new ethers.utils.Interface(WETHAbi)
  const wethContract = new ethers.Contract(addresses.WETH, wethInterface, signer)
  const wethBalance = await wethContract.balanceOf(account)

  console.log(wethBalance)
  console.log(BigNumber.from(wethBalance).toString())

  // let gasLimit = 200000 
  // if (BigNumber.from(wethBalance).toString() !== '0') {
  //   await wethContract.approve(addresses.EXCHANGE, makerOrder.price)
  // }
  let gasLimit = await exchangeContract.estimateGas.matchAskWithTakerBidUsingETHAndWETH(takerOrder, makerOrder, {
    value: makerOrder.price,
  })

  const tx = await exchangeContract.matchAskWithTakerBidUsingETHAndWETH(takerOrder, makerOrder, {
    value: makerOrder.price,
    gasLimit: gasLimit
  })

  // try {
  //   await tx.wait()
  // } catch (error) {
  //   console.log(error)
  // }
}


</script>
<style scoped>
.read-the-docs {
  color: #888;
}
</style>




<!-- function matchAskWithTakerBidUsingETHAndWETH(
  OrderTypes.TakerOrder calldata takerBid,
  OrderTypes.MakerOrder calldata makerAsk
) external payable override nonReentrant {
  require((makerAsk.isOrderAsk) && (!takerBid.isOrderAsk), "Order: Wrong sides");
  require(makerAsk.currency == WETH, "Order: Currency must be WETH");
  require(msg.sender == takerBid.taker, "Order: Taker must be the sender");

  // If not enough ETH to cover the price, use WETH
  if (takerBid.price > msg.value) {
      IERC20(WETH).safeTransferFrom(msg.sender, address(this), (takerBid.price - msg.value));
  } else {
      require(takerBid.price == msg.value, "Order: Msg.value too high");
  }

  // Wrap ETH sent to this contract
  IWETH(WETH).deposit{value: msg.value}();

  // Check the maker ask order
  bytes32 askHash = makerAsk.hash();
  _validateOrder(makerAsk, askHash);

  // Retrieve execution parameters
  (bool isExecutionValid, uint256 tokenId, uint256 amount) = IExecutionStrategy(makerAsk.strategy)
      .canExecuteTakerBid(takerBid, makerAsk);

  require(isExecutionValid, "Strategy: Execution invalid");

  // Update maker ask order status to true (prevents replay)
  _isUserOrderNonceExecutedOrCancelled[makerAsk.signer][makerAsk.nonce] = true;

  // Execution part 1/2
  _transferFeesAndFundsWithWETH(
      makerAsk.strategy,
      makerAsk.collection,
      tokenId,
      makerAsk.signer,
      takerBid.price,
      makerAsk.minPercentageToAsk
  );

  // Execution part 2/2
  _transferNonFungibleToken(makerAsk.collection, makerAsk.signer, takerBid.taker, tokenId, amount);

  emit TakerBid(
      askHash,
      makerAsk.nonce,
      takerBid.taker,
      makerAsk.signer,
      makerAsk.strategy,
      makerAsk.currency,
      makerAsk.collection,
      tokenId,
      amount,
      takerBid.price
  );
} -->