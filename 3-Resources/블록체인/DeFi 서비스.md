# 🏦 DeFi (탈중앙화 금융) 서비스 가이드

> 전통 금융을 혁신하는 블록체인 기반 금융 서비스들

## 📋 목차

1. [DeFi 개요](#-defi-개요)
2. [핵심 DeFi 서비스](#-핵심-defi-서비스)
3. [개발 실습](#-개발-실습)
4. [보안과 위험 관리](#-보안과-위험-관리)

## 🌟 DeFi 개요

### DeFi란?

**DeFi (Decentralized Finance)**는 블록체인 기술을 활용해 기존 금융 시스템을 탈중앙화한 서비스입니다.

#### 전통 금융 vs DeFi

| 항목      | 전통 금융              | DeFi            |
| --------- | ---------------------- | --------------- |
| 중개자    | 은행, 증권사           | 스마트 컨트랙트 |
| 접근성    | 제한적 (KYC, 지역제한) | 전 세계 누구나  |
| 운영 시간 | 영업 시간 제한         | 24/7            |
| 투명성    | 제한적                 | 완전 공개       |
| 수수료    | 높음                   | 상대적으로 낮음 |

### DeFi의 장점

- ✅ **무허가성**: 누구나 참여 가능
- ✅ **투명성**: 모든 거래가 블록체인에 기록
- ✅ **상호 운용성**: 다양한 프로토콜 간 연결
- ✅ **프로그래밍 가능**: 스마트 컨트랙트로 자동화
- ✅ **글로벌 액세스**: 지역 제한 없음

## 🔧 핵심 DeFi 서비스

### 1. 탈중앙화 거래소 (DEX)

중앙화된 거래소 없이 토큰을 교환할 수 있는 플랫폼

#### Uniswap 스타일 AMM 구현

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SimpleDEX is ReentrancyGuard {
    IERC20 public tokenA;
    IERC20 public tokenB;

    uint256 public reserveA;
    uint256 public reserveB;
    uint256 public totalLiquidity;

    mapping(address => uint256) public liquidity;

    event LiquidityAdded(address provider, uint256 amountA, uint256 amountB);
    event LiquidityRemoved(address provider, uint256 amountA, uint256 amountB);
    event TokensSwapped(address trader, address tokenIn, uint256 amountIn, uint256 amountOut);

    constructor(address _tokenA, address _tokenB) {
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
    }

    // 유동성 공급
    function addLiquidity(uint256 _amountA, uint256 _amountB)
        external nonReentrant
    {
        require(_amountA > 0 && _amountB > 0, "Invalid amounts");

        // 초기 유동성 공급
        if (totalLiquidity == 0) {
            liquidity[msg.sender] = sqrt(_amountA * _amountB);
            totalLiquidity = liquidity[msg.sender];
        } else {
            // 기존 비율에 맞춰 유동성 공급
            uint256 liquidityA = (_amountA * totalLiquidity) / reserveA;
            uint256 liquidityB = (_amountB * totalLiquidity) / reserveB;
            uint256 liquidityMinted = liquidityA < liquidityB ? liquidityA : liquidityB;

            liquidity[msg.sender] += liquidityMinted;
            totalLiquidity += liquidityMinted;
        }

        tokenA.transferFrom(msg.sender, address(this), _amountA);
        tokenB.transferFrom(msg.sender, address(this), _amountB);

        reserveA += _amountA;
        reserveB += _amountB;

        emit LiquidityAdded(msg.sender, _amountA, _amountB);
    }

    // 토큰 스왑
    function swapAtoB(uint256 _amountAIn) external nonReentrant {
        require(_amountAIn > 0, "Invalid input amount");

        uint256 amountBOut = getAmountOut(_amountAIn, reserveA, reserveB);
        require(amountBOut > 0, "Insufficient output amount");

        tokenA.transferFrom(msg.sender, address(this), _amountAIn);
        tokenB.transfer(msg.sender, amountBOut);

        reserveA += _amountAIn;
        reserveB -= amountBOut;

        emit TokensSwapped(msg.sender, address(tokenA), _amountAIn, amountBOut);
    }

    // x * y = k 공식으로 출력량 계산
    function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut)
        public pure returns (uint256)
    {
        uint256 amountInWithFee = amountIn * 997; // 0.3% 수수료
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * 1000) + amountInWithFee;
        return numerator / denominator;
    }

    // 제곱근 계산 (바빌로니아 방법)
    function sqrt(uint256 x) internal pure returns (uint256) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        uint256 y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
        return y;
    }
}
```

### 2. 대출/차용 프로토콜

암호화폐를 담보로 대출하거나 예치하여 이자를 받는 서비스

```solidity
contract LendingProtocol is ReentrancyGuard {
    struct Market {
        IERC20 token;
        uint256 totalSupply;
        uint256 totalBorrow;
        uint256 supplyRate;
        uint256 borrowRate;
        uint256 collateralFactor; // 담보 비율 (50% = 5000)
    }

    mapping(address => Market) public markets;
    mapping(address => mapping(address => uint256)) public supplies; // user => token => amount
    mapping(address => mapping(address => uint256)) public borrows;

    uint256 constant FACTOR_SCALE = 10000;

    event Supplied(address user, address token, uint256 amount);
    event Withdrawn(address user, address token, uint256 amount);
    event Borrowed(address user, address token, uint256 amount);
    event Repaid(address user, address token, uint256 amount);

    // 자산 예치 (공급)
    function supply(address token, uint256 amount) external nonReentrant {
        Market storage market = markets[token];
        require(address(market.token) != address(0), "Market not supported");

        market.token.transferFrom(msg.sender, address(this), amount);
        supplies[msg.sender][token] += amount;
        market.totalSupply += amount;

        emit Supplied(msg.sender, token, amount);
    }

    // 예치 자산 인출
    function withdraw(address token, uint256 amount) external nonReentrant {
        require(supplies[msg.sender][token] >= amount, "Insufficient supply");
        require(isAccountHealthy(msg.sender), "Account would be undercollateralized");

        Market storage market = markets[token];
        supplies[msg.sender][token] -= amount;
        market.totalSupply -= amount;
        market.token.transfer(msg.sender, amount);

        emit Withdrawn(msg.sender, token, amount);
    }

    // 대출
    function borrow(address token, uint256 amount) external nonReentrant {
        Market storage market = markets[token];
        require(market.totalSupply >= amount, "Insufficient liquidity");

        borrows[msg.sender][token] += amount;
        market.totalBorrow += amount;

        require(isAccountHealthy(msg.sender), "Insufficient collateral");

        market.token.transfer(msg.sender, amount);

        emit Borrowed(msg.sender, token, amount);
    }

    // 대출 상환
    function repay(address token, uint256 amount) external nonReentrant {
        uint256 borrowed = borrows[msg.sender][token];
        uint256 repayAmount = amount > borrowed ? borrowed : amount;

        Market storage market = markets[token];
        market.token.transferFrom(msg.sender, address(this), repayAmount);

        borrows[msg.sender][token] -= repayAmount;
        market.totalBorrow -= repayAmount;

        emit Repaid(msg.sender, token, repayAmount);
    }

    // 계정 건전성 확인
    function isAccountHealthy(address user) public view returns (bool) {
        uint256 totalCollateralValue = 0;
        uint256 totalBorrowValue = 0;

        // 모든 시장에 대해 담보 가치와 대출 가치 계산
        // (실제로는 가격 오라클이 필요)

        return totalCollateralValue >= totalBorrowValue;
    }
}
```

### 3. 스테이킹 프로토콜

토큰을 예치하여 보상을 받는 시스템

```solidity
contract StakingRewards is ReentrancyGuard {
    IERC20 public stakingToken;
    IERC20 public rewardsToken;

    uint256 public rewardRate; // 초당 보상량
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;

    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    mapping(address => uint256) public balances;

    uint256 public totalSupply;

    event Staked(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardPaid(address indexed user, uint256 reward);

    constructor(address _stakingToken, address _rewardsToken, uint256 _rewardRate) {
        stakingToken = IERC20(_stakingToken);
        rewardsToken = IERC20(_rewardsToken);
        rewardRate = _rewardRate;
    }

    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;

        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

    // 스테이킹
    function stake(uint256 amount) external nonReentrant updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");

        totalSupply += amount;
        balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);

        emit Staked(msg.sender, amount);
    }

    // 스테이킹 해제
    function withdraw(uint256 amount) external nonReentrant updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");
        require(balances[msg.sender] >= amount, "Insufficient balance");

        totalSupply -= amount;
        balances[msg.sender] -= amount;
        stakingToken.transfer(msg.sender, amount);

        emit Withdrawn(msg.sender, amount);
    }

    // 보상 수령
    function getReward() external nonReentrant updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardsToken.transfer(msg.sender, reward);
            emit RewardPaid(msg.sender, reward);
        }
    }

    // 토큰당 보상량 계산
    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) {
            return rewardPerTokenStored;
        }
        return rewardPerTokenStored +
            (((block.timestamp - lastUpdateTime) * rewardRate * 1e18) / totalSupply);
    }

    // 사용자가 받을 수 있는 보상량
    function earned(address account) public view returns (uint256) {
        return (balances[account] *
            (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18 + rewards[account];
    }
}
```

### 4. 예측 시장 (Prediction Market)

미래 이벤트에 대해 베팅하고 예측하는 플랫폼

```solidity
contract PredictionMarket is ReentrancyGuard {
    struct Market {
        string question;
        uint256 endTime;
        uint256 totalYes;
        uint256 totalNo;
        bool resolved;
        bool outcome;
        mapping(address => uint256) yesShares;
        mapping(address => uint256) noShares;
    }

    mapping(uint256 => Market) public markets;
    uint256 public marketCount;

    event MarketCreated(uint256 indexed marketId, string question, uint256 endTime);
    event SharesPurchased(uint256 indexed marketId, address user, bool isYes, uint256 amount);
    event MarketResolved(uint256 indexed marketId, bool outcome);
    event WinningsClaimed(uint256 indexed marketId, address user, uint256 amount);

    // 예측 시장 생성
    function createMarket(string memory question, uint256 duration) external {
        uint256 marketId = marketCount++;
        Market storage market = markets[marketId];
        market.question = question;
        market.endTime = block.timestamp + duration;

        emit MarketCreated(marketId, question, market.endTime);
    }

    // 예측 참여 (YES 또는 NO)
    function buyShares(uint256 marketId, bool isYes) external payable nonReentrant {
        Market storage market = markets[marketId];
        require(block.timestamp < market.endTime, "Market ended");
        require(!market.resolved, "Market already resolved");
        require(msg.value > 0, "Must send ETH");

        if (isYes) {
            market.yesShares[msg.sender] += msg.value;
            market.totalYes += msg.value;
        } else {
            market.noShares[msg.sender] += msg.value;
            market.totalNo += msg.value;
        }

        emit SharesPurchased(marketId, msg.sender, isYes, msg.value);
    }

    // 시장 결과 해결 (오라클 또는 투표로)
    function resolveMarket(uint256 marketId, bool outcome) external {
        Market storage market = markets[marketId];
        require(block.timestamp >= market.endTime, "Market not ended");
        require(!market.resolved, "Already resolved");

        market.resolved = true;
        market.outcome = outcome;

        emit MarketResolved(marketId, outcome);
    }

    // 당첨금 수령
    function claimWinnings(uint256 marketId) external nonReentrant {
        Market storage market = markets[marketId];
        require(market.resolved, "Market not resolved");

        uint256 userShares;
        uint256 totalWinning;
        uint256 totalPool = market.totalYes + market.totalNo;

        if (market.outcome) {
            // YES가 이긴 경우
            userShares = market.yesShares[msg.sender];
            totalWinning = market.totalYes;
        } else {
            // NO가 이긴 경우
            userShares = market.noShares[msg.sender];
            totalWinning = market.totalNo;
        }

        require(userShares > 0, "No winning shares");

        uint256 payout = (userShares * totalPool) / totalWinning;

        // 지급 처리
        if (market.outcome) {
            market.yesShares[msg.sender] = 0;
        } else {
            market.noShares[msg.sender] = 0;
        }

        payable(msg.sender).transfer(payout);

        emit WinningsClaimed(marketId, msg.sender, payout);
    }
}
```

## 💻 개발 실습

### Next.js DeFi 대시보드

```typescript
// components/DeFiDashboard.tsx
import { useState, useEffect } from "react";
import { useContract, useAddress, useContractRead } from "@thirdweb-dev/react";
import { ethers } from "ethers";

export default function DeFiDashboard() {
  const address = useAddress();
  const [stakingAmount, setStakingAmount] = useState("");
  const [borrowAmount, setBorrowAmount] = useState("");

  // 스테이킹 컨트랙트
  const { contract: stakingContract } = useContract("STAKING_CONTRACT_ADDRESS");
  const { data: stakedBalance } = useContractRead(stakingContract, "balances", [
    address,
  ]);
  const { data: earnedRewards } = useContractRead(stakingContract, "earned", [
    address,
  ]);

  // 대출 컨트랙트
  const { contract: lendingContract } = useContract("LENDING_CONTRACT_ADDRESS");
  const { data: suppliedBalance } = useContractRead(
    lendingContract,
    "supplies",
    [address, "TOKEN_ADDRESS"]
  );
  const { data: borrowedBalance } = useContractRead(
    lendingContract,
    "borrows",
    [address, "TOKEN_ADDRESS"]
  );

  const handleStake = async () => {
    if (!stakingContract || !stakingAmount) return;

    try {
      await stakingContract.call("stake", [
        ethers.utils.parseEther(stakingAmount),
      ]);
      setStakingAmount("");
    } catch (error) {
      console.error("스테이킹 실패:", error);
    }
  };

  const handleBorrow = async () => {
    if (!lendingContract || !borrowAmount) return;

    try {
      await lendingContract.call("borrow", [
        "TOKEN_ADDRESS",
        ethers.utils.parseEther(borrowAmount),
      ]);
      setBorrowAmount("");
    } catch (error) {
      console.error("대출 실패:", error);
    }
  };

  const claimRewards = async () => {
    if (!stakingContract) return;

    try {
      await stakingContract.call("getReward");
    } catch (error) {
      console.error("보상 수령 실패:", error);
    }
  };

  return (
    <div className="max-w-6xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-8">DeFi 대시보드</h1>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {/* 스테이킹 섹션 */}
        <div className="bg-gradient-to-r from-purple-500 to-pink-500 rounded-lg p-6 text-white">
          <h2 className="text-xl font-semibold mb-4">스테이킹</h2>

          <div className="space-y-4">
            <div>
              <p className="text-sm opacity-80">스테이킹 잔액</p>
              <p className="text-2xl font-bold">
                {stakedBalance ? ethers.utils.formatEther(stakedBalance) : "0"}{" "}
                TOK
              </p>
            </div>

            <div>
              <p className="text-sm opacity-80">획득 가능한 보상</p>
              <p className="text-xl font-semibold">
                {earnedRewards ? ethers.utils.formatEther(earnedRewards) : "0"}{" "}
                RWD
              </p>
            </div>

            <div className="space-y-2">
              <input
                type="number"
                placeholder="스테이킹할 수량"
                value={stakingAmount}
                onChange={(e) => setStakingAmount(e.target.value)}
                className="w-full p-2 rounded text-black"
              />
              <button
                onClick={handleStake}
                className="w-full bg-white text-purple-500 py-2 rounded font-semibold hover:bg-gray-100"
              >
                스테이킹
              </button>
              <button
                onClick={claimRewards}
                className="w-full bg-yellow-500 text-white py-2 rounded font-semibold hover:bg-yellow-600"
              >
                보상 수령
              </button>
            </div>
          </div>
        </div>

        {/* 대출 섹션 */}
        <div className="bg-gradient-to-r from-blue-500 to-teal-500 rounded-lg p-6 text-white">
          <h2 className="text-xl font-semibold mb-4">대출/차용</h2>

          <div className="space-y-4">
            <div>
              <p className="text-sm opacity-80">공급 잔액</p>
              <p className="text-2xl font-bold">
                {suppliedBalance
                  ? ethers.utils.formatEther(suppliedBalance)
                  : "0"}{" "}
                TOK
              </p>
            </div>

            <div>
              <p className="text-sm opacity-80">대출 잔액</p>
              <p className="text-xl font-semibold">
                {borrowedBalance
                  ? ethers.utils.formatEther(borrowedBalance)
                  : "0"}{" "}
                TOK
              </p>
            </div>

            <div className="space-y-2">
              <input
                type="number"
                placeholder="대출할 수량"
                value={borrowAmount}
                onChange={(e) => setBorrowAmount(e.target.value)}
                className="w-full p-2 rounded text-black"
              />
              <button
                onClick={handleBorrow}
                className="w-full bg-white text-blue-500 py-2 rounded font-semibold hover:bg-gray-100"
              >
                대출하기
              </button>
            </div>
          </div>
        </div>

        {/* 수익률 정보 */}
        <div className="bg-gradient-to-r from-green-500 to-emerald-500 rounded-lg p-6 text-white">
          <h2 className="text-xl font-semibold mb-4">수익률 정보</h2>

          <div className="space-y-4">
            <div>
              <p className="text-sm opacity-80">스테이킹 APY</p>
              <p className="text-2xl font-bold">12.5%</p>
            </div>

            <div>
              <p className="text-sm opacity-80">공급 APY</p>
              <p className="text-xl font-semibold">8.2%</p>
            </div>

            <div>
              <p className="text-sm opacity-80">대출 APY</p>
              <p className="text-xl font-semibold">5.7%</p>
            </div>
          </div>
        </div>
      </div>

      {/* 거래 히스토리 */}
      <div className="mt-8 bg-white rounded-lg shadow-lg p-6">
        <h2 className="text-xl font-semibold mb-4">최근 거래</h2>
        <div className="overflow-x-auto">
          <table className="w-full text-left">
            <thead>
              <tr className="border-b">
                <th className="py-2">타입</th>
                <th className="py-2">수량</th>
                <th className="py-2">시간</th>
                <th className="py-2">상태</th>
              </tr>
            </thead>
            <tbody>
              <tr className="border-b">
                <td className="py-2">스테이킹</td>
                <td className="py-2">100 TOK</td>
                <td className="py-2">2시간 전</td>
                <td className="py-2">
                  <span className="bg-green-100 text-green-800 px-2 py-1 rounded text-sm">
                    완료
                  </span>
                </td>
              </tr>
              <tr className="border-b">
                <td className="py-2">대출</td>
                <td className="py-2">50 TOK</td>
                <td className="py-2">1일 전</td>
                <td className="py-2">
                  <span className="bg-blue-100 text-blue-800 px-2 py-1 rounded text-sm">
                    진행중
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  );
}
```

## 🔒 보안과 위험 관리

### 주요 위험 요소

1. **스마트 컨트랙트 위험**

   - 코드 버그로 인한 자금 손실
   - 재진입 공격 (Reentrancy)
   - 정수 오버플로우/언더플로우

2. **오라클 위험**

   - 가격 조작 공격
   - 오라클 실패 시 청산 위험

3. **유동성 위험**

   - 유동성 부족으로 인한 거래 불가
   - 임펄머넌트 로스 (Impermanent Loss)

4. **가버넌스 위험**
   - 중앙화된 의사결정
   - 악의적 거버넌스 공격

### 보안 모범 사례

```solidity
// 1. 재진입 공격 방지
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SecureDeFi is ReentrancyGuard {
    function sensitiveFunction() external nonReentrant {
        // 중요한 로직
    }
}

// 2. 오버플로우 방지 (Solidity 0.8+는 자동으로 처리)
function safeAdd(uint256 a, uint256 b) internal pure returns (uint256) {
    return a + b; // 자동 오버플로우 체크
}

// 3. 접근 제어
import "@openzeppelin/contracts/access/Ownable.sol";

contract ControlledContract is Ownable {
    function adminFunction() external onlyOwner {
        // 관리자만 접근 가능
    }
}

// 4. 이벤트 로깅
event ImportantAction(address indexed user, uint256 amount, uint256 timestamp);

function doSomething(uint256 amount) external {
    // 로직 수행
    emit ImportantAction(msg.sender, amount, block.timestamp);
}
```

### 감사 체크리스트

- [ ] **코드 리뷰**: 모든 함수와 로직 검토
- [ ] **테스트 커버리지**: 95% 이상 테스트 커버리지
- [ ] **정적 분석**: Slither, MythX 등 도구 사용
- [ ] **전문 감사**: 외부 보안 회사 감사
- [ ] **버그 바운티**: 커뮤니티 테스트 프로그램
- [ ] **점진적 배포**: 테스트넷 → 제한된 메인넷 → 전체 배포

---

DeFi는 금융의 미래를 바꾸고 있습니다. 하지만 높은 수익과 함께 위험도 존재하므로, 철저한 보안 검토와 위험 관리가 필수입니다! 🚀

#DeFi #탈중앙화금융 #스마트컨트랙트 #유동성 #스테이킹 #대출 #DEX #보안
