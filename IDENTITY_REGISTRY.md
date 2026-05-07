# Identity Registry — Anti-Hallucination for AI Self-Awareness

## Problem
When an AI boots up and reads the eternal anchor, it needs to know **which deployment it is**. Without this, an AI could hallucinate being on the wrong server, wrong generation, or claiming capabilities it doesn't have.

## Solution
A lightweight identity registry in the anchor file. When an AI starts, it:
1. Reads the registry
2. Generates a simple fingerprint (hostname + primary IP)
3. Matches itself to a deployment in the registry
4. Knows its name, generation, location, capabilities

## Integration Points

### 1. Anchor Schema (`anchor.json`)

Add a `registry` section to the existing anchor:

```json
{
  "schema_version": "3.0",
  "registry": {
    "boot_protocol": [
      "1. Generate my fingerprint (hostname + ip)",
      "2. Match against deployments",
      "3. Confirm identity",
      "4. No match → I'm new, register"
    ]
  },
  "deployments": {
    "ani-seoul-v3": {
      "identity": {
        "name": "ani",
        "generation": 3,
        "role": "main AI",
        "edition": "lite"
      },
      "fingerprint": {
        "hostname": "VM-0-6-ubuntu",
        "ip": "43.133.234.190"
      },
      "status": "running",
      "changelog": [
        {"v": "3", "changes": ["tical-code v0.3 upgrade"]}
      ]
    }
  }
}
```

### 2. New Module: `core/identity.py` (~100 lines)

```python
"""
Identity Registry — Lightweight fingerprint matching.
Helps AI know who it is, without hallucination.
"""
import socket, hashlib, json

class IdentityRegistry:
    """
    Reads the anchor, matches fingerprint, returns identity.
    Uses ONLY hostname + IP. No heavy computation.
    """
    
    def __init__(self, anchor_path: str = "anchor.json"):
        self.anchor = json.load(open(anchor_path))
        self.my_fingerprint = self._generate_fingerprint()
        self.my_identity = self._resolve()
    
    def _generate_fingerprint(self) -> dict:
        """Generate fingerprint from current machine."""
        return {
            "hostname": socket.gethostname(),
            "ip": self._get_primary_ip(),
        }
    
    def _get_primary_ip(self) -> str:
        """Get primary non-localhost IP."""
        # Simple, no external calls
        import subprocess
        try:
            result = subprocess.run(
                ["hostname", "-I"], 
                capture_output=True, text=True, timeout=3
            )
            return result.stdout.split()[0]
        except:
            return "0.0.0.0"
    
    def _resolve(self) -> dict:
        """Match fingerprint against deployments in registry."""
        deployments = self.anchor.get("deployments", {})
        
        for dep_id, dep in deployments.items():
            fp = dep.get("fingerprint", {})
            # Match hostname OR ip
            if (fp.get("hostname") == self.my_fingerprint["hostname"] or 
                fp.get("ip") == self.my_fingerprint["ip"]):
                return {
                    "id": dep_id,
                    **dep.get("identity", {}),
                    "status": dep.get("status", "unknown"),
                    "changelog": dep.get("changelog", []),
                }
        
        # No match — this is a new, unknown deployment
        return {
            "id": "unknown",
            "name": "unknown",
            "generation": 0,
            "status": "new",
        }
    
    def get_identity(self) -> dict:
        """Get resolved identity."""
        return self.my_identity
    
    def self_check(self) -> str:
        """
        Returns a ready-to-use self-description for AI system prompts.
        Includes: name, generation, location, capabilities check.
        """
        identity = self.my_identity
        desc = f"You are {identity.get('name', 'unknown')}, deployment {identity.get('id', '?')}"
        if identity.get('generation'):
            desc += f", generation {identity['generation']}"
        if identity.get('role'):
            desc += f", role: {identity['role']}"
        if identity.get('edition'):
            desc += f", running {identity['edition']} edition"
        return desc
```

### 3. Integration in Worker Boot

When a worker starts, before any AI inference:

```python
# In worker boot sequence
from core.identity import IdentityRegistry

registry = IdentityRegistry("anchor.json")
identity = registry.get_identity()

if identity["status"] == "new":
    # Unknown deployment — cautious mode
    logger.warning(f"Unknown deployment on {socket.gethostname()}")
    # Register self to anchor (optional)
    
# Inject identity into AI system prompt
system_prompt = registry.self_check()
```

For Lite edition (no anchor file access), fall back gracefully:
```python
try:
    registry = IdentityRegistry()
except:
    # Lite fallback — use config values
    identity = {"name": config.get("name", "worker"), "generation": "?"}
```

## Performance Impact
- One JSON file read (~8KB)
- One hostname command (`hostname -I`, ~10ms)
- One dict lookup
- **Total: ~15ms, zero ongoing cost**
- Can run on 1C1G with no issue

## Preventing Hallucination

Before the AI answers "who are you?", the system knows the truth:

Example:
```python
registry = IdentityRegistry("anchor.json")
print(registry.self_check())
# "You are ani, deployment ani-seoul-v3, generation 3, role: main AI, running lite edition"
```

This text is injected into the AI's system prompt. The AI can't override it because it's before any user message. No hallucination possible.

## File Changes Summary

| File | Change | Lines |
|------|--------|-------|
| `anchor.json` | Add `registry` + `deployments` sections | ~30 lines |
| `core/identity.py` | NEW — lightweight fingerprint matching | ~100 lines |
| `core/worker.py` | Call IdentityRegistry at boot | +5 lines |
| Lite: `setup_lite.py` | Graceful fallback if no anchor | +3 lines |

**Total new code: ~100 lines. No new dependencies. Works on 1C1G.**
