// SPDX-License-Identifier: MIT 
pragma solidity ^0.8.0;

contract Crowdfunding {
    string public nameOfCampaign;
    string public description;
    uint256 public goal;
    uint256 public deadlineDate;
    address public owner;
    bool public paused;

    enum CampaignState { Active, Successful, Failed }
    CampaignState public state;

    struct Tier {
        string name;
        uint256 amount;
        uint256 backers;
    }

    struct Backer {
        uint256 totalContribution;
        mapping (uint256 => bool) fundedTiers;
    }

    Tier[] public tiers;
    mapping (address => Backer) public backers;

    // Events for frontend / monitoring
    event Funded(address indexed backer, uint256 amount, uint256 tierIndex);
    event Refunded(address indexed backer, uint256 amount);
    event Withdrawn(address indexed owner, uint256 amount);
    event TierAdded(string name, uint256 amount);
    event TierRemoved(uint256 index);
    event CampaignPaused(bool isPaused);
    event DeadlineExtended(uint256 newDeadline);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }

    modifier campaignOpen() {
        require(state == CampaignState.Active, "Campaign is not active.");
        _;
    }

    modifier notPaused() {
        require(!paused, "Contract is paused");
        _;
    }

    modifier updateState() {
        checkAndUpdateCampaignState();
        _;
    }

    constructor(
        string memory _name,
        string memory _description,
        uint256 _goal,
        uint256 _durationInDays
    ) {
        nameOfCampaign = _name;
        description = _description;
        goal = _goal;
        deadlineDate = block.timestamp + (_durationInDays * 1 days);
        owner = msg.sender;
        state = CampaignState.Active;
    }

    function checkAndUpdateCampaignState() internal {
        if (state == CampaignState.Active) {
            if (block.timestamp >= deadlineDate) {
                state = address(this).balance >= goal ? CampaignState.Successful : CampaignState.Failed;
            } else {
                state = address(this).balance >= goal ? CampaignState.Successful : CampaignState.Active;
            }
        }
    }

    function fund(uint256 _tierIndex) public payable campaignOpen notPaused updateState {
        require(_tierIndex < tiers.length, "Invalid tier.");
        require(msg.value == tiers[_tierIndex].amount, "Incorrect amount.");

        tiers[_tierIndex].backers++;
        backers[msg.sender].totalContribution += msg.value;
        backers[msg.sender].fundedTiers[_tierIndex] = true;

        emit Funded(msg.sender, msg.value, _tierIndex);
    }

    function addTier(string memory _name, uint256 _amount) public onlyOwner {
        require(_amount > 0, "Amount should be greater than zero.");
        tiers.push(Tier(_name, _amount, 0));

        emit TierAdded(_name, _amount);
    }

    function removeTier(uint256 _index) public onlyOwner {
        require(_index < tiers.length, "Tier does not exist.");
        tiers[_index] = tiers[tiers.length - 1];
        tiers.pop();

        emit TierRemoved(_index);
    }

    function withdraw() public onlyOwner updateState {
        require(state == CampaignState.Successful, "Campaign not successful yet.");
        uint256 balance = address(this).balance;
        require(balance > 0, "No balance to withdraw.");

        payable(owner).transfer(balance);

        emit Withdrawn(owner, balance);
    }

    function getContractBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function refund() public updateState {
        require(state == CampaignState.Failed, "Refund not available.");
        uint256 amount = backers[msg.sender].totalContribution;
        require(amount > 0, "No contribution to refund");

        backers[msg.sender].totalContribution = 0;

        payable(msg.sender).transfer(amount);

        emit Refunded(msg.sender, amount);
    }

    function hasFundedTier(address _backer, uint256 _tierIndex) public view returns (bool) {
        return backers[_backer].fundedTiers[_tierIndex];
    }

    function getTiers() public view returns (Tier[] memory) {
        return tiers;
    }

    function togglePause() public onlyOwner {
        paused = !paused;
        emit CampaignPaused(paused);
    }

    function getCampaignStatus() public view returns (CampaignState) {
        if (state == CampaignState.Active && block.timestamp > deadlineDate) {
            return address(this).balance >= goal ? CampaignState.Successful : CampaignState.Failed;
        }
        return state;
    }

    function extendDeadline(uint256 _daysToAdd) public onlyOwner campaignOpen {
        deadlineDate += _daysToAdd * 1 days;
        emit DeadlineExtended(deadlineDate);
    }
}


