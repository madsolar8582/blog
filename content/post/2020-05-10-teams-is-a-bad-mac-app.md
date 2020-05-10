---
title: "Teams is a Bad Mac App"
subtitle: ""
date: 2020-05-10T07:00:00-05:00
tags: ["microsoft", "teams"]
---

Before we dive too deep into this, I need to preface this by saying that Microsoft has done a good job shifting their priorities and getting features out to help everyone work from home during the pandemic.

So if you are like me and your company uses Microsoft 365 (née Office 365), you are probably using [Microsoft Teams](https://www.microsoft.com/en-us/microsoft-365/microsoft-teams/group-chat-software) for chat and collaboration. If you are using the Windows version or the mobile version (iOS/Android), it works really well. However, if you are using the macOS version you are probably running into some very frustrating design decisions and limitations.

### Notifications

My major complaint is that Teams uses a custom notifications system instead of using the native notification API (i.e. integrating with Notification Center). This means that if you have Do Not Disturb on, it ignores it and you cannot turn off Teams notifications in System Preferences. The notifications are presented using a hidden window. Now, this leads to additional issues as the window can technically be selected and set as the active window, but it doesn't do anything regarding user input. This is especially annoying when you have Spaces enabled since the window installs itself on each virtual desktop and takes over on every switch. This also means when you switch back to Teams, macOS thinks Teams is the active application (it is), but it is the notification window instead of the application window which makes pasting or responding to things require an additional click.

Another gripe is that you can turn off notifications on chats occurring in a channel, but not private chats. Teams allows you to "mute" the conversation, but you still receive email notifications of missed messages which defeats the purpose of ignoring a conversation. This also means that you can't mute meeting conversations entirely without leaving the chat.

### Chat History

Since Teams is built using [Electron](https://www.electronjs.org/) instead of native code on macOS, you run into limitations due to the web-based architecture. One of these limitations is chat history. If you have a long thread in a channel or private conversation, only a limited number of messages is kept in memory. This means that when you are scrolling through chat/thread history, only the recently viewed messages are kept loaded and when you scroll past that, even if you viewed some of those messages a few seconds ago, they have to be reloaded again. This is a major usability problem and performance problem since not enough messages are kept in the cache. Since iOS does not allow Electron apps, it is written entirely natively. As such, it doesn't actually suffer from this problem since it uses a UITableView and the cells' data source has all of the messages and allows for much more performant scrolling while keeping many messages available to the user without reloading. This goes without saying but if your mobile app does something better than your desktop app, that is unacceptable.

Also, you cannot delete personal conversations which leads to you needing to use the search function frequently to get find people you want to chat with rather than having to scroll through an almost endless list of chats. Now, you can pin contacts as a frequently contacted so that helps a bit, but I would still like the ability to delete chats that I do not care about.

### Chat UI

Teams is built using a singular window and does not allow you to have chats or meetings in their own window. This again leads to some usability issues as every context change requires the app to re-render and depending on what you are navigating to, takes times. When in a meeting with screen sharing or active video, Teams will use a small floating window to show the current content while you interact with other channels or chats, but as soon as you re-enter the meeting/call, it has to re-render all of the participants. This can easily be solved by having the meeting or call use its own window like its predecessor Skype for Business did. By having separate windows, you can also pin chats or meetings to different virtual desktops or use the native Picture-in-Picture APIs of macOS to allow you to continue to work in another application while keeping the meeting content visible.

### Video Performance and Quality

Now I am aware that due to the increase in usage due to everyone working from home, Microsoft did reduce the quality of people's video during calls, but this feedback is coming from before that change.

When meetings or calls have video content, Teams zooms in to try to focus on the participant. This leads to a very weird usage of space as you need to right click to zoom out to get a 16:9 view of the video. With that change active, you have room for more participants, but Teams is limited to 4 people and then rotates out based on speaking history (unless you pin certain participants). Allegedly, Microsoft will be issuing an update this month to change to a grid-based layout like Zoom to allow for more participants to be on screen. Hopefully this includes defaulting to a 16:9 layout as well to get more people on screen. 

The thing is though is that the quality is poor for incoming video. Again, the iOS application is not Electron based, so it uses the native [AVFoundation](https://developer.apple.com/documentation/avfoundation?language=objc) & [AVKit](https://developer.apple.com/documentation/avkit?language=objc) APIs for video and audio which leads to much better quality and performance. Teams on macOS on the other hand uses webRTC, which should be fine, but Teams is stuck using a very old version of Electron (4.2.12 - October 2019; Chromium 69) instead of the latest stable branch (8.2.5 - April 2020; Chromium 80) which has significant performance improvements (not to mention even more coming in 9.x; Chromium 83). So maybe the desktop app gets equivalent or better quality and performance when they update Electron, but I have no idea when that is coming so I sometimes use the iPad app as it us a much more enjoyable experience.

### System Audio

When screen sharing, Teams cannot capture system audio. This means that if you have a presentation with sound effects or a video, you cannot transmit that audio. Supposedly that is a feature that is coming soon™, but the latest update (as of the time of this writing) broke additional audio settings (i.e. call control sync on external certified audio devices) in preparation for this feature. I really hope this is fixed soon as it defeats the purpose of having an audio conferencing device if Teams cannot synchronize with it.

### Updates

Since teams does not use the normal Microsoft app delivery system, it does not integrate with Microsoft AutoUpdate. This makes installing/managing updates a bit more cumbersome but that also means that you cannot have Teams be a part of the distribution channel you are using (Stable, Insider Slow, or Insider Fast). Hopefully they switch to using that distribution mechanism to align with the rest of their software (not to mention that is how the predecessor updated as well).

---

With all of that being said, all of these problems are fixable as it just takes time. The interesting thing to note here is that if Microsoft decided to port their iPad app over using Catalyst, a lot of problems would be solved very quickly. The reason I think that they are not doing that is that would raise the minimum macOS version to macOS 10.15 (2019) instead of 10.11 (2015). It just boggles my mind that a good Electron app ([VS Code](https://code.visualstudio.com/)) and such a bad one can come from the same company.


