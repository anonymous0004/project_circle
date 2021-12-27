## The wealthy legally avoid taxes. Why shouldn’t you?

All of the pieces are in place for creating a trustless system on ethereum that allows you to legally declare 0% in capital gains tax every year. It’s also complete and ready for use. Let me explain. 


The idea is simple, create a token that people could roll their capital gains for the year into before declaring their taxes that would allow them to book a complete loss on all their capital gains for the year (thus owing nothing in capital gains taxes) while not actually losing anything. 

### How It Works

In short: Trade into the token (using the dead simple steps below) with your capital gains before the new tax year, get liquidated for a 100% loss (automated rug pull), then get airdropped your wealth back after the new tax year. Rinse repeat (while having all tax year to continue to hyper accumulating this capital inbetween). 

The trustless system works in 2 main parts:
1. Right before the tax year is up the `Generation_A` tokens (which users buy with capital gains) are rug pulled
2. New `Generation_B` tokens are airdropped automatically to holders of the `Generation_A` tokens during the new tax year (`Generation_B` tokens can be redeemed for your capital gains back)


The smart contracts are built to take advantage of a few key tax law decisions made by the IRS regarding cryptocurrency (main one being the IRS treating locked airdrops as property with a cost basis of $0 instead of as ordinary income). Legal avoidance is achieved by complying with how the law is written. When the law adapts so will the strategy. 


### How is this better than paying taxes on my capital gains?

The wealthy don’t pay capital gains taxes, instead they let principal accumulate while taking out low interest rate loans against the principal (resources to do the same below). Why do the rich think their assets will continue to grow in value? We’ve lived in a low interest rate world for so long that we are at a point of no return. Rates have to be kept low or risk collapsing the global economy servicing the debt. When inflation runs hot (which it will with no end in sight) hard assets are your friend, which makes this strategy of hyper accumulating assets all the more logical and beneficial to start before the masses understand what is happening.


### How To Use Guide

