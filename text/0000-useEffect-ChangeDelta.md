* **Start Date:** 2026-06-03
* **RFC PR:** (leave this empty)
* **React Issue:** (leave this empty)

### Summary

I propose extending the callback signature of `useEffect` (and subsequently `useLayoutEffect`) to accept an optional metadata argument containing the transaction details of the dependency change that triggered the runtime execution loop. This payload exposes the current dependency snapshots, the previous dependency snapshots, and an array of numeric indices pointing directly to the specific elements that failed equality checks.

### Basic example

```javascript
import { useEffect, useState } from 'react';

export function DataStreamController() {
  const [streamUrl, setStreamUrl] = useState('https://api.v1.source');
  const [filterConfig, setFilterConfig] = useState({ limit: 50 });

  useEffect((changeDelta) => {
    // Initial mount pass has no delta payload
    if (!changeDelta) return;

    const { currentDepValues, previousDepValues, updatedDepIndices } = changeDelta;

    // Index 0 mapping: streamUrl
    if (updatedDepIndices.includes(0)) {
      console.log(`Stream target shifted from ${previousDepValues[0]} to ${currentDepValues[0]}. Resetting socket connections.`);
      // hardResetSocket(currentDepValues[0]);
    }

    // Index 1 mapping: filterConfig
    if (updatedDepIndices.includes(1)) {
      console.log('Filters updated. Dispatching configuration payload downstream.');
      // applyPatchFilter(currentDepValues[1]);
    }
  }, [streamUrl, filterConfig]); // Index 0, Index 1
}

```

### Motivation

In the legacy class-component architecture, the `componentDidUpdate(prevProps, prevState)` lifecycle method provided granular, explicit insight into exactly *why* a component re-rendered. This allowed developers to write straightforward differential execution logic inside a single, cohesive lifecycle block.

Modern functional components lack an ergonomic equivalent. When a multi-dependency `useEffect` fires, the callback executes blindly, completely unaware of which dependency triggered the execution frame.

To bypass this limitation, developers currently resort to two anti-patterns:

1. **Fragmentation:** Splitting a single conceptually unified side-effect into 3 or 4 disconnected `useEffect` calls simply because different trigger sources require slightly different handling. This scatters state logic and multiplies reconciliation hooks.
2. **Boilerplate Bloat:** Manually introducing `useRef` snapshots to store previous values and running custom, unoptimized diffing logic directly inside user-land scripts.

By providing a structured change descriptor—conceptually similar to how `useReducer` delivers an action payload to categorize execution—we allow developers to safely branch execution paths within a unified, high-performance side effect.

### Detailed Design

The proposed typing signature updates the standard `EffectCallback` definition to support an optional introspection parameter:

```typescript
type DependencyList = ReadonlyArray<unknown>;

interface DependencyChangeDelta<T extends DependencyList> {
  currentDepValues: T;
  previousDepValues: T;
  updatedDepIndices: number[]; // Array of indices where Object.is() evaluated to false
}

type EffectCallback<T extends DependencyList> = (
  delta?: DependencyChangeDelta<T>
) => void | Destructor;

```

#### Fiber Reconciler Core Mechanics:

During the commit phase, React already iterates over the dependency array to compare the `memoizedState` of the hook against the newly passed values using `Object.is`.

Because this comparison occurs natively inside the core architecture loop, capturing the indices of failing elements adds virtually **zero computational overhead**. Instead of discarding this change matrix after evaluation, the reconciler compiles the results into a temporary transaction object and injects it directly into the user-allocated effect callback.

### Drawbacks

* **Array Index Fragility:** Relying on hardcoded numeric indices (`updatedDepIndices.includes(0)`) inside the logic block can become brittle if developers reorder items within the dependency array. However, this risk is mitigated by standard linting matrices (`react-hooks/exhaustive-deps`) and robust unit testing patterns.
* **Argument Complexity:** Adding arguments to an historically empty callback signature increases the conceptual surface layer of the hook for newer developers, though it remains entirely opt-in.

### Alternatives

* **User-land Custom Hook Wrapper:** Developers can build abstractions utilizing tracking references (`useRef`). However, performing custom snapshot arrays and running secondary evaluation loops in user-land scripts entirely bypasses the internal optimizations of the React Fiber reconciliation diffing algorithm, introducing unnecessary performance overhead and rendering latency.
* **Object-keyed Dependencies:** Instead of numeric indices, the change payload could map to named keys. However, because React dependency lists are natively ordered arrays rather than keyed dictionaries, utilizing indices preserves direct architectural alignment with the existing hook specification.
To illustrate the user-land overhead, a developer would have to implement a heavy tracking abstraction similar to the following:
 - Example Custom Hook Declaration
