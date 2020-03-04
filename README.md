# VPP trailing branches - a proposal for consuming VPP in a CI/CD environment.


## Introduction

The classic way to consume a software project like VPP would be a custom branch.
The typical drivers behind this are:

1) Absence of any new bugs
2) Absence of any changes to the APIs

The first factor is usually very important in the product-style scenarios,
where the software is shipped in a singular "box" that performs some 
forwarding functions, thus there is very few chances to update it in the field
afterwards.

In fact, this "slow update cycle / one box" combination is the underlying driver:
given only a few touchpoints at when to update the system, the desire is to make
the code as static as possible so as to increase the real or perceived stability.

The price to pay for this is the major version upgrades, which inevitably have
to happen as the old branch becomes divergent enough to make any updates impossible,
and also inability to easily get the new features.

As the software is increasingly moving to the cloud and ceases to become "boxes" with
"no possibility to update", but becomes a fleet of units with potential for 
a 1/10/100/... frequent upgrade strategy, a change in the approach may allow for
smaller overhead and better overall velocity.


## Assumptions/Prerequisite

The main assumption made is that the CI/CD/Observability tooling in the deployment
allows to make a fully automated grading of a given VPP build within the staggered (1/10/100/...)
deployment fashion.

Another assumption: the granularity of 1 day is long enough period to spot an issue in the testing/limited
deployments. Realistically, this granularity unit might need to be bumped up to 1 week or more, but we
keep it 1 day here for the simplicity of reasoning.

## The building block - "Trailing-T" branch, "Trailing-T+fixes" branch.

The main building block of this proposal is the "trailing-T branch": a branch that mechanically cherry-picks
all of the commits from the master branch, with a certain delay "T". 

The secondary building block is "Trailing-T + fixes" branch, which constitutes the Trailing-T 
branch with certain fixes that would otherwise be "in-flight", rebased on top of it.

This building block allows the creation of detect-fix-apply-detect loops for a given criteria.

The "T" in a given scenario would be maximum duration for the 95%-ile of issues/fixes, and can be tuned
dynamically by this process: If we imagine the set of all "Trailing-T" branches for all 
the possible values of "T", then if the timeline for the fix is too long, we can freeze 
the "Trailing-T + fixes" branch, which effectively means moving it every day onto a different
Trailing-T branch with ever increasing value of T.

After the fix is done, and provided the testing at the start reference point goes well, one can
perform the reverse motion - e.g. adding 2 days worth of commits every day rather than 1, thus
effectively moving to a Trailing-T branch with ever decreasing value of T.
Thus the point of the Trailing-T+fixes branch will eventually catch up with the desired target.
As the fixes are being cherry-picked from the master branch, eventually all the fixes that
were "early-cherry-picked" will be absorbed from the master branch - thus bringing the Trailing-T+fixes
in complete sync with Trailing-T, thus just becoming a master branch delayed by T days.

This gives a self-regulating property in that the operation will just reflect the state of
the matters, with the values of T being more the approximation and the indicator, rather
than hard deadlines, and avoids dealing with long-term divergencies.

Caveat: notice that the history of the "Trailing-T + fixes" branch is being continuously rewritten if there
are fixes that are cherry-picked on top of "Trailing-T" - they get rebased with each update of Trailing-T,
thus changing the commit-IDs. This can make it seemingly difficult to get "back to a known state".
However, this problem can be solved fairly trivially by having an append-only version controlled file 
within the "Trailing-T" with the commit-IDs that are being applied out-of-order. The "Git blame" history
on that file wlil tell exactly which fixes were cherry-picked when, as well as to allow the reconstruction
of a state of "Trailing-T + fixes" branch at any point in time in the past, by mechanically cherry-picking
the commits already mentioned in that file, that are newer than that particular point in time.


## Key event points

As mentioned above, there are two key considerations when upgrading the VPP code by a downstream:
API changes and general "new feature/new bug" issues.

Therefore, we need to define a few key "event points" - where a testing/qualification 
will take place, and where additional actions will be triggered, if necessary.


### Event point A: latest master

At this point the user will want to detect any relevant API changes that are made,
and optionally run some standalone integration tests that can be easily in the non-production environment.
The bugs found here will trigger the bug report, with the fairly straightforward
troubleshooting phase - if something "worked" yesterday and "does not work" today,
the scope of changes will be very small, thus aiding rapid triage and troubleshooting.

