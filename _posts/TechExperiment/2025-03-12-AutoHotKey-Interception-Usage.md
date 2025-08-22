---
title: AutoHotKey-Interception
date: 2025-03-12
categories: [device]
tags: [device]     # TAG names should always be lowercase
---
Problem: I got a keyboard which is 68 key without the key remapping of `Home` and `End`.
I want to use AutoHotKey to remap some cmob-key to this.
=====================

1. AutoHotKey seems is a good option, But seems it cannot specify one specific device (Keyboard or Mouse).
2. I checked the available tools, and a package of AHK, named [AHI](https://github.com/evilC/AutoHotInterception?tab=readme-ov-file), seems a good option.
3. I download and unzip the `z2` version of it under a work directory.
4. following the `README`, I installed other setups (Interception Driver, AHK (version AutoHotkey_2.0.19_setup.exe)).
5. And modified a script.
```ahk
#SingleInstance force
Persistent 
; (Interception hotkeys do not stop AHK from exiting, so use this)
#include Lib\AutoHotInterception.ahk

; Part 0
AHI := AutoHotInterception()
magegeeID := AHI.GetKeyboardId(0x1A2C, 0x95F6, 1)
cm1 := AHI.CreateContextManager(magegeeID)
return


;Magegee keyboard 68 keys without home and end
;PID,VID
;0x1A2C, 0x95F6
;HID
;HID\VID_1A2C&PID_95F6&REV_0110&MI_00


#HotIf cm1.IsActive
; Part one
::aaa::JACKPOT
1::
{
	ToolTip("KEY DOWN EVENT @ " A_TickCount)
	return
}	
1 up::
{
	ToolTip("KEY UP EVENT @ " A_TickCount)
	return
}
#HotIf cm1.IsActive

; Part Two
; Remap Ctrl+Ins to Home
^Insert::HOME
;{
;    AHI.SendKey(magegeeID, "Home") ; Send Home key
;    return
;}

; Remap Ins to End
Insert::END
;{
;    AHI.SendKey(magegeeID, "End") ; Send End key
;    return
;}
#HotIf


; Part 3
^Esc::
{
	ExitApp
}
``` 

Focus on these 4 parts.
-----------
a. Part 0 is using the `monitor.ahk` to get the PID and VID of the wanted device (magegee keyboard)
b. Part 1, is used to demonstrate how it works.
   - `#HotIf` is a directive in AutoHotkey that allows us to define context-sensitive hotkeys. It means that the hotkeys below the `#HotIf` line will only work if the specified condition is true. In the script, the condition is `cm1.IsActive`, which checks if the active keyboard is the one managed by the cm1 context manager (Magegee keyboard).
    - `::aaa::JACKPOT` This is an AutoHotkey text replacement (auto-replace) rule. It means: Whenever typing `aaa`, it will automatically replace it with `JACKPOT`. It’s useful for creating shortcuts for frequently typed phrases or words. For example, you could set ::btw::by the way to save time while typing.
    - `1::` and `1 up::` These are hotkey definitions for the `1` key. `1::` triggers when the 1 key is pressed down. `1 up::` triggers when the 1 key is released.

c. Part 2, this is a direct hotkey remapping of `Ctrl+Ins -> Home` and `Ins -> End`
d. Part 3, if ctrl + Esc is typed, then the script will stop.

If Part 3 is removed, then stop of the hotkey should be stop the running of the AutoHotKey Exe in the computer.