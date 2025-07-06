// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// BlackOpalToken is a deflationary ERC20-like token with added security features.
// Features include:
// - Fixed initial supply assigned to the owner at deployment.
// - Transfer fees configurable by the owner (expressed in basis points).
// - Optional maximum transaction and wallet limits (anti-whale measures).
// - Blacklist functionality to block specific malicious addresses.
// - Pausable transfers for emergency response.
// - Governance address that can be transferred for decentralized control.
// - Emits standard ERC20 events (Transfer, Approval) and custom events for state changes.
// - Can safely receive ETH and allow owner/governance to withdraw it.
// Compatible with wallets, dApps, and exchanges that support ERC20.

abstract contract Context {
    function _msgSender() internal view virtual returns (address) {
        return msg.sender;
    }
}

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

interface IERC20Metadata is IERC20 {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}

contract BlackOpalToken is Context, IERC20, IERC20Metadata {
    string private constant _NAME = "BlackOpal";
    string private constant _SYMBOL = "BOP";
    uint8 private constant _DECIMALS = 10;

    uint256 private _totalSupply;
    uint256 public constant MAX_FEE_BPS = 200;  // 2%

    address public owner;
    address public governance;

    bool public paused;
    uint256 public feeBasisPoints = 100;  // 1%
    uint256 public maxTxAmount;
    uint256 public maxWalletAmount;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    mapping(address => bool) private _blacklist;

    event Paused(address indexed by);
    event Unpaused(address indexed by);
    event BlacklistUpdated(address indexed account, bool status);
    event FeeUpdated(uint256 bps);
    event MaxTxUpdated(uint256 amount);
    event MaxWalletUpdated(uint256 amount);
    event OwnershipTransferred(address indexed oldOwner, address indexed newOwner);
    event GovernanceTransferred(address indexed oldGov, address indexed newGov);
    event ETHReceived(address indexed sender, uint256 amount);
    event ETHWithdrawn(address indexed by, uint256 amount);

    modifier onlyGovOrOwner() {
        require(_msgSender() == owner || _msgSender() == governance, "Not authorized");
        _;
    }

    modifier notPaused() {
        require(!paused, "Token is paused");
        _;
    }

    constructor(address initialOwner) {
        require(initialOwner != address(0), "Zero owner");
        owner = initialOwner;
        _totalSupply = 1_000_000_000 * 10 ** _DECIMALS;
        _balances[initialOwner] = _totalSupply;
        emit Transfer(address(0), initialOwner, _totalSupply);
    }

    // Metadata
    function name() external pure override returns (string memory) {
        return _NAME;
    }

    function symbol() external pure override returns (string memory) {
        return _SYMBOL;
    }

    function decimals() external pure override returns (uint8) {
        return _DECIMALS;
    }

    // ERC20 core
    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function allowance(address owner_, address spender) external view override returns (uint256) {
        return _allowances[owner_][spender];
    }

    function approve(address spender, uint256 amount) external override notPaused returns (bool) {
        require(spender != address(0), "Zero address");
        _allowances[_msgSender()][spender] = amount;
        emit Approval(_msgSender(), spender, amount);
        return true;
    }

    function transfer(address to, uint256 amount) external override notPaused returns (bool) {
        _transfer(_msgSender(), to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override notPaused returns (bool) {
        uint256 currentAllowance = _allowances[from][_msgSender()];
        require(currentAllowance >= amount, "Allowance exceeded");
        unchecked {
            _allowances[from][_msgSender()] = currentAllowance - amount;
        }
        _transfer(from, to, amount);
        return true;
    }

    function _transfer(address from, address to, uint256 amount) internal {
        require(to != address(0), "Zero address");
        require(!_blacklist[from] && !_blacklist[to], "Blacklisted");
        require(_balances[from] >= amount, "Insufficient balance");

        if (maxTxAmount != 0) require(amount <= maxTxAmount, "Exceeds max tx");
        if (maxWalletAmount != 0) require(_balances[to] + amount <= maxWalletAmount, "Exceeds max wallet");

        uint256 fee = (amount * feeBasisPoints) / 10_000;
        uint256 sendAmount = amount - fee;

        unchecked {
            _balances[from] -= amount;
            _balances[to] += sendAmount;
        }

        emit Transfer(from, to, sendAmount);

        if (fee != 0) {
            _balances[owner] += fee;
            emit Transfer(from, owner, fee);
        }
    }

    // Governance
    function pause() external onlyGovOrOwner {
        require(!paused, "Already paused");
        paused = true;
        emit Paused(_msgSender());
    }

    function unpause() external onlyGovOrOwner {
        require(paused, "Not paused");
        paused = false;
        emit Unpaused(_msgSender());
    }

    function updateBlacklist(address account, bool status) external onlyGovOrOwner {
        require(account != address(0), "Zero address");
        if (_blacklist[account] != status) {
            _blacklist[account] = status;
            emit BlacklistUpdated(account, status);
        }
    }

    function updateFee(uint256 bps) external onlyGovOrOwner {
        require(bps <= MAX_FEE_BPS, "Fee too high");
        if (feeBasisPoints != bps) {
            feeBasisPoints = bps;
            emit FeeUpdated(bps);
        }
    }

    function updateMaxTx(uint256 amount) external onlyGovOrOwner {
        if (maxTxAmount != amount) {
            maxTxAmount = amount;
            emit MaxTxUpdated(amount);
        }
    }

    function updateMaxWallet(uint256 amount) external onlyGovOrOwner {
        if (maxWalletAmount != amount) {
            maxWalletAmount = amount;
            emit MaxWalletUpdated(amount);
        }
    }

    function transferOwnership(address newOwner) external onlyGovOrOwner {
        require(newOwner != address(0), "Zero address");
        if (newOwner != owner) {
            emit OwnershipTransferred(owner, newOwner);
            owner = newOwner;
        }
    }

    function setGovernance(address newGov) external onlyGovOrOwner {
        require(newGov != address(0), "Zero address");
        if (newGov != governance) {
            emit GovernanceTransferred(governance, newGov);
            governance = newGov;
        }
    }

    // ETH handling
    receive() external payable {
        emit ETHReceived(_msgSender(), msg.value);
    }

    fallback() external payable {
        emit ETHReceived(_msgSender(), msg.value);
    }

    function withdrawETH(uint256 amount) external onlyGovOrOwner {
        require(address(this).balance >= amount, "Insufficient ETH");
        payable(_msgSender()).transfer(amount);
        emit ETHWithdrawn(_msgSender(), amount);
    }
}
