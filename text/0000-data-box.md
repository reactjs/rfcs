Start Date: (2026-04-30)
RFC PR: (leave this empty)
React Issue: (leave this empty)

# Summary

This RFC introduces a new primitive called Box — a sealed, referentially stable wrapper around data passed through the
React tree. Box does not trigger re-renders of components when its contents change. Instead, consumers must explicitly
"unwrap" the data they need using the `useUnwrap` hook, which subscribes only to the required slice of data. This
decouples data transport from data consumption, reducing unnecessary re-renders without manual memoization.

# Basic Example

## Simple Example

```tsx
const Child = ({ box }) => {
    const value = useUnwrap(box);

    return <div>{value}</div>;
};

// This component does not re-render when the parent's state changes
const Middle = React.memo(({ box }) => {
    return <Child box={box} />;
});

const Parent = () => {
    const [value, setValue] = useState(0);
    const box = useWrap(value);
    return (
        <>
            <Middle box={box} />
            <button onClick={() => setValue(value + 1)}>Increment</button>
        </>
    );
};
```

## Extended Example

```tsx
interface RowData {
    id: string;
    isChecked: boolean;
    title: string;
}

const TableRow: FC<{ row: Box<RowData> }> = props => {
    const title = useUnwrap(props.row, row => row.title);
    return (
        <>
            // Changing isChecked will only re-render a single checkbox, not the entire row or table.
            <Unwrap box={props.row} selector={row => row.isChecked}>
                {isChecked => <Checkbox isChecked={isChecked} />}
            </Unwrap>
            <div>{title}</div>
        </>
    );
};

const TableBody: FC<{ rows: Box<RowData[]> }> = props => {
    // Extract only a slice of the data. If the list of rows hasn't changed, the component won't re-render.
    // The selector must guarantee referential stability to prevent re-renders, but let's keep the example simple for now.
    const ids = useUnwrap(props.rows, rows => rows.map(row => row.id));

    return (
        <>
            {ids.map(id => (
                <ReWrap key={id} box={props.rows} selector={rows => rows.find(row => row.id === id)}>
                    {rowBox => <TableRow row={rowBox} />}
                </ReWrap>
            ))}
        </>
    );
};

const Table: FC<{ rows: RowData[] }> = props => {
    // Wrap the data in a Box. Direct access to the data is no longer available.
    const rowsBox = useWrap(props.rows);

    return (
        <>
            <TableHeader />
            <TableBody rows={rowsBox} />
        </>
    );
};
```

# Motivation

Imagine an analytics dashboard table. Each row represents a metric whose value changes every second. Each row contains
both display configuration (name, format) and the data itself (current value, percent change). The data-fetching logic
is either unknown to us or we lack the expertise to modify it without breaking other parts of the system.

```typescript
interface MetricRow {
    id: string;
    name: string; // rarely changes
    format: 'number' | 'percent' | 'bytes'; // rarely changes
    warningThreshold: number; // rarely changes
    criticalThreshold: number; // rarely changes
    currentValue: number; // updates every second
    trend: number[]; // updates every second
    lastUpdated: Date; // updates every second
}
```

The standard data model is a single `MetricRow[]` list. This code already exists in the application, and other similar
components may be written in the same way. But when used in React, a dilemma arises.

```tsx
interface CellProps {
    metric: MetricRow;
}

const NameCell: FC<CellProps> = ({ metric }) => {
    return <td>{metric.name}</td>;
};

const ValueCell: FC<CellProps> = ({ metric }) => {
    return <td>{formatValue(metric.currentValue, metric.format)}</td>;
};

const TrendCell: FC<CellProps> = ({ metric }) => {
    return (
        <td>
            <Chart value={metric.trend} />
        </td>
    );
};

const columnsMeta = [
    { name: 'Metric', key: 'name', Component: NameCell },
    { name: 'Value', key: 'currentValue', Component: ValueCell },
    { name: 'Trend', key: 'trend', Component: TrendCell },
];

const Dashboard: FC = () => {
    const metrics = useRealtimeMetrics(); // MetricRow[], updates every second

    return (
        <table>
            <thead>
                <tr>
                    {columnsMeta.map(column => (
                        <th key={column.key}>{column.name}</th>
                    ))}
                </tr>
            </thead>
            <tbody>
                {metrics.map(metric => (
                    <MetricRowView key={metric.id} metric={metric} />
                ))}
            </tbody>
        </table>
    );
};

const MetricRowView: FC<{ metric: MetricRow }> = ({ metric }) => {
    return <tr>{columnsMeta.map(column => React.createElement(column.Component, { metric, key: column.key }))}</tr>;
};
```

