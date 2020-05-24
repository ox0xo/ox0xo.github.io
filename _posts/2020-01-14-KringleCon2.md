---
layout: post
title:  "KringleCon2 writeup"
date:   2020-01-14
permalink: /ctf/:title
categories: CTF
tags: SANS KringleCon DeepBlueCLI Splunk Zeek RITA TensorFlow HTML5 CSS3
excerpt: This is Writeup for SANS Holiday Hack Challenge 2019. Unfortunately, I did not get all the correct answers, but I have listed the nine answers I have solved. I will continue when my college exam is over.
mathjax: false
---

* content
{:toc}

# Introduction

This is a writeup for SANS Holiday Hack Challenge 2019 - KringleCon 2 -.
The contest is set at Elf University where Santa Claus and his friends gather. This is a sequel to KringleCon held last year. The contest includes 12 questions and multiple exercises.
Unfortunately I could not solve all the problems, but I will write about the 9 questions that I could solve.

# Writeup

## 0. Talk to Santa in the Quad
```
Enter the campus quad and talk to Santa.
```

This is the welcome quiz. Talk to Santa in the Quad to clear it.

![](/images/2020-01-14-KringleCon2/2020-01-13-22-57-40.png)

## 1. Find the Turtle Doves
```
Find the missing turtle doves.
```

Santa asked I to look for two missing turtle dove. This objective becomes obvious when all of the objective 2-5 are solved.

## 2. Unredact Threatening Document
```
Someone sent a threatening letter to Elf University. What is the first word in ALL CAPS in the subject line of the letter? Please find the letter in the Quad.
```

A letter is falling in the northwest corner of the Quad.

![](/images/2020-01-14-KringleCon2/2020-01-13-23-05-25.png)

This appears to be a letter of intimidation addressed to Elf University, but most of the text is hidden and details are unknown. Let's look at this letter a little more. This letter is a PDF but looks like it was created in MS Word. Character information is stored and I can select the text. And the hidden part seems to have been created by Auto Shape. Therefor, it is considered that the characters hidden in Auto Shape can be copied by using the procedure of [Ctrl + A] > [Ctrl + C].

![](/images/2020-01-14-KringleCon2/2019-12-22-04-57-53.png)

The copied information is as follows. So the answer to this objective is **DEMAND**.

```
To the Administration, Faculty, and Staff of Elf University
17 Christmas Tree Lane
North Pole

From: A Concerned and Aggrieved Character

Subject: DEMAND: Spread Holiday Cheer to Other Holidays and Mythical Characters… OR ELSE!

Attention All Elf University Personnel,

It remains a constant source of frustration that Elf University and the entire operation at the North Pole focuses exclusively on "Mr. S. Claus" and his year-end holiday spree. We URGE you to consider lending your considerable resources and expertise in providing merriment, cheer, toys, candy, and much more to other holidays year-round, as well as to other mythical characters.

For centuries, we have expressed our frustration at your lack of willingness to spread your cheer beyond the inaptly-called “Holiday Season.” There are many other perfectly fine holidays and mythical characters that need your direct support year-round.

If you do not accede to our demands, we will be forced to take matters into our own hands.
We do not make this threat lightly. You have less than six months to act demonstrably.

Sincerely,

--A Concerned and Aggrieved Character
```

## 3. Windows Log Analysis: Evaluate Attack Outcome
```
We're seeing attacks against the Elf U domain! Using the event log data, identify the user account that the attacker compromised using a password spray attack. Bushy Evergreen is hanging out in the train station and may be able to help you out.
```

I need to examine the windows event log to determine which account was compromised by the attacker who invaded Elf University. To get hint for this objective from Bushy Evergreen, let's start by solving his problem.

![](/images/2020-01-14-KringleCon2/2020-01-13-23-23-55.png)

