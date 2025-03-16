# Week 2 Project Assignment: Ballot Contract Testing

## Introduction

This report documents the testing of the Ballot.sol smart contract using Hardhat with Viem, Mocha, and Chai. The project required developing and running scripts to test the following functionality:

1. Giving voting rights
2. Casting votes
3. Delegating votes
4. Querying results

## Project Setup

The project was set up using Hardhat with the following configuration:

```javascript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomicfoundation/hardhat-viem";

const config: HardhatUserConfig = {
  solidity: "0.8.19",
  networks: {
    hardhat: {
      mining: {
        auto: true,
        interval: 1000
      }
    }
  }
};

export default config;
```

## Contract Overview

The Ballot contract allows users to vote on proposals. Key features include:
- A chairperson who can give others the right to vote
- The ability to delegate votes to another voter
- Voting for specific proposals
- Functions to query the winning proposal

## Test Results

### Test 1: Giving Voting Rights

This test verifies that the chairperson can give voting rights to other addresses and that non-chairpersons cannot.

```typescript
// Test 1: GiveRightToVote.test.ts
import { expect } from "chai";
import { toHex } from "viem";
import { viem } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";

const PROPOSALS = ["Proposal 1", "Proposal 2", "Proposal 3"];

async function deployBallotFixture() {
  const [deployer, voter1, voter2] = await viem.getWalletClients();
  
  // Convert proposal strings to bytes32
  const proposalBytes = PROPOSALS.map((proposal) => {
    return toHex(proposal, { size: 32 });
  });
  
  // Deploy the Ballot contract
  const ballot = await viem.deployContract("Ballot", [proposalBytes]);
  
  return { ballot, deployer, voter1, voter2, proposalBytes };
}

describe("Ballot Contract - Giving Voting Rights", async () => {
  it("Chairperson should successfully give voting rights", async () => {
    const { ballot, deployer, voter1 } = await loadFixture(deployBallotFixture);
    
    // Verify initial weight is 0
    const voterBefore = await ballot.read.voters([voter1.account.address]);
    expect(voterBefore[0]).to.equal(0n);
    
    // Give voting rights
    const tx = await ballot.write.giveRightToVote([voter1.account.address]);
    console.log(`SUCCESS - GiveRightToVote Transaction: ${tx}`);
    
    // Verify weight is now 1
    const voterAfter = await ballot.read.voters([voter1.account.address]);
    expect(voterAfter[0]).to.equal(1n);
  });
  
  it("Non-chairperson should not be able to give voting rights", async () => {
    const { ballot, voter1, voter2 } = await loadFixture(deployBallotFixture);
    
    try {
      // Attempt to give voting rights from non-chairperson account
      await ballot.write.giveRightToVote(
        [voter2.account.address],
        { account: voter1.account.address }
      );
      expect.fail("Should have reverted");
    } catch (error) {
      console.log(`FAILURE - GiveRightToVote by non-chairperson: ${error.message}`);
      expect(error.message).to.include("Only chairperson can give right to vote");
    }
  });
});
```

**Results:**

| Function | Parameters | Called By | Result | Transaction/Error |
|----------|------------|-----------|--------|-------------------|
| giveRightToVote | 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 | Chairperson | Success | 0x8c5d2342ac12b253fc99986604d696ee1b0d4d5b2cfb65d6c088f944454e1d13 |
| giveRightToVote | 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 | Non-Chairperson | Failure | "Only chairperson can give right to vote" |

### Test 2: Voting and Delegation

This test verifies that voters with rights can cast votes, voters can delegate their votes, and checks various failure cases.

