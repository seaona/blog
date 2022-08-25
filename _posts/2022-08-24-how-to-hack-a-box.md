# How to Hack a Box


## Information Gathering
- **Online Tools**
    - [Security Trails](https://securitytrails.com/)
    - [Shodan](https://www.shodan.io/)


- **Custom Word List generator (CeWL)** : CeWL spiders a given URL, up to a specified depth, and returns a list of words which can then be used for password crackers. 

    `cewl -m 8 -e --email_file ./emails.txt -d 0 -w ./diccio https://twitter.com/hackbysecurity?lang=es`


- **Common User Passwords Profiler (CUPP)**: generate commmon passwords dictionaries based on personal information from the target. You can also import and mutate existing dictionaries. Download from source: [cupp](https://github.com/Mebus/cupp.git)
    
    `python3 cupp.py -i`


- **The Harvester**: using different search engines. Must first configure API Keys:

    `nano /etc/theHarvester/api-keys.yaml`

    `theHarvester -d hackbysecurity.com -s -b google -n -f ./report`

    `--dns-lookup`: check to which IP address corresponds the domain name and will look at the range of IP addresses if there are subdomains.
Direct and inverse domain resolutions


- **Dmitry**: perform whois, subdomaind and mails search.

    `dmitry -w hackbysecurity.com -n -s -e`

## Port and Vulnerability Scanning
- **Wireshark**: packet sniffer to visualize traffic in our network interface. Passive method
    - eth0: local wired network
        - ARP packages
        - DHCPv6 packages: with TCP/IP v6
- **netdiscover**: send ARP packages actively or passively
    
    `netdiscover -i eth0 -r 10.0.2/24`

- Check ARP tables from my machine `arp -a`. 
- ARP requests: only works at local network level
- ICMP requests: for exernal petitions, sent 1 package to a sequential network range 
    
    `for i in $(seq 3 254); do ping -c 1 10.0.2.$i; done`
- DNS requests: 
    1. Configure DNS servers include our target dns and ip address 
        
        `nano /etc/resolv.conf`
    2. **wfuzz**: scan, choose a dictionary to use, ignore empty responses  
        `wfuzz -Z -c -w /usr/share/wordlists/wfuzz/general/common.txt --hh=0 FUZZ.hackbysecurity.com`
        - dictionary [secList](https://github.com/danielmiessler/SecLists/tree/master/Discovery/DNS): common subdomains we can use
- NetBIOS requests
    
    `nbtscan -r 10.0.2.0/24`