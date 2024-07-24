To implement the project as illustrated in the provided diagram using the existing repository, follow the steps below. We will integrate the fund management system, involving contributions, proposal creation, approval, and execution with IPFS storage for proposal documents.

### Directory Structure

We will use the existing directories where possible and create new directories only if necessary.

### 1. **Smart Contract: FundManager.sol**

Update the `FundManager.sol` smart contract to match the requirements of the project. This will involve creating multiple custom token contracts and implementing the roles of End Users, Executives, and Managers.

#### Updated `FundManager.sol` Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FundManager is Ownable {
    struct Proposal {
        address proposer;
        uint256 amount;
        string ipfsHash;
        bool approvedByUser;
        bool approvedByManager;
        bool executed;
    }

    IERC20 public token1;
    IERC20 public token2;
    IERC20 public token3;
    Proposal[] public proposals;
    mapping(uint256 => mapping(address => bool)) public approvals;

    address[] public managers;

    constructor(address token1Address, address token2Address, address token3Address) Ownable() {
        token1 = IERC20(token1Address);
        token2 = IERC20(token2Address);
        token3 = IERC20(token3Address);
    }

    function setManagers(address[] memory _managers) public onlyOwner {
        managers = _managers;
    }

    function contribute(uint256 amount, uint8 tokenType) external {
        if (tokenType == 1) {
            token1.transferFrom(msg.sender, address(this), amount);
        } else if (tokenType == 2) {
            token2.transferFrom(msg.sender, address(this), amount);
        } else if (tokenType == 3) {
            token3.transferFrom(msg.sender, address(this), amount);
        }
    }

    function submitProposal(uint256 amount, string memory ipfsHash) external {
        proposals.push(Proposal({
            proposer: msg.sender,
            amount: amount,
            ipfsHash: ipfsHash,
            approvedByUser: false,
            approvedByManager: false,
            executed: false
        }));
    }

    function approveProposalByUser(uint256 proposalId) external {
        require(!proposals[proposalId].approvedByUser, "Proposal already approved by user");
        proposals[proposalId].approvedByUser = true;
    }

    function approveProposalByManager(uint256 proposalId) external {
        require(!proposals[proposalId].approvedByManager, "Proposal already approved by manager");
        bool isManager = false;
        for (uint i = 0; i < managers.length; i++) {
            if (managers[i] == msg.sender) {
                isManager = true;
                break;
            }
        }
        require(isManager, "Only managers can approve");
        proposals[proposalId].approvedByManager = true;
    }

    function executeProposal(uint256 proposalId) external {
        Proposal storage proposal = proposals[proposalId];
        require(proposal.approvedByUser, "Proposal not approved by user");
        require(proposal.approvedByManager, "Proposal not approved by manager");
        require(!proposal.executed, "Proposal already executed");
        proposal.executed = true;
        // Transfer funds to the proposer. This example assumes all contributions are in token1
        token1.transfer(proposal.proposer, proposal.amount);
    }

    function getProposal(uint256 proposalId) external view returns (Proposal memory) {
        return proposals[proposalId];
    }
}
```

### 2. **Update Configuration Files**

Update `packages/valory/agents/learning_agent/aea-config.yaml`:

```yaml
name: learning_agent
skills:
  - valory/learning_abci:0.1.0
  - valory/learning_chained_abci:0.1.0
contracts:
  - your_namespace/fund_manager:0.1.0
```

### 3. **Integrate Fund Management Logic**

Add the fund management logic to existing files:

#### Update `behaviours.py`

Edit `packages/valory/skills/learning_abci/behaviours.py`:

```python
from aea.skills.behaviours import TickerBehaviour
from aea.helpers.ipfs.base import IPFSClient
from web3 import Web3