- Bushy Evergreen
  ```
  Hi, I'm Bushy Evergreen. Welcome to Elf U!
  I'm glad you're here. I'm the target of a terrible trick.
  Pepper Minstix is at it again, sticking me in a text editor.
  Pepper is forcing me to learn ed.
  Even the hint is ugly. Why can't I just use Gedit?
  Please help me just quit the grinchy thing.
  ```

![](/images/2020-01-14-KringleCon2/2019-12-22-03-38-52.png)

Bushy is using Gedit, a text editor. I read the manual because I never used that. To explain to the vi user, the key to enter command mode is [P] and the command to exit the gedit is [* q]. In other words, I can quit gedit by doing the following.

![](/images/2020-01-14-KringleCon2/2019-12-22-03-46-36.png)

References: https://www.gnu.org/software/ed/manual/ed_manual.html

- Bushy Evergreen
  ```
  Wow, that was much easier than I'd thought.
  Maybe I don't need a clunky GUI after all!
  Have you taken a look at the password spray attack artifacts?
  I'll bet that DeepBlueCLI tool is helpful.
  You can check it out on GitHub.
  It was written by that Eric Conrad.
  He lives in Maine - not too far from here!
  ```

Now, let's return to the subject.
Install the DeepBlueCLI tool with the help of Bushy. It is a powerful hunting tool for analyzing Windows event logs, and I can detect the presence of threats without knowing the details of the event logs. According to the analysis of the event log at Elf University, logon failures are recorded for a large number of user accounts, but pminstix and supatree have successfully logged on. **supatree** is included in the list of users eligible for Password Spray, so I can determine that it was the first account compromised by Password Spray.

![](/images/2020-01-14-KringleCon2/2019-12-22-04-10-19.png)

References：https://github.com/sans-blue-team/DeepBlueCLI

## 4. Windows Log Analysis: Determine Attacker Technique
```
Using these normalized Sysmon logs, identify the tool the attacker used to retrieve domain password hashes from the lsass.exe process. For hints on achieving this objective, please visit Hermey Hall and talk with SugarPlum Mary.
```

Next, the attacker appears to have obtained a password hash on the compromised system. Search the tool used at this time from the Sysmon log. I went to see SugarPlum Mary to hear tips for this objective.

![](/images/2020-01-14-KringleCon2/2020-01-13-23-46-20.png)

- SugarPlum Mary
  ```
  Oh me oh my - I need some help!
  I need to review some files in my Linux terminal, but I can't get a file listing.
  I know the command is ls, but it's really acting up.
  Do you think you could help me out? As you work on this, think about these questions:

  1. Do the words in green have special significance?
  2. How can I find a file with a specific name?
  3. What happens if there are multiple executables with the same name in my $PATH?
  ...

  Oh me oh my - I need some help!
  ```

His terminal has been compromised and appears to refer to fake ls binaries by alias. Finding the correct binary path and running it directly will clear the problem.

![](/images/2020-01-14-KringleCon2/2019-12-22-07-29-49.png)

![](/images/2020-01-14-KringleCon2/2019-12-22-07-33-44.png)

- SugarPlum Mary
  ```
  Oh there they are! Now I can delete them. Thanks!
  Have you tried the Sysmon and EQL challenge?
  If you aren't familiar with Sysmon, Carlos Perez has some great info about it.
  Haven't heard of the Event Query Language?
  Check out some of Ross Wolf's work on EQL or that blog post by Josh Wright in your badge.
  ```

Return to the subject.
I installed EQL according to SugarPlum's tips, but I am not familiar with Sysmon logs. What should I start with? The goal this time was to look for traces dumping the password hash from lsass.exe. It seems good to start with lsass.exe. When grep, lsass.exe is found in parent_process_name, so I identified pid=3440 of the event. Next I searched ppid=3440 to find the child process called from pid=3440. There are no child processes beyond this event. Therefore, process_name=**ntdsutil.exe** is the tool used to dump password hashes.

