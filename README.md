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
jbo@McJbo ~ # csrutil status
System Integrity Protection status: enabled.
jbo@McJbo ~ # csrutil disable
csrutil: This tool needs to be executed from Recovery OS.
jbo@McJbo ~ #
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
