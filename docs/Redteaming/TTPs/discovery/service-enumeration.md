# Service Discovery

## Windows

```powershell
C: \> sc; tasklist /svc
C: > net start
C: \> wmic service
PS C: \> Get-Service
```

## *nix

```bash
$ service --status-all
$ systemctl list-units --type=service -all # systemd
$ ls -1 /etc/init.d/* # SystemV
```