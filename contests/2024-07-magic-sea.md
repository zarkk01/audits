# Magic Sea DEX
## Contest Summary
Code under review: [2024-07-magic-sea](https://audits.sherlock.xyz/contests/437)

Contest Page: [magic-sea-contest](https://audits.sherlock.xyz/contests/437)

Placement: #39/296

# Findings

# [H-01] vote function does not correctly checks if the remaining duration of a LockingPosition is greater than 14 days.

## Vulnerability Detail
When a user has a LockingPosition and wants to to vote, according to docs, "the remaining lock period needs to be longer then the epoch time". However, the checks in vote function is like this :
```
    function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
        // ...

        // check if _minimumLockTime >= initialLockDuration and it is locked
@>        if (_mlumStaking.getStakingPosition(tokenId).initialLockDuration < _minimumLockTime) {
            revert IVoter__InsufficientLockTime();
        }
@>        if (_mlumStaking.getStakingPosition(tokenId).lockDuration < _periodDuration) {
            revert IVoter__InsufficientLockTime();
        }

        // ...
    }
```
As we can see, the function checks if the lockDuration is greater than the _periodDuration which is 14 days. However, the lockDuration of the LockingPosition can indeed be greater than the 14 days but the remaining lock time to be actually seconds.

## Impact
This vulnerability leads to someone to be actually to double vote since he can vote with a LockingPosition which has remaining lock time some seconds and then withdraw his staked MLUM and stake them again and vote again. Also, the invariant of the protocol that "the remaining lock period needs to be longer then the epoch time" is not enforced.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L175

## Tool used
Manual Review

## Recommendation
Consider checking that the remaining lock periods is longer than 14 days by using the block.timestamp.


# [M-02] A malicious user can execute a Denial of Service (DoS) attack on the registration of legitimate BribeRewarder contracts in the Voter contract by registering 5 worthless BribeRewarder contracts in each VotingPeriod.

## Summary
A malicious user can execute a Denial of Service (DoS) attack on the registration of legitimate BribeRewarder contracts in the Voter contract by registering 5 worthless BribeRewarder contracts in each VotingPeriod.

## Vulnerability Detail
When a user wants to create a BribeRewarder, they must call the bribe function in the BribeRewarder contract and specify the VotingPeriod for the bribe activation. However, the Voter contract checks if the number of registered BribeRewarder contracts for the specified period is not above 5. If this limit is reached, no further BribeRewarder can be registered for that VotingPeriod. This limitation allows a malicious user to create and register worthless BribeRewarder contracts that distribute negligible amounts (e.g., 1 wei of ETH) to voters, thereby blocking the registration of legitimate BribeRewarder contracts.
The relevant check during registration is shown below:
```
    function _bribe(uint256 startId, uint256 lastId, uint256 amountPerPeriod) internal {
        _checkAlreadyInitialized();
        if (lastId < startId) revert BribeRewarder__WrongEndId();
@>        if (amountPerPeriod == 0) revert BribeRewarder__ZeroReward();

        IVoter voter = IVoter(_caller);

        // ...
    }
```
By exploiting this check, a malicious user can create numerous BribeRewarder contracts that distribute trivial amounts, effectively preventing the registration of legitimate BribeRewarder contracts.

## Impact
This vulnerability allows a malicious user to completely disrupt the Bribes system, making it impossible for legitimate BribeRewarder contracts to be registered and function as intended. The attack can be executed at minimal cost (e.g., 5 wei for each pair for each VotingPeriod), causing significant disruption to the voting and bribe distribution processes.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L141

## Tool used
Manual Review

## Recommendation
To mitigate this vulnerability, consider implementing a minimum reward threshold that a BribeRewarder must meet before being accepted. This will ensure that only BribeRewarder contracts distributing meaningful rewards can be registered, preventing the registration of worthless BribeRewarder contracts.
```
    function _bribe(uint256 startId, uint256 lastId, uint256 amountPerPeriod) internal {
        _checkAlreadyInitialized();
        if (lastId < startId) revert BribeRewarder__WrongEndId();
        if (amountPerPeriod == 0) revert BribeRewarder__ZeroReward();
+       if (amountPerPeriod < MINIMUM_REWARD) revert BribeRewarder__InsufficientReward();

        IVoter voter = IVoter(_caller);

        // ...
    }
```
Define MINIMUM_REWARD based on the desired minimum amount to ensure the effectiveness of bribe distribution.


# [M-03] _requireOnlyOperatorOrOwnerOf does not correctly check the owner or the operator of the position leading to anyone can adjust the duration of a LockingPosition by adding to it. 

## Vulnerability Detail
The requireOnlyOperatorOrOwnerOf function is supposed to check if the caller of addPosition in MlumStaking contract is, actually, the owner of the LockingPosition or authorized. However, in the way that the function call of _isAuthorized call is implemented the requireOnlyOperatorOrOwnerOf will always return true. We can see the the _isAuthorized function of ERC721 here :
```
    /**
     * @dev Returns whether `spender` is allowed to manage `owner`'s tokens, or `tokenId` in
     * particular (ignoring whether it is owned by `owner`).
     *
     * WARNING: This function assumes that `owner` is the actual owner of `tokenId` and does not verify this
     * assumption.
     */
    function _isAuthorized(address owner, address spender, uint256 tokenId) internal view virtual returns (bool) {
        return
            spender != address(0) &&
            (owner == spender || isApprovedForAll(owner, spender) || _getApproved(tokenId) == spender);
    }
```
In MlumStaking, msg.sender is passed in both owner and spender params without checking if the msg.sender is the owner of the NFT as stated in the comments of the _isAuthorized function of ERC721. This results to anyone can call addPosition function for whichever NFT they want to. By adding to the position, an attacker can adjust the duration of any NFT position and can prevent the actual owner of the NFT to withdraw their funds or vote in Voter contract.

## Impact
Anyone can change the duration of a LockingPosition can lead to a DoS attack on the actual owner of the position since the lockDuration of the position was selected by the owner so to serve his needs. By extending or reducing the duration of the position, the attacker can prevent the owner from withdrawing his funds or voting in the Voter contract among other problems for the actual owner which does not equal the extra amount in the position that the attacker added.

## Proof of concept
This PoC demonstrates the scenario where an attacker DoS the withdrawal of the actual owner of the LockingPosition by adding to it a very tiny amount and extending the duration of it.
To understand better this vulnerability, add this test in MlumStakingTest.sol and run forge test --mt testWithdrawDOSbyAddingToPosition:
```
function testWithdrawDOSbyAddingToPosition() public {
        _stakingToken.mint(ALICE, 100 ether);
        _stakingToken.mint(BOB, 1 ether);

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 50 ether);
        _pool.createPosition(1 ether, 2 days);
        vm.stopPrank();

        skip(1 days);

        vm.expectRevert();
        vm.prank(ALICE);
        _pool.withdrawFromPosition(1, 0.5 ether);

        skip(1 days);

        // as long as the malicious Bob deposits amountAdd > amountStaked / (secondsInitDuration - 1) the withdraw of Alice will fail because the duration gonna be extended.
        vm.startPrank(BOB);
        _stakingToken.approve(address(_pool), 5787070527029);
        _pool.addToPosition(1, 5787070527029);
        vm.stopPrank();

        vm.expectRevert();
        vm.prank(ALICE);
        _pool.withdrawFromPosition(1, 1 ether);
    }
```
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L140

## Tool used
Manual Review

## Recommendation
Consider making this change in the _requireOnlyOperatorOrOwnerOf function so to implement correctly the check :
```
    function _requireOnlyOperatorOrOwnerOf(uint256 tokenId) internal view {
        // isApprovedOrOwner: caller has no rights on token
-        require(ERC721Upgradeable._isAuthorized(msg.sender, msg.sender, tokenId), "FORBIDDEN");
+        require(ERC721Upgradeable._isAuthorized(ERC721Upgradeable.ownerOf(tokenId), msg.sender, tokenId), "FORBIDDEN");
    }
```


# [M-04] Rewards in MlumStaking are distributed unfairly not taking into consideration the time someone has been locked. 

## Vulnerability Detail
In MlumStaking contract, as stated in the docs rewards will be distributed every few days(see here). However, the contract does not take into consideration the time that someone has been locked for between the reward distrbutions and this opens the door to some attacks making the rewards to be distributed unfairly. Also, the contract allows users to create StakingPosition with 0 lockDuration which means that this position is available to be withdrawn anytime the user wants and just the multiplier will be 0. This can cause serious problems in the reward distribution process since someone can time the updatePool call and create a position with 0 lockDuration, minutes/hours before the reward distribution and then instantly withdraw it after the updatePool call.

## Impact
This vulnerability leads to unfair reward distribution since an attacker can get a huge portion of the rewards without having the appropriate time commitment. As a result, users and MLUM stakers will get less rewards than they should due to the fact that the attacker locked some minutes/hours before the distribution and then instantly withdraw after the updatePool call.

## Proof of concept
This PoC demonstrates the scenario where someone frontruns the updatePool function by creating a StakingPosition with lockDuration set to 0 and then backruns it by calling withdrawFromPosition instantly. However, this can happen, also, with some time delay and the attacker create a StakingPosition some minutes/hours before the reward distribution and then instantly withdraw it after the updatePool call. To understand better this vulnerability, add this test in the MlumStaking.sol file and run forge test --mt test_attackWithFrontrunningInMintRewards :
```
    function test_attackWithFrontrunningInMintRewards() public {
        _stakingToken.mint(ALICE, 2 ether);
        _stakingToken.mint(BOB, 100 ether);

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 1 days);
        vm.stopPrank();

        skip(3600);

        vm.startPrank(BOB);
        _stakingToken.approve(address(_pool), 100 ether);
        _pool.createPosition(100 ether, 0);
        vm.stopPrank();
        _rewardToken.mint(address(_pool), 100_000_000);
        vm.startPrank(BOB);
        _pool.withdrawFromPosition(2, 100 ether);
        vm.stopPrank();

        vm.prank(ALICE);
        _pool.harvestPosition(1);

        console.log("BOB reward: ", _rewardToken.balanceOf(BOB));
        console.log("ALICE reward: ", _rewardToken.balanceOf(ALICE));

        vm.startPrank(ALICE);
        vm.expectRevert();
        _pool.withdrawFromPosition(1, 0.5 ether);

        skip(1 days);

        _pool.withdrawFromPosition(1, 1 ether);

        vm.stopPrank();
    }
```    
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L354
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L574

## Tool used
Manual Review

## Recommendation
Consider checking how much time has someone been locked for an distribute the rewards accordingly.


# [M-05] BribeRewarder contract funtionality is broken with low-decimals tokens.

## Vulnerability Detail
The duration of a VotingPeriod is 14 days which is 1209600 in seconds. Also, a user can create a BribeRewarder which whatever token he likes and these kind of issues are in-scope according to the docs. In _calculateRewards of BribeRewarder, a division is executed and we can see it here :
```
    function _calculateRewards(uint256 periodId) internal view returns (uint256) {
        (uint256 startTime, uint256 endTime) = IVoter(_caller).getPeriodStartEndtime(periodId);

        if (endTime == 0 || startTime > block.timestamp) {
            return 0;
        }

        uint256 duration = endTime - startTime;
@>        uint256 emissionsPerSecond = _amountPerPeriod / duration;

        uint256 lastUpdateTimestamp = _lastUpdateTimestamp;
        uint256 timestamp = block.timestamp > endTime ? endTime : block.timestamp;
        return timestamp > lastUpdateTimestamp ? (timestamp - lastUpdateTimestamp) * emissionsPerSecond : 0;
    }
```    
Given the VotingPeriod duration of 1,209,600 seconds, any _amountPerPeriod less than 1,209,600 will result in emissionsPerSecond being 0 due to Solidity's rounding down behavior. This is particularly problematic for tokens with low decimal places, such as GUSD (Gemini Dollar), which has 2 decimals. For instance, if the _amountPerPeriod is 10,000 GUSD, the emissionsPerSecond will always be 0, preventing voters from receiving any rewards.

## Impact
This vulnerability prevents the distribution of rewards to voters when using tokens with low decimal places. Consequently, the reward distribution mechanism is rendered ineffective, and voters do not receive the rewards they are entitled to for their participation.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L300

## Tool used
Manual Review

## Recommendation
To address this issue, consider implementing a check to ensure that the _amountPerPeriod is sufficient to generate a non-zero emissionsPerSecond for the given VotingPeriod duration. Additionally, consider adjusting the calculation method to accommodate tokens with low decimal places, ensuring accurate reward distribution. One potential solution is to use a higher precision for internal calculations to avoid rounding down to zero.