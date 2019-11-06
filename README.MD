
# etvGame
The first game for blockchain, see https://game.etheralliance.io/
# Rule analysis
* 61% static
* 9% dynamic within 3 generations
* 12% leader distribution
* 5% Insurance pool
* 3% lucky prize pool
* 3% platform fee
* 7% Team node distribution

```Solidity
  uint256 public static = 61;
  uint256 public dynamic = 9;
  uint256 public leader = 12;
  uint256 public insurance = 5;
  uint256 public lucky = 3;
  uint256 public platform = 3;
  uint256 public team = 7;
```
# Source code analysis
## lucky income calculation

```Solidity

  //get all dogs
  function allLuckDogs()
  public view returns (uint256, address [] memory){
    address [] memory dogs = new address [](luckNum);
    for(uint256 idx=0; idx<luckNum; idx++){
      dogs[idx] = luckyDogs[idx];
    }

    return (luckNum, dogs);
  }


  function randLoc(uint256 _uplimit)
  internal returns(uint256) {
    randidx = randidx.add(1);
    uint256 random = uint256(keccak256(abi.encodePacked(block.difficulty,
      now, msg.sender, randidx)));
    return random % _uplimit;
  }


  function luckyDogs()
  public onlyWhitelisted isActivated {

    luckNum = LUCK_NUM;
    uint256 singleLuck = lucky.div(luckNum);

    luckyDogs = new address[](luckNum);
    for(uint256 idx = 0; idx < luckNum; idx++){
      uint256 curIdx = randLoc(totalPlayers);
      address curLuck = playAddrs[curIdx];

      if(!isAddrInLuck(curLuck)){
        luckyDogs[idx] = curLuck;
      }else{
        while(true){
          curIdx = curIdx.add(1) % totalPlayers;
          if(!isAddrInLuck(playAddrs[curIdx])){
            break;
          }
        }
        luckyDogs[idx] = playAddrs[curIdx];
      }

      if(!chkOuterOK(luckyDogs[idx])){
        addTargetWin(luckyDogs[idx], singleLuck, false);
      }
    }
  }

```
* Random number generation
* Random seed acquisition

## Static income calculation
```Solidity
  function staticIncome()
  public onlyWhitelisted isActivated {

    //calc cur staticIncome rand: 1-2
    staticRand = randLoc(MAX_STATIC_RAND).add(1);

    for(uint256 idx = 0; idx < plays.length; idx++){
      addTargetWin(plays[idx], staticEths[idx], true);
    }

  }
```
* Calculate the expected return and the actual earned income based on the current latest participation, number of days, and income that can be received

## Win the prize in the dynamic prize pool
```Solidity

  struct Player {
    address payable self;
    address payable parent;
    address payable grand;
    address payable great;
    uint256 income;
    uint256 allIncome;
    uint256 totalInvest;
    uint256 sons;
    uint256 time;
  }


  function invest(address payable _parentAddr)
  public payable isActivated {
    require(msg.value >= minInvest && msg.value <= maxInvest && msg.sender != _parentAddr);

    bool newPlayFlag = false;
    if(allPlayers[msg.sender].self == ZERO_ADDR){
      if(_parentAddr != ZERO_ADDR){
        address payable parent = ZERO_ADDR;
        if(allPlayers[_parentAddr].time > 0){
          parent = _parentAddr;
          allPlayers[_parentAddr].sons = allPlayers[_parentAddr].sons.add(1);
        }

        allPlayers[msg.sender] = Player({
          self : msg.sender,
          parent : parent,
          grand : allPlayers[_parentAddr].parent,
          great : allPlayers[_parentAddr].grand,
          income:0,
          allIncome : 0,
          totalInvest : msg.value,
          sons : 0,
          time : now
          });
      }else{
        allPlayers[msg.sender] = Player({
          self : msg.sender,
          parent : ZERO_ADDR,
          grand : ZERO_ADDR,
          great : ZERO_ADDR,
          income:0,
          allIncome : 0,
          totalInvest : msg.value,
          sons : 0,
          time : now
          });
      }
      newPlayFlag = true;
    }else{//old player, calc total
      allPlayers[msg.sender].totalInvest = allPlayers[msg.sender].totalInvest.add(msg.value);
    }

    emit logUserInvest(msg.sender, _parentAddr, newPlayFlag, msg.value, now);
  }
```
