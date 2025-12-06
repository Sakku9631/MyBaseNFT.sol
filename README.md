# MyBaseNFT.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

/// Simple ERC-721 NFT for deploying on Base (owner-controlled mint + public sale).
/// Imports OpenZeppelin from raw GitHub URLs so Remix can fetch them.
///
/// Features:
/// - Owner-only mint (batch)
/// - Public mint (payable) with toggle and price
/// - Max supply cap
/// - Base URI management
/// - Owner withdraw
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/access/Ownable.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/utils/Counters.sol";

contract MyBaseNFT is ERC721Enumerable, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIdCounter;

    string private _baseTokenURI;
    uint256 public immutable maxSupply;
    uint256 public mintPrice;       // price in wei per token for public mint
    bool public publicSaleActive;

    event BaseURIChanged(string newBaseURI);
    event PublicSaleToggled(bool active);
    event Minted(address to, uint256 tokenId);

    constructor(
        string memory name_,
        string memory symbol_,
        string memory baseURI_,
        uint256 maxSupply_,
        uint256 mintPrice_
    ) ERC721(name_, symbol_) {
        require(maxSupply_ > 0, "maxSupply must be > 0");
        _baseTokenURI = baseURI_;
        maxSupply = maxSupply_;
        mintPrice = mintPrice_;
        publicSaleActive = false;
    }

    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }

    function setBaseURI(string calldata newBaseURI) external onlyOwner {
        _baseTokenURI = newBaseURI;
        emit BaseURIChanged(newBaseURI);
    }

    function ownerMint(address to, uint256 quantity) external onlyOwner {
        require(quantity > 0, "quantity must be > 0");
        require(totalSupply() + quantity <= maxSupply, "max supply reached");
        for (uint256 i = 0; i < quantity; i++) {
            _mintTo(to);
        }
    }

    function mint(uint256 quantity) external payable {
        require(publicSaleActive, "public sale not active");
        require(quantity > 0, "quantity must be > 0");
        require(totalSupply() + quantity <= maxSupply, "max supply reached");
        require(msg.value >= mintPrice * quantity, "insufficient ETH sent");

        for (uint256 i = 0; i < quantity; i++) {
            _mintTo(msg.sender);
        }

        uint256 cost = mintPrice * quantity;
        if (msg.value > cost) {
            uint256 refund = msg.value - cost;
            (bool sent, ) = payable(msg.sender).call{value: refund}("");
            require(sent, "refund failed");
        }
    }

    function togglePublicSale() external onlyOwner {
        publicSaleActive = !publicSaleActive;
        emit PublicSaleToggled(publicSaleActive);
    }

    function setMintPrice(uint256 newPrice) external onlyOwner {
        mintPrice = newPrice;
    }

    function withdraw() external onlyOwner {
        address payable to = payable(owner());
        uint256 balance = address(this).balance;
        require(balance > 0, "no balance");
        (bool sent, ) = to.call{value: balance}("");
        require(sent, "withdraw failed");
    }

    function _mintTo(address to) internal {
        _tokenIdCounter.increment();
        uint256 newId = _tokenIdCounter.current();
        _safeMint(to, newId);
        emit Minted(to, newId);
    }

    receive() external payable {}
    fallback() external payable {}
}
MyBaseNFT.sol
```markdown
# Deploy MyBaseNFT to Base using Remix

Quick summary:
- Paste `MyBaseNFT.sol` into Remix.
- Use MetaMask (Injected Web3) configured for the Base network.
- Compile with Solidity 0.8.17 (or compatible 0.8.x).
- Deploy using your wallet account on Base.

Steps

1) Preparation
- Install MetaMask and unlock it.
- Add the Base network to MetaMask (Mainnet Chain ID: 8453). Use the official RPC URL listed in Base docs (do not rely on untrusted RPCs). If you want a testnet, get Base testnet details from the Base docs.
- Make sure the account has ETH (on Base) to pay gas.

2) Open Remix
- Visit https://remix.ethereum.org.
- Create a new file `MyBaseNFT.sol` and paste the contract above.

3) Compiler settings
- Go to the "Solidity Compiler" plugin.
- Select compiler version 0.8.17 (or exact 0.8.x matching the pragma).
- Enable Optimization (recommended) with 200 runs.
- Click "Compile MyBaseNFT.sol".
- If Remix fails to fetch OpenZeppelin imports, either:
  - Use a matching OpenZeppelin release URL, or
  - Copy required OZ contracts into Remix as separate files.

4) Deploy
- In "Deploy & Run Transactions", set Environment to "Injected Provider - MetaMask".
- Connect Remix to MetaMask when prompted; ensure MetaMask is set to the Base network and the correct account.
- Select the `MyBaseNFT` contract from the dropdown.
- Fill constructor arguments:
  - name_ (string) — e.g. "MyBaseNFT"
  - symbol_ (string) — e.g. "MBNFT"
  - baseURI_ (string) — e.g. "ipfs://Qm.../" or a placeholder to update later
  - maxSupply_ (uint256) — e.g. 1000
  - mintPrice_ (uint256) — in wei (for 0.01 ETH use 10000000000000000)
- Click "Deploy". Confirm the transaction in MetaMask and wait for confirmation on Base.

5) Post-deploy actions
- After deploy, copy the contract address shown in Remix.
- From Remix (Deployed Contracts section) you can:
  - setBaseURI(...)
  - togglePublicSale()
  - setMintPrice(...)
  - ownerMint(to, qty)
  - withdraw()
  - mint(quantity) (for public users when sale is active; include value)
- If you need to refund or send ETH, verify balances and gas.

6) Verify on Base block explorer
- Use the explorer's "Verify & Publish" to publish source. Make sure to set the same compiler version and optimizer settings used in Remix.
- If verification requires a flattened file, generate one (I can generate it for you).

7) Safety & tips
- For production add ReentrancyGuard on withdraw/refund flows, consider EIP-2981 (royalties), and run tests/audit.
- Use official Base RPC endpoints or a reputable provider (Alchemy, Blast, Infura).
- Test on Base testnet before mainnet deployment.
- Keep private keys secure and double-check constructor args before confirming the deploy transaction.

Common Troubleshooting
- "Wrong Network" in MetaMask: re-check Chain ID and selected network.
- Remix import errors: try a different OpenZeppelin tag matching your compiler.
- Gas failures: increase gas limit or switch RPC if network issues persist.

If you want, I can:
- Produce a flattened source file for explorer verification.
- Add EIP-2981 royalties to the contract.
- Provide a Hardhat or Foundry deployment script (with Base RPC & wallet config).
- Walk you through an actual testnet deployment step-by-step while you perform actions in Remix/MetaMask.
```
