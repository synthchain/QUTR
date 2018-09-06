

pragma solidity ^0.4.18;


contract QUTR {

    uint256 constant MAX_UINT256 = 2**256 - 1;

    uint256 MAX_SUBMITTED = 50006715761945;

    // (no premine)
    uint256 _totalSupply = 0;

    // The following 2 variables are essentially a lookup table.
    // They are not constant because they are memory.
    // I came up with this because calculating it was expensive,
    // especially so when crossing tiers.

    // Sum of each tier by qtum submitted.
   uint256[] levels = [
      877192982456,
     1989552533017,
     3735007078472,
     6411477666707,
     9840049095279,
    14840049095279,
    21840049095279,
    30840049095279,
    41506715761945,
    50006715761945
    ];

    // Token amounts for each tier.
    uint256[] ratios = [
      114,
      89,
      55,
      34,
      21,
      13,
       8,
       5,
       3,
       2 ];

    // total qtum submitted before fees.
    uint256 _submitted = 0;

    uint256 public tier = 0;

    // QRC20 events.
    event Transfer(address indexed _from, address indexed _to, uint _value);
    event Approval(address indexed _owner, address indexed _spender, uint _value);

    // QUTR events.
    event Mined(address indexed _miner, uint _value);
    event WaitStarted(uint256 endTime);
    event SwapStarted(uint256 endTime);
    event MiningStart(uint256 end_time, uint256 swap_time, uint256 swap_end_time);
    event MiningExtended(uint256 end_time, uint256 swap_time, uint256 swap_end_time);


    // Optional ERC20 values.
    string public name = "Quantumator";
    uint8 public decimals = 8;
    string public symbol = "QUTR";

    // Public variables so the curious can check the state.
    bool public swap = false;
    bool public wait = false;
    bool public extended = false;

    // Public end time for the current state.
    uint256 public endTime;

    // These are calculated at mining start.
    uint256 swapTime;
    uint256 swapEndTime;
    uint256 endTimeExtended;
    uint256 swapTimeExtended;
    uint256 swapEndTimeExtended;

    // Pay rate calculated from balance later.
    uint256 public payRate = 0;

    // Fee variables.  Fees are reserved and then withdrawn  later.
    uint256 submittedFeesPaid = 0;
    uint256 penalty = 0;
    uint256 reservedFees = 0;

    // Storage.
    mapping (address => uint256) balances;
    mapping (address => mapping (address => uint256)) allowed;


   // Fallback function mines the tokens.
   // Send from a wallet you control.
   // DON'T send from an exchange wallet!
   // We recommend sending using a method that calculates gas for you.
   // Here are some estimates (not guaranteed to be accurate):
   // It usually costs around 90k gas.  It cost more if you cross a tier.
   // Maximum around 190k gas.
   function () external payable {

       require(msg.sender != address(0) &&
                tier != 10 &&
                swap == false &&
                wait == false);

        uint256 issued = mint(msg.sender, msg.value);

        Mined(msg.sender, issued);
        Transfer(this, msg.sender, issued);
    }

    // Constructor.
    function QUTR() public {
        _start();
    }

    // This gets called by constructor AND after the swap to restart evertying.
    function _start() internal
    {
        swap = false;
        wait = false;
        extended = false;

        endTime = now + 366 days;
        swapTime = endTime + 30 days;
        swapEndTime = swapTime + 5 days;
        endTimeExtended = now + 1096 days;
        swapTimeExtended = endTimeExtended + 30 days;
        swapEndTimeExtended = swapTimeExtended + 5 days;

        submittedFeesPaid = 0;
        _submitted = 0;

        reservedFees = 0;

        payRate = 0;

        tier = 0;

        MiningStart(endTime, swapTime, swapEndTime);
    }

    // Restarts everything after swap.
    // This is expensive, so we make someone call it and pay for the gas.
    // Any holders that miss the swap get to keep their tokens.
    // Qtum stays in contract, minus 20% penalty fee.
    function restart() public {
        require(swap && now >= endTime);

        penalty = this.balance * 2000 / 10000;

        payFees();

        _start();
    }

    // ERC20 standard supply function.
    function totalSupply() public constant returns (uint)
    {
        return _totalSupply;
    }

    // Mints new tokens when they are mined.
    function mint(address _to, uint256 _value) internal returns (uint256)
    {
        uint256 total = _submitted + _value;

        if (total > MAX_SUBMITTED)
        {
            uint256 refund = total - MAX_SUBMITTED - 1;
            _value = _value - refund;

            // refund money and continue.
            _to.transfer(refund);
        }

        _submitted += _value;

        total -= refund;

        uint256 tokens = calculateTokens(total, _value);

        balances[_to] += tokens;

        _totalSupply += tokens;

        return tokens;
    }

    // Calculates the tokens mined based on the tier.
    function calculateTokens(uint256 total, uint256 _value) internal returns (uint256)
    {
        if (tier == 10)
        {
            // This just rounds it off to an even number.
            return 7400000000;
        }

        uint256 tokens = 0;

        if (total > levels[tier])
        {
            uint256 remaining = total - levels[tier];
            _value -= remaining;
            tokens = (_value) * ratios[tier];

            tier += 1;

            tokens += calculateTokens(total, remaining);
        }
        else
        {
            tokens = _value * ratios[tier];
        }

        return tokens;
    }

    // This is basically so you don't have to add 1 to the last completed tier.
    //  You're welcome.
    function currentTier() public view returns (uint256) {
        if (tier == 10)
        {
            return 10;
        }
        else
        {
            return tier + 1;
        }
    }

    // Qtum remaining for tier.
    function leftInTier() public view returns (uint256) {
        if (tier == 10) {
            return 0;
        }
        else
        {
            return levels[tier] - _submitted;
        }
    }

    // Total sumbitted for mining.
    function submitted() public view returns (uint256) {
        return _submitted;
    }

    // Balance minus oustanding fees.
    function balanceMinusFeesOutstanding() public view returns (uint256) {
        return this.balance - (penalty + (_submitted - submittedFeesPaid) * 1530 / 10000);  // fees are 15.3 % total.
    }

    // Calculates the amount of qtum per token from the balance.
    // This is calculated once by the first account to swap.
    function calculateRate() internal {
        reservedFees = penalty + (_submitted - submittedFeesPaid) * 1530 / 10000;  // fees are 15.3 % total.

        uint256 tokens = _totalSupply / 100000000;
        payRate = (this.balance - reservedFees);

        payRate = payRate / tokens;
    }

    // This function is called on token transfer and fee payment.
    // It checks the next deadline and then updates the deadline and state.
    //
    // It uses the block time, but the time periods are days and months,
    // so it should be pretty safe  ¯\_(ツ)_/¯
    function _updateState() internal {
        // Most of the time, this will just be skipped.
        if (now >= endTime)
        {
            // We are not currently swapping or waiting to swap
            if(!swap && !wait)
            {
                if (extended)
                {
                    // It's been 36 months.
                    wait = true;
                    endTime = swapTimeExtended;
                    WaitStarted(endTime);
                }
                else if (tier == 10)
                {
                    // Tiers filled
                    wait = true;
                    endTime = swapTime;
                    WaitStarted(endTime);
                }
                else
                {
                    // Extended to 36 months
                    endTime = endTimeExtended;
                    extended = true;

                    MiningExtended(endTime, swapTime, swapEndTime);
                }
            }
            else if (wait)
            {
                // It's time to swap.
                swap = true;
                wait = false;

                if (extended)
                {
                    endTime = swapEndTimeExtended;
                }
                else
                {
                    endTime = swapEndTime;
                }

                SwapStarted(endTime);
            }
        }
    }

    // Standard ERC20 transfer plus state check and token swap logic.
    //
    // We recommend sending using a method that calculates gas for you.
    //
    // Here are some estimates (not guaranteed to be accurate):
    // It usually costs around 37k gas.  It cost more if the state changes.
    // State change means around 55k - 65k gas.
    // Swapping tokens for qtum costs around 46k gas. (around 93k for the first account to swap)
    function transfer(address _to, uint256 _value) public returns (bool success) {

        require(balances[msg.sender] >= _value);

         // Normal transfers check if time is expired.
        _updateState();

        // Check if sending in for swap.
        if (_to == address(this))
        {
            // throw if they can't swap yet.
            require(swap);

            if (payRate == 0)
            {
                calculateRate(); // Gas to calc the rate paid by first unlucky soul.
            }

            uint256 amount = _value * payRate;
            // Adjust for decimals
            amount /= 100000000;

            // Burn tokens.
            balances[msg.sender] -= _value;
             _totalSupply -= _value;
            Transfer(msg.sender, _to, _value);

            //send qtum
            msg.sender.transfer(amount);
        } else
        {
            balances[msg.sender] -= _value;
            balances[_to] += _value;
            Transfer(msg.sender, _to, _value);
        }
        return true;
    }

    // Standard ERC20.
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {

        uint256 allowance = allowed[_from][msg.sender];
        require(balances[_from] >= _value && allowance >= _value);
        balances[_to] += _value;
        balances[_from] -= _value;
        if (allowance < MAX_UINT256) {
            allowed[_from][msg.sender] -= _value;
        }
        Transfer(_from, _to, _value);
        return true;
    }

    // Standard ERC20.
    function balanceOf(address _owner) view public returns (uint256 balance) {
        return balances[_owner];
    }

    // Standard ERC20.
    function approve(address _spender, uint256 _value) public returns (bool success) {

        allowed[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
        return true;
    }

    function allowance(address _owner, address _spender) view public returns (uint256 remaining) {
      return allowed[_owner][_spender];
    }

    // ********************
    // Fee stuff.

    // Addresses for fees.
    address public foundation = 0xa5bcf3c46545163abb34ff646eeebbd2e99fcbb9;
    address public owner = 0x1dae000b8e578ce7a54a7bf82f6143b5659ade86;
    address public dev = 0x40d5eb30a95c315a9f197c9b0f187c3152b480ec;
    address public dev2 = 0x382568ba1742ac31aa6a7f46b858d132c5043528;

    // Pays fees to the foundation, the owner, and the devs.
    // It also updates the state.  Anyone can call this.
    function payFees() public {
         // Check state to see if swap needs to happen.
         _updateState();

        uint256 fees = penalty + (_submitted - submittedFeesPaid) * 1530 / 10000;  // fees are 15.3 % total.
        submittedFeesPaid = _submitted;

        reservedFees = 0;
        penalty = 0;

        if (fees > 0)
        {
            foundation.transfer(fees / 4);
            owner.transfer(fees / 4);
            dev.transfer(fees / 4);
            dev2.transfer(fees / 4);
        }
    }

    function changeFoundation (address _receiver) public
    {
        require(msg.sender == foundation);
        foundation = _receiver;
    }


    function changeOwner (address _receiver) public
    {
        require(msg.sender == owner);
        owner = _receiver;
    }

    function changeDev (address _receiver) public
    {
        require(msg.sender == dev);
        dev = _receiver;
    }

    function changeDev2 (address _receiver) public
    {
        require(msg.sender == dev2);
        dev2 = _receiver;
    }

}
