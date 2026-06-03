* **Start Date:** 2026-06-03
* **RFC PR:** (leave this empty)
* **React Issue:** (leave this empty)

### Summary

`useEffectAll` is a proposed addition to the React hooks primitive suite that alters the dependency resolution logic from an inclusive **OR** operation to a strict, concurrent **AND** operation. The hook triggers its effect callback exclusively when every single member of the provided dependency array has mutated compared to its value in the preceding render loop.

### Basic example

```javascript
import { useEffectAll } from 'react';

useEffectAll(() => {
  // Executes strictly when BOTH state variables change in the same transaction
  console.log("Paradigm shift detected. Synchronizing execution engines...");
  
  return () => {
    // Teardown resource lanes
  };
}, [operationalMatrix, routingPolicy]);

```

### Motivation

React's standard `useEffect` evaluates dependencies individually. If any item within the array undergoes a change, the scheduler immediately queues the effect callback.

In optimized enterprise layers, this inclusive evaluation causes architectural execution bottlenecks when dealing with compound, batched updates. High-overhead side effects—such as resetting web worker pools, wiping local offline databases, or initiating expensive analytical workflows—frequently depend on a concurrent threshold. They should only fire if all watched metrics alter collectively, rather than responding incrementally to a single isolated field change. Currently, developers must manage duplicate historical snapshots with multiple tracking references (`useRef`) to bypass partial frames, polluting component layout architecture.

### Detailed Design

`useEffectAll` mirrors the signature of `useEffect` but wraps the frame evaluation cycle inside a strict conditional checking pass.

```typescript
export function useEffectAll(
  effect: EffectCallback, 
  deps: DependencyList
): void;

```

#### Reconciler Lifecycle Rules:

1. **Initial Mount Phase:** The reconciler captures and caches the initial dependency array state. The effect callback is skipped to establish a valid comparative baseline.
2. **Evaluation Phase:** Upon successive render loops, the hook processes the array using a strict logical assertion loop.
3. **Equality Evaluation:** The hook performs reference comparisons (`Object.is`) against the cached historical state. If any single item passes this check (meaning it did not change), the entire evaluation fails early.
4. **Trigger Condition:** The side effect executes if and only if every element in the array breaks equality with its past state simultaneously.
5. **Reference Update:** The cached dependency snapshot is updated at the conclusion of every execution pass to maintain accuracy against the immediate past frame.

### Drawbacks

* **Reference Stability Risks:** Passing unstable object or array literals that accidentally regenerate on every render pass will break the **AND** validation logic. This hazard exists natively in `useEffect` but is amplified when multiple variables must align cleanly.
* **Library Complexity:** Introducing another core primitive tracking hook slightly increases the API surface layer of the library, though the processing loop adds negligible overhead to the internal fiber reconciliation loop.

### Alternatives

Developers can maintain custom wrapper configurations in user space. However, managing user-land snapshot synchronization outside the unified state reconciler loop introduces frame timing discrepancies and forces duplicate custom caching utility logic across disconnected projects.

Developers can attempt to maintain custom wrapper configurations using `useRef` snapshots in user space. However, a user-land implementation introduces significant structural limitations:

* **Reference Breakdowns:** Traditional user-land reference caching struggles when comparing complex user-defined objects or tracking mutations on ref-attached DOM elements, frequently leading to stale closures or missed evaluations.
* **Fiber Engine Optimization:** This is where React’s internal comparison engine shines. By implementing this natively within the React Fiber reconciliation diffing algorithm, the reconciler can evaluate dependency arrays natively and performantly, without forcing developers to execute expensive, high-overhead deep-equality utility loops in user-land scripts.

