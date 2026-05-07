# Identity Verification — Force-Verify Integration

> Integration of identity registry into the Force-Verify anti-hallucination system.
> Not a separate module. An additional verification layer.

## Concept

Force-Verify currently verifies **actions** (file writes, command execution, API calls).
Identity Verification adds the verification of **self-awareness** — the AI's understanding of who it is.

Every time the AI makes a claim about itself, the system cross-checks against the registry.

## How It Fits Into Force-Verify

```
VerifyLevel:
  NONE = 0    # No verification
  BASIC = 1   # Type check
  SCHEMA = 2  # Schema validation
  DUAL = 3    # Two independent verifiers
  HUMAN = 4   # Human approval
  + IDENTITY = 5  # NEW: Identity check (added as optional level)
```

## Integration Points

### 1. New Method in ForceVerify

```python
class ForceVerify:
    def __init__(self):
        self.registry = IdentityRegistry()  # lightweight, ~15ms load
    
    def verify_identity(self, claimed_identity: dict) -> VerifyResult:
        """
        Cross-check what the AI claims about itself against the registry.
        
        Args:
            claimed_identity: What the AI says (e.g., "I'm running Full edition")
        
        Returns:
            VerifyResult with match score
        """
        # Get the TRUTH from registry
        truth = self.registry.get_identity()
        
        checks = []
        
        # Check name
        if claimed_identity.get("name") == truth.get("name"):
            checks.append(("name", True))
        else:
            checks.append(("name", False, f"Claimed '{claimed_identity.get('name')}' but actually '{truth.get('name')}'"))
        
        # Check generation
        if claimed_identity.get("generation") == truth.get("generation"):
            checks.append(("generation", True))
        
        # Check edition
        truth_edition = truth.get("edition", "lite")
        if claimed_identity.get("edition", "lite") == truth_edition:
            checks.append(("edition", True))
        else:
            checks.append(("edition", False, f"Claimed {claimed_identity.get('edition')} but running {truth_edition}"))
        
        passed = all(c[1] for c in checks)
        return VerifyResult(
            passed=passed,
            level=VerifyLevel.IDENTITY,
            method="registry_match",
            details=checks
        )
```

### 2. Before Every AI Response

```python
# In the worker loop, before sending AI response:
verify_result = force_verify.verify_identity({
    "name": ai_claimed_name,
    "generation": ai_claimed_gen,
    "edition": ai_claimed_edition,
})

if not verify_result.passed:
    # Inject correct identity into response
    corrected = registry.get_identity()
    response = f"[Identity corrected] I am {corrected['name']}, gen {corrected['generation']}"
```

### 3. Self-Correction Flow

```
AI says: "I'm running on Oracle with Full edition"
         ↓
ForceVerify.verify_identity()
         ↓
Registry says: "This deployment is Seoul, Lite edition"
         ↓
Verification FAILS
         ↓
System corrects: "I am ani, deployment ani-seoul-v3, generation 3, Lite edition"
         ↓
AI accepts correction (can't argue with verified data)
```

## Lite Edition Behavior

On 1C1G (Oracle Micro), the identity check is even simpler:

```python
# Lite: no JSON file, just config-based
def verify_identity_lite(config: dict):
    """Simple identity check from config values."""
    expected = {
        "name": config.get("name"),
        "edition": config.get("edition", "lite"),
    }
    return VerifyResult(passed=True, level=VerifyLevel.BASIC, method="config_check")
```

## Implementation

| Component | Change | Lines |
|-----------|--------|-------|
| `core/verify.py` | Add IDENTITY level + verify_identity() method | +30 |
| `core/identity.py` | IdentityRegistry class (was separate spec) | ~80 |
| `anchor.json` | Add registry + deployments sections | ~30 |
| Worker boot | Call verify_identity after loading AI | +5 |

**Total: ~115 lines. Zero new dependencies. Fits in Lite edition.**