Every second `metrics` updates → every `MetricRowView` re-renders → every cell re-renders. That's n × m (n = total
number of rows, m = number of columns) component updates every second, even if only 5 rows actually changed.

## Why Memoization Is Not a Silver Bullet

`React.memo` on MetricRowView is useless in this case. The object reference for `metric` changes every second because
the entire array is recreated. To make `memo` work, significant refactoring would be required:

1. Refactor useRealtimeMetrics to guarantee immutability of `MetricRow` objects whose values haven't changed. If the
   data comes via WebSocket, an additional reconciliation layer would be needed.

2. Optionally, if step 1 is not feasible: stop passing the existing MetricRow object as props and instead describe all
   used fields as flat props on the component.

```tsx
<MetricRowView id={metric.id} name={metric.name} currentValue={metric.currentValue} {...otherProps} />
```

3. Refactor all cell components so they accept only the fields they need.

```tsx
const NameCell: FC<{ name: MetricRow['name'] }> = ({ name }) => {
    return <td>{name}</td>;
};

const ValueCell: FC<{ currentValue: MetricRow['currentValue']; format: MetricRow['format'] }> = ({
    currentValue,
    format,
}) => {
    return <td>{formatValue(currentValue, format)}</td>;
};

const TrendCell: FC<{ trend: MetricRow['trend'] }> = ({ trend }) => {
    return (
        <td>
            <Chart value={trend} />
        </td>
    );
};

const columnsMeta = [
    { name: 'Metric', key: 'name', Component: NameCell, propsMapper: (metric: MetricRow) => ({ name: metric.name }) },
    {
        name: 'Value',
        key: 'currentValue',
        Component: ValueCell,
        propsMapper: (metric: MetricRow) => ({ currentValue: metric.currentValue, format: metric.format }),
    },
    {
        name: 'Trend',
        key: 'trend',
        Component: TrendCell,
        propsMapper: (metric: MetricRow) => ({ trend: metric.trend }),
    },
];
```

Each of these steps requires significant refactoring. Step 1 is often impractical because the `useRealtimeMetrics` hook
may be used in other parts of the system that we don't control. The internal logic of `useRealtimeMetrics` itself may
also be inaccessible to us.

Good React performance currently relies on the entire system being set up correctly from end to end. If there's a
problem in the most foundational code, it negatively affects the entire application. In large, long-lived applications
there is always legacy code, and it's often used across a large portion of the application. This is precisely why teams
are reluctant to rewrite it — the risk of breaking one of the existing scenarios is too high.

## What Box Offers

_Box_ allows you to keep the convenient/existing data model and defer its reactivity to the point of consumption.

```tsx
interface CellProps {
    metric: Box<MetricRow>;
}

const NameCell: FC<CellProps> = ({ metric }) => {
    const name = useUnwrap(metric, m => m.name);
    return <td>{name}</td>;
};

const ValueCell: FC<CellProps> = ({ metric }) => {
    const value = useUnwrap(metric, m => formatValue(m.currentValue, m.format));
    return <td>{value}</td>;
};

const TrendCell: FC<CellProps> = ({ metric }) => {
    const trend = useUnwrap(metric, m => m.trend);
    return (
        <td>
            <Chart value={trend} />
        </td>
    );
};

const columnsMeta = [
    { name: 'Metric', key: 'name', Component: NameCell },
    { name: 'Value', key: 'currentValue', Component: ValueCell },
    { name: 'Trend', key: 'trend', Component: TrendCell },
];

const Dashboard: FC = () => {
    const metrics = useRealtimeMetrics(); // MetricRow[], updates every second

    return (
        <table>
            <thead>
                <tr>
                    {columnsMeta.map(column => (
                        <th key={column.key}>{column.name}</th>
                    ))}
                </tr>
            </thead>
            <tbody>
                {metrics.map(metric => (
                    <Wrap key={metric.id} data={metric}>
                        {metricBox => <MetricRowView metric={metricBox} />}
                    </Wrap>
                ))}
            </tbody>
        </table>
    );
};

const MetricRowView: FC<{ metric: Box<MetricRow> }> = React.memo(({ metric }) => {
    return <tr>{columnsMeta.map(column => React.createElement(column.Component, { metric, key: column.key }))}</tr>;
});
```

`metricBox` is an object with a stable reference. It does not cause `MetricRowView` or any of the row's cells to
re-render. `useUnwrap` triggers a re-render of individual cells only when the value returned by the selector has
changed. There was no need to adapt the data to React's standard memoization model, yet the updates are maximally
localized!

# Detailed Design

The core API consists of 3 hooks:

**useWrap** — wraps data in an inert wrapper

