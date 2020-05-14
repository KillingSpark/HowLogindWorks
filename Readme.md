# How does Logind work
This document tries to collect all info I can gather about the workings and magic of session management, as performed by logind.
Since logind is the predominant session manager this is kinda the 'current state of the art".

Note that I did not read any source-code. So everything here could be wrong, if the available documentation disagress with the code.
I had two ways of figuring stuff out: 
1. Reading blogs and systemd doc
1. Poking my own system and looking for clues 

## Big picture
So session management needs to work on a number of different things to work correctly. Lets first try to formulate the goal session management tries to achieve.

* A `User` should be able to login on a `Loginmanager`. This should put the `User` into a newly created `Session`. 
* The login could happen on a set of hardware (as opposed to loggin in over ssh or something similar). Lets call this set of hardware a `Seat`.
* This seat should be accessible to this session and not be available to other sessions until this session releases the seat.
* The `SessionManager` needs to keep track of the sessions and their assigned seats. It also watches for exited sessions and reclaims their seats.
* The `SessionManager` can remove a session from a seat and give access to another session if needed.

Immediatly we have a whole bunch of concepts session management needs to deal with. All of this has of course many details that need to be talked about. This will be the next few sections.

## Lets get detailed!
In the following sections I will describe in a lot of detail how the stuff in the "big picture" section work.

### Users
Users and their identification and authentification are luckily out of the scope of session management. We expect this to be handled by something else. Logind
requires PAM to be used anyways so probably some PAM module will be used to authenticate a users when he logs in.


### Seats
Reasoning about which hardware belongs to which seat is hard. Thats why logind does not do it. This is part of udev. I still want to talk about it because there
is one weird interaction between user ACLs set by logind and udev.

Ok so shot intro for udev: linux exposes a very messy and complicated device file tree. Udev looks at it and makes some sense out of it and provides a simpler 
abstraction. It also has a collection of device-ids, vendor-ids, etc... that help it classify these devices. It then creates device files in /dev for you to 
interact with the devices (or device drivers). It also listens to events that indicate new/removed devices and takes appropriate actions.

Udev also tags devices with useful info. The tags that interest logind are the seat related ones. Udev tags some devices with `seat` if they belong to a seat.
Which seat is stated in another property of this device. Note that not all sub-devices of a device are tagged with `seat`. This still means that they belong to the 
same seat as their tagged parent. There is also the `seat-master` tag that makes a seat exist (in most cases a monitor / graphics card).

