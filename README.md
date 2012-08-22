ACPIPS2Nub
==========

Want to run Leopard (Darwin 9) in VMware? Oops.. uhh.. where's the keyboard? With Darwin 8 you merely needed to compile ApplePS2Controller.kext and it would magically work. Now it doesn't. What gives!?

Well, you see ApplePS2Controller.kext as the name implies is intended to work with a legacy PS/2 keyboard/mouse controller. You know, keyboard interrupt 1, mouse interrupt 12? Now of course most hardware still includes something that works like the old i8042 but that's not really the issue. The issue is that the driver for it expects it to claim interrupts 1 and 12. But that's more a function of the i8259 interrupt controller, or rather the cascaded pair of them as found on the PC-AT and compatible systems. Of course, modern systems have no such thing but like the keyboard controller, they pretend to. Unfortunately, once you venture into ACPI land (e.g. by using the ACPI platform expert) you give this up.

This is of course a significant improvement, but how do you make the pre-existing i8042 code work? Enter the power of IOKit. The ApplePS2Controller driver only needs to have a provider that allows it to register with interrupts 1 and 12. As long as it can receive those interrupts, it's happy. But ACPI forms a tree structure and the ACPI Platform Expert sets up a hierarchy in the IORegistry to reflect this. Worse yet, the Plug'n'Play and ACPI specs together specify that the keyboard will have one PNP ID and the mouse a different one. That gets represented as two different nubs in the IORegistry.

Damn.. so how do you make a driver that expects to attach to one IOPlatformDevice nub work when you have two nubs? The answer is that you simply write a nub class which attaches to the other two nubs and thus provides a combined nub. It's a bit tricky since a typical IOKit instance only has one provider but it's not impossible. The trick is that you pick one of the nubs as your primary provider. In this case, we'll pick the keyboard nub. That's going to be provided by the ACPI PE with either PNP0303 or PNP030B as its name. Once you've started on that nub, you can then go find the mouse nub and attach yourself to it.

But then what about interrupts? Each of your providers only provides one interrupt but you need to provide two to your client (the ApplePS2Controller class). Furthermore, you need to provide them such that when it asks for interrupt 1 you give it the interrupt information from your keyboard provider and when it asks for interrupt 12 you give it the interrupt information from your mouse provider. That's not too difficult and the trick there is that at the IORegistry level, interrupt numbers are only indices into an array.

What's perhaps confusing about this to IOKit newbies is that the nub I'm discussing here used to be part of the AppleACPIPlatformExpert.kext. Yes Virginia, extensions can have multiple classes. And often times, one class will be the provider and another class will be a client to that provider. That is exactly the situation until OS X Leopard was released with an AppleACPIPlatformExpert.kext that did not contain the AppleACPIPS2Nub class.

Not a problem. All we need to do is rewrite the AppleACPIPS2Nub class and we're in business. Unfortunately, the IOKit documentation on this is pretty nonexistant so I chose instead to painstakingly reverse engineer the binary. The text above is of course the description of my findings and the source is of course based on this knowledge. That means the source is not a "clean-room" reimplementation of the nub.

Hopefully this kernel extension, unlike the class present in Apple's binary, will not only serve its functional purpose but also illustrate some of the trickier parts of the IOKit.

Without further ado, enjoy.