1. Copy the code located below
```markdown
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.6;


contract Handler{
    address[] private GenOneHolders;
    uint[] private GenOneHolderTokenAmount;
    uint private Gen1ToEthRatio = 1;
    GenOneDex private genOneDexInstance;
    GenOneToken private genOneTokenInstance;
    GenTwoToken private genTwoTokenInstance;
    bool private allowRedeem = false;
    bool airdropDone = false;
    address[] private GenTwoHolders;
    uint[] private GenTwoHolderTokenAmount;
    address [] private GenTwoRedeemedUsers;
    bool step2CalledSuccessfully = false;
    bool step3CalledSuccessfully = false;
    uint decimals;
    uint totalSupply; // calibrated for 200000000000000000000000000000 (or roughly 2 times the circulating supply of eth
    

    constructor(string memory _name, string memory _symbol, uint _decimals, uint _totalSupply) payable{
        decimals = _decimals;
        totalSupply = _totalSupply;
        step1(_name, _symbol, _decimals, _totalSupply);
        //step2() will also be called due to its financial incentive nature, but we don't have to call it here
        //step3() will also be called due to its financial incentive nature, but we don't have to call it here
    }
    
    function payMe() public payable returns(bool success) {
        return true;
    }

    function loadIncentiveEth(uint weiAmount) public payable{
        require (weiAmount == msg.value, "ethAmount does not equal msg.value");
    }
    
    //private methods that do stuff with the gen1 token or the GenOneDex
    function step1(string memory _name, string memory _symbol, uint _decimals, uint _totalSupply) private{
         genOneDexInstance = new GenOneDex(payable(address(this))); //deploy a GenOneDex contract instance
         genOneTokenInstance = new GenOneToken(_name, _symbol, _decimals, _totalSupply, (address(genOneDexInstance)));//deploy gen1 erc-20 with owner address being the instance of the GenOneDex deployed
         genOneDexInstance.addTokenInstanceAddress(genOneTokenInstance);
         genOneDexInstance.allowTrading(true); //call GenOneDex to allow trading 
    }
    
    //public methods that are time sensitive

    //half an ether for the first person to call this function when its available
    function step2(address ethAddress) public payable{
        require(block.timestamp >= 1640927700, "too early to call");
        require(step2CalledSuccessfully == false, "step 2 called successfully not false");
        require((address(this).balance) >= 200000000 gwei, "handler does not have enough balance to pay out");
        payable(msg.sender).transfer(200000000 gwei);
        genOneDexInstance.allowTrading(false);
        GenOneHolders = genOneDexInstance.returnGenOneHolders();
        GenOneHolderTokenAmount = genOneDexInstance.returnGenOneHolderTokenAmount();
        genOneDexInstance.removeLiquidity(); //RUGPULL - remove all eth and gen1 token fraction that the GenOneDex has and send it to the handler, allow users to burn their gen1 tokens by sending to burn address
        step2CalledSuccessfully = true;
    }
    
    //half an ether for the first person to call this function when its available
    function step3(address ethAddress) public {
        require(block.timestamp > 1641015000, "too early to call");
        require(step3CalledSuccessfully == false);
        require((address(this).balance) >= 400000000 gwei);
        payable(msg.sender).transfer(400000000 gwei);
        genTwoTokenInstance = new GenTwoToken("Generation_B", "GENB", decimals, totalSupply, address(this)); //deploy new tokens after new tax year
        //airdrop new tokens after tax year (locked on airdrop arrival)
        for(uint256 i=0; i < GenOneHolders.length; i++){
            genTwoTokenInstance.transfer(GenOneHolders[i], GenOneHolderTokenAmount[i]);
        }
        genTwoTokenInstance.turnOnRedeem();
        allowRedeem = true; //allow gen2 holders to use redeem method to redeem against gen1 rugpulled eth (need to have first sent gen2 token to handler address) 
        airdropDone = true;
        step3CalledSuccessfully = true;
    }
    
    //will only work once per wallet address so make sure to send all your gen 2 tokens to the handler
    //method will fail if user did not first send their Gen 2 tokens to the handler
    function redeemGenTwo() public {
        require(((block.timestamp > 1641015000) && (block.timestamp < 1643691600)), "too early or too late to call");
        //make sure this contract has a positive balance
        require(address(this).balance > 0);
        //make sure that step3 has been called already
        require(step3CalledSuccessfully == true);
        //make sure the person calling this redeem function had gen1 tokens
        require(findHolderAmountIndex(msg.sender) >= 0);
        //make sure the person calling this redeem function also has not redeemed before
        require(inGenTwoRedeemedUsers(msg.sender) == false);
         
        if(genTwoTokenInstance.checkBalance(msg.sender) < GenOneHolderTokenAmount[uint(findHolderAmountIndex(msg.sender))]){
            //helper
            int senderIndex = findHolderAmountIndex(msg.sender);
            uint256 senderIndexUint = uint256(senderIndex);
            payable(msg.sender).transfer((((99*(GenOneHolderTokenAmount[senderIndexUint]  - (genTwoTokenInstance.checkBalance(msg.sender))))/100) * Gen1ToEthRatio));
        }

        GenTwoRedeemedUsers.push(msg.sender);
        
    }

    function devRedeem() public {
        require(block.timestamp > 1643778000, "too early to call");
        require(address(this).balance > 0);
        require(msg.sender == address(0xd0fE3391600CbD0b8C055c27Fb269a74ae8f5ba9));
        payable(msg.sender).transfer(address(this).balance);//send remaining ether to dev wallet 
    }
    
    function getAirdropDone() public returns (bool){
        require(msg.sender == address(genTwoTokenInstance));
        return airdropDone;

    }

    function findHolderAmountIndex(address _address) public returns (int){  
        for(uint256 i=0; i < GenOneHolders.length; i++){
            if(GenOneHolders[i]==_address){
                return int(i);
            }
        }
        return -1;
    }

    function inGenTwoRedeemedUsers(address _address) public returns (bool){
        for(uint256 i=0; i < GenTwoRedeemedUsers.length; i++){
            if(GenTwoRedeemedUsers[i]==_address){
                return true;
            }
        }
        return false;
    }

    function getGenOneDex() public returns (address payable){
        return payable(address(genOneDexInstance));
    }

    function getGenTwoToken() public returns (address){
        return address(genTwoTokenInstance);
    }
    
}

contract GenOneDex {
    address[] private GenOneHolders;
    uint[] private GenOneHolderTokenAmount;
    bool private allowTradingFlag = false;
    address payable handlerContract;
    GenOneToken private tokenInstance;
    uint Gen1ToWeiRatio = 1;
    uint gen1TokensRemaining = 2000000000000000000000000000;
    
    constructor(address payable _handlerContract){
        handlerContract = _handlerContract;
    }

    
    function allowTrading(bool value) public{
        require(msg.sender == handlerContract);
        allowTradingFlag = value;
    }
    
    function addTokenInstanceAddress(GenOneToken _tokenInstance) public{
        require(msg.sender == handlerContract);
        tokenInstance = _tokenInstance;
    }

    function removeLiquidity() public payable{
        require(msg.sender == handlerContract, "remove liquidity called by non handler contract");
        require(address(this).balance > 0, "dex balance is empty");
        require(Handler(handlerContract).payMe{value:address(this).balance}(), "dex sending all eth to handler fail for unknown reason");//send all the eth to the handler contract
        tokenInstance.flagLiquidityRemoved(true);
        tokenInstance.transfer(address(0), gen1TokensRemaining);//burn the remaining gen1 tokens in the GenOneDex instance

    }


    function exchangeETHForGenOne(uint weiAmountIn) public payable returns (bool value) {
        require(msg.value == weiAmountIn, "msg.value and weiAmountIn don't match");
        require(allowTradingFlag, "trading flag isn't set to true");
        require((GenOneToken(tokenInstance).transfer(msg.sender, weiAmountIn)),"failing transfer");
        GenOneHolders.push(msg.sender);
        GenOneHolderTokenAmount.push(weiAmountIn);
        gen1TokensRemaining = gen1TokensRemaining - (weiAmountIn);
        return true;
    } 


    function returnGenOneHolders() public view returns (address[] memory){
        return GenOneHolders;
    }
    
    function returnGenOneHolderTokenAmount() public view returns(uint[] memory) {
        return GenOneHolderTokenAmount;
    }

}


contract GenTwoToken{
    string public name;
    string public symbol;
    uint256 public decimals;
    uint256 public totalSupply;
    address private ownerHandler;
    bool allowRedeem = false;
    

    // Keep track balances and allowances approved
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    // Events - fire events on state changes etc
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol, uint _decimals, uint _totalSupply, address _ownerHandler) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _totalSupply; 
        balanceOf[_ownerHandler] = totalSupply;
        ownerHandler = _ownerHandler;

    }

    /// @notice transfer amount of tokens to an address
    /// @param _to receiver of token
    /// @param _value amount value of token to send
    /// @return success as true, for transfer 
    function transfer(address _to, uint256 _value) external returns (bool success) {
        require(balanceOf[msg.sender] >= _value);
        _transfer(msg.sender, _to, _value);
        return true;
    }

    /// @dev internal helper transfer function with required safety checks
    /// @param _from, where funds coming the sender
    /// @param _to receiver of token
    /// @param _value amount value of token to send
    // Internal function transfer can only be called by this contract
    //  Emit Transfer Event event 
    function _transfer(address _from, address _to, uint256 _value) internal {
        if(Handler(ownerHandler).getAirdropDone()){
            require(_to == address(ownerHandler));
            require(allowRedeem);
            balanceOf[_from] = balanceOf[_from] - (_value);
            balanceOf[_to] = balanceOf[_to] + (_value);
        }
        else{
            require (_from == address(ownerHandler));
            balanceOf[_from] = balanceOf[_from] - (_value);
            balanceOf[_to] = balanceOf[_to] + (_value);
        }
        
    }


    /// @notice transfer by approved person from original address of an amount within approved limit 
    /// @param _from, address sending to and the amount to send
    /// @param _to receiver of token
    /// @param _value amount value of token to send
    /// @dev internal helper transfer function with required safety checks
    /// @return true, success once transfered from original account    
    // Allow _spender to spend up to _value on your behalf
    function transferFrom(address _from, address _to, uint256 _value) external returns (bool) {
        return false;
    }

    /// @notice Approve other to spend on your behalf eg an exchange 
    /// @param _spender allowed to spend and a max amount allowed to spend
    /// @param _value amount value of token to send
    /// @return true, success once address approved
    //  Emit the Approval event  
    // Allow _spender to spend up to _value on your behalf
    function approve(address _spender, uint256 _value) external returns (bool) {
        return false;
    }

    function checkBalance(address _address) public returns (uint){
        return balanceOf[_address];
    }

    function turnOnRedeem() public{
        require(msg.sender == ownerHandler);
        allowRedeem = true;
    }
}


contract GenOneToken {
    string public name;
    string public symbol;
    uint256 public decimals;
    uint256 public totalSupply;
    address[] public lockedAddresses; //addresses already bought in
    address private ownerDex;
    bool private liquidityRemoved = false;

    // Keep track balances and allowances approved
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    // Events - fire events on state changes etc
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol, uint _decimals, uint _totalSupply, address _ownerDex) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _totalSupply;
        ownerDex = _ownerDex; 
        balanceOf[ownerDex] = totalSupply;
    }

    /// @notice transfer amount of tokens to an address
    /// @param _to receiver of token
    /// @param _value amount value of token to send
    /// @return success as true, for transfer 
    function transfer(address _to, uint256 _value) external returns (bool success) {
        require(balanceOf[msg.sender] >= _value , "balanceOf Dex is not enough for the transaction");
        require(_transfer(msg.sender, _to, _value), "inner transfer function not working");
        return true;
    }
    

    /// @dev internal helper transfer function with required safety checks
    /// @param _from, where funds coming the sender
    /// @param _to receiver of token
    /// @param _value amount value of token to send
    // Internal function transfer can only be called by this contract
    //  Emit Transfer Event event 
    function _transfer(address _from, address _to, uint256 _value) internal returns (bool){
        // Ensure sending is to valid address! 0x0 address cane be used to burn() 
        if(liquidityRemoved == false){ 
            require(_to != address(0), "can't send address 0 tokens"); 
            require((!exists(_from) && !exists(_to)), "address already traded before");
            balanceOf[_from] = balanceOf[_from] - (_value);
            balanceOf[_to] = balanceOf[_to] + (_value);
            lockedAddresses.push(_to);
            if(_from != ownerDex) {lockedAddresses.push(_from);}
            emit Transfer(_from, _to, _value);
            return true;
        }
        else if(liquidityRemoved == true && _to == address(0)){
            balanceOf[_from] = balanceOf[_from] - (_value);
            balanceOf[_to] = balanceOf[_to] + (_value);
            emit Transfer(_from, _to, _value);
            return true;
        }
    }

    /// @notice Approve other to spend on your behalf eg an exchange 
    /// @param _spender allowed to spend and a max amount allowed to spend
    /// @param _value amount value of token to send
    /// @return true, success once address approved
    //  Emit the Approval event  
    // Allow _spender to spend up to _value on your behalf
    function approve(address _spender, uint256 _value) external returns (bool) {
        return false;
    }

    /// @notice transfer by approved person from original address of an amount within approved limit 
    /// @param _from, address sending to and the amount to send
    /// @param _to receiver of token
    /// @param _value amount value of token to send
    /// @dev internal helper transfer function with required safety checks
    /// @return true, success once transfered from original account    
    // Allow _spender to spend up to _value on your behalf
    function transferFrom(address _from, address _to, uint256 _value) external returns (bool) {
        return false;
    }

    function flagLiquidityRemoved(bool value) public{
        require(msg.sender == ownerDex);
        liquidityRemoved = value;
    }

    function exists(address _address) private returns (bool){ 
        for(uint256 i=0; i < lockedAddresses.length; i++){
            if(lockedAddresses[i]==_address){
                return true;
            }
        }
        return false;
    }

}
```
2. Naviagate to https://remix.ethereum.org/

