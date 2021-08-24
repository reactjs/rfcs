# React RFCs

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put
through a bit of a design process and produce a consensus among the React
core team.

The "RFC" (request for comments) process is intended to provide a
consistent and controlled path for new features to enter the project.

[Active RFC List](https://github.com/reactjs/rfcs/pulls)


## Contributor License Agreement (CLA)

In order to accept your pull request, we need you to submit a CLA. You only need
to do this once, so if you've done this for another Facebook open source
project, you're good to go. If you are submitting a pull request for the first
time, just let us know that you have completed the CLA and we can cross-check
with your GitHub username.

**[Complete your CLA here.](https://code.facebook.com/cla)**

## When to follow this process

You should consider using this process if you intend to make "substantial"
changes to React or its documentation. Some examples that would benefit
from an RFC are:

  - A new feature that creates new API surface area, and would
     require a feature flag if introduced.
  - The removal of features that already shipped as part of the release
     channel.
  - The introduction of new idiomatic usage or conventions, even if they
     do not include code changes to React itself.

Some changes do not require an RFC:

  - Rephrasing, reorganizing or refactoring
  - Addition or removal of warnings
  - Additions that strictly improve objective, numerical quality
  criteria (speedup, better browser support)
  - Additions only likely to be _noticed by_ other implementors-of-React,
  invisible to users-of-React.

## What to expect

It is very hard to write an RFC that would get accepted. Nevertheless, this shouldn't
discourage you from writing one.

React has a very limited API surface area, and each feature needs to work seamlessly with all other features.
Even among the team members who work on React full time every day, ramping up
and gaining enough context to write a good RFC takes more than a year.

In practice, React RFCs serve two purposes:

* **React Team RFCs** are submitted by [React Team members](https://reactjs.org/community/team.html) after extensive (sometimes,
multi-month or multi-year) design, discussion, and experimentation. In practice, they comprise
majority of the RFCs that got merged so far. The purpose of these RFCs is to preview the design
for the community and to provide an opportunity for feedback. We read every comment on the RFCs
we publish, respond to questions, and sometimes incorporate the feedback into the proposal.
Since our time is limited, we don't tend to write an RFC for a React feature unless we're very
confident that it fits the design. Although it might look like most React Team RFCs easily
get accepted, in practice it's because 98% of ideas were left on the cutting floor. The remaining
2% that we feel very confident and have team consensus on about are the ones that we announce as RFCs for community feedback.

* **Community RFCs** can be submitted by anyone. In practice, most community RFCs do not get merged.
The most common reasons we reject an RFC is that it has significant design gaps or flaws, does not work
cohesively with all the other features, or does not fall into our view of the scope of React. However,
getting merged is not the only success criteria for an RFC. Even when the API design does not match
the direction we'd like to take, we find RFC discussions very valuable for research and inspiration.
We don't always review community RFCs timely, but whenever we start work on a related area, we check
the RFCs in that area, and review the use cases and concerns that the community members have posted.
When you send an RFC, your primary goal should not be necessarily to get it merged into React as is,
but to generate a rich discussion with the community members. If your proposal later becomes accepted,
that's great, but even if it doesn't, it won't be in vain if the resulting discussion informs the next
proposal in the same problem space, whether it comes from the community or from the React Team.

We apply the same level of rigour both to React Team RFCs and Community RFCs. The primary difference
between them is in the design phase: React Team RFCs tend to be submitted at the end of the design
process whereas the Community RFCs tend to be submitted at the beginning as a way to kickstart it.

## What the process is

In short, to get a major feature added to React, one usually first gets
the RFC merged into the RFC repo as a markdown file. At that point the RFC
is 'active' and may be implemented with the goal of eventual inclusion
into React.

* Fork the RFC repo http://github.com/reactjs/rfcs
* Copy `0000-template.md` to `text/0000-my-feature.md` (where
'my-feature' is descriptive. Don't assign an RFC number yet).
* Fill in the RFC. Put care into the details: **RFCs that do not
present convincing motivation, demonstrate understanding of the
impact of the design, or are disingenuous about the drawbacks or
alternatives tend to be poorly-received**.
* Submit a pull request. As a pull request the RFC will receive design
feedback from the larger community, and the author should be prepared
to revise it in response.
* Build consensus and integrate feedback. RFCs that have broad support
are much more likely to make progress than those that don't receive any
comments.
* Eventually, the team will decide whether the RFC is a candidate
for inclusion in React. Note that a team review may take a long time,
and we suggest that you ask members of the community to review it first.
* RFCs that are candidates for inclusion in React will enter a "final comment
period" lasting 3 calendar days. The beginning of this period will be signaled with a
comment and tag on the RFCs pull request.
* An RFC can be modified based upon feedback from the team and community.
Significant modifications may trigger a new final comment period.
* An RFC may be rejected by the team after public discussion has settled
and comments have been made summarizing the rationale for rejection. A member of
the team should then close the RFCs associated pull request.
* An RFC may be accepted at the close of its final comment period. A team
member will merge the RFCs associated pull request, at which point the RFC will
become 'active'.


## The RFC lifecycle

Once an RFC becomes active, then authors may implement it and submit the
feature as a pull request to the React repo. Becoming 'active' is not a rubber
stamp, and in particular still does not mean the feature will ultimately
be merged; it does mean that the core team has agreed to it in principle
and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor whether anybody is currently working on it.

Modifications to active RFCs can be done in followup PRs. We strive
to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged RFC to actually reflect what the end result will be at
the time of the next major release; therefore we try to keep each RFC
document somewhat in sync with the language feature as planned,
tracking such changes via followup pull requests to the document.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active'
RFC, but cannot determine if someone else is already working on it,
feel free to ask (e.g. by leaving a comment on the associated issue).

## Reviewing RFCs

Currently, the React Team cannot commit to reviewing RFCs in a timely manner.
When you submit an RFC, your primary goal should be to solicit community feedback
and generate a rich discussion. The React Team reevaluates the current list of
projects and priorities every several months. Even if an RFC is well-designed,
we often can't commit to integrating it right away. However, we find it very
valuable to revisit the open RFCs every few months, and see if anything catches
our eye. Whenever we start working on a new problem space, we also make sure
to check for prior work and discussion in any related RFCs, and engage with them.

We read all RFCs within a few weeks of submission. If we think the design fits React well,
and if we're ready to evaluate it, we will try to review it sooner. If we're hesitant about
the design or if we don't have enough information to evaluate it, we will leave it open
until it receives enough community feedback. We recognize it is frustrating to not receive
a timely review, but you can be sure that none of the work you put into an RFC is in vain.

## Inspiration

React's RFC process owes its inspiration to the [Yarn RFC process], [Rust RFC process], and [Ember RFC process]

[Yarn RFC process]: https://github.com/yarnpkg/rfcs
[Rust RFC process]: https://github.com/rust-lang/rfcs
[Ember RFC process]: https://github.com/emberjs/rfcs

We've changed it in the past in response to [feedback](https://github.com/reactjs/rfcs/issues/182), and we're willing to change it again if needed. Please file an issue in this repository if you have ideas or suggestions.