```javascript
import { useEffect, useRef } from 'react';

function isDeepEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== 'object' || a === null || typeof b !== 'object' || b === null) return false;
  if (Array.isArray(a) && Array.isArray(b)) {
    return a.length === b.length && a.every((val, i) => isDeepEqual(val, b[i]));
  }
  const keysA = Object.keys(a), keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every(k => Object.prototype.hasOwnProperty.call(b, k) && isDeepEqual(a[k], b[k]));
}

export function useEffectWithDetails(effect, deps) {
  const prevDepsRef = useRef(null);

  useEffect(() => {
    // Treat the initial mount as a baseline initialization step
    if (prevDepsRef.current === null) {
      prevDepsRef.current = deps;
      return;
    }

    // Determine exactly which indices broke deep equality
    const updatedDeps = [];
    deps.forEach((currentDep, index) => {
      const previousDep = prevDepsRef.current[index];
      if (!isDeepEqual(currentDep, previousDep)) {
        updatedDeps.push(index);
      }
    });

    const changeMetadata = {
      currentDepValues: deps,
      previousDepValues: prevDepsRef.current,
      updatedDeps: updatedDeps // Array of indices denoting which items changed
    };

    // Keep the reference current for the subsequent render loop
    prevDepsRef.current = deps;

    // Only fire the callback if an actual mutation occurred
    if (updatedDeps.length > 0) {
      return effect(changeMetadata);
    }
  }, deps);
}
```
- Example Custom Hook Implementation

```javascript
import React, { useState } from 'react';
import { useEffectWithDetails } from './useEffectWithDetails';

export function TelemetryViewer() {
  const [sessionToken, setSessionToken] = useState('auth_abc');
  const [dataPoints, setDataPoints] = useState([10, 20, 30]);
  const [deviceConfig, setDeviceConfig] = useState({ model: 'X', firmware: 1.0 });

  // Map out your dependencies clearly to know what the index positions mean
  // Index 0: sessionToken, Index 1: dataPoints, Index 2: deviceConfig
  useEffectWithDetails((meta) => {
    console.log('🔄 Effect triggered with changes:', meta);
    
    if (meta.updatedDeps.includes(0)) {
      console.log('Session token rotated. Re-authenticating background streams...');
    }
    
    if (meta.updatedDeps.includes(1)) {
      console.log('New real-time stream coordinates appended. Updating map vectors...');
    }
    
    if (meta.updatedDeps.includes(2)) {
      console.log('Hardware specs mutated. Re-allocating graphics memory context...');
    }
  }, [sessionToken, dataPoints, deviceConfig]);

  return (
    <div>
      <button onClick={() => setDataPoints([10, 20, 30, 40])}>
        Append Coordinates (Changes Index 1)
      </button>
      <button onClick={() => setDeviceConfig(prev => ({ ...prev, firmware: 1.1 }))}>
        Upgrade Firmware (Changes Index 2)
      </button>
    </div>
  );
}
```

Ultimately, this user-land abstraction fails because duplicating dependency tracking and performing iterative identity-checking passes inside individual component lifecycles forces unoptimized computation loops outside the native Fiber tree evaluation cycle, introducing completely unnecessary execution overhead and micro-render delays across complex layout structures.

### Adoption Strategy

This feature is completely non-breaking and backwards-compatible. If an existing `useEffect` declaration omits the callback parameter, the argument is ignored entirely. This can be cleanly introduced in any minor version release without requiring breaking structural transformations or codemods.

### How we teach this

This capability should be introduced under advanced documentation guidelines as **Effect Trigger Introspection**. It should be explicitly framed as an optimization pattern meant to consolidate complex, multi-variable synchronization logic inside enterprise dashboard or data-fetching layouts, drawing a clear functional parallel to how action payloads orchestrate execution inside `useReducer`.

### Unresolved questions

* **Batching Manifestation:** If multiple independent state parameters update within a single batched event loop transaction, how should concurrent index changes be sequenced? (The design specifies that `updatedDepIndices` naturally aggregates all mutations encountered during that specific render sweep).
