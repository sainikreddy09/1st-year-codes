// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract SupplyChainTracking {
    
    // Enum for different stakeholder roles
    enum Role { None, Manufacturer, Distributor, Retailer, Consumer }
    
    // Enum for product status throughout the supply chain
    enum ProductStatus { 
        Manufactured, 
        InTransitToDistributor, 
        ReceivedByDistributor, 
        InTransitToRetailer, 
        ReceivedByRetailer, 
        Sold 
    }
    
    // Struct to represent a product
    struct Product {
        uint256 id;
        string name;
        string description;
        address manufacturer;
        uint256 manufactureDate;
        ProductStatus status;
        bool exists;
        string batchNumber;
        uint256 price;
    }
    
    // Struct to represent a stakeholder
    struct Stakeholder {
        address stakeholderAddress;
        string name;
        Role role;
        bool isActive;
        uint256 registrationDate;
    }
    
    // Struct to track product movement/transactions
    struct Transaction {
        uint256 productId;
        address from;
        address to;
        uint256 timestamp;
        ProductStatus newStatus;
        string location;
        string notes;
    }
    
    // State variables
    address public owner;
    uint256 public productCount;
    uint256 public transactionCount;
    
    // Mappings
    mapping(uint256 => Product) public products;
    mapping(address => Stakeholder) public stakeholders;
    mapping(uint256 => Transaction[]) public productTransactions;
    mapping(address => uint256[]) public stakeholderProducts;
    
    // Events
    event StakeholderRegistered(address stakeholder, string name, Role role);
    event ProductCreated(uint256 productId, string name, address manufacturer);
    event ProductTransferred(uint256 productId, address from, address to, ProductStatus newStatus);
    event ProductStatusUpdated(uint256 productId, ProductStatus status, string location);
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only contract owner can perform this action");
        _;
    }
    
    modifier onlyActiveStakeholder() {
        require(stakeholders[msg.sender].isActive, "Only active stakeholders can perform this action");
        _;
    }
    
    modifier onlyRole(Role _role) {
        require(stakeholders[msg.sender].role == _role, "Insufficient role permissions");
        _;
    }
    
    modifier productExists(uint256 _productId) {
        require(products[_productId].exists, "Product does not exist");
        _;
    }
    
    // Constructor
    constructor() {
        owner = msg.sender;
        productCount = 0;
        transactionCount = 0;
    }
    
    // Core Function 1: Register Stakeholder
    function registerStakeholder(
        address _stakeholder,
        string memory _name,
        Role _role
    ) public onlyOwner {
        require(_stakeholder != address(0), "Invalid stakeholder address");
        require(_role != Role.None, "Invalid role");
        require(!stakeholders[_stakeholder].isActive, "Stakeholder already registered");
        require(bytes(_name).length > 0, "Name cannot be empty");
        
        stakeholders[_stakeholder] = Stakeholder({
            stakeholderAddress: _stakeholder,
            name: _name,
            role: _role,
            isActive: true,
            registrationDate: block.timestamp
        });
        
        emit StakeholderRegistered(_stakeholder, _name, _role);
    }
    
    // Core Function 2: Create Product (Manufacturing)
    function createProduct(
        string memory _name,
        string memory _description,
        string memory _batchNumber,
        uint256 _price
    ) public onlyActiveStakeholder onlyRole(Role.Manufacturer) {
        require(bytes(_name).length > 0, "Product name cannot be empty");
        require(bytes(_batchNumber).length > 0, "Batch number cannot be empty");
        require(_price > 0, "Price must be greater than zero");
        
        productCount++;
        
        products[productCount] = Product({
            id: productCount,
            name: _name,
            description: _description,
            manufacturer: msg.sender,
            manufactureDate: block.timestamp,
            status: ProductStatus.Manufactured,
            exists: true,
            batchNumber: _batchNumber,
            price: _price
        });
        
        // Add to manufacturer's product list
        stakeholderProducts[msg.sender].push(productCount);
        
        // Record initial transaction
        productTransactions[productCount].push(Transaction({
            productId: productCount,
            from: address(0),
            to: msg.sender,
            timestamp: block.timestamp,
            newStatus: ProductStatus.Manufactured,
            location: "Manufacturing Facility",
            notes: "Product manufactured"
        }));
        
        transactionCount++;
        
        emit ProductCreated(productCount, _name, msg.sender);
        emit ProductStatusUpdated(productCount, ProductStatus.Manufactured, "Manufacturing Facility");
    }
    
    // Core Function 3: Transfer Product
    function transferProduct(
        uint256 _productId,
        address _to,
        ProductStatus _newStatus,
        string memory _location,
        string memory _notes
    ) public onlyActiveStakeholder productExists(_productId) {
        require(_to != address(0), "Invalid recipient address");
        require(stakeholders[_to].isActive, "Recipient is not an active stakeholder");
        require(_validateTransfer(_productId, msg.sender, _to, _newStatus), "Invalid transfer");
        
        // Update product status
        products[_productId].status = _newStatus;
        
        // Record transaction
        productTransactions[_productId].push(Transaction({
            productId: _productId,
            from: msg.sender,
            to: _to,
            timestamp: block.timestamp,
            newStatus: _newStatus,
            location: _location,
            notes: _notes
        }));
        
        // Update stakeholder product lists
        stakeholderProducts[_to].push(_productId);
        
        transactionCount++;
        
        emit ProductTransferred(_productId, msg.sender, _to, _newStatus);
        emit ProductStatusUpdated(_productId, _newStatus, _location);
    }
    
    // Internal function to validate transfers based on roles and current status
    function _validateTransfer(
        uint256 _productId,
        address _from,
        address _to,
        ProductStatus _newStatus
    ) internal view returns (bool) {
        Product memory product = products[_productId];
        Role fromRole = stakeholders[_from].role;
        Role toRole = stakeholders[_to].role;
        
        // Manufacturer to Distributor
        if (fromRole == Role.Manufacturer && toRole == Role.Distributor) {
            return product.status == ProductStatus.Manufactured && 
                   _newStatus == ProductStatus.InTransitToDistributor;
        }
        
        // In transit to received by distributor
        if (fromRole == Role.Manufacturer && toRole == Role.Distributor) {
            return product.status == ProductStatus.InTransitToDistributor && 
                   _newStatus == ProductStatus.ReceivedByDistributor;
        }
        
        // Distributor to Retailer
        if (fromRole == Role.Distributor && toRole == Role.Retailer) {
            return product.status == ProductStatus.ReceivedByDistributor && 
                   _newStatus == ProductStatus.InTransitToRetailer;
        }
        
        // In transit to received by retailer
        if (fromRole == Role.Distributor && toRole == Role.Retailer) {
            return product.status == ProductStatus.InTransitToRetailer && 
                   _newStatus == ProductStatus.ReceivedByRetailer;
        }
        
        // Retailer to Consumer
        if (fromRole == Role.Retailer && toRole == Role.Consumer) {
            return product.status == ProductStatus.ReceivedByRetailer && 
                   _newStatus == ProductStatus.Sold;
        }
        
        return false;
    }
    
    // Update product status (for same stakeholder updates)
    function updateProductStatus(
        uint256 _productId,
        ProductStatus _newStatus,
        string memory _location,
        string memory _notes
    ) public onlyActiveStakeholder productExists(_productId) {
        require(_isAuthorizedForProduct(_productId, msg.sender), "Not authorized for this product");
        
        products[_productId].status = _newStatus;
        
        productTransactions[_productId].push(Transaction({
            productId: _productId,
            from: msg.sender,
            to: msg.sender,
            timestamp: block.timestamp,
            newStatus: _newStatus,
            location: _location,
            notes: _notes
        }));
        
        transactionCount++;
        
        emit ProductStatusUpdated(_productId, _newStatus, _location);
    }
    
    // Check if stakeholder is authorized for a product
    function _isAuthorizedForProduct(uint256 _productId, address _stakeholder) internal view returns (bool) {
        uint256[] memory userProducts = stakeholderProducts[_stakeholder];
        for (uint256 i = 0; i < userProducts.length; i++) {
            if (userProducts[i] == _productId) {
                return true;
            }
        }
        return false;
    }
    
    // Get product details
    function getProduct(uint256 _productId) public view productExists(_productId) returns (
        uint256 id,
        string memory name,
        string memory description,
        address manufacturer,
        uint256 manufactureDate,
        ProductStatus status,
        string memory batchNumber,
        uint256 price
    ) {
        Product memory product = products[_productId];
        return (
            product.id,
            product.name,
            product.description,
            product.manufacturer,
            product.manufactureDate,
            product.status,
            product.batchNumber,
            product.price
        );
    }
    
    // Get product transaction history
    function getProductHistory(uint256 _productId) public view productExists(_productId) returns (Transaction[] memory) {
        return productTransactions[_productId];
    }
    
    // Get stakeholder information
    function getStakeholder(address _stakeholder) public view returns (
        string memory name,
        Role role,
        bool isActive,
        uint256 registrationDate
    ) {
        Stakeholder memory stakeholder = stakeholders[_stakeholder];
        return (stakeholder.name, stakeholder.role, stakeholder.isActive, stakeholder.registrationDate);
    }
    
    // Get products owned by a stakeholder
    function getStakeholderProducts(address _stakeholder) public view returns (uint256[] memory) {
        return stakeholderProducts[_stakeholder];
    }
    
    // Verify product authenticity
    function verifyProduct(uint256 _productId, string memory _batchNumber) public view productExists(_productId) returns (bool) {
        return keccak256(abi.encodePacked(products[_productId].batchNumber)) == keccak256(abi.encodePacked(_batchNumber));
    }
    
    // Deactivate stakeholder (only owner)
    function deactivateStakeholder(address _stakeholder) public onlyOwner {
        require(stakeholders[_stakeholder].isActive, "Stakeholder is not active");
        stakeholders[_stakeholder].isActive = false;
    }
    
    // Reactivate stakeholder (only owner)
    function reactivateStakeholder(address _stakeholder) public onlyOwner {
        require(!stakeholders[_stakeholder].isActive, "Stakeholder is already active");
        stakeholders[_stakeholder].isActive = true;
    }
}
