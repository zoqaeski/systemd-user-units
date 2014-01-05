# UPDATE

As of systemd-206 and higher, most of this fails to work as expected due to how
loginctl creates user slices: user services run outside of the session, so NO
session data is available to them.

I’ll keep these documents and files here, but I personally am not using systemd
to control my session anymore.

# Using `systemd --user`

systemd is useful for system initialization, but it it also useful from a user
standpoint. Using `systemctl --system` to do anything requires root privileges
and is therefore not useful to unprivileged users. Starting their personal
services either requires them to be enabled or to be started by someone with
root privileges. Recently, I have begun working with the user part of systemd. 

I began using the information on the [Arch Linux wiki][1], and also followed
[KaiSforza’s guide][2] (this README was based on that guide). If anyone knows
where I can find [gtmanfred’s guide][3], please point it to me so I can improve
this.

All of your systemd user units must reside in `$HOME/.config/systemd/user`.
These units take precedence over the other systemd unit directories.

## Prerequisites

There are two packages you need to get this working, both currently available
from the AUR: [xorg-launch-helper][4] and [user-session-units][5]. You should
also have at least the ability to apply patches and edit pkgbuilds. Just remember
that this is no easy feat. Though once you get it working you will be rewarded
with a damn beautiful system.

Though I have installed `xorg-launch-helper`, I have adopted a different
approach due to difficulties integrating systemd and Xorg. At present I login
using LXDM, so Xorg cannot be started by a systemd user unit. None of my units
have dependency checking for Xorg as a result, so those that depend on it will
fail if it is not running.

## Creating some starting units

Next is setting up your targets. I’ve set up three,	one for environment
variables, one for my terminal multiplexer (tmux), and the other to emulate my
~/.xinitrc. There is also a fourth which, although not presently in use, is
intended to start my window manager (Openbox). 

    ~/.config/systemd/user/xinitrc.target
    [Unit]
    Description=Xinitrc Stuff
    After=environment.target
    #Wants=wm.target
    Wants=environment.target
    Wants=multiplexer.target
    
    [Install]
    Alias=default.target

Link this to `default.target`. Other guides may list this unit as
`mystuff.target`, but I thought `xinitrc.target` was a much better name.
	
    ~/.config/systemd/user/environment.target
    [Unit]
    Description=Set session environment variables
    Requires=dbus.socket
    IgnoreOnIsolate=true

This can be used to set environment variables. At the moment, I am using it to
ensure dbus dependency. I’d like to set more environment variables here, but
they don’t seem to take effect.

	~/.config/systemd/user/multiplexer.target
    [Unit]
    Description=Terminal multiplexer
    Documentation=info:screen man:screen(1) man:tmux(1)
    
    [Install]
    Alias=default.target

This can be installed as `default.target` so you can start from console straight
into a terminal multiplexer. 

You could also create a target similar to `xinitrc.target` for services that do
not depend on X (maybe call it `initrc.target`?), and add a line to your login
scripts that are run when you log in to console such as `systemd --user
-unit=initrc.target &`, though you would need to make sure this command is not
executed when you log into X. I haven’t tested this, but it shouldn’t be
difficult; I start systemd from my `~/.xinitrc` so it will never be started on
console. All my non-X services are wanted by `default.target`, so they will
always be started when `systemd --user` is run without a specified target unit. 

If you wish to try starting your window manager from systemd, you can write a
unit based on this one:

    [Unit]
    Description=Window manager
    Wants=xorg.target
    Wants=xinitrc.target
    Requires=dbus.socket
    AllowIsolate=true
    
    [Install]
    WantedBy=default.target

Due to the aforementioned difficulties with LXDM and Xorg, I am **not** doing
this. Your mileage may vary; if you do this, you will need a service that starts
your window manager of choice directly (Openbox in this example) :

    [Unit]
    Description=Openbox
    Wants=compton.service
    Conflicts=gnome-session.service
    After=xorg.target
    After=environment.target
    Requires=xorg.target
    
    [Service]
    ExecStart=/usr/bin/openbox
    ExecReload=/usr/bin/openbox --reload
    ExecStop=/usr/bin/openbox --exit
    
    KillMode=process
    Restart=always
    RestartSec=1
    
    [Install]
    WantedBy=wm.target

Note the [Install] section contains a ‘WantedBy’ part. When using 
`systemctl --user enable` it will link this as
`$HOME/.config/systemd/user/wm.target.wants/openbox.service`, allowing it 
to be started at login. I would recommend enabling this service, not linking it
manually.

You can fill your user unit directory with a plethora of services, I
currently have ones for tmux, urxvtd, compton, wallpaper (feh) and conky, to
name a few. This allows these programs to be tracked by systemd individually.
Some people have managed to get PulseAudio running this way, but it crashes for
me, so I still use the old initscript in `/etc/X11/xinit/xinitrc.d/` for the
time being.

## Some actual important stuff

Last but not least, add this line to /etc/pam.d/login and
/etc/pam.d/system-auth:

    session    required    pam_systemd.so

Now add `/usr/lib/systemd/systemd --user` to your shell's `$HOME/.*profile` file
and you are ready to go! (This takes a lot of tweaking, so when I say that, I
mean that you are ready to debug and find spelling mistakes.)

Using these services and target will give you a very minimal user environment.
I would recommend making service files for as many applications as possible and
then using those to start the applications. That way things can be tracked and
managed easily. 

## Auto Login

I am not using this. 

## Anecdotes

One of the most important things you can add to the service files you will be
writing is the use of Before= and After= in the [Unit] section. These two
parts will determine the order things are started. Say you have a graphical
application you want to start on boot, you would put `After=xorg.target` into
your unit. Say you start ncmpcpp, which requires mpd to start, you can put
`After=mpd.service` into your ncmpcpp unit. You will eventually figure out
exactly how this needs to go either from experience or from reading the
systemd manual pages. I would recommend starting with [systemd.unit(5)][6].

Just as a note to readers and users, I am by no means an expert on this stuff.
It's not easy, and it made me work. If you don't get it to work the first time,
don't just pass it off as a failed attempt. I completely botched it my first
try, and only got it working with some more reading and excellent help. Don't
give up!

[1]: https://wiki.archlinux.org/index.php/Systemd/User
[2]: https://bitbucket.org/KaiSforza/systemd-user-units
[3]: http://blog.gtmanfred.com/?p=26 (broken link)
[4]: https://aur.archlinux.org/packages/xorg-launch-helper/
[5]: https://aur.archlinux.org/packages/user-session-units/
[6]: http://www.freedesktop.org/software/systemd/man/systemd.unit.html