```
grep "lsass.exe" sysmon-data.json 
        "parent_process_name": "lsass.exe",
        "parent_process_path": "C:\\Windows\\System32\\lsass.exe",
```
```
eql query -f sysmon-data.json "process where parent_process_name = 'lsass.exe'" | jq
{
  "command_line": "C:\\Windows\\system32\\cmd.exe",
  "event_type": "process",
  "logon_id": 999,
  "parent_process_name": "lsass.exe",
  "parent_process_path": "C:\\Windows\\System32\\lsass.exe",
  "pid": 3440,
  "ppid": 632,
}
```
```
eql query -f sysmon-data.json "process where ppid = 3440" | jq
{
  "command_line": "ntdsutil.exe  \"ac i ntds\" ifm \"create full c:\\hive\" q q",
  "event_type": "process",
  "logon_id": 999,
  "parent_process_name": "cmd.exe",
  "parent_process_path": "C:\\Windows\\System32\\cmd.exe",
  "pid": 3556,
  "ppid": 3440,
  "process_name": "ntdsutil.exe",
}
```

References：https://pen-testing.sans.org/blog/2019/12/10/eql-threat-hunting/

## 5. Network Log Analysis: Determine Compromised System

```
The attacks don't stop! Can you help identify the IP address of the malware-infected system using these Zeek logs? For hints on achieving this objective, please visit the Laboratory and talk with Sparkle Redberry.
```

I examine network logs obtained by Zeek to identify devices infected with malware at Elf University. Check the given log set as it also contains RITA statistics. RITA is a tool for visualizing Zeek logs. Infected hosts with malware often send beacons to the C2 server, so they look for devices that have more network connections and longer connection times than other clients.

![](/images/2020-01-14-KringleCon2/2020-01-13-04-08-27.png)

![](/images/2020-01-14-KringleCon2/2020-01-13-04-08-51.png)

As a result, I can see terminals that record abnormal values in both Beacons and Long Connections. The IP of the infected host with malware is **192.168.134.130**.

References：https://www.activecountermeasures.com/free-tools/rita/

## 6. Splunk

```
Access https://splunk.elfu.org/ as elf with password elfsocks. What was the message for Kent that the adversary embedded in this attack? The SOC folks at that link will help you along! For hints on achieving this objective, please visit the Laboratory in Hermey Hall and talk with Prof. Banas.
```

According to Kent, the attackers seem to have compromised the Professor Banas computer. Use Splunk to examine the Professor's computer logs to determine the behavior of the attacker and the messages he sent to Kent.

1. What is the short host name of Professor Banas' computer?
　
Since I don't know in which field the username is stored, tried the simplest search query. However,  I should excluded logs for sourcetype=stoq because that hinder the search. The final query used is `sourcetype! = Stoq | search * banas *`. The computer name of Professor Banas is **sweetums**.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-00-37-30.png)

2. What is the name of the sensitive file that was likely accessed and copied by the attacker? Please provide the fully qualified location of the file. (Example: C:\temp\report.pdf)
　
There are few clues, so I will search for "keywords of interest" widely according to the hints. But I can not found the keywords such as pdf, doc, xls. Recalling Alice's story, Professor Banas had a deep relationship with Santa. If I were an attacker, I would try to steal information related to Santa from Professor Banas' computer. Therefore, I searching for the keyword `santa` then I found a message about Santa. The text file attached there is the correct answer.
**C:\Users\cbanas\Documents\Naughty_and_Nice_2019_draft.txt**
　
![](/images/2020-01-14-KringleCon2/2020-01-13-00-26-49.png)

3. What is the fully-qualified domain name(FQDN) of the command and control(C2) server? (Example: badguy.baddies.com)
　
This question uses the query that Alice told me. The FQDN of the C2 server is **144.202.46.214.vultr.com**.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-00-49-49.png)

