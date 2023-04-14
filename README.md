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
	<integer>86400</integer>
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
