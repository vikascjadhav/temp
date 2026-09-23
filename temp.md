             Merge Request
                   │
        ┌──────────┴──────────┐
        │                     │
 Deterministic            AI Review
 Checks                    Agent
        │                     │
 ├─ compile               ├─ logic
 ├─ unit tests             ├─ architecture
 ├─ SAST                   ├─ maintainability
 ├─ SCA                    ├─ cross-file reasoning
 ├─ secrets                ├─ domain rules
 └─ quality gate           └─ test gaps
        │                     │
        └──────────┬──────────┘
                   ↓
              Human reviewer
                   ↓
                 MERGE