4. What document is involved with launching the malicious PowerShell code? Please provide just the filename. (Example: results.txt)		
　
The suspicious Powershell started on 2019-08-25T09:18:35, so I should check the log from just before that. So, I found the record of process ID 3552 and 6268 on 2019-08-25T17:18:33.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-01-35-34.png)
　
The process ID is recorded in Sysmon in hexadecimal, so I converted to 0x187c and 0xde0 in advance. By searching for the process ID, I can find out the name of the file that called powershell. The file name is **19th Century Holiday Cheer Assignment.docm**.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-01-39-33.png)

5. How many unique email addresses were used to send Holiday Cheer essays to Professor Banas? Please provide the numeric value. (Example: 1)
　
Use an aggregate function for SMTP logs extracted by stoq, group email addresses and count the number. Except for Professor Banas, the email address is **21**.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-02-16-25.png)

6. What was the password for the zip archive that contained the suspicious file?
　
ATT&CK's T1193 is spearfishing.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-02-20-03.png)
　
I have remember the information looked up so far. The malware was triggered by the `19th Century Holiday Cheer Assignment.docm`. If I research the email with that document attached, maybe I will find the password there. 
　
![](/images/2020-01-14-KringleCon2/2020-01-13-02-27-15.png)
　
Bingo! The password was included in the email sent from Bradly Buttercups. That is **123456789**.

7. What email address did the suspicious file come from?
　
Looking at the log I checked earlier, the sender was another person impersonating Bradly Buttercups. The attacker's email address is **bradly.buttercups@eifu.org**.

8. What was the message for Kent that the adversary embedded in this attack?
　
The contents of maldoc are displayed by the query given by Alice.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-03-45-30.png)
　
The docm file is a archive of the xml file, so if I look at one of these xml, I can analyze the properties of the file. The base files for Office files are core.xml and app.xml. When I downloaded each from S3 and checked the contents, the message was included in core.xml.
　
![](/images/2020-01-14-KringleCon2/2020-01-13-03-43-31.png)
　
The message that the attacker sent to Kent is **Kent you are so unfair. And we were going to make you the king of the Winter Carnival.**.

Reference: https://www.splunk.com/pdfs/solution-guides/splunk-quick-reference-guide.pdf

## 7. Get Access To The Steam Tunnels

```
Gain access to the steam tunnels. Who took the turtle doves? Please tell us their first and last name. For hints on achieving this objective, please visit Minty's dorm room and talk with Minty Candy Cane.
```

Minty CandyCane knows a hint of this objective, but he is in a locked Dormitry. First I should unlock the door by operating the Frost Keypad.
I get tips from unlocking the Frosty Keypad from Tangle Calbox.

- Tangle Coalbox
  ```
  Hey kid, it's me, Tangle Coalbox.
  I'm sleuthing again, and I could use your help.
  Ya see, this here number lock's been popped by someone.
  I think I know who, but it'd sure be great if you could open this up for me.
  I've got a few clues for you.

  1. One digit is repeated once.
  2. The code is a prime number.
  3. You can probably tell by looking at the keypad which buttons are used.
  ...

  Hey kid, it's me, Tangle Coalbox.
  ```

![](/images/2020-01-14-KringleCon2/2019-12-22-18-09-24.png)

The four-digit prime numbers that can be created from the three types of keys 1, 3, and 7 are the correct answer, I wrote a script that enumerates the corresponding prime numbers. I can find more than one candidate, but this door has no lockout mechanism so I can try it as many times as I like. The correct answer is **7331**.

Here is the script I wrote.
```
import itertools
import math

def isPrime(n):
    for i in range(2, int(math.sqrt(n)) + 1):
        if n % i == 0:
            return False
    return True

def solv():
    r = []
    d = [1, 3, 7]
    for d2 in d:
        d.extend([d2])
        for n in itertools.permutations(d):
            n = n[0] * 1000 + n[1] * 100 + n[2] * 10 + n[3]
            if isPrime(n):
                r.append(n)
        d = list(set(d))
    print(list(set(r)))

if __name__ == "__main__":
    solv()
```

