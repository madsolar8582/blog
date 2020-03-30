---
title: "It's Finally Fixed"
subtitle: ""
date: 2020-03-30T16:00:00-05:00
tags: ["apple", "bugs"]
---

After 4 months, the bugs with UI automation tests involving web content are fixed. Mostly.

The primary issues with the accessibility service being unable to interact with web content has been fixed as well as the interaction delays due to autofocus and the beta only issue of 60 second idle timeouts on all actions due to com.apple.Spotlight. 

All that is left now is the somewhat unreliable text input into text fields. From what I have seen, this impacts iPhones more than iPads and only happens sometimes. Since the issue is flaky, tests need to be updated to handles this. The workaround that I implemented that is successful is to input the text, then delete all of the text and input it again. 

---

Again, a 4 month turnaround time for a critical defect in developer tools is unacceptable. However, I'm just happy Xcode 11.4 was able to ship with the whole pandemic thing going on right now.
