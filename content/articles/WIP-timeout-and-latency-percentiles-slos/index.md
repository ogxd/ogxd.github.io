Worklog

IMHO this Jira is too focused on a solution and is missing a whole context. At this point it’s a shot in the dark.

What are we trying to solve?

SLIs can vary a lot from pod to pod for a given service on a given cluster. Let’s call least performing pods “outliers“.

A few outliers may impact the service SLIs at the cluster level (eg 95p)
 SILA / QoS degradation

Outliers may produce a lot of noise an interrupt engineers in they day to day work
 Productivity loss / fatigue

A service capacity is scaled based on its worse performing instances (downward leveling)
 Wasted capacity

How can we solve this?

To solve this issue, we need to understand what an “outlier” is. The issue is that this definition is unclear: an outlier may be a pod or a node, it may show individually or as a set, it may due to software bugs, hardware differences or configuration discrepancies, and it may show up different symptoms depending on the rootcause. The process of identifying and solving such issues inherently requires an engineering effort.

Naoh is a tool that aims a automatically killing outliers pods based on a set of metrics. In some cases, it's a solution that may allievate engineers from having to investigate, but it’s not a silver bullet. Even worse, it comes we several drawbacks:
- It requires a lot of tuning to be effective
- It may kill pods that are not outliers, causing more harm than good (such as new versions)
- It considers pods as the unit of work, but the issue may be at the node level (it actually is in most cases)
- It hides the issue, making it harder to understand the rootcause

We think "outliers" is not an illness that comes and goes, but a symptom of an actual issue that we a bound to tackle, as engineers. Otherwise, we are just hiding the problem under the carpet, and these issues will stack up, making it even more difficult to solve and critical as time goes by.

We propose instead to "assist" engineers in identifying and solving these issues, by providing them with the right tools and information, along with a well defined process, which could be described as follows:
- **Identify**. What is the problem? Is it a pod, a node, a set of nodes? What do we observe? What is the impact?
- **Investigate**. What is the rootcause? Is it a software bug, a hardware issue, a configuration discrepancy?
- **Track**. Track and communicate of that issue, so we're aware of it an can plan accordingly, don't forget about it and don't waste time by investigating the same issue over and over again.
- **Plan**
- **Solve**. Fix the bug / replace the hardware / update the configuration / ...