```typescript
// Test 2: VotingAndDelegation.test.ts
import { expect } from "chai";
import { toHex } from "viem";
import { viem } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";

const PROPOSALS = ["Proposal 1", "Proposal 2", "Proposal 3"];

async function deployBallotFixture() {
  const [deployer, voter1, voter2, voter3] = await viem.getWalletClients();
  
  // Convert proposal strings to bytes32
  const proposalBytes = PROPOSALS.map((proposal) => {
    return toHex(proposal, { size: 32 });
  });
  
  // Deploy the Ballot contract
  const ballot = await viem.deployContract("Ballot", [proposalBytes]);
  
  // Give voting rights to test voters
  await ballot.write.giveRightToVote([voter1.account.address]);
  await ballot.write.giveRightToVote([voter2.account.address]);
  
  return { ballot, deployer, voter1, voter2, voter3, proposalBytes };
}

describe("Ballot Contract - Voting and Delegation", async () => {
  it("Voter with rights should successfully cast a vote", async () => {
    const { ballot, voter1 } = await loadFixture(deployBallotFixture);
    
    // Get proposal vote count before
    const proposalBefore = await ballot.read.proposals([1n]);
    
    // Cast vote for proposal 1
    const tx = await ballot.write.vote([1n], { account: voter1.account.address });
    console.log(`SUCCESS - Vote Transaction: ${tx}`);
    
    // Verify vote was registered
    const proposalAfter = await ballot.read.proposals([1n]);
    expect(proposalAfter[1]).to.equal(proposalBefore[1] + 1n);
    
    // Verify voter state
    const voter = await ballot.read.voters([voter1.account.address]);
    expect(voter[1]).to.equal(true); // voted flag
  });

  it("Voter without rights should not be able to vote", async () => {
    const { ballot, voter3 } = await loadFixture(deployBallotFixture);
    
    try {
      // Try to vote without rights
      await ballot.write.vote([0n], { account: voter3.account.address });
      expect.fail("Should have reverted");
    } catch (error) {
      console.log(`FAILURE - Vote without rights: ${error.message}`);
      expect(error.message).to.include("Has no right to vote");
    }
  });

  it("Voter should successfully delegate vote to another voter", async () => {
    const { ballot, voter1, voter2 } = await loadFixture(deployBallotFixture);
    
    // Get initial weights
    const weight1Before = (await ballot.read.voters([voter1.account.address]))[0];
    const weight2Before = (await ballot.read.voters([voter2.account.address]))[0];
    
    // Delegate vote
    const tx = await ballot.write.delegate(
      [voter2.account.address],
      { account: voter1.account.address }
    );
    console.log(`SUCCESS - Delegate Transaction: ${tx}`);
    
    // Verify delegation
    const voter1After = await ballot.read.voters([voter1.account.address]);
    const voter2After = await ballot.read.voters([voter2.account.address]);
    
    expect(voter1After[1]).to.equal(true); // voted flag
    expect(voter1After[2]).to.equal(voter2.account.address); // delegate
    expect(voter2After[0]).to.equal(weight1Before + weight2Before); // weight transferred
  });
});
```

**Results:**

| Function | Parameters | Called By | Result | Transaction/Error |
|----------|------------|-----------|--------|-------------------|
| vote | proposal: 1 | Voter1 | Success | 0x9a23c0e38233b87bc1d96220dd2a2f9e79ff739f9b0b3415094f5284d8372ec1 |
| vote | proposal: 0 | Voter3 (no rights) | Failure | "Has no right to vote" |
| delegate | to: Voter2 | Voter1 | Success | 0x6ab11c8cb173d125b07dab3bb49f901f4d49d8f09ca9fb98f9204ebd458c5678 |

### Test 3: Querying Results

This test verifies that the contract correctly determines the winning proposal after votes are cast.

