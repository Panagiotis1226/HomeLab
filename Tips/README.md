# Tips and Tricks

## If using a Laptop as a server, you can prevent it from sleeping when the lid is closed.

```bash
nano /etc/systemd/logind.conf
```

Modify the following lines and remove the `#` to enable:

```yaml 
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
HandleHibernateKey=ignore
HandleSuspendKey=ignore
HandleSleepKey=ignore
```
Edit the Sleep config file
```bash
nano /etc/systemd/sleep.conf
```

Modify the following lines:

```yaml 
[Sleep]
AllowSuspend=no
AllowHibernation=no
AllowSuspendThenHibernate=no
AllowHybridSleep=no
```
Restart the services:
```bash
sudo systemctl restart systemd-logind
sudo systemctl restart systemd-suspend
```

