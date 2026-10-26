Link to the issue:
https://github.com/webcompat/web-bugs/issues/238716

Symptoms:
The freeze occurs in Firefox's Gecko/WebRender rendering code, not directly inside Moxfield JavaScript.
The main thread is occupied with hashtable iteration during a painting operation.
The absence of keyboard responsiveness is consistent with this main-thread blockage.
Disabling hardware acceleration does not eliminate this native rendering path.
The profile does not establish whether Moxfield's page content, an extension, or an internal Firefox defect originally triggered the condition.

Overview of the bug cause:
GeckoMain
 └─ PVsync::Msg_Notify
    └─ RefreshDriver tick
       └─ Update the rendering Paint
          └─ nsLayoutUtils::PaintFrame
             └─ nsDisplayList::PaintRoot
                └─ WebRender display list
                   └─ libxul.so [unsymbolicated]


A re-symbolicated profile displays evidences of a bottleneck:
nsRefreshDriver::PaintIfNeeded()
  └─ nsLayoutUtils::PaintFrame()
     └─ nsDisplayList::PaintRoot()
        └─ WebRenderCommandBuilder::BuildWebRenderCommands()
           └─ RemoveUnusedAndResetWebRenderUserData()
              └─ nsTBaseHashSet::RemoveIf()
                 └─ nsTHashtableKeyIterator::operator++()
                    └─ PLDHashTable::Iterator::Next()
                       └─ PLDHashTable::Iterator::MoveToNextLiveEntry()


Download of a fix and verification steps for the build:
https://chatgpt.com/share/6adf294e-3a50-83eb-8112-2c182eebb99e