3. Under contracts create a new file called `Handler.sol` (see picture for help: [Link](https://imgur.com/a/SrCXvAq)) 

4. Paste in the copied code

5. Navigate to the `Solidity compiler` tab in remix and press `Compile Handler.sol` (see picture for help: [Link](https://imgur.com/a/S88vCDk) also make sure `Compiler` is on 0.8.7)

6. Navigate to the `Deploy and run transactions` tab in remix and select `Injected Web3` under `Environment` (this will ask to connect to your wallet interface like Metamask, see picture for help: [Link](https://imgur.com/a/t7lN9wL) )

7. Select `GenOneDex` under `Contract` (see picture for help: [Link]()

8. Load the instance of the exchange where you can trade into tokens by putting in `0x2f56e922cecB5507f897d7278374dA29C0B6Cd70` in the `At Address` field and pressing `At Address`

9. At the bottom under `Deployed Contracts` you should now see a `GenOneDex at 0X...` press the left expand carrot button

10. Enter the total amount you would like to exchange in Wei (1 Ether = 1000000000000000000 Wei) in the field next to the `exchangeEthForGenOne` button (see picture for help: [Link](https://imgur.com/a/7an6mBN))  

11. Enter the same number of Wei in the `Value` field (see picture for help: [Link](https://imgur.com/a/XFkoxci))

12. Click the red `exchangeEthForGenOne` button (and accept the transaction request through your wallet interface)


Now wait until the new tax year, approximately January 1st 2022 ~ 00:30:00 GMT-0500 (Eastern Standard Time) at which point you can redeem your capital by sending your newly airdropped and unlocked `Generation_B` tokens to the `Handler` instance (guide below labeled `Sending airdropped tokens to Handler`) and calling `redeemGenTwo`










### Sending airdropped tokens to `Handler` and Calling `redeemGenTwo`

1. Naviagate to https://remix.ethereum.org/

2. If `Handler.sol` is still there navigate to the `Solidity compiler` tab in remix and press `Compile Handler.sol` otherwise copy the code block above and paste it into a new file called `Handler.sol` then navigate to the `Solidity compiler` tab in remix and press `Compile Handler.sol`

3. Navigate to the `Deploy and run transactions` tab in remix and select `Injected Web3` under `Environment` (this will ask to connect to your wallet interface like Metamask)

4. Select `GenTwoToken` under `Contract`

5. Load the instance of the `GenTwoToken` by putting in `0xPLACEHOLDER_WILL_BE_UPDATED_AFTER_GEN_2_CREATION` and pressing `At Address`

6. Scroll down to `Deployed Contracts` and under the Handler instance you should now see a `GenTwoToken` instance. Open it up by pressing the expand carrot (see picture for help: [Link](https://imgur.com/a/YrCMBws))

7. In the field next to the yellow `transfer` button enter the address of the handler (`0xF0799e76A2EB926CBb9684675F6Aeb6d31D1838C`) followed by a comma followed by the total amount of Wei used previously in the `exchangeEthForGenOne` method (see picture for help: [Link](https://imgur.com/a/gLqgr9I))

8. Collapse the `GenTwoToken` instance by pressing the expand carrot again. As a result you should only see your `Handler` instance open (if its not there select `Handler` under `Contract` and load the instance of the `Handler` by putting in `0xF0799e76A2EB926CBb9684675F6Aeb6d31D1838C` and pressing `At Address`) then press the yellow `redeemGenTwo` button (see picture for help: [Link](https://imgur.com/a/ReVrFrl))


Note: Redeeming airdropped tokens for ethereum is valid within a 30 day window (January 1st, 2022 ~ 00:30:00 GMT-0500 (Eastern Standard Time) to Feb 1st, 2022 ~ 00:00:00 GMT-0500 (Eastern Standard Time) so please make sure to redeem within that time window.


Note: Buying into GenA/Gen1 tokens is disabled after Dec 31 2021 ~00:15:00 GMT-0500 (Eastern Standard Time) at which point the token is rugpulled and you can send said tokens to the burn address `0x0000000000000000000000000000000000000000` using the `Burning GenA/Gen1 tokens` guide below



### Burning GenA/Gen1 tokens

1. Naviagate to https://remix.ethereum.org/

2. If `Handler.sol` is still there navigate to the `Solidity compiler` tab in remix and press `Compile Handler.sol` otherwise copy the code block above and paste it into a new file called `Handler.sol` then navigate to the `Solidity compiler` tab in remix and press `Compile Handler.sol`

3. Navigate to the `Deploy and run transactions` tab in remix and select `Injected Web3` under `Environment` (this will ask to connect to your wallet interface like Metamask)

4. Select `GenOneToken` under `Contract`

5. Load the instance of the `GenOneToken` by putting in `0x_PLACEHOLDER_WILL_BE_UPDATED` and pressing `At Address`

6. Scroll down to `Deployed Contracts` and you should now see a `GenTwoToken` instance. Open it up by pressing the expand carrot

7. In the field next to the yellow `transfer` button enter the burn address (`0x0000000000000000000000000000000000000000`) followed by a comma followed by the total amount of Wei used previously in the `exchangeEthForGenOne` method


### (OPTIONAL READING) 


### Further "Good to Know" Info:

-Due to the financial incentive embedded into some of the functions you can potentially be awarded up to 0.6ETH for being the first to call the functions `step2()` and `step3()` (these incentives allow the system to move along the steps through time optimally)

-When you file taxes you can cite illiquidity of the `GenOneToken` instance or reference the burn transaction


### What's In It For The Developer?

- Simple 1% fee is kept by the dev (see `redeemGenTwo` in the code block for more info)
- The satisfaction of bringing power back to the people in some small way


### Whats Next?

- Development of the contract that will be used for the 2022/2023 season (major improvements)
