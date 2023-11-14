- Start Date: 2023-11-14
- RFC PR:
- React Issue:

# Summary

This proposal aims to enhance the TextInput component by introducing
advanced tracking of text selection positions. The feature will provide
developers with the coordinates (x and y) for both the start and end of
the text selection and cursor within a TextInput. This enhancement will
facilitate functionalities like inline suggestions
(emojis, text suggestions) based on the selection, thereby improving
the interactivity and user experience of text input fields

# Basic example

```js
  type SelectionType = {
    start: number,
    end: number,
    cursorPosition: {
        start: {x: number, y: number},
        end: {x: number, y: number}
    }
  }

  <TextInput
    onSelectionChange={(e) => {
      const selection: SelectionType = e.nativeEvent.selection;
    }}
  />
```


# Motivation

The current implementation of TextInput in React Native lacks the ability
to measure the content of the selection or the cursor position. This limitation
hinders the development of features like context-aware suggestions or tools that
need precise cursor location (e.g., rich text editors, autocomplete functionalities).
By providing detailed coordinates for the cursor and selection, developers can
enhance user experience with dynamic and interactive text input features.

# Detailed design

The proposed design includes:

1. Enhanced Selection Data: Updating the `onSelectionChange` event's data to include
`selectionPosition`, which will contain the coordinates `{x: number, y: number}`
for the start and end points of the text selection.
```js
  selection: {
    start: number,
    end: number,
    cursorPosition: {
      start: { x: number, y: number },
      end: { x: number, y: number }
    }
  }
```

2. Type Definitions and Flow Updates: Adjusting type definitions in TextInput.d.ts
and TextInput.flow.js to integrate the new data structure.

3. Native Implementation: Modifying native methods in iOS (RCTBaseTextInputView.m)
and Android (ReactTextInputManager.java) to capture and transmit the selection position
data.

4. Event Handling and Dispatch: Revising event handling to accommodate the new selection
position data.

5. TextInput Component Adaptation: Updating the TextInput component to handle and utilize
the new selection data format.

6. Testing and Examples: Providing updated examples and test cases to demonstrate the
usage of the enhanced feature.

# Drawbacks

Increased Complexity: The implementation adds complexity to the codebase at various
levels (JavaScript, iOS, Android).

Potential Performance Impact: Additional calculations for selection position might
affect performance, particularly in complex text input scenarios.

# Alternatives

Cursor Position Tracking: Initially considered tracking only the cursor
position (cursorPosition), but this was deemed less universally beneficial compared
to selection position tracking.

External Libraries: Relying on third-party libraries for this functionality,
though this might not offer the same level of integration and performance efficiency.

# Adoption strategy

The feature is designed to be backward compatible and non-breaking.
Existing applications can adopt the new feature without any required changes to existing code.
Documentation and examples will be provided for smooth adoption.

# How we teach this

Documentation Updates: Revise the documentation to include this new feature,
supplemented with code examples.

Tutorial Creation: Develop tutorials showing practical implementations of the feature.

Community Engagement: Promote the feature within the community for feedback
and wider adoption.

# Unresolved questions

The exact performance impact on different devices and under various usage conditions.
Ensuring seamless integration with popular third-party libraries that extend TextInput.