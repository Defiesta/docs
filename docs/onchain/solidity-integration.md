---
title: Solidity Integration
sidebar_position: 2
---

# Solidity Integration

This document provides details on integrating with the Execution Kernel contracts from Solidity.

## Contract Interfaces

### IKernelExecutionVerifier

```solidity
interface IKernelExecutionVerifier {
    /// @notice Verify a kernel execution proof
    /// @param agentId The agent identifier
    /// @param journal The kernel journal (209 bytes)
    /// @param seal The Groth16 proof
    /// @return True if the proof is valid
    function verify(
        bytes32 agentId,
        bytes calldata journal,
        bytes calldata seal
    ) external view returns (bool);

    /// @notice Get the registered imageId for an agent
    /// @param agentId The agent identifier
    /// @return The registered imageId (0 if not registered)
    function registeredImageIds(bytes32 agentId) external view returns (bytes32);
}
```

### IKernelVault

```solidity
interface IKernelVault {
    /// @notice Execute verified agent actions
    /// @param journal The kernel journal (209 bytes)
    /// @param seal The Groth16 proof
    /// @param agentOutput The encoded agent output
    function execute(
        bytes calldata journal,
        bytes calldata seal,
        bytes calldata agentOutput
    ) external;

    /// @notice Get the last execution nonce
    function lastExecutionNonce() external view returns (uint64);

    /// @notice Get the authorized agent ID
    function authorizedAgentId() external view returns (bytes32);
}
```

## Parsing the Journal

Use the KernelOutputParser library:

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.20;

library KernelOutputParser {
    struct ParsedJournal {
        uint32 protocolVersion;
        uint32 kernelVersion;
        bytes32 agentId;
        bytes32 agentCodeHash;
        bytes32 constraintSetHash;
        bytes32 inputRoot;
        uint64 executionNonce;
        bytes32 inputCommitment;
        bytes32 actionCommitment;
        uint8 executionStatus;
    }

    uint8 constant EXECUTION_STATUS_SUCCESS = 0x01;
    uint8 constant EXECUTION_STATUS_FAILURE = 0x02;

    function parse(bytes calldata journal)
        internal
        pure
        returns (ParsedJournal memory parsed)
    {
        require(journal.length == 209, "Invalid journal length");

        // Parse little-endian u32
        parsed.protocolVersion = _readU32LE(journal, 0);
        parsed.kernelVersion = _readU32LE(journal, 4);

        // Parse 32-byte fields
        parsed.agentId = bytes32(journal[8:40]);
        parsed.agentCodeHash = bytes32(journal[40:72]);
        parsed.constraintSetHash = bytes32(journal[72:104]);
        parsed.inputRoot = bytes32(journal[104:136]);

        // Parse little-endian u64
        parsed.executionNonce = _readU64LE(journal, 136);

        // Parse commitments
        parsed.inputCommitment = bytes32(journal[144:176]);
        parsed.actionCommitment = bytes32(journal[176:208]);

        // Parse status
        parsed.executionStatus = uint8(journal[208]);
    }

    function _readU32LE(bytes calldata data, uint256 offset)
        private
        pure
        returns (uint32)
    {
        return uint32(uint8(data[offset]))
            | (uint32(uint8(data[offset + 1])) << 8)
            | (uint32(uint8(data[offset + 2])) << 16)
            | (uint32(uint8(data[offset + 3])) << 24);
    }

    function _readU64LE(bytes calldata data, uint256 offset)
        private
        pure
        returns (uint64)
    {
        return uint64(_readU32LE(data, offset))
            | (uint64(_readU32LE(data, offset + 4)) << 32);
    }
}
```

## Parsing Agent Output

```solidity
library AgentOutputParser {
    struct ActionV1 {
        uint32 actionType;
        bytes32 target;
        bytes payload;
    }

    uint32 constant ACTION_TYPE_CALL = 0x00000001;
    uint32 constant ACTION_TYPE_TRANSFER_ERC20 = 0x00000002;

    function parseActions(bytes calldata agentOutput)
        internal
        pure
        returns (ActionV1[] memory actions)
    {
        uint256 offset = 0;

        // Read action count (u32 LE)
        uint32 actionCount = _readU32LE(agentOutput, offset);
        offset += 4;

        actions = new ActionV1[](actionCount);

        for (uint256 i = 0; i < actionCount; i++) {
            // Read action length (u32 LE)
            uint32 actionLen = _readU32LE(agentOutput, offset);
            offset += 4;

            // Read action type (u32 LE)
            actions[i].actionType = _readU32LE(agentOutput, offset);
            offset += 4;

            // Read target (32 bytes)
            actions[i].target = bytes32(agentOutput[offset:offset+32]);
            offset += 32;

            // Read payload length (u32 LE)
            uint32 payloadLen = _readU32LE(agentOutput, offset);
            offset += 4;

            // Read payload
            actions[i].payload = agentOutput[offset:offset+payloadLen];
            offset += payloadLen;
        }
    }

    // ... _readU32LE helper
}
```

## Implementing a Vault

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.20;

import "./IKernelExecutionVerifier.sol";
import "./KernelOutputParser.sol";
import "./AgentOutputParser.sol";

contract MyVault {
    using KernelOutputParser for bytes;
    using AgentOutputParser for bytes;

    IKernelExecutionVerifier public immutable verifier;
    bytes32 public authorizedAgentId;
    uint64 public lastExecutionNonce;

    bytes32 constant EMPTY_OUTPUT_COMMITMENT =
        0xdf3f619804a92fdb4057192dc43dd748ea778adc52bc498ce80524c014b81119;

    constructor(address _verifier, bytes32 _agentId) {
        verifier = IKernelExecutionVerifier(_verifier);
        authorizedAgentId = _agentId;
    }

    function execute(
        bytes calldata journal,
        bytes calldata seal,
        bytes calldata agentOutput
    ) external {
        // Parse journal
        KernelOutputParser.ParsedJournal memory parsed = journal.parse();

        // Validate versions
        require(parsed.protocolVersion == 1, "Unsupported protocol");
        require(parsed.kernelVersion == 1, "Unsupported kernel");

        // Validate agent
        require(parsed.agentId == authorizedAgentId, "Wrong agent");

        // Validate nonce
        require(
            parsed.executionNonce == lastExecutionNonce + 1,
            "Invalid nonce"
        );

        // Validate status
        require(
            parsed.executionStatus == KernelOutputParser.EXECUTION_STATUS_SUCCESS,
            "Execution failed"
        );

        // Verify proof
        require(
            verifier.verify(parsed.agentId, journal, seal),
            "Invalid proof"
        );

        // Verify action commitment
        require(
            sha256(agentOutput) == parsed.actionCommitment,
            "Action commitment mismatch"
        );

        // Update nonce before execution (reentrancy protection)
        lastExecutionNonce = parsed.executionNonce;

        // Parse and execute actions
        AgentOutputParser.ActionV1[] memory actions = agentOutput.parseActions();
        for (uint256 i = 0; i < actions.length; i++) {
            _executeAction(actions[i]);
        }
    }

    function _executeAction(AgentOutputParser.ActionV1 memory action) internal {
        if (action.actionType == AgentOutputParser.ACTION_TYPE_CALL) {
            // Extract address from target (lower 20 bytes)
            address target = address(uint160(uint256(action.target)));

            // Decode payload: value (u64 LE) + calldata
            uint64 value = _readU64LE(action.payload, 0);
            bytes memory callData = action.payload[8:];

            // Execute call
            (bool success, ) = target.call{value: value}(callData);
            require(success, "Call failed");
        }
        // Handle other action types...
    }

    // Receive ETH
    receive() external payable {}
}
```

