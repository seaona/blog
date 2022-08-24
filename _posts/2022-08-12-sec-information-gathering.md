# SEC: Information Gathering
## Social Media and Dictionaries
### 1. Custom Word List generator (CeWL)
CeWL spiders a given URL, up to a specified depth, and returns a list of words which can then be used for password crackers. 

`cewl -m 8 -e --email_file ./emails.txt -d 0 -w ./diccio https://twitter.com/hackbysecurity?lang=es`

![cewl](../_media/cewl.gif)

### 2. Common User Passwords Profiler (CUPP)
Generate commmon passwords based on personal information from the target.

`python3 cupp.py -i`

![cewl](../_media/cupp.gif)

Download from source: [cupp](https://github.com/Mebus/cupp.git)