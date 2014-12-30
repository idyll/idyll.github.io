---
layout: post
title:  "Keymapping on Yosemite for your Keyboard with Karabiner"
date:   2014-12-29 18:00:00
categories: osx yosemite devtools
---
If you were a good developer this year Santa probably brought you a keyboard for Christmas. Alas, it probably doesn't have the magical Apple key that you use every day. Good news. I have your solution

### Logitech K811 ###

In my case, I like a "chicklette" style keyboard with the small keys and a short action. Santa got me the [Logitech K811](http://support.logitech.com/product/9938). It's pretty good so far -- supports 3 devices and without a numberpad it allows your mouse to be super close.

But as you can see, since it is intended for mobile and desktop use, it is missing some function keys.
In my case, I really like to have a "show desktop" shortcut. But I didn't want to switch over to using "F" keys.

In the [Logitech forums](http://forums.logitech.com/t5/Keyboards-and-Keyboard-Mice/K811-missing-shortcuts/td-p/1018819),
[Karabiner](https://pqrs.org/osx/karabiner/) was suggested!

__NOTE:__ _there's no reason why we can't use this with another keyboard. For example, a [Microsoft Sculpt Ergonomic Keyboard](http://www.microsoft.com/hardware/en-us/b/sculpt-ergonomic-keyboard-for-business/5KV-00001), is going to be my next attempt._

### Using Karabiner ###

You will need to download and install [Karabiner](https://pqrs.org/osx/karabiner/).

Next you need log out the keys you intent to use with [DEBUG MODE](https://pqrs.org/osx/karabiner/document.html.en#debugmode). In my case it took a while to figure it out, but the logitech keyboard uses a combination of a key and a modifier key to activate mission control. 

To determine this I first noticed that the single keypress actually activated 2 keys. The only way that would work is if at least one of them was a modifier key. So then I mashed on the modifier keys until I figured out which one it was.

I then set out to modify ```private.xml``` [(get the details here)](https://pqrs.org/osx/karabiner/document.html.en#privatexml).

### Private.xml ###

Private.xml is the custom keymapping file. I started with the two entries suggested in the [Logitech forums](http://forums.logitech.com/t5/Keyboards-and-Keyboard-Mice/K811-missing-shortcuts/td-p/1018819). I then added two more (good old guess and test style). I mapped ```<command>+<key>``` to each of the shortcuts I wanted by simply mapping them to the default keys used by OSX to activate the shortcut. I did this to avoid conflicts with the behaviour of the built in keyboard, although Karabiner does support different profiles and ways of keyboard specific mappings.

After you decide what to map, next is how to map it.

Karabiner has a couple of included files that translate key codes into names. If possible use this because then your XML file makes sense when you read it. If not you can use the ```KeyCode::RawValue``` option.

For a full list of what you can do and map, there's a [man page just for the private.xml file](https://pqrs.org/osx/karabiner/xml.html.en).

In the end, here's my private.xml:

{% highlight xml %}
<?xml version="1.0"?>
<root>
  <item>
    <name>Logitech K811 Brightness Down to Prev</name>
    <appendix>Change Keyboard Brightness Down key to Previous Track function</appendix>
    <identifier>private.brightnessDown_to_prevTrack</identifier>
    <autogen>--KeyToConsumer-- KeyCode::F14, ConsumerKeyCode::MUSIC_PREV</autogen>
  </item>
  <item>
    <name>Logitech K811 Brightness Up to Next</name>
    <appendix>Change Keyboard Brightness Up key to Next Track function</appendix>
    <identifier>private.brightnessUp_to_nextTrack</identifier>
    <autogen>--KeyToConsumer-- KeyCode::F15, ConsumerKeyCode::MUSIC_NEXT</autogen>
  </item>
  <item>
    <name>Logitech K811 Show Desktop</name>
    <appendix>Add show desktop to command F4/Expose</appendix>
    <identifier>private.comand_mc_to_show_desktop</identifier>
    <autogen>--KeyToConsumer-- KeyCode::RawValue::0x00a0, ModifierFlag::CONTROL_L | ModifierFlag::COMMAND_L, KeyCode::F11, ModifierFlag::NONE</autogen>
  </item>
  <item>
    <name>Logitech K811 Mission Control</name>
    <appendix>Add mission control to command F5/Dashboard</appendix>
    <identifier>private.command_dash_to_mission_control</identifier>
    <autogen>--KeyToConsumer-- KeyCode::RawValue::0x0083, ModifierFlag::COMMAND_L, KeyCode::CURSOR_UP, ModifierFlag::CONTROL_L</autogen>
  </item>
</root>
{% endhighlight %}

And that is that.

Hope this makes you as happy as it made me today. :)