- Minty Candycane
  ```
  Hi! I'm Minty Candycane!
  I just LOVE this old game!
  I found it on a 5 1/4" floppy in the attic.
  You should give it a go!
  If you get stuck at all, check out this year's talks.
  One is about web application penetration testing.
  Good luck, and don't get dysentery!
  ```

I found Minty CandyCane in the passage on the right side of the dormitory. He seems to be distracted by his favorite games. I will play with this game a little.

![](/images/2020-01-14-KringleCon2/2019-12-22-22-10-10.png)

![](/images/2020-01-14-KringleCon2/2020-01-05-00-42-09.png)

It's like a game where I reach the goal using limited resources such as days, food and physical strength. EASY passes parameters in URL queries, so it can be easily tampered with. I send `&distance=8000` then the game is clear.
Next, MEDIUM has been improved to use POST, but there is no problem I send POST data with the distance of 8000 as before. The only difference is that it is sent as POST request data.
Last, HARD has a hash for falsification detection and cannot be cleared by simply falsifying the distance. Falsify various parameters to see how they affect the hash. Trail member names can be falsified but have no meaning. A little worried, I gave up clearing HARD.

Minty Candycane gives a hint about objective 7 when I clear up to MEDIUM. Oh? I didn't need to clear the hardware.

- Minty Candycane
```
You made it - congrats!
Have you played with the key grinder in my room? Check it out!
It turns out: if you have a good image of a key, you can physically copy it.
Maybe you'll see someone hopping around with a key here on campus.
Sometimes you can find it in the Network tab of the browser console.
Deviant has a great talk on it at this year's Con.
He even has a collection of key bitting templates for common vendors like Kwikset, Schlage, and Yale.
```

I enter the room next to Minty CandyCane and I see man moving to the next room. I immediately chased, but unfortunately he seems to have go away behind a locked door. Fortunately, I know the shape of the key hanging on his waist. Duplicate his key, following the tips of Deviant Ollam.

I can judge that it is Schlage from the shape of the key, so measure the height of the key pile along the guide. The combination is 1-2-2-5-2-0.

![](/images/2020-01-14-KringleCon2/2020-01-12-04-16-32.png)

![](/images/2020-01-14-KringleCon2/2020-01-12-04-17-23.png)

I passing through a locked door and past the Steam Tunnel, I discovered **Krampus Hollyfeld**. He stole the turtle doves.

##  8. Bypassing the Frido Sleigh CAPTEHA
```
Help Krampus beat the Frido Sleigh contest. For hints on achieving this objective, please talk with Alabaster Snowball in the Speaker Unpreparedness Room.
```

- Krampus Hollyfeld
  ```
  Hello there! I’m Krampus Hollyfeld.
  I maintain the steam tunnels underneath Elf U, Keeping all the elves warm and jolly.
  Though I spend my time in the tunnels and smoke, In this whole wide world, there's no happier bloke!
  Yes, I borrowed Santa’s turtle doves for just a bit.
  Someone left some scraps of paper near that fireplace, which is a big fire hazard.
  I sent the turtle doves to fetch the paper scraps.
  But, before I can tell you more, I need to know that I can trust you.
  Tell you what – if you can help me beat the Frido Sleigh contest (Objective 8), then I'll know I can trust you.

  The contest is here on my screen and at fridosleigh.com.
  No purchase necessary, enter as often as you want, so I am!
  They set up the rules, and lately, I have come to realize that I have certain materialistic, cookie needs.
  Unfortunately, it's restricted to elves only, and I can't bypass the CAPTEHA.
  (That's Completely Automated Public Turing test to tell Elves and Humans Apart.)
  I've already cataloged 12,000 images and decoded the API interface.
  Can you help me bypass the CAPTEHA and submit lots of entries?
  ```

The objective 8 was given by Krampus. I need to bypass the image recognition system to win his trust and uncover the truth of the case. Alabaster Snowball knows hints for this objective.