- Example Custom Hook Declaration
```javascript
import { useEffect, useRef } from 'react';

// Native, optimized deep equality checker
function isDeepEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== 'object' || a === null || typeof b !== 'object' || b === null) return false;
  
  if (Array.isArray(a) && Array.isArray(b)) {
    if (a.length !== b.length) return false;
    return a.every((val, index) => isDeepEqual(val, b[index]));
  }

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => Object.prototype.hasOwnProperty.call(b, key) && isDeepEqual(a[key], b[key]));
}

/**
 * useEffectAll
 * Strictly executes the callback ONLY when every single item in the deps array has changed.
 */
export function useEffectAll(effect, deps) {
  const prevDepsRef = useRef(null);

  useEffect(() => {
    // 1. Establish the baseline on initial mount
    if (prevDepsRef.current === null) {
      prevDepsRef.current = deps;
      return;
    }

    // 2. Strict evaluation: Has every single item broken deep equality?
    const allChanged = deps.every((currentDep, index) => {
      return !isDeepEqual(currentDep, prevDepsRef.current[index]);
    });

    // 3. Keep memory fresh for the next render loop
    prevDepsRef.current = deps;

    // 4. Fire the callback only if condition is completely met
    if (allChanged) {
      return effect();
    }
  }, deps);
}
```

- Example Implementation
 ```javascript
import React, { useState } from 'react';
import { useEffectAll } from './useEffectAll';

export function CreditUnderwritingConsole() {
  // A single enterprise state object holding multiple underwriting metrics
  const [loanApplication, setLoanApplication] = useState({
    applicantName: "Global Logistics Inc",
    requestedAmount: 5000000,
    // The 3 critical risk vectors we want to watch collectively:
    collateralValue: 3500000,
    outstandingDebt: 1200000,
    guarantorScore: 710,
    // Non-critical metadata
    legalJurisdiction: "DE",
    riskTier: "Medium"
  });

  // Target exactly the 3 metrics that must ALL change together
  useEffectAll(() => {
    console.warn("💰 CRITICAL ACTION: Running Expensive Comprehensive Underwriting Algorithm...");
    // triggerHeavyRiskAPI(loanApplication);
    
  }, [
    loanApplication.collateralValue, 
    loanApplication.outstandingDebt, 
    loanApplication.guarantorScore
  ]);

  // SCENARIO 1: A system webhook or "Apply Template" action mutates all 3 risk factors at once
  const handleBulkFinancialRestructure = () => {
    setLoanApplication(prev => ({
      ...prev,
      collateralValue: 6000000,   // Changed
      outstandingDebt: 4500000,   // Changed
      guarantorScore: 640,        // Changed
      riskTier: "High"
    }));
  };

  // SCENARIO 2: Underwriter edits fields individually. Standard useEffect would fire here; we block it.
  const handleSingleFieldAdjustment = () => {
    setLoanApplication(prev => ({
      ...prev,
      collateralValue: 3900000,   // Changed
      // outstandingDebt remains 1200000 (Unchanged)
      // guarantorScore remains 710 (Unchanged)
    }));
  };

  return (
    <div style={{ padding: '20px', border: '2px solid #222' }}>
      <h3>Risk Underwriting Matrix</h3>
      <div style={{ background: '#eee', padding: '10px', margin: '10px 0' }}>
        <p>Collateral: ${loanApplication.collateralValue}</p>
        <p>Debt: ${loanApplication.outstandingDebt}</p>
        <p>Score: {loanApplication.guarantorScore}</p>
      </div>

      <button onClick={handleBulkFinancialRestructure} style={{ backgroundColor: 'red', color: 'white' }}>
        Import Restructured Financial Spreadsheet (Triggers Hook)
      </button>

      <button onClick={handleSingleFieldAdjustment} style={{ marginLeft: '10px' }}>
        Tweak Collateral Value Only (Safely Ignored)
      </button>
    </div>
  );
}
```

Ultimately, managing snapshot validation outside the unified state reconciler loop introduces frame timing discrepancies and forces duplicate, unoptimized caching logic across enterprise codebases.

### Adoption Strategy

This is a non-breaking, purely additive feature introduction. It can be safely introduced as part of a minor React core update. No codemods are necessary, and existing implementations of standard hooks remain completely untouched.

### How we teach this

`useEffectAll` should be taught as a specialized primitive engineered for **Transactional State Evaluation**. It bridges the gap between high-frequency interface rendering updates and low-frequency backend synchronization side effects. The concept fits naturally into the official documentation under an advanced hooks synchronization guide, emphasizing its clean usage alongside identity-stable primitives or memoized configurations.

### Unresolved questions

* **Custom Comparator Functionality:** Should `useEffectAll` accept an optional third argument allowing developers to pass a custom comparator function (e.g., recursive structural deep checking) to bypass standard shallow reference limitations for complex configurations?
