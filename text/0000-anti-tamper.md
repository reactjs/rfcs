- Start Date: 2020-06-29
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add anti-tamper mechanism to React to prevent HTML tampering if desired.

# Basic example

Developers can add flags such as `--anti-tamper` to the toolchain process to indicate that they want to activate this feature in the build.

# Motivation

HTML editing is being abused by scammers to deceive gullible victims and extort their money. This usually happens in a fake tech support setting where the scammer first gains remote access to the victim's computer and pull up the victim's online banking website. Then, the scammer tries to distract the victim while at the same time open up the developer console and manually edit the HTML of the banking website to convince the victim that money is added to their account. Finally, the scammer proceeds to claim they added more money than s/he should and demands the victim to return the additional amount by purchasing the scammer gift cards, etc. The scam concludes if the victim falls for it.

Source: 
https://www.youtube.com/watch?v=03fvkJ27eOs (An Internet vigilante records and exposes how such scam works from start to finish.)

This is by no means a fringe problem. It is proven that scammers use similar techniques to conduct scam systematically in a career fashion, and millions of dollars per year are scammed out of gullible victims such as the computer illiterate and elderly population.

Source:
https://www.youtube.com/watch?v=7rmvhwwiQAY (BBC documentary on the fake tech support scamming industry)

Adding a kind of anti-tamper mechanism would significantly reduce the efficacy of such abuse if not totally eliminate it, which is our primary motivation. Clearly, the benefit is not only limited to tech support scams where manual tampering is the issue, but the feature will apparently also benefit the security of any React client-side application in general via the extra layer of integrity check and protection.

While many banking websites are not currently implemented in React, addition of this feature will immediately benefit existing security-sensitive websites in React and set a positive example to the community. The fact that React is a leading framework in the web development world will only inspire more frameworks developers to follow suit.


# Detailed design

To implement this feature is not hard on the conceptual level. We will need to:

1) Extend the current set of event listeners to be comprehensive such that all possible DOM tampering would be captured by React.

1) Catagorize user input and DOM change events as "normal" and "abnormal".

-  Normal changes are defined to be the canonical set of interactions expected by the web standards and principles in UI design, such as typing in a textbox, pressing a button, dragging/clicking a div and etc.

- Abnormal changes are any other changes where no semantic interpretation exists in terms of web interaction, like disappearance of elements, modified content of read-only elements etc.

3. Implement a module in React which is able to determine which event belongs to which category in a blacklist or whitelist manner, whichever is easier.

4. In case a normal change is detected, trigger the routine React logic.

5. In case an abnormal change is detected, localize the DOM tree where the change occurs. Then dispatch the React reconciliation algorithm to re-render the specific tree, effectively reverting any potential abnormal change.

Now technically many "abnormal changes" are the results of routine DOM reconciliation produced by client-side logic and are in fact not malicious. Can see, in such cases, this design will produce a one-time wasteful rendering. Heuristics can be developed in the future to minimize this cost.

In summary, the algorithm would modify the React rendering system to not only operate in a reactive cycle, but to compute a stable fixpoint at all times during the entire webpage session to ensure DOM-model synchrony.

If implemented, this can be added as a build option flag `ex. --anti-tamper` to enable the feature in build. By default this flag is not enabled. In case it is disabled, bypass all the above setup and logic.

# Drawbacks

- Performance overhead if the feature is enabled. However, we believe the cost is worthy of the security concern presented. It is also not a mandatory feature, so existing users do not have to worry about this change at all provided it is implemented properly and decoupled from the main rendering routine.

- Potential corner cases in extreme circumstances that go into conflict with the design. In that case, further design is required such as the ability to mark a DOM tree to escape the anti-tampering routine (ex. `<Component anti-tamper="off" />`).

# Alternatives

We believe there is no good way of achieving the stated goal without changes at the framework level, which would be the cleanest and most community-friendly way to integrate this feature.

Just for the record, a crude way is to force update the entire DOM at a fixed time interval. This is easy and could be a quick fix to the security issue at hand. However, it is such an anti-pattern that no serious develepment team would consider in the long run.

# Adoption strategy

Implementation of this feature would only require a non-mandatory build flag to be set at the building stage to activate the feature in the build.  It is a low-level change completely transparent otherwise and should not pose any additional effort to learn and use React.

# How we teach this

This is a low-level add-on feature that should not require additional teaching effort to the community in general. A special documentation page is nevertheless required to show the existence of this feature and how to use it at the API level.

Moreover, since a change of rendering system is required, this change needs to be communicated to advanced React developers through news channels and advanced guides.

# Unresolved questions

None so far. But we expect some unresolved problems in terms of implementation details regarding interaction with specific React internals.