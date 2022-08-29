---
layout:     post
title:      OpenZeppelin分析三
subtitle:   Governance源码解析
date:       2022-08-29
author:     Emo
header-img: img/unimelb-bd.png
catalog: true
mermaid: true
tags:
- OpenZeppelin
---

> OpenZeppelin
> 
> [EMO's Blog](https://emosama.github.io/)

# 简介
Governance板块，主要是为了方便用户构建自己的治理合约。所谓的治理合约，其实就是围绕 Proposal 展开的，简要归纳下来它的流程如下
1. 提起 Proposal
2. Proposal 进入投票阶段，成员利用 token 对 Proposal 进行投票
3. 投票阶段结束后，进行计票
4. Proposal 如果通过了投票，需要等待一段时间的公示期 (如果设置了timelock的话)
5. 执行 Proposal


Governance 中的核心文件是 `Governor.sol`，这个是一个抽象的合约，其也是 `IGovernor.sol` 的实现。`Governor.sol` 无法直接使用，但是其中对于整个治理合约的流程和功能进行了统一的规范和一定程度上的实现。对于没有实现的函数，我们在使用这个文件去构建我们自己的治理合约的时候需要自行进行设计实现，当然 `OpenZeppelin` 在一定程度上也提供了一些模块的实现，我们也可以自己选择应用在自己的治理合约当中。**对于函数我们进行以下模块分类**
- 辅助模块：这是我自己命的名，这一部分的对外函数都是 `view` 或者 `pure` 的
    - Core Module
    - User-Config Module
    - Reputation Module
    - Voting Module
- 执行模块：这一部分才是实际的功能函数，会改变 `state variable` 也会抛出 `event` 的

- 扩展模块 (非必须的，对于治理提出一些新的需求)
	- timelock：这是一个扩展的例子，他的主要功能就是给通过投票的 Proposal 增加一个执行前的公示期，只有当公示期过了以后，才能进入执行。

上面的 **模块设计** 部分中的三个模块是实现一个治理合约必须要去完成的地方
# 详细解析
接下来的分析主要着重于 `Governor.sol` 这个合约，其继承于
- Context 主要是封装了对 `msg.value & msg,data` 的访问
- ERC165 标准接口检测
- EIP712 对类型化结构化数据进行散列和签名的标准
- IGovernor `Governor.sol` 的接口文件，主要定义了对外提供的功能函数
- IERC721Receiver 
- IERC1155Receiver

接下的分析主要以文件 `IGovernor.sol` 为起点，结合其在 `Governor.sol` 中的实现进行分析

## IGovernor解析
### 流程状态分析
我们知道 Governance 中的主要流程就是针对于 Proposal 进行处理，那么每一个 Proposal 就代表一个流程，而流程的周期被在 Proposal 的状态描述中可以看出来
```solidity
enum ProposalState {
    Pending, // 等待状态，开始投票之前
    Active, // 开始投票了
    Canceled, // 提案被取消了
    Defeated, // 提案失败了
    Succeeded, // 提案成功了
    Queued, // 提案进入排队
    Expired, // 提案到期了
    Executed // 提案被执行
}
```

### 事件分析
Governor中的event事件主要有两类，这些时间的抛出都是在 **功能模块** 中的函数里抛出的

- **和Proposal状态相关的事件**
```solidity
event ProposalCreated( // 当一个proposal被创建的时候抛出
	uint256 proposalId,
	address proposer,
	address[] targets,
	uint256[] values,
	string[] signatures,
	bytes[] calldatas,
	uint256 startBlock,
	uint256 endBlock,
	string description
);

event ProposalCanceled( // 当proposal被取消的时候抛出
	uint256 proposalId
);

event ProposalExecuted( // 当proposal被执行的时候抛出
	uint256 proposalId
);
```
- **和vote投票相关的事件**
```solidity
event VoteCast( // 投票的时候抛出
	address indexed voter, 
	uint256 proposalId, 
	uint8 support, 
	uint256 weight, 
	string reason
);
 
event VoteCastWithParams( // 带有参数的投票的时候抛出
    address indexed voter,
    uint256 proposalId,
    uint8 support,
    uint256 weight,
    string reason,
    bytes params
);
```

### 函数分析
我们知道 `Governor.sol` 继承了 `IGovernor.sol` ，那么我们先关注于 `IGovernor.sol` 中定义的函数，然后再去看 `Governor.sol` 自己提出了哪些新的对外的函数
在分析具体的函数实现之前，先对所有对外开放的函数的大致功能和作用进行一个总览，基本上对内的inetrnal的函数都是被对外的函数调用去具体的执行功能的。
这一部分函数的数量比较多。我们主要根据 `Governor.sol`实现与否分为两类
- 已经完整实现的
- 需要用户自己或者引入外部模块来实现的

而后在这两个大类中，我们又根据上面 [简介](#简介) 中定义的模块进行了小分类。下面是一个小的预览目录
- [已经实现的](#已经实现的)
    - [辅助模块](#辅助模块) (pure或view函数)
        - [Core Module](#core-module)
    - [功能模块](#功能模块) (非pure或view函数)
- [待完善的](#待完善的)
    - [辅助模块](#辅助模块) (pure或view函数)
        - [Core Module](#core-moudle)
        - [User-Config Module](#user-config-module)
        - [Reputation Module](#reputation-module)
        - [Voting Module](#voting-module)
    - [功能模块](#功能模块) (非pure或view函数)

我们可以看到，得到了完整实现的模块，主要就是 **core Module** 和 **功能模块** 中的一部分，其他模块基本上全部需要自己去完善。而且我们还可以看到
- 辅助模块中都是 pure 或者 view 类型的函数
- 功能模块中则都是会对 state variable 产生影响的函数

#### 已经实现的
##### 辅助模块
###### Core Module
```solidity
// 返回名字
function name() public view virtual returns (string memory);

// 返回版本
function version() public view virtual returns (string memory);

// 传入 Proposal 的所有信息，然后返回一个他们整合后的keccak256加密后的hash值
// 这个值就是这个 proposal 的 id
function hashProposal(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 descriptionHash
) public pure virtual override returns (uint256);

// snapshot其实就是返回vote开始的区块的高度
function proposalSnapshot(uint256 proposalId) public view virtual returns (uint256);

// deadline其实就是返回vote结束的区块的高度
function proposalDeadline(uint256 proposalId) public view virtual returns (uint256);
```

##### 功能模块
```solidity
// 发起一个Proposal
function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) public virtual returns (uint256 proposalId);

// 执行一个Proposal
// 涉及到三个内部函数 _beforeExecute(), _execute(), _afterExecute()
function execute(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) public payable virtual returns (uint256 proposalId);
```

#### 待完善的
这里面的外部函数，要么是 `Governor.sol` 中根本就没有去实现，要么就是其关联的内部函数没有被实现
##### 辅助模块
###### Core Moudle
```solidity
function state(uint256 proposalId) public view virtual returns (ProposalState);
```

###### User-Config Module
```solidity
function votingDelay() public view virtual returns (uint256);
function votingPeriod() public view virtual returns (uint256);
function quorum(uint256 blockNumber) public view virtual returns (uint256);
```

###### Reputation Module
```solidity
// 没有传入 params，使用默认的_defaultParams()
function getVotes(address account, uint256 blockNumber) public view virtual returns (uint256);

function getVotesWithParams(
        address account,
        uint256 blockNumber,
        bytes memory params
    ) public view virtual returns (uint256);
```

###### Voting Module
```solidity
function COUNTING_MODE() public pure virtual returns (string memory);

function hasVoted(uint256 proposalId, address account) public view virtual returns (bool);
```

##### 功能模块
```solidity
function castVote(uint256 proposalId, uint8 support) public virtual returns (uint256 balance);

function castVoteWithReason(
        uint256 proposalId,
        uint8 support,
        string calldata reason
    ) public virtual returns (uint256 balance);

function castVoteWithReasonAndParams(
        uint256 proposalId,
        uint8 support,
        string calldata reason,
        bytes memory params
    ) public virtual returns (uint256 balance);


function castVoteBySig(
        uint256 proposalId,
        uint8 support,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual returns (uint256 balance);


function castVoteWithReasonAndParamsBySig(
        uint256 proposalId,
        uint8 support,
        string calldata reason,
        bytes memory params,
        uint8 v,
        bytes32 r,state
        bytes32 s
    ) public virtual returns (uint256 balance);
```

## Governor解析
### Proposal结构
```solidity
struct ProposalCore {
    Timers.BlockNumber voteStart; // 记录这个Proposal的投票开始时间(区块号)
    Timers.BlockNumber voteEnd; // 记录这个Proposal的投票结束时间(区块号)
    bool executed; // 这个Proposal是否被执行
    bool canceled; // 这个Proposal是否被取消
}
```
我们可以看到对于一个 *Proposal*，它包含的信息其实是没有被保存在 *state variables* 中的。我们再来看看，发起一个 *Proposal* 的时候，实际提交的信息有哪些，这些信息可以在 `propose()` 这个函数的定义中找到
```solidity
function propose(
        address[] memory targets, // 提案操作的一系列地址
        uint256[] memory values, // 提案操作的一系列value，也就是转多少钱
        bytes[] memory calldatas, // 提案操作的一系列地址中的具体函数
        string memory description // 对于提案的描述
    ) public virtual override returns (uint256);
```
可以看到除了一个 **描述** 以外，其他的变量都是和 **操作** 相关的。所以我们可以知道一个发起的 Proposal 就是以下组成
- 描述 (description)
- 操作 (targets values calldatas)

描述就不用说了，我们来看看操作到底是什么吧。其实简单来说这些操作就是一个个 *transaction*，也就是定义了一系列对于外部合约的调用。*targets，values，calldatas* 都是数组，而且他们的长度必须是相等的，因为每一个 `(target, value, calldata)` 定义了一个 *transaction*。他的具体行为如下
> target.call {value: value} (calldata) // 就是对应组成一个 call function

### State Variables解析
我们知道合约本质上就是一个数据库模型，我们的一切行为都是在和这个数据库里定义的 **State Variable** 进行互动。那么我们就先来看看 Governor 这个数据库存了哪些数据吧
```solidity
bytes32 public constant BALLOT_TYPEHASH = keccak256("Ballot(uint256 proposalId,uint8 support)");

bytes32 public constant EXTENDED_BALLOT_TYPEHASH =
    keccak256("ExtendedBallot(uint256 proposalId,uint8 support,string reason,bytes params)");

// 治理合约的名字
string private _name;

// 用来存 Proposal 的结构 (proposalId --> proposalCore)
// 我们知道 proposalCore 里实际是没有 Proposal 的执行信息的
mapping(uint256 => ProposalCore) private _proposals;

// 这个队列是一个追踪队列，我们知道操作的本质是一系列的Transaction
// 而Transaction则是 (target, value, calldata) 组成的 target.call {value: value} (calldata)
// 当 target = this，即治理合约本身的时候，我们会将 calldata 记录下来
// 而当合约中被 onlyGovernance 保护的函数被外部合约调用的时候，我们需要确保这个操作是被记录在这个队列当中的
// 这个变量是和 Modifier onlyGovernance() 协作的，在后面将 onlyGovernance 会详细讲
DoubleEndedQueue.Bytes32Deque private _governanceCall;
```
我们可以看到其实 Governor 合约看似复杂，但是本质上的信息交互并不复杂。这是因为投票，计票等相关的功能都委托给其他模块去设计了，Governor 本身只包含了和 Proposals 相关的东西。
- _proposals，这里存储的是所有被提出的 proposal，但是这里的 proposal 只包含一些流程相关的信息(如[Proposal结构](#Proposal结构)中描述)，其本身的详细信息如，操作和描述等并没有记载下来。
- _governanceCall，这是一个白名单，proposal 中的 **operation (target, value, calldata)** 如果满足
    - **target == this** => target 是合约本身
    - **_executor() != address(this)** executor不是合约本身，因为如果executor是合约本身的话，就是可以说是一个内部调用了，根本不需要限制什么。

    这个 **operation** 里的 **calldata** 便会被记录到 **_governanceCall** 中。我们知道这里的 **calldata** 记录的是对本合约的特定函数的访问。这便代表这个访问操作被添加到了白名单中了。而对这个白名单的访问，或者说白名单权限确认的行为发生在后面的 **Modifier onlyGovernance()** 中。

### Modifier解析
#### onlyGovernance()
```solidity
modifier onlyGovernance() {
    require(_msgSender() == _executor(), "Governor:  onlyGovernance"); // 调用者必须是executor
    if (_executor() != address(this)) {
        bytes32 msgDataHash = keccak256(_msgData());
        // loop until popping the expected operation - throw if deque is empty (operation not authorized)
        while (_governanceCall.popFront() != msgDataHash) {} // 调用函数必须在白名单里
    }
    _;
}
```
我们可以看到，当对一个被 onlyGoverance() 保护的函数被访问的时候，需要满足以下条件
- **msg.sender == _executor()**
- **msg.data record in _governanceCall**

才能够进入函数中。

#### relay()
```solidity
function relay(
    address target,
    uint256 value,
    bytes calldata data
) external virtual onlyGovernance {
    Address.functionCallWithValue(target, data, value); // 转钱函数
}
```
上面这个函数是一个本合约往外部合约转账的函数，需要对外暴露供外部合约调用，但是有需要给其设计一个权限控制方案。**onlyGovernance** 便是这个权限控制方案。描述出来就是，想要调用这个函数，只有通过提出 proposal 的方式。

#### execute()
**_governanceCall** 白名单的注册，就发生在这个函数里面，我们可以在这里看到，**Governor** 是如何设计出的一个安全的白名单注册流程
```solidity
function execute(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) public payable virtual override returns (uint256) {
        uint256 proposalId = hashProposal(targets, values, calldatas, descriptionHash);

        ProposalState status = state(proposalId);
        require(
            status == ProposalState.Succeeded || status == ProposalState.Queued,
            "Governor: proposal not successful"
        );
        _proposals[proposalId].executed = true;

        emit ProposalExecuted(proposalId);

        _beforeExecute(proposalId, targets, values, calldatas, descriptionHash); // 注册到_governanceCall中
        _execute(proposalId, targets, values, calldatas, descriptionHash); // 执行 operations
        _afterExecute(proposalId, targets, values, calldatas, descriptionHash); // 清空_governanceCall，防止权限泄露

        return proposalId;
    }
```
我们可以看到 **_beforeExecute** 和 **_afterExecute** 是如何配合的
- **beforeExecute**
```solidity
function _beforeExecute(
    uint256, /* proposalId */
    address[] memory targets,
    uint256[] memory, /* values */
    bytes[] memory calldatas,
    bytes32 /*descriptionHash*/
) internal virtual {
    if (_executor() != address(this)) { // 1. executor不是合约本身
        for (uint256 i = 0; i < targets.length; ++i) {
            if (targets[i] == address(this)) { // 2. target是合约本身
                // calldata注册到白名单中，这里没有限制calldata必须是被onlyGovernance包裹的
                // 因为无法判断，calldata只是一个函数的签名，用来定位函数的
                // 所以无法从过calldata判断其是不是onlyGovernance的
                _governanceCall.pushBack(keccak256(calldatas[i]));
            }
        }
    }
}
```
- **_afterExecute**
```solidity
function _afterExecute(
    uint256, /* proposalId */
    address[] memory, /* targets */
    uint256[] memory, /* values */
    bytes[] memory, /* calldatas */
    bytes32 /*descriptionHash*/
) internal virtual {
    if (_executor() != address(this)) { // 1. executor不是合约本身，感觉这个条件不是很必要，和下面的条件2一定意义上重复了
        if (!_governanceCall.empty()) { // 2. 如果不是空的
            _governanceCall.clear(); // 清空
        }
    }
}
```
我们可以看到，_governanceCall的注册，执行和清空是在一个事务里面发生的，流程如下
- 符合条件的operations注册白名单
- 执行operations
- 清空白名单

这么做的目的是，如果流程之间有间隙话，可以看如下 operation：
> (target: this, value: 10, calldata: relay())

keccak256('relay()')被存入了白名单，如果在执行这个操作之前，exector 自发的发起了如下操作
> (target: this, value: 20, calldata: relay())

我们可以看到 value 被修改为了 20，可是我们可以看到其依旧可以通过白名单检测，因为白名单只对两个信息进行检测
- 是不是executor
- calldata和msg.data能不能对应上

其没有对 value 进行什么特别的检测。所以我们需要避免其被冒名顶替。而解决的方法就是，即时注册，即时执行，即时清空，避免白名单泄露。

### Function解析
这里主要对功能模块和已经实现的辅助模块进行分析。

#### 功能模块
这里主要分析 **功能模块** 中的两个函数，propse 和 execute。如果按照面向对象的设计模式来理解的话，可以把这两个函数都理解成 **模板方法模式**
##### propose
```solidity
function propose(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    string memory description
) public virtual override returns (uint256) {
    // 有没有资格成为proposer
    require(
        getVotes(_msgSender(), block.number - 1) >= proposalThreshold(),
        "Governor: proposer votes below proposal threshold"
    );
    
    // 计算proposal的id
    uint256 proposalId = hashProposal(targets, values, calldatas, keccak256(bytes(description)));

    // targets，values，calldatas的需要相等
    require(targets.length == values.length, "Governor: invalid proposal length");
    require(targets.length == calldatas.length, "Governor: invalid proposal length");
    require(targets.length > 0, "Governor: empty proposal");

    // 去_proposals里创建一个ProposalCore，里面只记录proposal流程相关的信息，而不会去记录proposal执行相关的具体信息。
    ProposalCore storage proposal = _proposals[proposalId];
    // 判断这个ProposalCore的voteStart有没有设置，其实就是判断这个Proposal有没有被提出过
    require(proposal.voteStart.isUnset(), "Governor: proposal already exists");

    // 获取两个信息，开始投票的时间 和 结束投票的时间。
    // 相关的 votingDelay() 和 votingPeriod() 函数需要用户自己去设计
    uint64 snapshot = block.number.toUint64() + votingDelay().toUint64();
    uint64 deadline = snapshot + votingPeriod().toUint64();

    // 把上面获取到的信息，存到ProposalCore中对应的voteStart和voteEnd
    proposal.voteStart.setDeadline(snapshot);
    proposal.voteEnd.setDeadline(deadline);

    // 抛出ProposalCreated事件
    emit ProposalCreated(
        proposalId,
        _msgSender(),
        targets,
        values,
        new string[](targets.length),
        calldatas,
        snapshot,
        deadline,
        description
    );

    return proposalId;
}
```

##### execute
```solidity
function execute(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 descriptionHash
) public payable virtual override returns (uint256) {
    // 根据proposal的执行信息，计算proposl的id
    uint256 proposalId = hashProposal(targets, values, calldatas, descriptionHash);

    // 获取propsal当前的state
    ProposalState status = state(proposalId);
    
    // state需要处在 Succeeded 或者 Queued
    require(
        status == ProposalState.Succeeded || status == ProposalState.Queued,
        "Governor: proposal not successful"
    );
    _proposals[proposalId].executed = true;

    // 抛出ProposalExecuted事件
    emit ProposalExecuted(proposalId);

    // _beforeExecute前面说过了，就是添加txs白名单
    _beforeExecute(proposalId, targets, values, calldatas, descriptionHash);
    // 执行proposal中提出的operation中的txs
    _execute(proposalId, targets, values, calldatas, descriptionHash);
    // 清空白名单
    _afterExecute(proposalId, targets, values, calldatas, descriptionHash);

    return proposalId;
}
```
上面 _beforeExecute，_execute，_afterExecute 都是可以被 *virtual* 的，但是 Governor 中也已经对他们进行了完整实现的。这里只讲一下最重要的 **_execute** ，其他两个在之前已经讲过了。
```solidity
function _execute(
    uint256, /* proposalId */
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 /*descriptionHash*/
) internal virtual {
    string memory errorMessage = "Governor: call reverted without message";
    for (uint256 i = 0; i < targets.length; ++i) { // 遍历operation里的txs
        (bool success, bytes memory returndata) = targets[i].call{value: values[i]}(calldatas[i]); // 通过call发起tx
        Address.verifyCallResult(success, returndata, errorMessage); // 对call的返回结果进行处理，其实如果success为true，便没啥特别的处理。但是如果是false的话，就会特别处理以决定抛出什么异常信息
    }
}
```

#### core module
这里的函数都是些 pure 或者 view 的函数，整体逻辑比较简单，直接看一看代码吧。
- [name](#name)
- [version](#version)
- [hashProposal](#hashProposal)
- [proposalSnapshot](#proposalSnapshot)
- [proposalDeadline](#proposalDeadline)

#### name
```solidity
function name() public view virtual override returns (string memory) {
    return _name; // 返回_name
}
```

#### version
```solidity
function version() public view virtual override returns (string memory) {
    return "1"; // 直接返回写死的值
```

#### hashProposal
```solidity
function hashProposal(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 descriptionHash
) public pure virtual override returns (uint256) {
    return uint256(keccak256(abi.encode(targets, values, calldatas, descriptionHash))); // 使用keccak256进行hash
}
```

#### proposalSnapshot
```solidity
function proposalSnapshot(uint256 proposalId) public view virtual override returns (uint256) {
    return _proposals[proposalId].voteStart.getDeadline(); // 获取voteStart
}
```

#### proposalDeadline
```solidity
function proposalDeadline(uint256 proposalId) public view virtual override returns (uint256) {
    return _proposals[proposalId].voteEnd.getDeadline(); // 获取voteEnd
}
```