- Alabaster Snowball
  ```
  Welcome to the Speaker UNpreparedness Room!
  My name's Alabaster Snowball and I could use a hand.
  I'm trying to log into this terminal, but something's gone horribly wrong.
  Every time I try to log in, I get accosted with ... a hatted cat and a toaster pastry?
  I thought my shell was Bash, not flying feline.
  When I try to overwrite it with something else, I get permission errors.
  Have you heard any chatter about immutable files? And what is sudo -l telling me?
  ```

![](/images/2020-01-14-KringleCon2/2019-12-22-07-37-31.png)

![](/images/2020-01-14-KringleCon2/2019-12-22-07-39-19.png)

This cute cat seems to be called by /bin/nsh. I want to overwrite it with /bin/bash but it is rejected despite having access permission. Why?

![](/images/2020-01-14-KringleCon2/2019-12-22-15-04-40.png)

When I tried sudo -l according to the hint, I found the execution permission of the chattr command. I learned about the chattr command for the first time, so I looked up it in man command. This is a command for assigning any attributes (e.g. immutable ), and the result can be checked with lsattr command.

![](/images/2020-01-14-KringleCon2/2019-12-22-14-46-15.png)

According to lsattr, it turns out that /bin/nsh is immutable. Therefor, rewriting the immutable attribute of /bin/nsh and overwriting it with /bin/bash can solve his problem.

![](/images/2020-01-14-KringleCon2/2019-12-22-15-07-55.png)

- Alabaster Snowball
  ```
  Who would do such a thing?? Well, it IS a good looking cat.
  Have you heard about the Frido Sleigh contest?
  There are some serious prizes up for grabs.
  The content is strictly for elves. Only elves can pass the CAPTEHA challenge required to enter.
  I heard there was a talk at KCII about using machine learning to defeat challenges like this.
  I don't think anything could ever beat an elf though!
  ```

This service introduces CAPTEHA (CAPTCHA?) that identifies only elves, and I need to select the specified image from 100 images. First, to clarify the capture protocol, I perform some operations on the browser and capture the communication with the local proxy.

![](/images/2020-01-14-KringleCon2/2020-01-12-02-41-53.png)

The identified protocols are as follows.

1. Send an empty request to https://fridosleigh.com/api/capteha/request
2. The session cookie is set in the response, returning 100 base64 encoded images, uuid, and the category of image to select
3. Submit the uuid of the selected image to https://fridosleigh.com/api/capteha/submit
4. CAPTEHA result is returned

![](/images/2020-01-14-KringleCon2/2020-01-12-02-39-09.png)

Since the CAPTCHA has only 5 seconds, there is no time to observe slowly. I thought about making a correspondence table between image data and uuid in advance, but the same data rarely appears because the image data is selected from a huge number of samples. I need to use machine learning to automatically determine the image.

Following the tips, I downloaded the machine learning demo script from the chrisjd20 github repository. After confirming that the image can be identified as in the tutorial, customize the details for CAPTEHA. Train time-consuming machine learning models first. The processing required to break through CAPTCHA is as follows.

1. Acquire a set of image data by sending a request to the server
2. Label image data using machine learning model
3. Select the specified image based on the label and send the uuid to the server

Once I break through the CAPTEHA, I can send requests as many times as I want using that session. If the e-mail address included in the request is chose by lot, a correct flag send me. The request sending function has also been incorporated into the script to increase the probability of chosing. After about 100 entries, I was selected for the lottery.

![](/images/2020-01-14-KringleCon2/2020-01-12-02-36-17.png)

The following methods have been added to the chrisjd20 script to clear this challenge.

