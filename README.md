# Linux Mint 18.3-19.1 'yelp' command injection bug
## Root cause
The URI handlers `help://`, `ghelp://`, and `man://` are defined in the file `/usr/share/applications/yelp.desktop` which will execute `/usr/local/bin/yelp` via `Exec=yelp %u` whenever one of those URI handlers is invoked.

The file `/usr/local/bin/yelp` is a simple python script that parses the incoming URI handler request. This script first searches for the substrings **"gnome-help"** and **"ubuntu-help"**, and if it doesn't find either one of those substrings then it'll execute `/usr/local/bin/yelp` without wrapping the argument in quotes.

Contents of `/usr/local/bin/yelp`:
```
#!/usr/bin/python

import os
import sys

if (len(sys.argv) > 1):
    args = ' '.join(sys.argv[1:])
    if ('gnome-help' in args) and not os.path.exists('/usr/share/help/C/gnome-help'):
        os.system ("xdg-open http://www.linuxmint.com/documentation.php &")
    elif ('ubuntu-help' in args) and not os.path.exists('/usr/share/help/C/ubuntu-help'):
        os.system ("xdg-open http://www.linuxmint.com/documentation.php &")
    else:
        os.system ("/usr/bin/yelp %s" % args)  # uh oh
else:
    os.system ('/usr/bin/yelp')
```

## From PoC to Shell

Exploitation took a little bit of creativity since Google Chrome URI encodes the space **\x20** character, curly brackets **{}**, the plus **+** character, the backtick character **\`**, and some others. Since `${IFS}` wouldn't work it was discovered that using `$IFS$()` works just as well since the `$()` statement prevents from the `$IFS` environment variable from concatenating with other ASCII characters and accidentally becoming `$IFSaddedword` or so.

# Mitigations
## Overview

The file `yelp.patch` modifies the vulnerable statement from `os.system ("/usr/bin/yelp %s" % args)` to `os.system ("/usr/bin/yelp %s" % quote(args))` which leverages the `quote()` method from the `shlex` library (https://docs.python.org/3/library/shlex.html#shlex.quote).

## Instructions for patching

Either patch `yelp` or remove it.

### Patching yelp

* Install patch: `sudo apt install patch`
* Patch yelp script: `sudo patch /usr/local/bin/yelp yelp.patch`

### Removing yelp and its associated URI handlers

* Removing URI handlers: `sudo rm /usr/share/applications/yelp.desktop`
* Removing yelp python script: `sudo rm /usr/local/bin/yelp`


[[b1ack0wl]](https://twitter.com/b1ack0wl)