There is another relevant tag: `uaccess`. When trying to find out what that tag means I stumbled upon this gem of an issue on the systemd repo: [Document the uaccess mechanism / dynamic device permissions](https://github.com/systemd/systemd/issues/4288) which is open since 2016. Documenting stuff is hard.

So after searching the net for the doc that systemd doesnt provide I found [this link from 2012](https://enotty.pipebreaker.pl/2012/05/23/linux-automatic-user-acl-management/). It shows how udev at that time runs a command `/usr/lib/systemd/systemd-uaccess $env{DEVNAME} $env{ID_SEAT}` for each device 
with the tag `uaccess`. This utility does not exist anymore (at least not on my arch installation) but I am guessing that something similar is still happening.
What that command did is creating the appropriate ACLs base on which seat the device was assigned to.

So logind is not the only entity granting users access to devices but udev might do that too, when a new device gets plugged in. I hope that is not causing any 
race conditions...

So all devives that are tagged `uaccess` are tagged `seat` but not all `seat` are `uaccess`.
From what I gathered from poking around in the output of `udevadm info -e` sound output devices and video cards are the only devices marked `uaccess`.
It makes sense that not everyone can open input devices like keyboards (else keylogging would be easy). See in the sections 
below, how access to these devices is realized with logind as a proxy.

How exactly multi-seat is done in udev-rules is not clear to me. I think an admin has to manually make rules to say which device belongs to which seat. Otherwise 
all devices belong to the default seat `seat0`.


### Create a session on login
Having Users and Seat management out of the way, we can start worrying about the stuff we really want to do: create and manage sessions!
So what happens when a `DisplayManager` like SDDM wants to start a session for a newly logged in user? 

The goal is simple: notify the session manager that a new session exists. Logind uses a 
dedicated PAM module that notifies logind about the new session.
("Creating/closing sessions is exclusively the job of PAM and its pam_systemd module" [\[1\]](https://www.freedesktop.org/wiki/Software/systemd/logind/)).

The PAM modules are executed by the `LoginManager` or `DisplayManager`, which can have funny side effects, see below.

The CreateSession call [\[1\]](https://www.freedesktop.org/wiki/Software/systemd/logind/) on the dbus API of logind gives a good overview of
1. What is needed to create a new session
2. How messy systemd APIs are

There will be a lot of guessing, and me saing I don't know what is happening. I promise the later parts of this document will be better informed. 
There are more and better sources on other parts of loginds inner workings. If anyone can shed light on what some of the more mysterious 
parameters do please let me know!

```
CreateSession(in  u uid,
    in  u pid,
    in  s service,
    in  s type,
    in  s class,
    in  s desktop,
    in  s seat_id,
    in  u vtnr,
    in  s tty,
    in  s display,
    in  b remote,
    in  s remote_user,
    in  s remote_host,
    in  a(sv) properties,
    out s session_id,
    out o object_path,
    out s runtime_path,
    out h fifo_fd,
    out u uid,
    out s seat_id,
    out u vtnr,
    out b existing);
```

#### CreateSession in detail

So lets pick that apart. Creating a session needs info about which user that session is being created for, identified by a `uid`. The `pid` is probably
the `root` or `parent` process for this new session. So far this is relatively straight forward. 

Now it gets weirder, why is a session associated with a service?
According to [\[1\]](https://www.freedesktop.org/wiki/Software/systemd/logind/) this is "Service encodes the PAM service name that registered the session". I am not entirely sure why this is necessary info to the session manager, but it is not entirely weird. Maybe a session created by some PAM module need special handling.

"Type encodes the session type. It's one of "unspecified" (for cron PAM sessions and suchlike), "tty" (for text logins) or "x11"/"mir"/"wayland" (for graphical logins)".
I don't really get why this is important, a session might switch between text and graphical mode without involving the session manager so that should not really matter.

"Class encodes the session class. It's one of "user" (for normal user sessions), "greeter" (for display manager pseudo-sessions), "lock-screen" (for display lock screens)".
How are sessions created that are not user? I thought only systemd_pam calls CreateSession. But the greeter and lockscreen probably do not go through a PAM login session do they?

The `desktop` parameter is entirely undescribed. I guess it was intended to tell logind which DE a user is using and was not needed in the end?

Ok so the `seat_id` makes sense. A session needs to be placed in a seat. This does mean that the systemd_pam is able to figure out on which seat the current process is
running (or somehow else infer on which seat it should be placed).

The `vtnr` is necessary for the good old session switching with alt+ctrl+fx ("VTNr encodes the virtual terminal number of the session"). 
There might be more than one session on `seat_id` but only one active one. I always thought that ttyX is always associated with the virtual terminal with 
number X but I might be wrong, so the `tty` parameter encodes that.

`remote`, `remote_user`, and `remote_host` confuse me a little. I thought `remote` would be inferrable from `service`. Maybe not. Why logind is interested in 
the `remote_user` and `remote_host` and how it gets these values is unclear to me.

`properties` is another opaque parameter, probably used to make extensions to the API without actually changing the API. I dont what this might contain.

The output is relatively straight forward. I am guessing that the `uid`, `seat_id`, and `vtnr` are only relevant if `existing` is true. How a session would be created a 
second time is unclear to me, it seems like that should rather be an error, but thats a design choice.

The `fifo_fd` is really interesting to me. I don't know what this is for. Remember this is called by the systemd_pam module. Why does that need another
way of communicating with logind? I dont even know in which direction this fifo is passing data. 
Is it for pushing events only directed to systemd_pam? Is it for sending more data to logind?  
And if yes: Why does it not use the dbus calling/signaling mechanics, since it is already connected to dbus? 
I would love to know more about this fifo, but I don't, sorry. 
Maybe it is used to communicate with a whole other component that gets passed the other end of the fifo, but I cannot guess what that would be.

#### Effects of creating a session
Creating a session with logind has a number of important effects.

1. Systemd creates a new scope in the slice of the user specified by `uid`
1. Systemd moves the `pid` into that scope. 
    * I am pretty sure that the systemd_pam module reports its own PID
    * This means the process that ran the PAM modules is now in a fresh otherwise empty cgroup, that has nothing to do anymore with the cgroup the `LoginManager`
        was in originally.
    * This means if you do not fork+exec before you execute the PAM modules your `LoginManager` is suddenly the session root. This could prove to be problematic
        for detecting exited sessions. I hope logind checks that no pid is a root for multiple sessions but I did not check that.
    * This is not properly documented in [\[2\]](https://www.freedesktop.org/wiki/Software/systemd/writing-display-managers/). This is mostly concerned with porting
        from ConsoleKit, maybe it was required to have fork+exec there too.
1. It sets ACLs to some device files, but not all. Which devices and how this is determined will be described in a later section.
    Anyways you can now in some way or another get access to all the devices on the seat you logged into. 

Great! Now we have created a session and have been seated (haha get it...) by logind. Our `LoginManager` may now start the actual session and drop us into KDE or Gnome
or i3 or whatever it was we originally wanted to start before being sidetracked by the session shenanigans.

### Accessing devices with logind
X11 and wayland need to access devices. To mouse/keyboard/touch/... for input and to screens for output. Traditionally the X11 server needed to get access to them
by being root and then drop privileges. This is obviously a bad choice since we are starting a session for a user. And that user better not get root access!

In comes logind providing the possibility to start the session completely as the dedicated used and not rely on that process to properly drop privileges.
This is possible by logind playing device-broker. There is a nice article with good graphics here: [\[3\]](https://dvdhrm.wordpress.com/2013/08/25/sane-session-switching/).

The summary is:
1. One process needs to call `TakeControl(...)` on the dbus interface of the created session
1. This process can call `TakeDevice(...)` which returns a filedescriptor
1. This filedescriptor can be revoked, if this happens the process can call `TakeDevice(...)` again to try to regain access

Logind can open these devices and only logind needs to run with eleviated priviliges. Everything else needs to go through logind to get access to devices (at least for input. There are some exceptions, where devices are accessible to normal users.)

#### Libinput
Now, X11 and Wayland compositors do not themselves deal with devices. Both rely on abstractions. 
X11 has multiple input drivers (a libinput based one exists and is widely used), while wayland relies solely on libinput.

Libinput normally doesn't care about sessions and whatnot, it just cares about getting device-input from the kernel to the user of the library. It does
provide a convenient notion of `seats` from udev which allows the user to get events from the whole seat instead of handling all devices separatly.

It makes the user to open a devices themselves though, side-stepping the whole permission problems and handing them to the user of the library. The user needs 
to provide a function called `open_restricted` which returns a filedescriptor for a given device path. This way we can integrate logind support without 
libinput directly depending on logind. 

See [here](https://wayland.freedesktop.org/libinput/doc/latest/api/structlibinput__interface.html#ae445aaa330e4eb7df6650fbc6428022a) for the doc
for `open_restricted`.

As an example of how this can be used see these two files from wlroots, a popular wayland compositor library
* https://github.com/swaywm/wlroots/blob/master/backend/libinput/backend.c
    * setup of libinput and uses the sessions routines to open the device files 
* https://github.com/swaywm/wlroots/blob/master/backend/session/logind.c
    * uses the logind dbus call `TakeDevice(...)` to open devices and return the filedescriptors

### Switching Sessions / Removing a Session from a Seat
Ok so far we were able to log into the system, create a session and be able to use out mouse, keyboard and display to look at funny cat videos on 
youtube [example](https://www.youtube.com/watch?v=dQw4w9WgXcQ). 
This is fine and dandy, but our boss is already coming dangerously close to realizing we are not working as much as we should, so we want to prepare a session
with $SERIOUS_USER and be able to switch to that quickly if he comes around to check in with us.

So we all know how this works, press alt+ctrl+F1 and login again as $SERIOUS_USER, start some busy looking screens and go back to the session with the cat videos.

Now it gets tricky. Remember how we said only one session should have access to a seat at a given time? How is this enforced? We cant rely on 
every session being nice and cooperative and bug-free.
The filedescriptors for the devices are open, logind gave them to the session. And we cannot close the filedescriptors in other processes. Or can we?

Not exactly. But the kernel provides us with something similar for filedescriptors associated with evdev devices: an ioctl command `EVIOCREVOKE`. 
This basically closes the filedescriptor but not completely. It stays valid in all processes that still also have it but all reads after this ioctl() will not return 
any events but will indicate that access has been revoked.  

So now logind can remove a session from a seat and enforce that this session can no longer read any input events, since (hopefully) all 
input device filedescriptors have been handed to the session through logind. When the removed session gets reactivated it has to regain access by asking logind for
new, fresh filedescriptors.

### Watching for exiting sessions
This is a nontrivial part of session management. You need to somehow be able to tell which processes belong to which sesssion be able to receive 
notifications when they exit. This could probably be done by using the posix sessionids in most cases. But those are not reliable since processes can leave their
sessions and make a new one. Now we dont know anymore what is going on!

That is why logind pushes the first process into a cgroup right at creation of a session. With cgroups we can reliably track which process
belongs to which session, and we also can easily use inotify to get the needed notifications. Cgroups are nice!
(as long as no root user fiddles around with those cgroups 
[WHICH ARE SYSTEMD PROPERTY AND IF YOU DARE TOUCH THEM YOU ARE A BAD PERSON](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/))

# Sources
* \[1\] https://www.freedesktop.org/wiki/Software/systemd/logind/
* \[2\] https://www.freedesktop.org/wiki/Software/systemd/writing-display-managers/
* \[3\] https://dvdhrm.wordpress.com/2013/08/25/sane-session-switching/
