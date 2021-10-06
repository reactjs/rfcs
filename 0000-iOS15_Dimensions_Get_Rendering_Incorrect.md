- Start Date: 2021-10-05
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Some customers who have upgraded to iOS 15 (Released 2021-09-20) are
experiencing rendering issues for components relying on window dimension
information from `useWindowDimensions` or `Dimensions.get('window')`.

For users on iOS 15 after a fresh app launch, `useWindowDimensions` and
`Dimensions.get('window')` would return either 0 or a negative value.
Depending on how these APIs were used in a component, the regression would
last until the next app active / other Dimensions set event in the best case,
and in the worst case would not be able to be resolved.

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?


Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.


On iOS, RCTDeviceInfo.mm signals the RN-side to update its set Dimensions
dictionary if certain iOS lifecycle events happen, such as application did
become active and accessiblity events.

I suggest adding another listener for `RCTContentDidAppearNotification` in
RCTDeviceInfo.mm. Below is the diff:

```
--- node_modules/react-native/React/CoreModules/RCTDeviceInfo.mm
+++ node_modules/react-native/React/CoreModules/RCTDeviceInfo.mm
@@ -14,6 +14,7 @@
 #import <React/RCTEventDispatcher.h>
 #import <React/RCTUIUtils.h>
 #import <React/RCTUtils.h>
+#import <React/RCTRootView.h>
 
 #import "CoreModulesPlugins.h"
 
@@ -71,6 +72,11 @@
                                                name:RCTUserInterfaceStyleDidChangeNotification
                                              object:nil];
 
+  [[NSNotificationCenter defaultCenter] addObserver:self
+                                           selector:@selector(interfaceFrameDidChange)
+                                               name:RCTContentDidAppearNotification
+                                             object:nil];
+
 #endif
 }
```

The `RCTContentDidAppearNotification` event is already emitted by React's iOS library when
the React Native bundle has been initialized and the root app component view can be added
and displayed in `RCTRootView.m`'s view. Emitting this event insures the RN side is ready
to receive dimension update events and that components can reliably get a value they
expect for the window size.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
    - Small

- whether the proposed feature can be implemented in user space
    - Although there are some workarounds (such as using `Dimensions.get('screen')`) exist,
      given the scope of how many customers have been impacted I feel a core library fix
      is needed to insure stability throughout the RN community.

- the impact on teaching people React
    - n/a

- integration of this feature with other existing and planned features
    - unknown

- cost of migrating existing React applications (is it a breaking change?)
    - this is not a breaking change

Other things I have considered:
    - Is there a more reliable event to listen for.
    - Is this stepping on other RN setup processes in between the RCTRootView
      mounting to the iOS app's controller view and whent he RN bundle has
      finished processing and mounts to the RCTRootView for display.
    - Why is this only happen only on iOS 15? I have a gut feeling (haven't root
      caused) that with iOS 15 being the first iOS to support ProMotion 120Hz
      displays the rendering issue is more refined and efficient which could
      be creating a race condition with the React mounting process.

# Alternatives

What other designs have been considered? What is the impact of not doing this?


We have considered making a patch in the RN side of the library however doing so
for an issue that was iOS only didn't make sense and would result in fragmented
and difficult to maintain code long term for iOS and other platforms.

Another approach would be to rely on `Dimensions.get('screen')` which on iOS always
returned the expected screen size values. However, advising that API is something
I would not recommend for an RFC as it is more of a cheap workaround of the problem.

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?


This proposal would be implemented by applying the above diff to RCTDeviceInfo.mm.
This is not a breaking change and can go out to developers via a standard React
library release.

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?


This proposal impacts only iOS and does not impact any existing documentation.

As far as I can tell there is no official React Native documentation / teaching
resources describing when, why, and how a window's dimensions change for React
Native's supported platforms. Following this RFC it might be a good idea to
add some documentation regarding the `Dimensions.set` lifecycle.
