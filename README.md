# show-tx-pkt-shell-prompt

Simple (but still a bit too complex) way to show an updated transmit packet count for an interface in your shell prompt

## Dependencies

This utilizes `sar`. There are a lot of clever ways to do this, some are lightweight, some are heavy, some are more accurate than others, etc, etc. This is just the quickest way I hacked it together that works well enough for me. You can probably do it much better with a small `.c` reading from `/proc/net`. You could certainly use a shell script, perl script, etc. There are a million ways to do this

## Components

While this can of course be done with one-liners, you don't want to wait a second each time your prompt rendors to pull a second's worth of sample data, so the best way to do this seems to have a simple service/long-running process in the background, keeping a flat file up to date, and then accessing that in the prompt using `cat`

### net-tx-mon-daemon

A very simple shell script that continuously samples the transmit rate using `sar`. It does *not* spin the CPU since `sar` does polling with sleeps. The content:

```
$ cat net-tx-mon-daemon
#!/bin/bash
export TEMP_TX=/dev/shm/tx-tmp
export DONE_TX=/dev/shm/tx-now
export RAW_TX=/dev/shm/tx-raw
export INTERFACE=eth0
while [ 1 ]; do
  sar -n DEV 1 1 > "${RAW_TX}"
  grep -E "^Average: +${INTERFACE}" "${RAW_TX}" | awk '{print $4}' > "${TEMP_TX}"
  cp "${TEMP_TX}" "${DONE_TX}"
done
```

You might be curious ... "why so many files?" or "why not a one-liner?" .. the answer is because you don't want to get caught reading a truncated file, or one that's in the process of being updated. So..

1. Get the sample data by running `sar` and output it to `$RAW_RX`
2. Use `grep` to pull out the line you want and awk to grab the column and output it to `$TEMP_TX`
3. Finally use `cp`, which is "atomic enough" to make this all work cleanly, copying the final result into `$DONE_TX`

This process should run as a low-privileged user, ideally a dedicated user with no ownership of any files. You can run it however you like. If you want to do it the "clean" and "proper" way, you can use a simple `systemd` service

### Using systemd to start/stop/status check the daemon

First, a caveat. There are a significant amount of security features now available in systemd, especially if your kernel supports namespaces. You can essentially create a container for this simple little service. Just remember you'll need to be able to access the one file it produces outside of that container. I'm not covering all the hardening options here because it's a very simple script and when running with minimal privileges it's really not much of a security issue. But if you're nervous, don't use it :>

OK, here is the barebones service file

```
$ cat /etc/systemd/system/net-tx-mon.service
[Unit]
Description=Network Transmission Packet Counter Service
After=network.target

[Service]
User=nobody     # It is safer to use a dedicated user, e.g. nettx-nobody
Group=nogroup   # It is safer to use a dedicated group, e.g. nettx-nogroup
ExecStart=/usr/bin/net-tx-mon-daemon
ExecStop=/bin/kill -s QUIT $MAINPID

[Install]
WantedBy=multi-user.target
```

You can now start the daemon and enable it for run at boot-time

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable net-tx-mon
$ sudo systemctl start net-tx-mon
```

## Using

As long as this glorified script "service" is running, you should be able to check the average transmitted packets per second using:

```
$ cat /dev/shm/tx-now 
620673.00
```

## Integrating into a prompt

This is most useful if you have an interactive shell and always want to know this rate. You can simple embed the following in your prompt:

```
$(cat /dev/shm/tx-now)
```

It depends on what shell you are using and if you're using a framework for your shell rc files when it comes to doing this properly. Here is a simple example of how to use this with an `oh-my-zsh` theme file

### Changing an oh-my-zsh theme (aussiegeek)

Simply comment out the current prompt, and insert how/where you please the `cat` statement. For example:

```
$ cat ~/.oh-my-zsh/themes/aussiegeek.zsh-theme | head -2
# PROMPT='$fg_bold[blue][ $fg[red]%t $fg_bold[blue]] $fg_bold[blue] [ $fg[red]%n@%m:%~$(git_prompt_info)$fg[yellow]$(rvm_prompt_info)$fg_bold[blue] ]$reset_color
PROMPT='$fg_bold[green]TX PPS:($(cat /dev/shm/tx-now)) $fg_bold[blue][ $fg[red]%t $fg_bold[blue]] $fg_bold[blue] [ $fg[red]%n@%m:%~$(git_prompt_info)$fg[yellow]$(rvm_prompt_info)$fg_bold[yellow] $fg_bold[blue] ]$reset_color
```

You can now login again and you will see the packet rate reflected on the left side of your prompt in green, like this (except with color):

```
TX PPS:(623136.00) [ 10:09PM ]  [ user@host:~  ]
 $ 
```

## Why would you want to do this?

I don't know. I needed to do it because I do heavy network I/O and I want to know if things are crashed or melting.

## Can I show rx count?

Sure. Change the value in the `awk` column selection

## Can I show tx or rx bandwidth, instead of packet count?

Yeah. Same thing.

