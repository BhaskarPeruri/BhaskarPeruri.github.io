---
layout: post
title: Projects
---

<style>
  .project-card {
    background-color: var(--bg-color);
    border: 1px solid var(--border-color);
    border-radius: 10px;
    padding: 1.5rem;
    width: 300px;
    height: 350px;
    color: var(--text-color);
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    text-decoration: none;
    transition: transform 0.2s ease;
  }

  .project-card:hover {
    transform: translateY(-4px);
  }

  .project-title {
    color: var(--title-color);
    text-align: center;
    margin: 0;
  }

  .project-badge {
    background-color: #06b6d4;
    color: black;
    padding: 2px 6px;
    border-radius: 4px;
    font-size: 0.8rem;
    margin-left: 8px;
  }

  .project-desc {
    text-align: center;
    font-size: 0.9rem;
  }

  .light-theme {
    --bg-color: #ffffff;
    --border-color: #d1d5db;
    --text-color: #1f2937;
    --title-color: #1d4ed8;
  }

  .dark-theme {
    --bg-color: #1f2937;
    --border-color: #4b5563;
    --text-color: #d1d5db;
    --title-color: #8ab4f8;
  }
</style>

<div class="dark-theme" style="display: flex; flex-wrap: wrap; gap: 20px;">
  <a href="https://github.com/BhaskarPeruri/AccountAbstraction_Ethereum" target="_blank" class="project-card">
    <h3 class="project-title">Account Abstraction Ethereum <span class="project-badge">new</span></h3>
    <hr />
    <p class="project-desc">This is a minimal implementation of an ERC-4337 Account Abstraction smart contract wallet. It provides basic account functionality that allows users to execute transactions through the Account Abstraction infrastructure while maintaining ownership control.</p>
  </a>

  <a href="https://github.com/BhaskarPeruri/StableCoin_DSC" target="_blank" class="project-card">
    <h3 class="project-title">StableCoin_DSC <span class="project-badge">new</span></h3>
    <hr />
    <p class="project-desc">A decentralized stablecoin system with WETH/WBTC collateral backing DSC tokens pegged to USD.</p>
  </a>

  <a href="https://github.com/BhaskarPeruri/NFTStaking" target="_blank" class="project-card">
    <h3 class="project-title">NFT Staking <span class="project-badge">new</span></h3>
    <hr />
    <p class="project-desc">Stake NFTs to earn ERC20 rewards, with staking management and security protections for users and the protocol</p>
  </a>

  <a href="https://github.com/BhaskarPeruri/TrustSig_Wallet" target="_blank" class="project-card">
    <h3 class="project-title">TrustSig Wallet <span class="project-badge">new</span></h3>
    <hr />
    <p class="project-desc">The TrustSig contract is a minimal, secure, and gas-efficient multi-signature wallet that allows a group of trusted owners to collaboratively manage and execute transactions.</p>
  </a>

  <a href="https://github.com/BhaskarPeruri/Dynamic_and_Static_NFTs" target="_blank" class="project-card">
    <h3 class="project-title">Dynamic & Static NFTs <span class="project-badge">new</span></h3>
    <hr />
    <p class="project-desc">A collection of ERC721 contracts: BasicNFT for static URIs and MoodNFT for toggleable emotions and dynamic metadata.</p>
  </a>

  <a href="https://github.com/BhaskarPeruri/DAO_DApp" target="_blank" class="project-card">
    <h3 class="project-title">DAO DApp</h3>
    <hr />
    <p class="project-desc">Allows users to propose, vote, and manage DAO treasury through secure and transparent mechanisms.</p>
  </a>

  <a href="https://github.com/BhaskarPeruri/Decentralized-Wallet" target="_blank" class="project-card">
    <h3 class="project-title">Decentralized Wallet</h3>
    <hr />
    <p class="project-desc">A smart contract wallet to transfer ETH built on Ganache with a React frontend.</p>
  </a>
</div>