class FundManagementBehaviour(TickerBehaviour):
    def __init__(self, **kwargs):
        super().__init__(tick_interval=10.0, **kwargs)
        self.web3 = Web3(Web3.HTTPProvider('<your_infura_or_alchemy_url>'))
        self.contract = self.web3.eth.contract(address='<contract_address>', abi=<contract_abi>)
        self.ipfs_client = IPFSClient()

    def act(self) -> None:
        # Implement logic for submitting proposals, approving, and executing them
        pass

    def submit_proposal(self, amount, ipfs_hash):
        tx = self.contract.functions.submitProposal(amount, ipfs_hash).buildTransaction({
            'from': self.context.agent_address,
            'nonce': self.web3.eth.getTransactionCount(self.context.agent_address),
        })
        signed_tx = self.web3.eth.account.signTransaction(tx, private_key='<private_key>')
        tx_hash = self.web3.eth.sendRawTransaction(signed_tx.rawTransaction)
        self.context.logger.info(f"Proposal submitted with tx hash: {tx_hash.hex()}")

    def approve_proposal_by_user(self, proposal_id):
        tx = self.contract.functions.approveProposalByUser(proposal_id).buildTransaction({
            'from': self.context.agent_address,
            'nonce': self.web3.eth.getTransactionCount(self.context.agent_address),
        })
        signed_tx = self.web3.eth.account.signTransaction(tx, private_key='<private_key>')
        tx_hash = self.web3.eth.sendRawTransaction(signed_tx.rawTransaction)
        self.context.logger.info(f"Proposal approved by user with tx hash: {tx_hash.hex()}")

    def approve_proposal_by_manager(self, proposal_id):
        tx = self.contract.functions.approveProposalByManager(proposal_id).buildTransaction({
            'from': self.context.agent_address,
            'nonce': self.web3.eth.getTransactionCount(self.context.agent_address),
        })
        signed_tx = self.web3.eth.account.signTransaction(tx, private_key='<private_key>')
        tx_hash = self.web3.eth.sendRawTransaction(signed_tx.rawTransaction)
        self.context.logger.info(f"Proposal approved by manager with tx hash: {tx_hash.hex()}")

    def execute_proposal(self, proposal_id):
        tx = self.contract.functions.executeProposal(proposal_id).buildTransaction({
            'from': self.context.agent_address,
            'nonce': self.web3.eth.getTransactionCount(self.context.agent_address),
        })
        signed_tx = self.web3.eth.account.signTransaction(tx, private_key='<private_key>')
        tx_hash = self.web3.eth.sendRawTransaction(signed_tx.rawTransaction)
        self.context.logger.info(f"Proposal executed with tx hash: {tx_hash.hex()}")
```

#### Update `handlers.py`

Edit `packages/valory/skills/learning_abci/handlers.py`:

```python
from aea.skills.base import Handler

class FundManagementHandler(Handler):
    def setup(self):
        pass

    def handle(self, message):
        pass

    def teardown(self):
        pass
```

#### Update `models.py`

Edit `packages/valory/skills/learning_abci/models.py`:

```python
from aea.skills.base import Model

class FundManagementModel(Model):
    def setup(self):
        pass

    def teardown(self):
        pass
```

#### Update `skill.yaml`

Edit `packages/valory/skills/learning_abci/skill.yaml`:

```yaml
name: learning_abci
authors: valory
version: 0.1.0
license: Apache-2.0
behaviours:
  fund_management_behaviour:
    class_name: FundManagementBehaviour
handlers:
  fund_management_handler:
    class_name: FundManagementHandler
models:
  fund_management_model:
    class_name: FundManagementModel
contracts:
  - your_namespace/fund_manager:0.1.0
```

### 4. **FSM Specification and Additional Configurations**

Create an FSM specification file `fsm_specification.yaml` in the skill directory with relevant states and transitions. This will depend on the specific states and logic you want to implement for your fund management.

### 5. **Logging and Transaction Hashes**

Ensure that each transaction's hash is logged for tracking. Modify logging configuration if necessary to capture desired transaction hashes.

### 6. **Testing and Verification**

Test the new functionalities locally using a testnet. Ensure all transactions are correctly logged and the system behaves as expected.

### 7. **Final Adjustments**

Make any necessary adjustments based on test results. Document the new features and changes in the `README.md` and other relevant files.

### Summary

1. Update and deploy the `FundManager.sol` contract.
2. Modify `behaviours.py`, `handlers.py`, and `models.py` to integrate with the smart contract.
3. Update the skill configuration and FSM specification.
4. Test and verify the implementation.

This approach ensures your project aligns with the provided diagram while making the necessary changes and additions to the existing repository.
