---
description: >-
  Smart contract system for Toncast DAO that allows users to stake TONCAST
  tokens, receive membership NFTs, and earn epoch rewards.
---

# Toncast dao

### Core Staking Flows

1\. Deposit & Stake

* User transfers TONCAST tokens to the DAO.
* DAO validates the transfer and updates the total staked amount.
* A unique Membership NFT is minted to the user as a staking certificate.<br>

2\. Withdraw & Claim Rewards

* User transfers their staking NFT back to the DAO.
* DAO verifies the NFT and parses the staking data.
* The staked TONCAST tokens are returned.
* Rewards from all eligible epochs are automatically calculated and paid out.

### Epoch Reward System

* Epoch Duration: Configurable (e.g., 30 days). A new epoch starts automatically when the previous one ends.
* Reward Pool: All TON accumulated by the DAO during an epoch is distributed to its stakers.
* Distribution: Rewards are proportional to your stake.
* Your Reward = (Your Staked Amount / Total Staked in Epoch) \* Total TON in Epoch
* Claiming: Rewards are automatically claimed when you withdraw your stake.

### Key Features

* On-Chain NFTs: Staking data (amount, timestamp, DAO address) is stored directly in the NFT metadata on-chain.
* Automatic & Decentralized: Epochs are created and funded automatically. The contract owner cannot withdraw user funds.
* Backwards Compatibility: The system includes a stop-and-redirect mechanism to safely upgrade the DAO without interrupting user withdrawals.



This system enables secure, transparent, and community-governed staking with automatic reward distribution.

<a href="https://github.com/TonCast/toncast-dao" class="button primary" data-icon="arrow-up-right-from-square">Toncast-dao</a>
