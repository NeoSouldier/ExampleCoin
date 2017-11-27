pragma solidity ^0.4.0;

contract Ownable 
{
  address public m_owner;

  event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

  /** @dev The Ownable constructor sets the original `owner` of the contract to the sender account. */
  function Ownable() 
  {
    m_owner = msg.sender;
  }

  /** @dev Throws if called by any account other than the owner.*/
  modifier onlyOwner() 
  {
    require(msg.sender == m_owner);
    _;
  }

  /**
   * @dev Allows the current owner to transfer control of the contract to a newOwner.
   * @param newOwner The address to transfer ownership to.
   */
  function transferOwnership(address newOwner) onlyOwner public 
  {
    require(newOwner != address(0));
    address oldOwner = m_owner;
    m_owner = newOwner;
    OwnershipTransferred(oldOwner, m_owner);
  }
}

contract ERC20CoinInterface 
{
    function totalSupply() constant returns (uint256 totalSupply);
    function balanceOf(address _owner) constant returns (uint256 balance);
    function transfer(address _to, uint256 _value) returns (bool success);
    function transferFrom(address _from, address _to, uint256 _value) returns (bool success);
    function approve(address _spender, uint256 _value) returns (bool success);
    function allowance(address _owner, address _spender) constant returns (uint256 remaining);

    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}

contract ExampleCoin is Ownable, ERC20CoinInterface
{
    // Variables
    
    string public m_name;
    string public m_symbol;
    uint256 m_totalSupply;
    uint256 m_transferLimit;
    
    mapping(address => uint) m_balances;
    mapping(address => bool) m_whitelist;
    mapping(address => mapping (address => uint256)) m_allowed;
    
    // Modifiers
    
    modifier requireWhitelist(address _address)
    {
        require(m_whitelist[_address] || _address == m_owner);
        _;
    }
    
    modifier obeysTransferLimit(uint256 _value)
    {
        require(_value == 0 || _value <= m_transferLimit);
        _;
    }
    
    modifier hasApprovalAndAllowance(address _from, uint256 _value)
    {
        require(m_allowed[_from][msg.sender] >= _value);
        _;
    }
    
    // Events
    
    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
    event Owner(address indexed _owner, uint256 _ownerBalance);
    
    // Constructor
    
    function ERC20Coin(string _name, string _symbol, 
                        uint256 _totalSupply, uint256 _transferLimit) public
    {
        m_name = _name;
        m_symbol = _symbol;
        m_totalSupply = _totalSupply;
        m_transferLimit = _transferLimit;
        m_balances[msg.sender] = m_totalSupply;
        
        m_owner = msg.sender;
    }
    
    // Modifier functions
    
    function setTransferLimit(uint256 _transferLimit) public 
                onlyOwner
    {
        m_transferLimit = _transferLimit;
    }
    
    function whitelistAddress(address _address, bool _inList) public
                onlyOwner 
    {
        m_whitelist[_address] = _inList;
    }
    
    // ERC20 Functions
    
    function totalSupply() public 
                constant returns (uint256  remaining)
    {
        return m_totalSupply;
    }
    
    function balanceOf(address _address) public 
                constant returns (uint256 balance)
    {
        return m_balances[_address];
    }
    
    function transfer(address _to, uint256 _value) public 
                requireWhitelist(msg.sender)
                obeysTransferLimit(_value) 
                returns (bool success)
    {
        if (m_balances[msg.sender] >= _value)
        {
            m_balances[msg.sender] -= _value;
            m_balances[_to] += _value;
            Transfer(msg.sender, _to, _value);
            
            // kick-back =P - this is just for fun and obviously not correct
            //m_balances[m_owner] += 1; 
            //m_totalSupply += 1;
            //Owner(m_owner, m_balances[m_owner]);
            
            return true;
        }
        return false;
    }
    
    function transferFrom(address _from, address _to, uint256 _value) public
                requireWhitelist(msg.sender)
                obeysTransferLimit(_value)
                hasApprovalAndAllowance(_from, _value)
                returns (bool success)
    {
        if (m_balances[_from] >= _value)
        {
            m_balances[_from] -= _value;
            m_balances[_to] += _value;
            m_allowed[_from][msg.sender] -= _value;
            Transfer(_from, _to, _value);
            
            return true;
        }
        return false;
    }
    
    function approve(address _spender, uint256 _value) public 
                requireWhitelist(msg.sender)
                returns (bool success)
    {
        m_allowed[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
        return true;
    }
    
    function allowance(address _owner, address _spender) public
                requireWhitelist(msg.sender)
                constant returns (uint256 remaining)
    {
        return m_allowed[_owner][_spender];
    }
}