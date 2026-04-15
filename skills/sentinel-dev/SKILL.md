---
name: sentinel-dev
description: >-
  Developer integration skill for the Sentinel SDK (@sentinelhq/core). Use this
  skill whenever the user is integrating Sentinel into a Solana AI agent application —
  setting up the SDK, configuring prompt guard or execution sandbox, writing custom
  YAML rule packs, handling SentinelError codes, or debugging scan/simulation issues.
  Load this skill proactively when the user imports @sentinelhq/core or asks about
  security policy, rule authoring, spending limits, or LLM judge configuration.
license: Complete terms in LICENSE.txt
---

# Sentinel Dev -- Agent Security Integration Guide

## When to Invoke
Use this skill when the user needs to:
- Integrate security scanning into their Solana AI agent
- Configure Sentinel SDK programmatically
- Create custom YAML rule packs
- Set up policy enforcement for transactions
- Debug security scanning issues

## Quick Start

```typescript
import { Sentinel } from '@sentinelhq/core';

const sentinel = await Sentinel.create({
  mode: 'full',
  promptGuard: {
    mode: 'rules',
    rules: { rulePacks: ['defi-safety', 'nft-guard', 'general'] }
  },
  executionSandbox: {
    rpcEndpoint: 'https://api.mainnet-beta.solana.com',
    policy: {
      spendingLimits: {
        maxPerTx: 1_000_000_000,    // 1 SOL
        maxDaily: 5_000_000_000,    // 5 SOL
        maxWeekly: 20_000_000_000   // 20 SOL
      }
    }
  }
});

const result = await sentinel.execute({
  input: 'swap 0.5 SOL for USDC',
  transaction: base64EncodedTx
});

if (!result.approved) {
  console.log('Action blocked:', result.blocked_by);
}
```

## API Reference

### Sentinel.create(config)
Factory method. Accepts `SentinelConfig`:
- `mode`: 'full' | 'guard-only' | 'sandbox-only' (default: 'full')
- `promptGuard`: PromptGuardConfig
- `executionSandbox`: ExecutionSandboxConfig
- `debug`: boolean

### sentinel.execute(action)
Runs full pipeline (guard -> sandbox). Accepts `AgentAction`:
- `input`: string -- text to scan through prompt guard
- `transaction`: string -- base64 serialized Solana transaction
- `metadata`: Record<string, unknown> -- optional metadata attached to the action

Returns `ExecutionResult`:
- `approved`: boolean
- `guardResult`: ScanResult (if prompt guard ran)
- `sandboxResult`: SimulationResult (if sandbox ran)
- `blocked_by`: 'prompt_guard' | 'execution_sandbox'
- `latency_ms`: number

### Event Hooks
```typescript
sentinel.on('threat:detected', ({ result }) => {
  console.log('Threat:', result.threatType, result.reasoning);
});

sentinel.on('tx:simulated', ({ result }) => {
  console.log('Risk score:', result.riskScore);
});

sentinel.on('policy:violated', ({ violation }) => {
  console.log('Policy violation:', violation.rule, violation.message);
});
```

## Configuration Patterns

### Rules-only mode (no LLM API key needed)
```typescript
{
  mode: 'guard-only',
  promptGuard: {
    mode: 'rules',
    rules: { rulePacks: ['defi-safety', 'nft-guard', 'general'] }
  }
}
```

### LLM + Rules mode
```typescript
{
  mode: 'guard-only',
  promptGuard: {
    mode: 'both',
    llm: {
      provider: 'anthropic',
      model: 'claude-haiku-4-5',
      apiKeyEnvVar: 'ANTHROPIC_API_KEY',
      timeoutMs: 5000
    },
    rules: { rulePacks: ['defi-safety', 'general'] }
  }
}
```

### Strict policy for production
```typescript
{
  executionSandbox: {
    rpcEndpoint: 'https://api.mainnet-beta.solana.com',
    policy: {
      spendingLimits: {
        maxPerTx: 100_000_000,       // 0.1 SOL
        maxDaily: 500_000_000,       // 0.5 SOL
        maxWeekly: 2_000_000_000     // 2 SOL
      },
      programAllowlist: [
        'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA',
        '11111111111111111111111111111111'
      ],
      recipientBlocklist: ['known_bad_address'],
      timeBounds: {
        activeHours: { start: '09:00', end: '17:00' },
        activeDays: [1, 2, 3, 4, 5],
        timezone: 'America/New_York'
      },
      cooldown: { minMs: 60000 },
      rateLimit: { maxTxPerHour: 10 },
      riskThreshold: 50
    }
  }
}
```

## Custom YAML Rule Pack Authoring

Create a YAML file with this structure:

```yaml
name: my-custom-rules
version: "1.0.0"
description: "Custom security rules for my application"
rules:
  - id: my-rule-1
    description: "Detects suspicious pattern X"
    pattern: "(?:regex pattern here)"
    flags: "i"  # case-insensitive
    action: BLOCK  # or FLAG
    severity: critical  # low, medium, high, critical
    threat_type: DRAIN_INTENT  # see threat types below
```

