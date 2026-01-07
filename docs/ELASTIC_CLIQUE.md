# Elastic Clique (ePoA) - Technical Documentation

## Overview

**Elastic Clique** (ePoA) is a modified Clique Proof-of-Authority consensus engine that dynamically adjusts block period based on network congestion. This enables faster block production during high-load periods (Burst Mode) and energy-efficient slower periods during idle times (Eco Mode).

## Design Goals

1. **Agentic AI Workloads**: Support bursty transaction patterns from autonomous AI agents
2. **Deterministic**: All nodes calculate the same period from the same parent block
3. **Backwards Compatible**: Standard Clique configurations work unchanged
4. **Configurable**: Parameters tunable via genesis.json

---

## Algorithm

### Period Calculation Formula

```
P_next = P_max - (U_parent / U_target) × (P_max - P_min)
```

Where:
- `P_next` = Block period for the next block (seconds)
- `P_max` = Maximum period (Eco Mode, default: 12s)
- `P_min` = Minimum period (Burst Mode, default: 2s)  
- `U_parent` = Parent block gas utilization (%)
- `U_target` = Target utilization threshold (default: 50%)

### Behavior Examples

| Parent Utilization | Calculated Period | Mode |
|-------------------|------------------|------|
| 0% | 12s | Eco Mode |
| 25% | 7s | Normal |
| 50% | 2s | Burst Mode |
| 100% | 2s | Burst Mode (clamped) |

---

## Implementation Details

### Modified Files

#### `params/config.go`

Extended `CliqueConfig` struct:

```go
type CliqueConfig struct {
    Period            uint64 `json:"period"`            // Base period (fallback)
    Epoch             uint64 `json:"epoch"`             // Epoch length
    MinPeriod         uint64 `json:"minPeriod,omitempty"`         // Burst Mode (default: 2)
    MaxPeriod         uint64 `json:"maxPeriod,omitempty"`         // Eco Mode (default: 12)
    TargetUtilization uint64 `json:"targetUtilization,omitempty"` // Target % (default: 50)
}
```

#### `consensus/clique/clique.go`

**1. `CalcDynamicPeriod(parent *types.Header) uint64`**

Core algorithm implementation with:
- Backwards compatibility check (returns `Period` if ePoA not configured)
- Genesis block fallback
- Zero gas limit protection
- Integer arithmetic with clamping

**2. `verifyCascadingFields` Modification**

```go
// Before (standard Clique):
if parent.Time + c.config.Period > header.Time {
    return errInvalidTimestamp
}

// After (Elastic Clique):
dynamicPeriod := c.CalcDynamicPeriod(parent)
log.Info("Elastic Clique Adjustment", "parent", parent.Number.Uint64(),
    "utilization", utilization, "period", dynamicPeriod)
if parent.Time + dynamicPeriod > header.Time {
    return errInvalidTimestamp
}
```

**3. `Prepare` Modification**

Uses `CalcDynamicPeriod(parent)` for header timestamp calculation.

**4. `Seal` Instrumentation**

Adds research logging before block sealing:
```go
log.Info("Elastic Clique Seal", "block", number,
    "parentUtilization", utilization, "period", dynamicPeriod)
```

---

## Configuration

### Genesis.json Example

```json
{
  "config": {
    "chainId": 1337,
    "clique": {
      "period": 5,
      "epoch": 30000,
      "minPeriod": 2,
      "maxPeriod": 12,
      "targetUtilization": 50
    }
  },
  "extradata": "0x0000000000000000000000000000000000000000000000000000000000000000<signer-address>0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "gasLimit": "0x1c9c380",
  "difficulty": "0x1",
  "alloc": {}
}
```

### Parameter Guidelines

| Parameter | Conservative | Aggressive | Description |
|-----------|-------------|------------|-------------|
| `minPeriod` | 3 | 1 | Minimum block time under high load |
| `maxPeriod` | 15 | 10 | Maximum block time when idle |
| `targetUtilization` | 60 | 40 | Threshold to trigger faster blocks |

---

## Backwards Compatibility

When all ePoA parameters (`minPeriod`, `maxPeriod`, `targetUtilization`) are **zero or omitted**, the engine operates in **standard Clique mode** using the fixed `period` value.

```go
// In CalcDynamicPeriod:
if c.config.MinPeriod == 0 && c.config.MaxPeriod == 0 && c.config.TargetUtilization == 0 {
    return c.config.Period  // Standard Clique behavior
}
```

---

## Logging Output

### Verification Log
```
INFO [01-07|11:00:00.000] Elastic Clique Adjustment parent=1234 utilization=75 period=2
```

### Sealing Log  
```
INFO [01-07|11:00:02.000] Elastic Clique Seal block=1235 parentUtilization=75 period=2
```

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Genesis block (block 0) | Uses `config.Period` |
| Parent is nil | Uses `config.Period` |
| Parent GasLimit = 0 | Uses `config.Period` |
| Utilization > 100% | Clamped to `minPeriod` |
| ePoA not configured | Uses `config.Period` (standard Clique) |

---

## Testing

```bash
# Build
go build ./...

# Run Clique tests
go test ./consensus/clique/... -v

# Run all tests
go test ./...
```

---

## Fork Considerations

> ⚠️ **IMPORTANT**: All nodes in the network MUST use the same ePoA-enabled binary with identical configuration. Mixing standard Clique nodes with ePoA nodes will cause consensus failures.

---

## Version

- **Base**: go-ethereum v1.13.15
- **Modification**: Elastic Clique (ePoA) v1.0.0
- **Author**: Research Implementation
- **Date**: 2026-01-07
