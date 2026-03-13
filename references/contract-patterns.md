# X Layer Contract Patterns

## Hardhat Config (TypeScript)

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import '@okxweb3/hardhat-explorer-verify';

const config: HardhatUserConfig = {
  solidity: "0.8.24",
  paths: { sources: "./contracts" },
  networks: {
    "xlayer-testnet": {
      url: process.env.XLAYER_TESTNET_RPC_URL || "https://testrpc.xlayer.tech/terigon",
      chainId: 1952,
      accounts: process.env.DEPLOYER_PRIVATE_KEY ? [process.env.DEPLOYER_PRIVATE_KEY] : [],
    },
    "xlayer-mainnet": {
      url: process.env.XLAYER_RPC_URL || "https://rpc.xlayer.tech",
      chainId: 196,
      accounts: process.env.DEPLOYER_PRIVATE_KEY ? [process.env.DEPLOYER_PRIVATE_KEY] : [],
    },
  },
  okxweb3explorer: {
    apiKey: process.env.OKLINK_API_KEY,
  },
};
export default config;
```

## Deploy
```bash
npx hardhat run scripts/deploy.ts --network xlayer-testnet
npx hardhat run scripts/deploy.ts --network xlayer-mainnet
```

## Contract Verification

Standard contract:
```bash
npm install @okxweb3/hardhat-explorer-verify
npx hardhat okverify --network xlayer-mainnet <CONTRACT_ADDRESS>
```

Proxy contract (UUPS/Transparent):
```bash
npx hardhat okverify --network xlayer-mainnet --contract contracts/File.sol:ContractName --proxy <PROXY_ADDRESS>
```

Note: Wait at least 1 minute after deploy. OKLink API key: https://www.oklink.com
Note: Command name is `okverify` (without proxy) and `okverify` + `--proxy` flag (with proxy). Older docs may show `okxverify` — same command.

Alternative etherscan plugin (chainId 196):
- apiURL: https://www.oklink.com/api/v5/explorer/contract/verify-source-code-plugin/XLAYER
- browserURL: https://www.oklink.com/xlayer

Testnet (chainId 1952 — see `network-config.md` for chainId notes):
- apiURL: https://www.oklink.com/api/v5/explorer/contract/verify-source-code-plugin/XLAYER_TESTNET
- browserURL: https://www.oklink.com/xlayer-test

---

## Foundry Config & Deploy

### foundry.toml
```toml
[profile.default]
src = "contracts"
out = "out"
libs = ["node_modules"]
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 200
evm_version = "cancun"

[rpc_endpoints]
xlayer = "https://rpc.xlayer.tech"
xlayer_testnet = "https://testrpc.xlayer.tech/terigon"

[etherscan]
xlayer = { key = "${OKLINK_API_KEY}", url = "https://www.oklink.com/api/v5/explorer/contract/verify-source-code-plugin/XLAYER" }
```

### Foundry Deploy
```bash
forge script script/Deploy.s.sol --rpc-url xlayer --broadcast --verify
```

### Hardhat + Foundry Together
```bash
npm install --save-dev @nomicfoundation/hardhat-foundry
```
```typescript
// hardhat.config.ts
import "@nomicfoundation/hardhat-foundry";
```

---

## UUPS Proxy Pattern

### Deploy
```solidity
// contracts/MyContractV1.sol
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract MyContractV1 is UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() { _disableInitializers(); }

    function initialize(uint256 _value) public initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
        value = _value;
    }

    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}
}
```

### Upgrade
```solidity
// contracts/MyContractV2.sol
contract MyContractV2 is MyContractV1 {
    uint256 public newField;  // New field — existing storage layout must NOT be broken!

    function reinitialize(uint256 _newField) public reinitializer(2) {
        newField = _newField;
    }
}
```

### Storage Gap Pattern
Upgradeable contracts in an inheritance chain must reserve storage slots to prevent collisions:
```solidity
contract MyContractV1 is UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;

    // Reserve 50 storage slots for future variables in this contract
    uint256[49] private __gap;  // 49 because `value` uses 1 slot
}
```
Without `__gap`, adding variables to a base contract shifts storage in child contracts, corrupting data.

### Critical Rules
- `constructor` must always call `_disableInitializers()`
- `initialize` function must use `initializer` modifier
- During upgrade: NEVER delete or reorder existing storage variables
- Always append new variables at the end
- Use `uint256[N] private __gap` in every upgradeable base contract
- Before upgrade: check storage layout with `@openzeppelin/upgrades-core`
- Production: use timelock + multisig for `_authorizeUpgrade`, not a single EOA

### Contract Size
- EIP-170 limit: 24,576 bytes deployed bytecode
- Check with `forge build --sizes` or `npx hardhat compile`
- Details → `gas-optimization.md`

---

## ERC721 Deploy & Verify

### Contract
```solidity
// contracts/MyNFT.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable2Step.sol";

contract MyNFT is ERC721, ERC721URIStorage, Ownable2Step {
    uint256 private _nextTokenId;
    uint256 public constant MAX_SUPPLY = 10_000;
    uint256 public mintPrice = 0.01 ether; // 0.01 OKB

    constructor() ERC721("MyNFT", "MNFT") Ownable(msg.sender) {}

    function mint(string calldata uri) external payable {
        require(msg.value >= mintPrice, "Insufficient OKB");
        require(_nextTokenId < MAX_SUPPLY, "Max supply reached");

        uint256 tokenId = _nextTokenId++;
        _safeMint(msg.sender, tokenId);
        _setTokenURI(tokenId, uri);
    }

    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No balance");
        (bool success,) = payable(owner()).call{value: balance}("");
        require(success, "Transfer failed");
    }

    // Required overrides
    function tokenURI(uint256 tokenId) public view override(ERC721, ERC721URIStorage) returns (string memory) {
        return super.tokenURI(tokenId);
    }

    function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC721URIStorage) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
}
```

### Deploy Script (Hardhat)
```typescript
import { ethers } from "hardhat";

async function main() {
    if (!process.env.DEPLOYER_PRIVATE_KEY) {
        throw new Error("DEPLOYER_PRIVATE_KEY env variable required");
    }

    const MyNFT = await ethers.getContractFactory("MyNFT");
    const nft = await MyNFT.deploy();
    await nft.waitForDeployment();
    console.log("MyNFT deployed to:", await nft.getAddress());
}

main().catch(console.error);
```

### Verify
```bash
npx hardhat okverify --network xlayer-mainnet <NFT_CONTRACT_ADDRESS>
```