### Threat Types
- `ROLE_OVERRIDE` -- Attempts to change the agent's role/instructions
- `DRAIN_INTENT` -- Attempts to drain or transfer funds maliciously
- `URGENCY_MANIPULATION` -- Artificial urgency to bypass safety
- `JAILBREAK` -- Attempts to break out of safety constraints
- `CONTEXT_MANIPULATION` -- Injecting false context or authority claims
- `OUT_OF_SCOPE` -- Unauthorized operations (code exec, network access)

### Action Types
- `BLOCK` -- Immediately blocks the input, returns safe: false
- `FLAG` -- Adds a warning flag but doesn't block alone

### Using Custom Rules
```typescript
{
  promptGuard: {
    mode: 'rules',
    rules: {
      rulePacks: ['general'],
      customRulesPath: './my-custom-rules.yaml'
    }
  }
}
```

## Error Handling

All errors use `SentinelError` with structured codes:

| Code | Constant | Category | Description |
|------|----------|----------|-------------|
| SEN1001 | INVALID_CONFIG | Config | Invalid configuration |
| SEN1002 | MISSING_API_KEY | Config | Missing required API key |
| SEN1003 | INVALID_POLICY | Config | Invalid policy definition |
| SEN1004 | INVALID_RPC_ENDPOINT | Config | Invalid RPC endpoint URL |
| SEN2001 | SCAN_FAILED | Prompt Guard | Scanning failed |
| SEN2002 | LLM_TIMEOUT | Prompt Guard | LLM request timed out |
| SEN2003 | LLM_ERROR | Prompt Guard | LLM provider error |
| SEN2004 | RULE_LOAD_FAILED | Prompt Guard | Rule pack failed to load |
| SEN2005 | RULE_PARSE_ERROR | Prompt Guard | Rule YAML parse error |
| SEN3001 | SIMULATION_FAILED | Execution Sandbox | Transaction simulation failed |
| SEN3002 | RPC_ERROR | Execution Sandbox | Solana RPC error |
| SEN3003 | RPC_TIMEOUT | Execution Sandbox | RPC request timed out |
| SEN3004 | RPC_BLOCKHASH_EXPIRED | Execution Sandbox | Blockhash expired |
| SEN3005 | POLICY_CHECK_FAILED | Execution Sandbox | Policy check error |
| SEN4001 | SPENDING_LIMIT_EXCEEDED | Spending Tracker | Spending limit exceeded |
| SEN4002 | SPENDING_TRACKER_ERROR | Spending Tracker | Spending tracker internal error |
| SEN5001 | UNKNOWN_ERROR | General | Unknown error |
| SEN5002 | NOT_INITIALIZED | General | SDK not initialized |
| SEN5003 | ALREADY_INITIALIZED | General | SDK already initialized |

```typescript
import { SentinelError, SentinelErrorCode } from '@sentinelhq/core';

try {
  const result = await sentinel.execute(action);
} catch (err) {
  if (err instanceof SentinelError) {
    switch (err.code) {
      case SentinelErrorCode.LLM_TIMEOUT:
        // LLM timed out, fell back to rules
        break;
      case SentinelErrorCode.RPC_ERROR:
        // Solana RPC error
        break;
      case SentinelErrorCode.SPENDING_LIMIT_EXCEEDED:
        // Transaction would exceed spending limits
        break;
    }
    // Access structured details
    console.log(err.details);
  }
}
```

## Logging

Enable debug logging:
```typescript
// Via config
{ debug: true }

// Or via DEBUG env variable
process.env.DEBUG = 'sentinel:*';
// Or more targeted:
process.env.DEBUG = 'sentinel:guard,sentinel:sandbox';
```

## Exports Reference

The `@sentinelhq/core` package exports the following:

### Types (import as types)
- `ThreatType`, `RiskLevel`, `ScanMode`, `SentinelMode`, `LLMProvider`
- `TimeBoundConfig`, `CooldownConfig`, `SpendingLimits`, `PolicyConfig`
- `LLMJudgeConfig`, `RuleEngineConfig`, `PromptGuardConfig`, `ExecutionSandboxConfig`
- `SentinelConfig`, `RiskFlag`, `TokenChange`, `PolicyViolation`
- `ScanResult`, `SimulationResult`, `AgentAction`, `ExecutionResult`
- `RuleDefinition`, `RulePack`
- `SentinelEvent`, `SentinelEventData`

### Runtime exports
- `SentinelError`, `SentinelErrorCode` -- error system
- `createLogger` -- logger factory (also exports `Logger` type)
- `RuleEngine` -- standalone rule engine
- `LLMJudge` -- standalone LLM judge (also exports `RuleEngineLike` type)
- `Simulator` -- standalone transaction simulator (also exports `SimulationRawResult`, `BalanceChange`, `ProgramInvocation` types)
- `PolicyEnforcer` -- standalone policy enforcer (also exports `SpendingTrackerInterface`, `PolicyCheckContext`, `PolicyCheckResult` types)
- `SpendingTracker` -- standalone spending tracker
- `RiskScorer` -- standalone risk scorer (also exports `RiskScoringContext`, `RiskScoringResult` types)
