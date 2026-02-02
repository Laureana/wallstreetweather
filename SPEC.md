# Wall Street Weather — Technical Specification

## Key Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                      ENTITY                                 │
│                   (e.g., Roku Inc.)                         │
│                                                             │
│   ┌─────────────┐                                           │
│   │  CROWN KEY  │ ← the institution itself                  │
│   │  (entity)   │   verified in-person, once                │
│   └──────┬──────┘                                           │
│          │                                                  │
│          │ vouches for                                      │
│          ▼                                                  │
│   ┌─────────────┐    ┌─────────────┐    ┌───────────┐      │
│   │  HAND KEY   │    │  HAND KEY   │    │ HAND KEY  │      │
│   │  (CEO)      │    │  (CFO)      │    │ (IR Lead) │      │
│   │  * first    │    │             │    │           │      │
│   │    verified │    │             │    │           │      │
│   │    in-person│    │             │    │           │      │
│   └─────────────┘    └─────────────┘    └───────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Crown key** = the institution (persists across leadership changes)  
**Hand key** = an agent authorized to speak for the institution  
**WSW key** = counter-signature confirming passage through the system

Crown vouches for hands. Hands sign updates. Crown is rarely used — only when adding/removing agents.

## Three Signatures

| Signature | Meaning |
|-----------|---------|
| Hand key | "I, the CFO, am saying this" |
| Crown key | "The entity authorized this person to speak" |
| WSW key | "Wall Street Weather received and verified this" |

## Data Flow

```
Entity (agent) signs update with hand key
        ↓
Pushes to entity's own repo
        ↓
Aggregator pulls from all entity repos
        ↓
Aggregator verifies signatures
        ↓
Aggregator counter-signs with WSW key
        ↓
Publishes to frontend + mirrors
```

## JSON Schema

### Entity Record

```json
{
  "entity": {
    "id": "roku-inc",
    "name": "Roku, Inc.",
    "ticker": "ROKU",
    "crown_key_public": "ed25519:abc123...",
    "registered": "2026-03-15T00:00:00Z"
  },
  
  "agents": [
    {
      "hand_key_public": "ed25519:def456...",
      "role": "CEO",
      "name": "Anthony Wood",
      "vouched_by": "crown",
      "added": "2026-03-15T00:00:00Z"
    },
    {
      "hand_key_public": "ed25519:ghi789...",
      "role": "CFO", 
      "name": "...",
      "vouched_by": "crown",
      "added": "2026-04-01T00:00:00Z"
    }
  ]
}
```

### Metrics Update

```json
{
  "type": "metrics_update",
  "entity_id": "roku-inc",
  "timestamp": "2026-02-01T06:00:00Z",
  "signed_by": "ed25519:def456...",
  "previous_hash": "sha256:abc...",
  
  "payload": {
    "past_facing": {
      "revenue": 3900000000,
      "net_income": 120000000,
      "gross_margin_pct": 44
    },
    "future_facing": {
      "yoy_customer_growth_pct": 14,
      "rd_pct_of_revenue": 23,
      "customer_retention_pct": 96
    },
    "entity_chosen": [
      { "label": "Active Accounts", "value": 85000000 },
      { "label": "Streaming Hours", "value": 32000000000 },
      { "label": "ARPU", "value": 46 }
    ]
  },
  
  "hand_signature": "ed25519sig:xyz789...",
  "wsw_signature": "ed25519sig:abc123..."
}
```

### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `entity_id` | string | Unique identifier, lowercase, no spaces |
| `timestamp` | ISO 8601 | When update was created |
| `signed_by` | string | Public key of signing agent |
| `previous_hash` | string | SHA256 hash of prior update (creates chain) |
| `past_facing` | object | Standard backward-looking metrics |
| `future_facing` | object | Standard forward-looking metrics |
| `entity_chosen` | array | 1-5 custom metrics selected by entity |
| `hand_signature` | string | Agent's signature over payload |
| `wsw_signature` | string | Aggregator's counter-signature |

## Verification Flow

Anyone can verify:

1. Get update from log
2. Get entity record → find `hand_key_public` matching `signed_by`
3. Verify `hand_signature` matches payload
4. Verify hand key was vouched by crown
5. Verify `previous_hash` chains to prior update
6. Verify `wsw_signature` matches WSW public key

If any step fails → update is invalid, ignore.

## Architecture

```
┌──────────────────┐
│  ENTITY REPOS    │  ← Each entity maintains their own
│  (distributed)   │     signed feed
└────────┬─────────┘
         │ pull
         ▼
┌──────────────────┐
│   AGGREGATOR     │  ← Verifies signatures
│   (priest runs)  │     Counter-signs
└────────┬─────────┘     Publishes
         │
         ▼
┌──────────────────┐
│    FRONTEND      │  ← Static display
│    ("glass")     │     No logic
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│     MIRRORS      │  ← Redundant copies
│  (high-trust)    │     Anyone can run one
└──────────────────┘
```

## Key Registration

| Key | Verification method |
|-----|---------------------|
| Crown key | In-person ceremony, once |
| First hand key (CEO) | In-person, same time as crown |
| Subsequent hand keys | Crown key vouches (no in-person needed) |

Options for in-person verification:
- Law firm witnesses key generation
- Notary network trained in protocol
- Dedicated verifier corps

## Key Compromise

**Hand key compromised:** Crown revokes it, authorizes new one.

**Crown key compromised:** Lost forever. Entity registers new crown key via new in-person ceremony. Old signed data remains valid but frozen.

No recovery mechanism. Like Bitcoin.

## Update Frequency

Overnight, like SOFR. Not real-time, not quarterly.

Entities can update more frequently if they choose. The *frequency* of updates is itself signal.

## UI Principles

- Black background, cream text
- No colors, no branding, no decoration
- Order randomizes on page load (defeats alphabetical bias)
- Toggle sorts: shuffle, A→Z, Z→A, by any metric
- Integers displayed raw (3,900,000,000 not "3.9B")
- Percentages where specified (gross margin, R&D %)

## Governance

**The Priest:**
- Maintains aggregator and frontend
- Cannot edit signed data
- Stewards architecture, not content
- Can be replaced — anyone can fork

**If priest is compromised:**
- Entity repos still exist independently
- Anyone can build new aggregator
- Data survives because it's distributed

## Open Questions

1. **Specific signing library** — Ed25519 assumed, implementation TBD
2. **Mirror coordination** — How do high-trust mirrors sync?
3. **Entity onboarding flow** — Exact steps for first signup
4. **Verification ceremony details** — What does in-person look like?

These resolve during implementation, not before.

## Principles

- The data is the data
- The choice of what to report is signal
- The choice of what NOT to report is signal
- The frequency of updates is signal
- Absence is signal
- Numbers, not narratives
- Verification, not trust
