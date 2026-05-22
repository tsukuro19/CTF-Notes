---
platform: picoCTF
category: General Skill
difficulty: Easy
points:
date: 2024-03-25
tags:
  - ctf
  - general_skill
solved: true
---
# Challenge Name: Undo
## Challenge Description
I have built my own Git server with my own rules!
## Initial Analysis
- In this challenge, we will clone a Git repository to find the flag in README.MD.
- When searching in README.md, we will realize that to get the flag, we need to use the username 'root' and the email address 'root@picoctf'.
- Based on that, we just need to adjust the username and email address to match the requirements to impersonate root and then push the file to get the flag.

## Solution

### Step 1: Clone the repository
- The challenge provided the following instructions:
![](../../../05-Assets/Pasted%20image%2020260522204414.png)
- We clone the repository:
```bash
git clone ssh://git@foggy-cliff.picoctf.net:62275/git/challenge.git
```
- And use the password as instructed above.
### Step 2: Check the README
- Navigating into the cloned repository:
```bash
cd challenge
ls
```
- Output:
![](../../../05-Assets/Pasted%20image%2020260522205042.png)
- Reading the README.md:
```bash
cat README.md
```
- Result:
![](../../../05-Assets/Pasted%20image%2020260522205203.png)
- As shown in the README file, we need to push the flag.txt file as username root and the email address root@picoctf.
### Step 3: Configure Git Identity
- Set the Git username and email to match the required identity:
```bash
git config user.name "root"  
git config user.email "root@picoctf"
```
- Verify configuration:
```bash
git config user.name outputs root  
git config user.email outputs root@picoctf
```
### Step 4: Create, Add, and Commit flag.txt
- We create a placeholder file:
```bash
echo "requesting flag" > flag.txt
```
- Add and commit the file:
```bash
git add flag.txt  
git commit -m "push flag as root"
```
![](../../../05-Assets/Pasted%20image%2020260522205730.png)
### Step 5: 
- Check the current branch:
```bash
git branch
```
- Output:
![](../../../05-Assets/Pasted%20image%2020260522210011.png)
- The default branch was master, not main, so we pushed to master:
```bash
git push origin master
```
- Output
![](../../../05-Assets/Pasted%20image%2020260522210159.png)

## Flag

`picoCTF{REDACTED}`

## Tools Used
- In this lesson, we will mostly learn about Git operations when cloning to a repository, as well as some basic errors that can occur when we impersonate a user in a repository.
- Can read more about git commands here:[Git-Commands-Penetration-Tester](../../../02-Techniques/Git-Commands-Penetration-Tester.md)

## Takeaway
- In this lesson, we learned that sometimes Git checks user.name and user.email locally. If the repository doesn't authenticate this, it becomes a vulnerable target for an attack.
- Always verify the main and master branches before pushing.