```
def for_elf_main():
    import requests
    import json
    import base64

    elf_url = "https://fridosleigh.com"
    elf_api_req = "/api/capteha/request"
    elf_api_sub = "/api/capteha/submit"
    elf_api_entry = "/api/entry"

    # Loading the Trained Machine Learning Model created from running retrain.py on the training_images directory
    graph = load_graph('/tmp/retrain_tmp/output_graph.pb')
    labels = load_labels("/tmp/retrain_tmp/output_labels.txt")

    # Load up our session
    input_operation = graph.get_operation_by_name("import/Placeholder")
    output_operation = graph.get_operation_by_name("import/final_result")
    sess = tf.compat.v1.Session(graph=graph)
    q = queue.Queue()

    # Get CAPTCHA resources
    session = requests.Session()
    res = session.get(elf_url + elf_api_req)
    elf_json = json.loads(res.text)
    selects = list(map(lambda x: x.replace("and ", ""), elf_json["select_type"].split(", ")))

    # Predict images
    for image in elf_json["images"]:
        while len(threading.enumerate()) > 10:
            time.sleep(0.0001)
        args = (
                q,
                sess,
                graph,
                base64.b64decode(image["base64"]),
                image["uuid"],
                labels,
                input_operation,
                output_operation
                )
        threading.Thread(target=predict_image, args=args).start()
    while q.qsize() < len(elf_json["images"]):
        time.sleep(0.001)

    # Pass authentication
    post_data = []
    prediction_results = [q.get() for x in range(q.qsize())]
    for prediction in prediction_results:
        if prediction["prediction"] in selects:
            post_data.append(prediction["img_full_path"])
    post_header = {
            "Origin": "https://fridosleigh.com",
            "X-Requested-With": "XMLHttpRequest",
            "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8",
            "Sec-Fetch-Site": "same-origin",
            "Sec-Fetch-Mode": "cors",
            "Referer": "https://fridosleigh.com/"
            }
    post_data = "answer=%s" % "%2C".join(post_data)
    res = session.post(elf_url + elf_api_sub, data=post_data, headers=post_header)

    # Entry
    if json.loads(res.text)["request"]:
        post_data  = "favorites=cupidcrunch%2Csugarcookiesantas%2Cdosidancers%2Cprancerspeanutbutterpatties%2Ccanesahoy%2Csnoweos%2Cfignorthtons%2Cthickmints&"
        post_data += "age=181&"
        post_data += "about=dummy&"
        post_data += "email=my-email-address%40gmail.com&"
        post_data += "name=luckyelf123"
        while True:
            res = session.post(elf_url + elf_api_entry, data=post_data, headers=post_header)
            print(res.text)
            time.sleep(1)
```

## 11. Open the Sleigh Shop Door
```
Visit Shinny Upatree in the Student Union and help solve their problem. What is written on the paper you retrieve for Shinny?
For hints on achieving this objective, please visit the Student Union and talk with Kent Tinseltooth.
```

The locked doors of the Sleigh Shop seem to hide important information about the incident. I went to Kent for a hint to unlock the door.

- Kent Tinseltooth
  ```
  OK, this is starting to freak me out!
  Oh sorry, I'm Kent Tinseltooth. My Smart Braces are acting up.
  Do... Do you ever get the feeling you can hear things? Like, voices?
  I know, I sound crazy, but ever since I got these... Oh!
  Do you think you could take a look at my Smart Braces terminal?
  I'll bet you can keep other students out of my head, so to speak.
  It might just take a bit of Iptables work.
  ```

![](/images/2020-01-14-KringleCon2/2019-12-22-06-22-18.png)

![](/images/2020-01-14-KringleCon2/2019-12-22-06-27-48.png)

His request is clear. I met the requirements by referring to the iptables manual. I referred to is in Japanese, but there are many other excellent manuals.

```
# 1. Set the default policies to DROP for the INPUT, FORWARD, and OUTPUT chains.
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT DROP

# 2. Create a rule to ACCEPT all connections that are ESTABLISHED,RELATED on the INPUT and the OUTPUT chains.
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 3. Create a rule to ACCEPT only remote source IP address 172.19.0.225 to access the local SSH server (on port 22).
sudo iptables -A INPUT -p tcp --dport 22 -s 172.19.0.225 -j ACCEPT

# 4. Create a rule to ACCEPT any source IP to the local TCP services on ports 21 and 80.
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# 5. Create a rule to ACCEPT all OUTPUT traffic with a destination TCP port of 80.
sudo iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT

# 6. Create a rule applied to the INPUT chain to ACCEPT all traffic from the lo interface.
sudo iptables -A INPUT -i lo -p all -j ACCEPT
```

