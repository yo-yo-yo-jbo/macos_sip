# Introduction to macOS - SIP
My previous blogpost about [the macOS sandbox](https://github.com/yo-yo-yo-jbo/macos_sandbox/) mentioned `LaunchAgents` and `LaunchDaemons`.  
Those of you that have been getting their hands dirty might have discovered there are multiple directories for them, for example, for `LaunchAgents`:
- `~/Library/LaunchAgents` - per-user LaunchAgents (might not exist on vanilla boxes).
- `/Library/LaunchAgents` - global LaunchAgents.
- `/System/Library/LaunchAgents` - the system's LaunchAgents.
For `LaunchDaemons` you will only have the last 2 directories, because, as I explained in an earlier blogpost - `LaunchDaemons` are not tied to individual users.

What is the difference between the different types of `LaunchAgents`? Well, Apple has good documentation [here](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html):  
> When a user logs in, a per-user launchd is started. It does the following:
> 1. It loads the parameters for each launch-on-demand user agent from the property list files found in /System/Library/LaunchAgents, /Library/LaunchAgents, and the user’s individual Library/LaunchAgents directory.
> 2. It registers the sockets and file descriptors requested by those user agents.
> 3. It launches any user agents that requested to be running all the time.
> 4. As requests for a particular service arrive, it launches the corresponding user agent and passes the request to it.
> 5. When the user logs out, it sends a SIGTERM signal to all of the user agents that it started.

It helps a bit, but we need more information. Experimenting with them by creating one of each type can certainly help.  
We can prepare a minimal `LaunchAgent` plist file:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.jbo.mylogger</string>
	<key>Program</key>
	<string>/private/tmp/my_logger.sh</string>
	<key>ProgramArguments</key>
	<array/>
	<key>RunAtLoad</key>
	<true/>
	<key>StartInterval</key>
	<integer>300</integer>
</dict>
</plist>
```

This will attempt to run `/private/tmp/my_logger.sh` as a `LaunchAgent` (note that `/tmp` is just a symbolic link to `/private/tmp`).  
While creating this kind of `plist` in the per-user directory and the global `/Library/LaunchAgents` directory works well, this fails:

```shell
jbo@McJbo ~ % sudo cp /tmp/com.jbo.mylogger.plist /System/Library/LaunchAgents/
Password:
cp: /System/Library/LaunchAgents/com.jbo.mylogger.plist: Operation not permitted
jbo@McJbo ~ %
```

Note that this copy command ran *as root* and yet I got an annoying `Operation not permitted` error! This means root is not an omnipotent user.
I hope this is a good motivation to research a technology that limits even the root user - also known as `SIP`.

## Introduction to SIP
The first time I noticed SIP was when I tried debugging Apple-signed binaries with `lldb` - I couldn't do that even as `root`. However, the most noticable SIP feature is by far blocking any writes on various directories, most of them under the `/System` path.  
`SIP` stands for `System Integrity Protection`, also known as `rootless`, and has been around for quite some time, and it's responsible of hardening the system against root-level attackers. By default, it's on, but you can always check with the `csrutil` utility:

```shell
root@McJbo ~ # csrutil status
System Integrity Protection status: enabled.
root@McJbo ~ # csrutil disable
csrutil: This tool needs to be executed from Recovery OS.
root@McJbo ~ #
```

Note how I tried to turn it off and ended up with an error - the only *legitimate* way to turn it off is by booting to [Recovery Mode](https://support.apple.com/guide/mac-help/intro-to-macos-recovery-mchl46d531d6/mac).  
What are SIP's responsibilities exactly? [My blogpost from 2021](https://www.microsoft.com/en-us/security/blog/2021/10/28/microsoft-finds-new-macos-vulnerability-shrootless-that-could-bypass-system-integrity-protection/) documents it nicely, but you can conclude its highlevel responsibilities by reading the [relevant XNU source code](https://opensource.apple.com/source/xnu/xnu-4570.71.2/bsd/sys/csr.h) - the SIP configuration is saved in [NVRAM variables](https://wikileaks.org/ciav7p1/cms/page_26968084.html). Specifically, the variable `csr-active-config` is what we care about. Itsaves a bitmap of capabilities:

```c
...

/* Rootless configuration flags */
#define CSR_ALLOW_UNTRUSTED_KEXTS		(1 << 0)
#define CSR_ALLOW_UNRESTRICTED_FS		(1 << 1)
#define CSR_ALLOW_TASK_FOR_PID			(1 << 2)
#define CSR_ALLOW_KERNEL_DEBUGGER		(1 << 3)
#define CSR_ALLOW_APPLE_INTERNAL		(1 << 4)
#define CSR_ALLOW_DESTRUCTIVE_DTRACE	(1 << 5) /* name deprecated */
#define CSR_ALLOW_UNRESTRICTED_DTRACE	(1 << 5)
#define CSR_ALLOW_UNRESTRICTED_NVRAM	(1 << 6)
#define CSR_ALLOW_DEVICE_CONFIGURATION	(1 << 7)
#define CSR_ALLOW_ANY_RECOVERY_OS	(1 << 8)
#define CSR_ALLOW_UNAPPROVED_KEXTS	(1 << 9)

#define CSR_VALID_FLAGS (CSR_ALLOW_UNTRUSTED_KEXTS | \
                         CSR_ALLOW_UNRESTRICTED_FS | \
                         CSR_ALLOW_TASK_FOR_PID | \
                         CSR_ALLOW_KERNEL_DEBUGGER | \
                         CSR_ALLOW_APPLE_INTERNAL | \
                         CSR_ALLOW_UNRESTRICTED_DTRACE | \
                         CSR_ALLOW_UNRESTRICTED_NVRAM | \
                         CSR_ALLOW_DEVICE_CONFIGURATION | \
                         CSR_ALLOW_ANY_RECOVERY_OS | \
                         CSR_ALLOW_UNAPPROVED_KEXTS)