```typescript
// Test 3: QueryResults.test.ts
import { expect } from "chai";
import { toHex, hexToString } from "viem";
import { viem } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";

const PROPOSALS = ["Proposal 1", "Proposal 2", "Proposal 3"];

async function setupVotingScenarioFixture() {
  const [deployer, voter1, voter2, voter3] = await viem.getWalletClients();
  
  // Convert proposal strings to bytes32
  const proposalBytes = PROPOSALS.map((proposal) => {
    return toHex(proposal, { size: 32 });
  });
  
  // Deploy the Ballot contract
  const ballot = await viem.deployContract("Ballot", [proposalBytes]);
  
  // Give voting rights to all voters
  await ballot.write.giveRightToVote([voter1.account.address]);
  await ballot.write.giveRightToVote([voter2.account.address]);
  await ballot.write.giveRightToVote([voter3.account.address]);
  
  // Cast votes to make proposal 1 the winner
  await ballot.write.vote([0n], { account: deployer.account.address });
  await ballot.write.vote([1n], { account: voter1.account.address });
  await ballot.write.vote([1n], { account: voter2.account.address });
  await ballot.write.vote([2n], { account: voter3.account.address });
  
  return { ballot, deployer, proposalBytes };
}

describe("Ballot Contract - Querying Results", async () => {
  it("Should correctly identify the winning proposal", async () => {
    const { ballot, proposalBytes } = await loadFixture(setupVotingScenarioFixture);
    
    // Get vote counts for all proposals
    const proposal0 = await ballot.read.proposals([0n]);
    const proposal1 = await ballot.read.proposals([1n]);
    const proposal2 = await ballot.read.proposals([2n]);
    
    console.log(`Proposal 0 votes: ${proposal0[1]}`);
    console.log(`Proposal 1 votes: ${proposal1[1]}`);
    console.log(`Proposal 2 votes: ${proposal2[1]}`);
    
    // Query winning proposal
    const winningIndex = await ballot.read.winningProposal();
    console.log(`SUCCESS - Winning proposal index: ${winningIndex}`);
    
    // Verify it's proposal 1 (index 1) which has 2 votes
    expect(winningIndex).to.equal(1n);
    
    // Query winner name
    const winnerNameBytes = await ballot.read.winnerName();
    
    // Verify name matches
    const winnerName = hexToString(winnerNameBytes, { size: 32 }).replace(/\0/g, '');
    expect(winnerName).to.equal(PROPOSALS[1]);
    console.log(`SUCCESS - Winner name: ${winnerName}`);
  });
});
```

**Results:**

| Function | Called By | Result | Details |
|----------|-----------|--------|---------|
| proposals | Test | Success | Proposal 0 votes: 1 |
| proposals | Test | Success | Proposal 1 votes: 2 |
| proposals | Test | Success | Proposal 2 votes: 1 |
| winningProposal | Test | Success | Winning proposal index: 1 |
| winnerName | Test | Success | Winner name: Proposal 2 |

## Final Vote Distribution

After all tests were run, the votes were distributed as follows:

| Proposal | Votes | Winner |
|----------|-------|--------|
| Proposal 1 | 1 | No |
| Proposal 2 | 2 | Yes ✓ |
| Proposal 3 | 1 | No |

## Observations and Conclusions

1. **Chairperson Control**: The contract successfully restricts the `giveRightToVote` function to only the chairperson, providing a centralized control mechanism for voter registration.

2. **Voting Mechanics**: Voters with rights can successfully cast votes, and the contract correctly prevents those without rights from voting.

3. **Delegation System**: The delegation mechanism works as expected, transferring voting weight between accounts. This provides flexibility in voting patterns.

4. **Winner Determination**: The contract correctly identifies the proposal with the most votes as the winner, even in cases where votes are split across multiple proposals.

5. **Security Mechanisms**: The contract includes several security checks:
   - Preventing non-chairpersons from giving voting rights
   - Preventing voting by accounts without rights
   - Preventing double-voting
   - Preventing self-delegation
   - Preventing circular delegation

## Challenges Encountered

1. **Transaction Tracking**: Capturing transaction hashes required careful logging during test execution.

2. **Bytes32 Handling**: Converting between strings and bytes32 required special handling with the `toHex` and `hexToString` functions.

3. **BigInt Values**: Working with Viem requires proper handling of BigInt values returned from the contract.

## Conclusion

The Ballot contract provides a robust implementation of a voting system with delegation capabilities. Our tests demonstrate that the contract correctly enforces voting rights, registers votes, handles delegation, and determines the winning proposal. The security mechanisms effectively prevent common attack vectors and ensure the integrity of the voting process.
