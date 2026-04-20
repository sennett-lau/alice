# Architecture

Present tense. What runs today.

## Overview

<2–4 paragraphs. Components, services, runtime shape. How a request / event flows through the system end-to-end.>

## Packages / services

- `<name>` — <one-line role + key tech>
- `<name>` — <one-line role + key tech>

## Runtime topology

```
<ASCII or mermaid diagram of services, data flow, external deps>
```

## External dependencies

- <provider> — <what it does for us, what it costs, fallback if it goes down>

## Boundaries

- **Trust boundaries:** <where untrusted input enters, where validation happens>
- **Process boundaries:** <which services run separately, what they share>
- **Deployment boundaries:** <prod / staging / local differences>

## Update when

- New service added or removed.
- A trust boundary moves.
- A primary external dependency changes.
- The runtime topology changes (new queue, new cache, new gateway).
