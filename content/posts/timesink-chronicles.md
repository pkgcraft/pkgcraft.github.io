---
title: "The Timesink Chronicles"
subtitle: "Or How I Learned to Stop Procastinating and Waste More Time"
date: 2022-01-18T16:01:26-07:00
tags: ["Gentoo"]
---

Before beginning these grand timesink chronicles, allow me to review my
historical use of Gentoo that helps explain this project's existence. Note that
this is pertains only to my experience and opinions.

##### 2005 -- Getting sucked in

I first installed Gentoo around 2005, near the probable peak of outward
interest in the project. At this point no EAPIs existed, portage was the only
choice, and the project felt exciting probably due to the level of interest at
the time. From an outsider's perspective, portage was doing an adequate job
since core counts were low for consumer machines. Therefore, parallelization
wasn't overly important and improving the package tree was generally a higher
priority.

In retrospect, the project should have capitalized more off the interest wave
to explore other ideas before, in effect, chaining itself to bash for life.
While I understand the upsides for selecting bash as a base, the downsides are
quite large from a developer's perspective as bash is highly focused on two
things, running scripts and interactive shell usage. Its underlying structure
leaves a lot to be desired when trying to force it outside those bounds.

##### 2010 -- New package managers on the block

Fast forward about 5 years to 2010 when EAPI 3 came out and two new package
managers (pkgcore and paludis) joined portage, evolving as part of the
specification process and proving its existence in aiding new development.
Having tried both, I was impressed by the speed of pkgcore in relation to my
experience with portage, both being mainly written in python. The main reasons
for this runtime difference come from pkgcore's more streamlined restriction
framework, overall cleaner design, and ebuild daemon functionality that
avoided re-execing bash as much as possible. Sadly enough, as more features
made it into new EAPIs, pkgcore slowly fell behind mostly due to getting
bus-factored into near stasis.

As could be discovered when trying out the alternative package managers,
portage was starting to lag behind in a number of areas that it still suffers
from including overall design and maintainability. While a number of features
and performance improvements did find their way from pkgcore to portage around
this time, in my opinion, Gentoo as a project should have thrown more support
behind moving to or subsuming pkgcore because it would quickly become a
nontrivial task in later years.

##### 2015 -- Reviving a dying project

2015 was about the time I started getting involved in pkgcore-related
development. For my part, most of my work on pkgcore and its related tools was
due to curiosity, interest, and a hesitancy to let the project entirely die.
Over the next few years I would drag pkgcore along, keeping it barely alive,
culminating in rewriting pkgcheck (the pkgcore-based ebuild linter) nearly from
scratch in an effort to parallelize it as much as its python-based nature
allowed.

During this time, it became apparent to me that Gentoo as a development
community often felt directionless and highly change averse. Democratizing
leadership while keeping the foundation separate lead to weak, overarching
vision and therefore a rudderless appearance for the project. Personally I
think the council should actively define priorities for the project and even
use funding where appropriate to aid in that effort rather than its mainly
reactionary and ratification style meetings.

##### 2020 onwards -- Accepting fate

By 2020, it had become clear to me that pkgcore and its related tools were
evolutionary dead ends. I felt enough work had been done to prove their worth
and underlying design was better than portage in a number of ways, but that
wasn't able to grow interest to a level where moving on from portage was
feasible. In order to have had a better future, more focus should have been
placed on merging pkgcore's design with portage during the 2010 era when it was
potentially feasible to do, by 2020 it was all but impossible.

With that in mind, I passed on maintenance to those with a vested interest in
the project mainly due to Gentoo beginning to seriously use pkgcheck for CI
against the main tree after the parallelization work was merged. I imagine the
project will continue on life-support style maintenance as long as its
alternative focuses on being an interactive commit tool, performing shockingly
terrible at linting runs on any significant scale.

Regarding my decision to drop pkgcore, in essence I never agreed with some of
the underlying design and didn't want to nearly rewrite it as I had been forced
to for pkgcheck. For example, continuing in portage's footsteps using an
interpreted language like python for the core package manager felt like a poor
long-term choice. At the time the fork occurred, it probably made sense but
doesn't anymore. In any case, if I was going to start over it would be
drastically different than pkgcore from the outset.

##### Enter... pkgcraft

Having kicked around the idea of rewriting pkgcore in rust since early 2017, it
slowly became a reality as the ecosystem grew enough where third party
libraries existed to support the intended design. Whether it will surpass
pkgcore's efforts or fade away as an ephemeral dream remains to be seen, but
hopefully these chronicles entertain, inform, or inspire others to support this
timesink and strive towards their own.