![](/images/2020-01-14-KringleCon2/2019-12-22-07-04-19.png)

References: http://web.mit.edu/rhel-doc/4/RH-DOCS/rhel-sg-ja-4/s1-firewall-ipt-basic.html

- Kent Tinseltooth
  ```
  Oh thank you! It's so nice to be back in my own head again. Er, alone.
  By the way, have you tried to get into the crate in the Student Union? It has an interesting set of locks.
  There are funny rhymes, references to perspective, and odd mentions of eggs!
  And if you think the stuff in your browser looks strange, you should see the page source...
  Special tools? No, I don't think you'll need any extra tooling for those locks.
  BUT - I'm pretty sure you'll need to use Chrome's developer tools for that one.
  Or sorry, you're a Firefox fan?
  Yeah, Safari's fine too - I just have an ineffible hunger for a physical Esc key.
  Edge? That's cool. Hm? No no, I was thinking of an unrelated thing.
  Curl fan? Right on! Just remember: the Windows one doesn't like double quotes.
  Old school, huh? Oh sure - I've got what you need right here...
  ```

- Shinny Upatree
  ```
  Psst - hey!
  I'm Shinny Upatree, and I know what's going on!
  Yeah, that's right - guarding the sleigh shop has made me privvy to some serious, high-level intel.
  In fact, I know WHO is causing all the trouble.
  Cindy? Oh no no, not that who. And stop guessing - you'll never figure it out.
  The only way you could would be if you could break into my crate, here.
  You see, I've written the villain's name down on a piece of paper and hidden it away securely!
  ```

I decided to use Google Chrome as per the tip I got from Kent. Use Google Chrome's web developer tools to unlock the 10 keys on the crate.

1. Open the console tab and get the unlock code.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-16-01.png)

2. Open Print Preview with Ctrl+P to get unlock code.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-17-41.png)

3. Find the unlock code by looking for the network tab.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-19-57.png)

4. Find the local storage from the application tab and find the unlock code.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-21-27.png)

5. I can find the unlock code by searching for the title tag of the HTML source from the element tab.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-23-17.png)

6. Right-click the holographic card and click Verify. I can check the CSS of the .hologram class by checking the style tab on the right side of the element tab. If I increase the value of perspective, you can find the unlock code.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-26-25.png)

7. I check the CSS of the description written in cursive in the same way as before, then I can find the unlock code from the list of font-family.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-28-38.png)

8. Now look at the eggs element drawn in green. Nothing is found in CSS, but suspicious events are registered in Event Listeners. I can find the unlock code in the spoil handler.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-29-25.png)

9. I read the HTML, so I will see that the description is split by several span tags. Change all of them to: active status to find the unlock code.
![](/images/2020-01-14-KringleCon2/2020-01-12-19-32-26.png)

10. First, move the "cover" class div tag out of the "lock c10" class div tag. Enter the unlock code printed on the board, but it will not unlock. Search for the keyword "macaroni" using the error code and put it in the div tag of the "lock c10" class. Since components are still missing, I will collect components in the same way. The final required components were three.

![](/images/2020-01-14-KringleCon2/2020-01-12-19-38-15.png)
![](/images/2020-01-14-KringleCon2/2020-01-12-19-39-08.png)
![](/images/2020-01-14-KringleCon2/2020-01-12-19-40-15.png)
![](/images/2020-01-14-KringleCon2/2020-01-12-19-41-05.png)
![](/images/2020-01-14-KringleCon2/2020-01-12-19-44-33.png)

Opening all keys will reveal the criminal's name. He is **The Tooth Fairy**.

![](/images/2020-01-14-KringleCon2/2020-01-12-19-43-20.png)
