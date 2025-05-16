# EnsoLend: Cross-Chain Lending & Borrowing Protocol (Solana & Sui)

EnsoLend is a decentralized, cross-chain lending and borrowing protocol built on Solana and Sui, leveraging Wormhole for interoperability and Pyth Network for reliable price feeds. It allows users on one chain to lend assets and users on another chain to borrow those assets by providing collateral.

## Table of Contents

- [Problem Solved](#problem-solved)
- [Features](#features)
- [Architecture Overview](#architecture-overview)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Setup and Installation](#setup-and-installation)
  - [Prerequisites](#prerequisites)
  - [Solana Smart Contract](#solana-smart-contract)
  - [Sui Smart Contract](#sui-smart-contract)
- [How It Works (Key Flows)](#how-it-works-key-flows)
  - [Cross-Chain Loan (Lender on Solana, Borrower on Sui - Example)](#cross-chain-loan-lender-on-solana-borrower-on-sui---example)
- [Wormhole Integration](#wormhole-integration)
  - [Wormhole Bounty Track](#wormhole-bounty-track)
- [Deployment](#deployment)
- [Testing](#testing)
- [Future Enhancements](#future-enhancements)
- [Contributing](#contributing)
- [License](#license)

## Problem Solved

Traditional DeFi lending protocols are often siloed within a single blockchain ecosystem. This limits liquidity and user access. EnsoLend aims to:
-   **Bridge Liquidity:** Enable assets on Solana to be lent to borrowers on Sui, and vice-versa.
-   **Enhance Capital Efficiency:** Allow users to utilize their assets across different chains for lending and borrowing.
-   **Improve User Experience:** Provide a seamless way to interact with a multi-chain lending market.

## Features

**Lending (Primarily on Solana via `enso-lending` program):**
-   Create Lend Offers: Lenders can create offers specifying asset, amount, interest rate, and duration.
-   Edit Lend Offers: Modify existing, unmatched lend offers.
-   Cancel Lend Offers: Withdraw unmatched lend offers.
-   System-Managed Offer Cancellation.

**Borrowing (Native & Cross-Chain):**
-   **Native Loans (Solana):**
    -   Create Loan Offers (matching a Lend Offer on Solana).
    -   Deposit/Withdraw Collateral (native Solana assets).
    -   Repay Loans.
    -   Liquidate unhealthy or expired loans.
-   **Cross-Chain Loans (e.g., Borrower on Sui, Lender on Solana):**
    -   **Solana Side:**
        -   Handles the core lending logic and VAA verification for cross-chain actions.
        -   `create_loan_offer_cross_chain`: Initiates a loan based on a VAA from Sui.
        -   `repay_loan_offer_cross_chain`: Processes repayment initiated from Sui via VAA, emits message for collateral release.
        -   Manages VAA processing for collateral deposits/withdrawals/cancellations initiated on Sui.
        -   Handles liquidation logic for cross-chain loans.
    -   **Sui Side:**
        -   `collateral_holder.move`: Manages user collateral deposited on Sui.
        -   `loan_crosschain.move`, `operator_loan_crosschain.move`: Handles logic for initiating loan requests to Solana, processing VAAs from Solana for fund transfer, initiating repayments, and managing collateral based on Wormhole messages.
        -   Deposit collateral to initiate a loan request to Solana (emits Wormhole message).
        -   Receive funds from Solana lender (after VAA verification).
        -   Initiate loan repayment (emits Wormhole message).
        -   Request collateral withdrawal/cancellation (emits Wormhole message).
        -   Process liquidation messages from Solana.

**Platform & Admin Features:**
-   Asset Management: Initialize and edit supported assets, their price feeds, and cross-chain properties.
-   Setting/Tier Management: Configure lending tiers with specific parameters (amount, duration, fees).
-   Wormhole Integration: Initialize and manage Wormhole foreign chain emitters and configurations.
-   Oracle Integration: Utilizes Pyth Network for real-time asset pricing.

## Architecture Overview

EnsoLend operates with two main smart contract components:

1.  **Solana (Anchor):** The `enso-lending` program acts as the primary hub for lending. It manages lend offers, native Solana loans, and the Solana-side logic for cross-chain loans. It verifies VAAs from Sui and emits VAAs to Sui.
2.  **Sui (Move):** The Sui smart contracts manage collateral deposits on Sui for cross-chain loans. They emit VAAs to Solana to initiate loan actions and process VAAs from Solana to update loan states or release funds/collateral.

**Interaction Flow (Simplified Cross-Chain Loan):**
1.  Lender creates a lend offer on Solana.
2.  Borrower on Sui wishes to take this loan:
    a.  Deposits collateral into the EnsoLend Sui contract.
    b.  Sui contract emits a Wormhole VAA indicating the loan request and collateral lock.
3.  A relayer picks up the VAA and posts it to the Solana contract.
4.  The Solana contract verifies the VAA, matches the loan with an available lend offer, locks the lender's funds, and emits a VAA confirming the loan and fund transfer.
5.  A relayer posts this VAA to the Sui contract.
6.  The Sui contract verifies the VAA and allows the borrower to access the (represented) funds or confirms the loan is active.
7.  Repayment and collateral release follow a similar VAA-based reverse flow.

```mermaid

Technology Stack

Solana:

Rust

Anchor Framework

pyth-solana-receiver-sdk for Pyth price feeds

wormhole-anchor-sdk for Wormhole integration

Sui:

Move Language

Sui SDK

Wormhole SDK/bindings (from Wormhole dependency in Move.toml and wormhole.move, vaa_utils.move)

Pyth SDK/bindings (from Pyth dependency in Move.toml and price_feed.move)

Interoperability: Wormhole Core Messaging (VAAs)

Frontend/Tooling: Node.js, TypeScript for scripting.

Project Structure
├── .gitignore
├── README.md
├── solana_smartcontract        # Solana Anchor Program
│   ├── programs
│   │   └── enso-lending        # Main lending program
│   │       ├── src
│   │       │   ├── common      # Constants, errors, events
│   │       │   ├── contexts    # Instruction handlers
│   │       │   ├── states      # Account state definitions
│   │       │   └── utils       # Helper functions (Pyth, VAA, etc.)
│   │       └── lib.rs          # Program entrypoint
│   ├── migrations
│   ├── tests                   # Anchor tests
│   ├── Anchor.toml
│   ├── Cargo.toml
│   └── package.json
└── sui_smartcontract           # Sui Move Package
    ├── sources                 # Move contract modules
    │   ├── loan.move           # Core loan logic
    │   ├── offer.move          # Offer logic
    │   ├── loan_crosschain.move # Cross-chain loan specifics
    │   ├── wormhole.move       # Wormhole interaction logic
    │   └── vaa_utils.move      # VAA parsing utilities
    ├── tests                   # Move tests
    ├── scripts                 # Deployment and interaction scripts
    ├── Move.toml
    └── package.json
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END
Setup and Installation
Prerequisites

Rust & Cargo

Node.js (v18+) & Yarn (or NPM)

Solana CLI: Install Guide

Sui CLI: Install Guide

Anchor CLI: cargo install --git https://github.com/coral-xyz/anchor anchor-cli --locked --force

Solana Smart Contract

Navigate to the Solana contract directory:

cd solana_smartcontract
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Install Node.js dependencies:

yarn install
# or npm install
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Build the Anchor program:

anchor build
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Configure Anchor.toml:

Set your localnet/devnet program ID if deploying.

Update [provider] wallet path and cluster.

(Optional) Run tests:

anchor test
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

(Refer to solana_smartcontract/README.md for more detailed build and deployment instructions for Solana.)

Sui Smart Contract

Navigate to the Sui contract directory:

cd sui_smartcontract
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Install Node.js dependencies (for scripts):

npm install
# or yarn install
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Set up your Sui environment:

Ensure you have an active Sui environment (e.g., sui client active-env).

Switch to the desired environment (e.g., sui client switch --env testnet).

Ensure your active address has SUI for gas.

Build the Move package:

sui move build
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

(Optional) Run tests:

sui move test
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

(Refer to sui_smartcontract/README.md for detailed deployment checklists for Sui.)

How It Works (Key Flows)
Cross-Chain Loan (Lender on Solana, Borrower on Sui - Example)

Lender (Solana):

Calls create_lend_offer on the Solana enso-lending program, specifying the asset, amount, desired interest, and duration. The funds are typically held or escrowed by the program/hot wallet.

Borrower (Sui):

Discovers the available lend offer (e.g., via an off-chain service or by querying Solana state).

Calls a function like deposit_collateral_to_take_loan on the Sui EnsoLend contract.

This function takes the details of the Solana lend offer (ID, lend amount, tier, etc.), the collateral they are providing, and target Wormhole chain/address information.

The Sui contract locks the borrower's collateral in a CollateralHolder object.

The Sui contract emits a Wormhole message (VAA) containing:

Target chain (Solana's Wormhole Chain ID).

Target address (EnsoLend Solana program ID).

Target function identifier (e.g., create_loan_offer_cross_chain).

Payload: Solana lend offer ID, collateral amount, collateral asset details (Sui type/address), borrower's Sui address (as bytes for Wormhole payload), etc.

Relaying to Solana:

A Wormhole relayer network observes the VAA emitted on Sui.

The relayer submits this VAA to the Wormhole core bridge contract on Solana, which verifies it and makes it available for consumption.

The relayer (or another service) calls the create_loan_offer_cross_chain instruction on the EnsoLend Solana program, providing the VAA hash.

Solana Contract Processing:

The create_loan_offer_cross_chain context:

Fetches the PostedVaa from Wormhole using the VAA hash.

Verifies the VAA's emitter chain and address against its configured foreign chain (Sui) details.

Parses the VAA payload to extract loan details.

Validates the loan request (e.g., matching lend offer exists, health ratio if applicable based on VAA data).

If valid, updates the Solana LendOfferAccount to Loaned and creates a LoanOfferAccount with status Matched.

Emits a new Wormhole message (VAA) destined for Sui, indicating the loan is matched and funds should be (conceptually) made available to the Sui borrower. Payload would include:

Target chain (Sui's Wormhole Chain ID).

Target address (EnsoLend Sui contract ID).

Target function identifier (e.g., confirm_loan_and_release_funds - actual name in Sui contract might differ).

Payload: Sui loan identifier (derived from original request), loan terms.

Relaying to Sui:

A Wormhole relayer observes the VAA from Solana.

The relayer submits this VAA to the Sui contract.

Sui Contract Processing:

The Sui contract's operator_loan_crosschain::system_fund_transfer (or similar) function:

Verifies the VAA from Solana.

Updates the state of the loan record on Sui (if any) to FundTransferred or active.

If the actual funds are bridged (e.g., wrapped asset from Solana), they are transferred to the borrower. More likely, the Sui contract now acknowledges the loan is active, and the borrower has a debt obligation.

Repayment and Liquidation would follow similar VAA-based communication patterns.

Wormhole Integration

EnsoLend uses Wormhole for core messaging to enable cross-chain state synchronization and action invocation.

Solana (solana_smartcontract):

Uses wormhole-anchor-sdk for interacting with Wormhole.

VAA Consumption: Parses and verifies VAAs from Sui in various contexts (e.g., create_loan_offer_cross_chain, update_deposit_collateral_cross_chain, request_cancel_collateral_cross_chain). This involves checking emitter chain ID, emitter address, and payload consistency.

VAA Emission: Constructs and emits VAAs to Sui in contexts like system_update_loan_offer (after matching a cross-chain loan) or when processing repayments/liquidations that require action on Sui (e.g., repay_loan_offer_cross_chain).

Relevant state accounts: WormholeConfig, WormholeEmitter, WormholeMessage, ForeignChain.

VAA parsing logic is present in src/utils/vaa.rs.

Sui (sui_smartcontract):

Uses the Wormhole package dependency (from Move.toml).

VAA Consumption: The enso_lending::wormhole and vaa_utils.move modules handle VAA parsing and verification (e.g., parse_and_verify_vaa). This is used to process messages from Solana, such as loan funding confirmation or liquidation instructions.

VAA Emission: Emits VAAs to Solana when a user on Sui initiates a cross-chain action, such as requesting a loan against a Solana lend offer (loan_crosschain::deposit_collateral_to_take_loan), requesting collateral withdrawal, or initiating repayment.

Key modules: wormhole.move, collateral_holder.move, loan_crosschain.move, operator_loan_crosschain.move.

Message Payloads: Custom string-based payloads are defined (e.g., target_chain,target_address,target_function,payload_specific_data1,payload_specific_data2,...) and parsed by the respective contracts. VAA parsing logic can be found in solana_smartcontract/programs/enso-lending/src/utils/vaa.rs and sui_smartcontract/sources/vaa_utils.move.

Wormhole Bounty Track

This project is best suited for Track 2: Build Multichain Apps Using Wormhole Native Token Transfers (NTT).

Although it might not be using the pre-built NTT manager contracts, it facilitates the cross-chain utilization of tokens (loan assets, collateral) in a way that aims to preserve their utility across chains, a core principle of NTT. The project demonstrates how Wormhole's core messaging can be used to build complex cross-chain applications involving token transfers and state synchronization, which is foundational to the NTT concept.

Deployment

Solana: Refer to the solana_smartcontract/README.md for detailed instructions on building, deploying, and extending program space. Key commands include anchor build and anchor deploy or solana program deploy.

Sui: Refer to the sui_smartcontract/README.md for the deployment checklist. Key commands include sui move build and sui client publish. The scripts/bash/publish.sh script automates this.

Testing

Solana:

Uses Anchor's testing framework. Run tests with:

cd solana_smartcontract
anchor test
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Sui:

Uses the built-in Move testing framework. Run tests with:

cd sui_smartcontract
sui move test
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
Future Enhancements

Support for More Chains: Extend protocol to other Wormhole-connected chains.

Wider Range of Assets: Integrate more assets for lending and collateral.

Governance Token: Introduce a token for protocol governance.

Advanced Loan Features: Variable interest rates, flash loans.

Improved UI/UX: A dedicated frontend to simplify user interaction.

Gas Abstraction: Explore options to simplify gas payments for cross-chain transactions.

Formal NTT Integration: Potentially refactor to use Wormhole's official NTT manager contracts for token bridging/wrapping if beneficial for standardization and security.

Contributing

Contributions are welcome! Please follow these steps:

Fork the repository.

Create a new branch (git checkout -b feature/your-feature-name).

Make your changes.

Commit your changes (git commit -m 'Add some feature').

Push to the branch (git push origin feature/your-feature-name).

Open a Pull Request.

Please ensure your code adheres to existing style guides and includes appropriate tests.

License

This project is licensed under the MIT License. (Please create a LICENSE.md file with the MIT License text if it doesn't exist).