#define CSR_ALWAYS_ENFORCED_FLAGS (CSR_ALLOW_DEVICE_CONFIGURATION | CSR_ALLOW_ANY_RECOVERY_OS)

...
```
As you can see, there are different capabilities, including (but not limited to):
- `CSR_ALLOW_UNTRUSTED_KEXTS`: The ability to load untrusted kernel extensions.
- `CSR_ALLOW_TASK_FOR_PID`: allowing the `task_for_pid` API, which is the macOS equivalent of the Windows `OpenProcess`. Yes, injection is very different in macOS!
- `CSR_ALLOW_KERNEL_DEBUGGER`: kernel debugging.
- `CSR_ALLOW_UNRESTRICTED_NVRAM`: arbitrarily setting `NVRAM variables`.
- `CSR_ALLOW_UNRESTRICTED_FS`: not enforcing filesystem protections.

I hope you are at least kind of convinced that defeating any of those can defeat SIP as a whole - for example, with kernel debugging you could turn off the SIP protections, with unrestricted `NVRAM variables` you could directly affect the `csr-active-config` variable and so on.

## Filesystem restrictions
Historically, bypassing the filesystem restrictions seem to be the #1 cause of SIP bypasses (unless we're considering kernel vulnerabilities, I guess). How are those enforced? Is the entire `/System` tree simply protected?  
Apparently not - SIP is totally configurable at the filesystem level - for each file there is some metadata that specifies whether it's SIP protected or not.  
If you remember my [Gatekeeper blogpost](https://github.com/yo-yo-yo-jbo/macos_gatekeeper/) then you might see where I'm going - `extended attributes` are used to manage SIP!  
Well, that's 90% true. There are several configuration files (e.g. `/System/Library/Sandbox/rootless.conf`) that control which files are SIP protected and which are not (and obviously, THAT file is SIP protected), but most files will simply have an extended attribute of `com.apple.rootless`.  
As you might have guessed, there's no legitimate way to even *create* SIP protected files (e.g. by creating a `com.apple.rootless` extended attribute) since such files will be effectively undeletable - imagine some malware doing that!  
It's possible to view which files are protected with the simple `ls` utility - provide a `-O` flag and the protected files will have a `restricted` output:

```shell
root@McJbo ~ # ls -laO /System/Library/ | grep Sp
drwxr-xr-x     4 root  wheel  sunlnk       128 Apr  8 14:38 Speech
drwxr-xr-x     3 root  wheel  restricted    96 Mar 24 18:26 SpeechBase
drwxr-xr-x    20 root  wheel  restricted   640 Mar 24 18:26 Spotlight
root@McJbo ~ # echo hi > /System/Library/Speech/hi.txt
root@McJbo ~ # echo hi > /System/Library/SpeechBase/hi.txt
zsh: operation not permitted: /System/Library/SpeechBase/hi.txt
root@McJbo ~ # log show --last=5m | grep SpeechBase
2023-04-14 14:42:46.423270-0700 0x144902   Error       0x0                  0      0    kernel: (Sandbox) System Policy: zsh(76917) deny(1) file-write-create /System/Library/SpeechBase/hi.txt
root@McJbo ~ #
```

As you can see:
- Under the `/System/Library` directory, the `Speech` directory is not SIP protected (says `sunlnk`) while the `SpeechBase` is protected (`restricted`).  
- Therefore, trying to create a file under `Speech` succeeds (note I am running as `root`) while a similar operation fails for the `SpeechBase` directory.
- Using the `log` utility shows the source of the error - `Kernel: (Sandbox) System Policy` is the mark of SIP enforcement.
Those of you who remember my [macOS App Sandbox blogpost](https://github.com/yo-yo-yo-jbo/macos_sandbox/) might wonder if there's a connection between SIP and the App Sandbox - and indeed, SIP is just the same Sandbox utility applied on the entire system!
