# Introduction to macOS - SIP
My previous blogpost about [the macOS sandbox](https://github.com/yo-yo-yo-jbo/macos_sandbox/) mentioned `LaunchAgents` and `LaunchDaemons`.  
Those of you that have been getting their hands dirty might have discovered there are multiple directories for them, for example, for `LaunchAgents`:
- `~/Library/LaunchAgents` - per-user LaunchAgents (might not exist on vanilla boxes).
- `/Library/LaunchAgents` - global LaunchAgents.
- `/System/Library/LaunchAgents` - the system's LaunchAgents.

What is the difference between all of them?
