# THM - Brooklyn Nine-Nine

*Disclaimer - I AM NOT AN EXPERT. Not even close. If you see something in this writeup that you think was dumb, or the wrong way to do something, it probably was.I'm pretty new to all of this and this is more of a reference for me to look back on.*

---
## The Room

Brooklyn Nine-Nine is a show that is on rotation in my house regularly. So when I stumbled on a room themed around the show on THM, I had to give it a shot. The room is labeled as easy, and it was. All we are looking for is a user flag, and the root flag. 

## Enumeration

With my Kali box and an openvpn tunnel up to THM, we need to do some enumeration. Let's start with nmap. This is the syntax that I used for this box and it got me the information that I needed.

```
nmap -sC -sV -oA nmapout -T4 $IP
```

Let's break down the flags I used:

- **-sC** - Script scan. This tells nmap to run a script scan using it's default set of scripts. Be careful with this, as some of these scripts are considered intrusive. Don't run this against a target without permission. More info can be found on nmap's official docs [here](https://nmap.org/book/nse-usage.html) .
- **-sV** - Service/Version Detection. The way that this detection works is still a bit hazy to me, but I do recommend heading back to nmap's official docs to read up more on it [here](https://nmap.org/book/man-version-detection.html)
- **-oA [filename]** - This will output the scan results to a normal text file, a greppable text file, and an XML. I like using this because I can go back and reference the scan later without having to scroll up in my terminal. I also like having an XML file handy because I recently found out that searchsploit can take an XML created from an nmap scan and search for exploits. 
- **-T4** - Setting the timing. 
- **$IP** - When I am doing these boxes I usually set a variable for the IP of the box I am attacking in my terminal. You can do this with `export IP=[IP you are attacking]` and then call it anytime you need it (within that terminal instance) by using `$IP`. 

**Note:** I should probably be scanning all ports here with `-p-`. 

Alright, lets take a look at the results.

![image1](https://user-images.githubusercontent.com/24508513/183257758-f3e357b7-60a5-403d-827f-d4ffa638e470.png)

It looks like we have ports 21, 22, and 80 open. 

Right away, port 21, running ftp jumps out at me. The default scripts have given us some decent information. It looks like we can log in to the ftp server anonymously and that there is a file located there called note-to-jake.txt. Lets take note of that and come back to it.  

It looks like SSH is also open. I'm sure this will come into play later. 

Port 80 is also open and it looks like it's running Apache. Let's see what this server is hosting by navigating to the IP in a web browser.

![image2](https://user-images.githubusercontent.com/24508513/183257770-e184e78b-eb48-4e91-8fe5-5d04f4478999.png)

Interesting. Let's check the page source. 

![image3](https://user-images.githubusercontent.com/24508513/183257771-ac1a88ce-8e98-4079-b419-9ee29926d0a0.png)

Well, I'm no FBI agent, but I would call that a significant clue. It looks like the path we are heading down is steganography. There has to be some data hidden in the image on the site. Let's download the image and see what we can pull out of it. 

*A quick note: I also ran a feroxbuster scan on the site, but the results were unremarkable, so they are not shown.*

## Steganography

The first thing we can try is `strings`. Strings simply shows you printable strings in a file. We can run it against the jpg with:

```
strings [filename]
```

The result was, just as I expected, nothing of note. 

Time to move on to `steghide`. Steghide is a program that you can use to hide, or extract data from certain types of image and audio files. Here's the command I used:

```
steghide --extract -sf [filename]
```

Break-down:

- **--extract** - Extracts data
- **-sf [filename]** - specifies the file that we want to extrac the data from.

And here is the result:

![image4](https://user-images.githubusercontent.com/24508513/183257776-6c9b13ad-e17d-4725-bba4-f58b77061850.png)

Well, damn. We need a passphrase. My first thought is that the passphrase might be in that file that our nmap scan picked up on the ftp server. So, let's take a look at it. 

## FTP

Connecting to the ftp server:

```
ftp $IP
```

**Name:** anonymous

**Password:** Hit "enter"

Now that we are in the ftp server we can use this command to retrieve the file. 

```
get [filename]
```

It will download that file (in this case, note-to-jake.txt) to the directory that your host is currently in. *Sidenote: this is why I create a directory on my desktop for each of these boxes that I attempt and I work in that directory. *

Now we can exit the ftp server with `exit` and take a look at the contents of the note-to-jake. 

![image5](https://user-images.githubusercontent.com/24508513/183257784-49ddec41-2649-4694-b740-b35671c22278.png)

Dead end. Maybe. Jake has a weak password. I put a pin in this when doing this box. As a last resort, I was going to come back and try to brute force ssh since Jake has a weak password, but I wanted to go back to...

## Steganography, Part Deux

We need a way to get that passphrase for the jpg. The first route I went down was John the Ripper. I thought that maybe there was something in the way of zip2john, but for steg files. While researching that I stumbled on some thing else that fit my needs perfectly. `Stegcracker` is a brute-force utility that will try to get passphrase from a steg file. The command I used is :

```
stegcracker [file]
```

Pretty simple. You can add another argument to specify a wordlist. I just went with the default, which is rockyou.txt

The results:

![image6](https://user-images.githubusercontent.com/24508513/183257793-2ef1e469-1db8-4428-a05f-5da0ebbce16d.png)

We have a passphrase. Time to rerun `steghide` and feed it the passphrase. The results will be written to note.txt in our present working directory, which looks like this:

![image7](https://user-images.githubusercontent.com/24508513/183257798-158e091f-2de8-4f48-8148-5486d36f8251.png)
###### Cheddar, you duplicitous bitch...

Alright! We have Holt's password. Lets give it a shot through ssh. 

![image8](https://user-images.githubusercontent.com/24508513/183257805-e3b60182-d838-40c1-8fbf-157f4ebae022.png)

We're in!

Now the only two submissions the tryhackme room is looking for is a user flag and a root flag. Poking around in Holt's account gets us the user flag. Now it's time for...

## Privilege Escalation

To start, I like to just try:

```
sudo -l
```

This command lists the allowed (and forbidden) sudo commands for the invoking user on the current host. Which returns:

![image9](https://user-images.githubusercontent.com/24508513/183257809-d8ae8b2f-fe1a-45cd-b457-ffdfdcf169a8.png)

Interesting find! We can run nano with sudo permissions without a sudo password. Maybe we can break out of nano once we are in. Lets check [GTFOBins](https://gtfobins.github.io/) to see what we can do. Looks like we are in luck:

![image10](https://user-images.githubusercontent.com/24508513/183257813-5f6018b1-ab98-4684-831d-3584125596e5.png)

So it looks like we need to call nano with sudo, hit Cntl+r, then Cntl+x, and type in `reset; sh 1>&0 2>&0` in the 'Command to Execute' prompt. Lets give it a shot. 

![image11](https://user-images.githubusercontent.com/24508513/183257817-ab6cd225-45f3-4d90-b88d-a2a7ce2bdb43.png)

At first, I didn't think that it worked, but if you look closely, you will see an octothorpe (yes, that's what it's called) after the '0' in the prompt in nano. Hit enter a few times and you'll see that you "leave" nano. Test it out with `whoami` and you'll find that you are root. 

![image12](https://user-images.githubusercontent.com/24508513/183257820-3c6a86d8-c18c-4802-bcf9-eabda2c42a3e.png)

We can stabilize the shell with a python one-liner:

```
python3 -c "import pty; pty.spawn('/bin/bash')"
```

And then go flag hunting!