```typescript
function useWrap<T>(data: T): Box<T>;

const box = useWrap(data);
```

**useUnwrap** — makes a slice of the data reactive again. Importantly, we can extract only part of the data and ignore
changes to the rest.

```typescript
function useUnwrap<T, R>(box: Box<T>, selector: (data: T) => R): R;

const dataSlice = useUnwrap(box, data => data.something);
```

**useReWrap** — narrows the scope of the wrapper.

```typescript
function useReWrap<T, R>(box: Box<T>, selector: (data: T) => R): Box<R>;

const rowBox = useReWrap(box, data => data.rows[index]);
```

And 3 helper components for cases where hooks cannot be used:

**Wrap**, **Unwrap**, and **ReWrap**

```tsx
<Wrap data={data}>
    {box => <Child box={box}>}
</Wrap>

<Unwrap box={box}>
    {data => <Child data={data}>}
</Unwrap>

<ReWrap box={box} selector={data => data.rows[index]}>
    {rowBox => <Child box={rowBox}>}
</ReWrap>
```

At the core of all these hooks lies the `Box` primitive. This primitive serves as a direct bridge between two components
in the same React tree with a parent-child relationship. During rendering, the parent directly notifies all dependent
children, even if an intermediate child has indicated it has no work to do in this render cycle.

```tsx
interface Box<T> {
    getState: () => T;
    subscribe: (callback: () => void) => () => void;
}
```

Components using the `useUnwrap` hook only update when the data returned by the selector differs from the previous
render. Comparison is done via Object.is.

Users can write their own custom selector that uses React.ObjectRef or any other mechanism to achieve a stable selector
result in complex scenarios where the added complexity is justified.

Box is compatible with Concurrent Mode. The data being wrapped lives inside a React component, unlike
useSyncExternalStore, so React can manage the current value returned by the `useUnwrap` hook.

Box is not available in React Server Components. The limitations are the same as for regular React.Context and
useSyncExternalStore.

# Drawbacks

- New mental model. A concept of "boxed" data is introduced. Previously all data was always reactive; now that's no
  longer the case.
- Increased React API surface. 3 new hooks + 3 new components are added.
- Competition with react-compiler. The current paradigm assumes that memoization should be sufficient. If this process
  is automated and made maximally correct, the render phase should be fast enough. Under this assumption, manual
  optimizations may seem like unnecessary micro-optimization. However, not everyone is ready to adopt react-compiler,
  and it still cannot handle situations with non-optimizable data models.
- Executing selectors is not a free operation either.

# Alternatives

- Provide a low-level API that allows scheduling an update for a deeply nested child component within the same render
  cycle (not updating state, but signaling that a component has work to do). This would enable a full-featured
  implementation in userland. Now it possible to implement via combination of useSyncExternalStore and useLayoutEffect, but it splits single render in two and have other limitations.

- useContextSelector (a long-requested API) https://github.com/reactjs/rfcs/pull/119. This only solves the problem of
  context subscribers updating on any context change, without considering that they only need a slice of the data. `Box`
  solves a more general problem, but a Context with useContextSelector can be built on top of it.

- react-compiler. Solves the memoization problem, but not all code is amenable to memoization without significant
  refactoring.

- signals. Focused on state management; you can't simply wrap arbitrary data used in React. Feels like magic; hides
  implementation details behind proxies.

- zustand. Also focused on state management rather than data transport.

# Adoption Strategy

Publishing this feature requires no preparatory work. Its presence in React will not affect existing code in any way.
Users must decide on their own to add it to their code — adoption is entirely optional.

# How We Will Teach This

For teaching purposes, a box analogy can be used. It doesn't matter what we put in the box — what matters is that it's
still the same box. Therefore, all places that don't need the box's contents work fast. They simply take the box and
pass it along. They don't check whether anything was lost in previous steps — the box is sealed. Only when the contents
of the box truly matter does it get opened and its contents examined.

This is why the name Box is proposed for the entity — it's the shortest and most descriptive option. An alternative
would be Wrapper, but it's twice as long. The process of sealing into a box and unsealing is represented by useWrap and
useUnwrap.

It's also necessary to be able to connect different hooks and components that operate on different boxes. A dedicated
hook, useReWrap, is needed for this. One could potentially use a combination of useUnwrap and useWrap, but this would
cause unnecessary re-renders. A specialized hook solves this problem.

# Unresolved Questions

- If users want to write their own selector function that maintains a stable object reference in complex cases, how will
  this work in Concurrent Mode?
- Is it possible to write a useUnwrap that combines multiple Boxes to further reduce the number of re-renders? Or should
  useReWrap be able to combine multiple Boxes?
