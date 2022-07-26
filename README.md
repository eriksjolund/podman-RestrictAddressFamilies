Title: "_Restricting Podman with the systemd directive RestrictAddressFamilies_"

Subtitle: "_Learn how to restrict network access for Podman_"

The previous blog post _Use socket activation with Podman to get improved security_ demonstrated that a socket-activated container can use the activated socket to serve the internet even when the network is disabled (i.e., when the option __--network=none__ is given to `podman run`). This blog post takes the approach one step further by also restricting the internet access for Podman and its helper programs (e.g., conmon and the OCI runtime).

Requirements:
* podman >= 3.4.0
* runc >= 1.1.3 or crun >= 1.5
* container-selinux >= 2.183.0 (if using SELinux)

When using Podman in a systemd service, the __systemd__ directive
[__RestrictAddressFamilies__](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictAddressFamilies=)
can be used to restrict Podman's access to sockets. The restriction only concerns the use of the system call __socket()__,
which means that [socket-activated sockets](https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md#socket-activation-of-containers)
are unaffected by the directive.

Containers that only need internet access via socket-activated sockets can still be run by Podman when
systemd is configured to restrict Podman's ability to use the system call `socket()` for the AF_INET and AF_INET6
socket families. Of course, Podman would then be blocked from pulling down any container images so the container
image needs to be present beforehand.

## Example: restrict a socket-activated echo server

Let's see how we could use __RestrictAddressFamilies__ for [socket-activate-echo](https://github.com/eriksjolund/socket-activate-echo/pkgs/container/socket-activate-echo),
a simple echo server container that supports socket activation.

If the `--pull=never` option is added to `podman run`, the echo server container will continue to work even with
the very restricted setting

```
RestrictAddressFamilies=AF_UNIX AF_NETLINK
```

All use of the system call `socket()` is then disallowed except for AF_UNIX sockets and AF_NETLINK
sockets.

In case there would be a security vulnerability in Podman, conmon or the OCI runtime, this configuration limits
the possibilities an intruder has to launch attacks on other PCs on the network.

### Create the systemd unit files

Create the container

```
$ podman pull -q ghcr.io/eriksjolund/socket-activate-echo
$ podman create --rm --name restricted-echo --network=none --sdnotify=conmon --pull=never ghcr.io/eriksjolund/socket-activate-echo
```

Generate the systemd service unit

```
$ mkdir -p ~/.config/systemd/user
$ podman generate systemd --name --new restricted-echo > ~/.config/systemd/user/restricted-echo.service
```

Add the two lines
```
RestrictAddressFamilies=AF_UNIX AF_NETLINK
NoNewPrivileges=yes
```
under the line `[Service]` with the program __sed__ (or just use an editor)

```
$ sed -i '/\[Service\]/a \
RestrictAddressFamilies=AF_UNIX AF_NETLINK\
NoNewPrivileges=yes' ~/.config/systemd/user/restricted-echo.service
```

Create the file _~/.config/systemd/user/restricted-echo.socket_ with
the contents

```
[Unit]
Description=restricted echo server
[Socket]
ListenStream=127.0.0.1:9933

[Install]
WantedBy=default.target
```

Add the two lines
```
After=podman-pause-process.service
BindTo=podman-pause-process.service
```
under the line `[Unit]` with the program __sed__ (or just use an editor)

```
$ sed -i '/\[Unit\]/a \
After=podman-pause-process.service\
BindTo=podman-pause-process.service' ~/.config/systemd/user/restricted-echo.service
```

Create the file _~/.config/systemd/user/podman-pause-process.service_ with the contents

```
[Unit]
Description=podman-pause-process.service

[Service]
Type=oneshot
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman unshare /bin/true
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

### Test the echo server

Start the socket unit

```
$ systemctl --user start restricted-echo.socket
```

Test the echo server with the program __socat__

```
$ echo hello | socat - tcp4:127.0.0.1:9933
hello
$
```

The echo server works as expected! It replies _"hello"_ after receiving the text _"hello"_.

Podman is blocked from establishing new connections to the internet but everything
works fine because Podman is configured to not pull the container image.

Now modify the service unit so that Podman always pulls the container image

```
$ grep -- --pull= ~/.config/systemd/user/restricted-echo.service
	--pull=never ghcr.io/eriksjolund/socket-activate-echo
$ sed -i s/pull=never/pull=always/ ~/.config/systemd/user/restricted-echo.service
$ grep -- --pull= ~/.config/systemd/user/restricted-echo.service
	--pull=always ghcr.io/eriksjolund/socket-activate-echo
```

After editing the unit file, systemd needs to reload its configuration

```
$ systemctl --user daemon-reload
```

Stop the service

```
$ systemctl --user stop restricted-echo.service
```

Test the echo server with the program __socat__

```
$ echo hello | socat - tcp4:127.0.0.1:9933
$
```

As expected the service fails because Podman is blocked from establishing a connection to the container registry.

__journalctl__ shows this error message

```
$ journalctl --user -xe -u restricted-echo.service | grep -A2 "Trying to pull" | tail -3
Jul 16 08:26:10 asus podman[28272]: Trying to pull ghcr.io/eriksjolund/socket-activate-echo:latest...
Jul 16 08:26:10 asus podman[28272]: Error: initializing source docker://ghcr.io/eriksjolund/socket-activate-echo:latest: pinging container registry ghcr.io: Get "https://ghcr.io/v2/": dial tcp 140.82.121.34:443: socket: address family not supported by protocol
Jul 16 08:26:10 asus systemd[10686]: test.service: Main process exited, code=exited, status=125/n/a
$
```

systemd has marked the service and the socket as being in the __failed__ unit state.

```
$ systemctl --user is-failed restricted-echo.service
failed
$ systemctl --user is-failed restricted-echo.socket
failed
```

Revert the change and use __--pull=never__ instead

```
$ sed -i s/pull=always/pull=never/ ~/.config/systemd/user/restricted-echo.service
$ systemctl --user daemon-reload
$ systemctl --user reset-failed restricted-echo.service
$ systemctl --user reset-failed restricted-echo.socket
$ systemctl --user start restricted-echo.socket
```

## The need for a separate service for creating the Podman pause process

Rootless Podman uses a pause process for keeping the unprivileged
namespaces alive. When running a command such as `podman run`,
Podman will first create the Podman pause process if it's missing. This is the case for
rootless Podman. When running as root there is no need for a Podman pause process.

Let's consider the situation when systemd starts the systemd user services for
an unprivileged user directly after a reboot. If lingering has been enabled for the user
(`loginctl enable-linger <username>`) and the user is not logged in, the first
started Podman systemd user service will notice that the Podman pause process is missing
and will thus try to create it. This normally succeeds, but when RestrictAddressFamilies
is used together with rootless Podman, creating the Podman pause process fails.

The reason is that using RestrictAddressFamilies in an unprivileged systemd user service
implies [`NoNewPrivileges=yes`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#NoNewPrivileges=),
which prevents __/usr/bin/newuidmap__ and __/usr/bin/newgidmap__ from running with elevated privileges.
Rootless Podman needs to execute __newuidmap__ and __newgidmap__  when setting up the user namespace. Both executables normally
run with elevated privileges as they perform operations not available to an unprivileged user.
The extra capabilities are

```
$ getcap /usr/bin/newuidmap
/usr/bin/newuidmap cap_setuid=ep
$ getcap /usr/bin/newgidmap
/usr/bin/newgidmap cap_setgid=ep
$
```

Services using `RestrictAddressFamilies` or `NoNewPrivileges=yes` can
be made to work by configuring them to start after a systemd user service that is responsible for
creating the Podman pause process.

For instance, the unit _restricted-echo.service_ depends on _podman-pause-process.service_:

```           paus-process
$ grep podman-pause-process.service ~/.config/systemd/user/restricted-echo.service
After=podman-pause-process.service
BindTo=podman-pause-process.service
```

The service _podman-pause-process.service_ is a `Type=oneshot` service that executes `podman unshare /bin/true`. That
command is normally used for other things, but a side effect of the command is that it creates the Podman pause
process if it's missing.

Enable lingering

```
$ loginctl enable-linger $USER
```

Enable the socket units and reboot

```
$ systemctl --user -q enable restricted-echo.socket
$ systemctl --user -q enable podman-pause-process.service
$ sudo reboot
```

After the reboot, test the echo server with the program __socat__

```
$ echo hello | socat - tcp4:127.0.0.1:9933
hello
$
```

The echo server works as expected even after a reboot!

### Wrap up

The systemd directive _RestrictAddressFamilies_ provides a way to restrict network access for Podman
and its helper programs while a socket-activated container still can serve the internet.

This could, for instance, improve security for a machine that is running a socket-activated web server container.
In case Podman, conmon or the OCI runtime would be compromised due to
a security vulnerability, the intruder would gain less privileges and therefore have less
possibilities to use the compromise as a starting point for attacks on other PCs.

Note, using the systemd directive _RestrictAddressFamilies_ to restrict Podman is an advanced use of Podman.
The method is not mentioned in the [Podman documentation](https://docs.podman.io/en/latest/) and might not be officially supported.

The [socket activation tutorial](https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md) provides
more information about socket activation support in Podman.
