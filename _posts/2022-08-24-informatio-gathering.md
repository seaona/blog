# Information Gathering
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
