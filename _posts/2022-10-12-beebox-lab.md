
# Beebox Lab
## Open Ports and Services
Check open ports and running services in all port range against our target with a SYN scan
- Command: `nmap -sS -sV -p- 10.0.2.14`
- Notes:
    - If cannot recognize a service, you'll see a `doom?` value
    - All Web Apps can be analyzed with Nikto and OWASP
## Exploits and vulnerabilities
Command `nmap -sS -p 21 --script=exploit,vuln 10.0.2.14`
Check and note down CVEs one by one on each port

### SMPT Services
For SMPT service, check which methods are enabled 
    - `nmap -sS -sV -p 25 --script=auth,default,exploit,vuln 10.0.2.14`
    - VRFY method: allows us to get existing user accounts in the domain
    - `smpt-user-enum --help`
### Web Apps
- Command: `nikto -host http://10.0.2.14`
- Which crossdomain policy has: go to browser `http://10.0.2.14/crossdomain.xml`. I.e. wildcard entry (wrong config)
- Check interesting routes found: `http://10.0.2.14/phpmyadmin` is accessible? (Later, use dictionary attack - 'guest' / '')
- Alternatively use OWASP ZAP automated scan

### Machine/Domain info
- Command on Samba ports: `enum4linux -a 10.0.2.14`
- Find versions, user accounts, shared resources, password policies