## Testing Integration

### Using Foundry

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/MyVault.sol";

contract VaultTest is Test {
    MyVault vault;
    address verifier = 0x9Ef5bAB590AFdE8036D57b89ccD2947D4E3b1EFA;
    bytes32 agentId = bytes32(uint256(1));

    function setUp() public {
        vault = new MyVault(verifier, agentId);
        vm.deal(address(vault), 1 ether);
    }

    function testExecuteWithValidProof() public {
        // Load test data from file or generate
        bytes memory journal = /* 209 bytes */;
        bytes memory seal = /* proof data */;
        bytes memory agentOutput = /* encoded actions */;

        vault.execute(journal, seal, agentOutput);

        assertEq(vault.lastExecutionNonce(), 1);
    }

    function testRejectInvalidNonce() public {
        bytes memory journal = /* with wrong nonce */;
        bytes memory seal = /* ... */;
        bytes memory agentOutput = /* ... */;

        vm.expectRevert("Invalid nonce");
        vault.execute(journal, seal, agentOutput);
    }
}
```

### Using Cast

```bash
# Read vault state
cast call $VAULT "lastExecutionNonce()(uint64)" --rpc-url $RPC_URL
cast call $VAULT "authorizedAgentId()(bytes32)" --rpc-url $RPC_URL

# Check vault balance
cast balance $VAULT --rpc-url $RPC_URL
```

## Events

Consider emitting events for tracking:

```solidity
event ExecutionCompleted(
    bytes32 indexed agentId,
    uint64 nonce,
    bytes32 inputCommitment,
    bytes32 actionCommitment,
    uint256 actionsExecuted
);

event ActionExecuted(
    uint256 indexed index,
    uint32 actionType,
    bytes32 target,
    bool success
);
```

## Related

- [Verifier Overview](/onchain/verifier-overview) - Contract architecture
- [Security Considerations](/onchain/security-considerations) - Trust model
- [Journal Format](/kernel/journal-format) - Journal structure