The fixes to the found bugs will need to be committed to the master and then subsequently cherry-picked
to the Trailing(A-B)+fixes branch.

The API changes found at this stage will cause the slide of point "A" from the tip of the master
branch to the Trailing-T branches, with T gradually incrementing to 1, 2, etc. days, until the 
control plane side verification passes, at which point this event point will need to begin catching
up to master HEAD.

Post-mortem activities: If it is possible to submit the results upstream to the VPP in the form of the "make test"
testcases, doing so will ensure that the future instances of that type of misbehavior will be avoided
even earlier.

### Event point B: limited scope alpha testing

This is the "1" moment in the "1/10/1000/..." deployment strategy - when the build has passed the automated unit testing
and is being deployed to the seed location - internal office, free subscribers, etc.

IF the impacting failure(s) is/are found at this point, this event point gets frozen in time, until the troubleshooting/fix coding/fix verification is complete. 

Post-mortem activities: if there is a way to make this failure part of an automated test at event point A, or part
of "make test", kick off that activity.

### Event point C: beta testing

This is the "10/100" point in deployment - when the new software gets wider testing, yet not deployed generally.

The process invoked at this point is the same as at point B - if issues are found, it gets frozen until the issues get fixed.
The fixes get tested here, as well as propagated via cherry-pick via a second run of the event points A and B.

Post-mortem activities: evaluate why the issues found here were missed at point B, and perform the necessary adjustments.
If the issues can be pushed into the automated testing at point A, or be made into a "make test" testcase, kick off the
respective activities.


### Event point D: Deployment


This is the "..." point in deployment, where the given VPP code is considered to be "trusted" enough to be deployed at scale.

Any large-scale impacting issues found at this stage will cause the freeze of this event point; 

They will as well require AUTOMATED collecting of the
necessary troubleshooting data (e.g. post-mortem API dump, full core dump as well as the ensured continuity of 
access to the built artifacts and the exact source tree and toolchain used to build them; etc. = this list is not exhaustive and
to be extended)

After the troubleshooting information has been collected, the running version is rolled back to a known good one on all of the fleet.

Post-mortem activities: analysis of why this issue was not detected at earlier points, and kick off of the activities to ensure
the future coverage of it is ensured there, rather than in full production.

### Event point E: End of use

This event point needs to be set out sufficiently far in the past so as to ensure no false negatives. At this event point the "old" version(s) of VPP that are deemed end of life are discarded. 



## API compatibility discussion and potential solutions

API compatibility is always at odds with the necessity to implement the new features - new functionality necessitates
new or changed semantics in the API.

Realistically, any control-plane managing many instances of VPP at scale will need to foresee an internal abstraction layer,
that would translate the higher-level semantics of the project into the lower-level API that VPP provides.
Having that component would allow rapid access to the new features that the VPP offers, without compromising the backwards compatibility.

Likewise, it is easily noticeable that between N different consumers there will be some subset of "basic" functionality requirements
that will not change for quite a long time, and will not benefit from new features. How big this subset would be is a bit tricky to
say - because what different projects consider to be "legacy" functionality will certainly differ.


One approach to this on the VPP side is to "freeze" the API via exposing a separate endpoint that is guaranteeing a given API,
and this guarantee would need to be part of the VPP development/testing cycle same as other new features.

Having the option for such an "insulation layer" would allow to minimize the repeated (between different consumer projects) work
on the control plane project side in its own API abstraction layer.

At the same time, having this insulation layer on VPP side is more of an optimization/convenience factor - because on its own it
does not solve the problem of access to the new features, it merely freezes the existing ones.


### API compatibility selection criteria


The client (control plane) side should use the "show_version" API so as to ascertain which exactly version the VPP is, and to select the corresponding translation layer for it.

If there is a VPP-side adaptation/translation layer, then the endpoints for it will be communicated in advance. Thus, the client-side
translation layers will be able to multiplex the "old" with "master" endpoints as necessary, using the "old" endpoint as a stable anchor,
and the "master" for the bleeding-edge API access.

At release time, the VPP will need to forge a second "stable" API adapter/layer, which then the client software can use as a base for
their API-related functionality migration independently of the VPP release cycle. Having these two "stable-to-be-migrated-from" and "stable-to-be-migrated-to" layers in VPP will allow other projects which are not interested in the latest feature set, but would
not mind to be on master, an easy path to do so.



