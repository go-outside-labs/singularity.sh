Title: Understanding the Shellshock Vulnerability
Date: 2014-10-1 12:21
Category: Vulnerabilities
Tags: Shellshock, Bash, Command_Injection


![cyber](http://i.imgur.com/kjkWTWV.png)

Almost a week ago, a new ([old]) type of [OS command Injection] was reported. The **Shellshock** vulnerability, also known as **[CVE-2014-6271]**, allows attackers to inject their own code into [Bash] using specially crafted **environment variables**, and it was disclosed with the following description:

 Bash supports exporting not just shell variables, but also shell functions to other bash instances, via the process environment to(indirect) child processes. Current bash versions use an environment variable named by the function name, and a function definition starting with “() {” in the variable value to propagate function definitions through the environment. The vulnerability occurs because bash does not stop after processing the function definition; it continues to parse and execute shell commands following the function definition.

 For example, an environment variable setting of
 VAR=() { ignored; }; /bin/id
 will execute /bin/id when the environment is imported into the bash process. (The process is in a slightly undefined state at this point. The PATH variable may not have been set up yet, and bash could crash after executing /bin/id, but the damage has already happened at this point.)

 The fact that an environment variable with an arbitrary name can be used as a carrier for a malicious function definition containing trailing commands makes this vulnerability particularly severe; it enables network-based exploitation.





Even scarier, the [NIST vulnerability database] has rated [this vulnerability “10 out of 10” in terms of severity]. At this point, there are claims that the [Shellshock attacks could already top 1 Billion]. [Shellshock-targeting DDoS attacks and IRC bots were spotted less than 24 hours after news about Shellshock went public last week!] [Honeypots are catching several exploit payloads]. Matthew Prince, from [Cloudflare], said yesterday that they are "[seeing north of 1.5 million Shellshock attacks across the CloudFlare network daily]". In the same day, the [Incapsula] team released several plots showing that their application firewall had deflected over 217,089 exploit attempts on over 4,115 domains although almost 70% were scanners (to attempt to verify the vulnerability), almost 35% where either payload to try to hijack the server or [DDoS] malware.

[Incapsula]:http://www.incapsula.com/blog/shellshock-bash-vulnerability-aftermath.html,
[DDoS]: http://en.wikipedia.org/wiki/Denial-of-service_attack

Pretty nasty stuff, huh?


[Shellshock-targeting DDoS attacks and IRC bots were spotted less than 24 hours after news about Shellshock went public last week!]: http://www.inforisktoday.co.uk/attackers-exploit-shellshock-bug-a-7361
[seeing north of 1.5 million Shellshock attacks across the CloudFlare network daily]: https://twitter.com/eastdakota/status/516457250332741632
[Cloudflare]: https://www.cloudflare.com/
[CVE-2014-6271]: http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-6271
[old]: http://blog.erratasec.com/2014/09/shellshock-is-20-years-old-get-off-my.html
[OS command Injection]: http://cwe.mitre.org/data/definitions/78.html
[CWE (Common Weakness Enumeration)]: http://cwe.mitre.org/index.html
[Bash]: http://www.gnu.org/software/bash/
[NIST vulnerability database]: http://nvd.nist.gov/
[this vulnerability “10 out of 10” in terms of severity]: http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-6271
[Honeypots are catching several exploit payloads]: http://www.alienvault.com/open-threat-exchange/blog/attackers-exploiting-shell-shock-cve-2014-6721-in-the-wild
[Shellshock attacks could already top 1 Billion]: http://www.securityweek.com/shellshock-attacks-could-already-top-1-billion-report?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%3A+Securityweek+%28SecurityWeek+RSS+Feed%29




------------------------------
## Understanding the Bash Shell


To understand this vulnerability, we need to know how Bash handles functions and environment variables.

The [GNU Bourne Again shell (BASH)] is a [Unix shell] and [command language interpreter]. It was released in 1989 by [Brian Fox] for the [GNU Project] as a free software replacement for the [Bourne shell] (which was born back in 1977).

```sh
$ man bash
NAME
 bash - GNU Bourne-Again SHell
SYNOPSIS
 bash [options] [file]
COPYRIGHT
 Bash is Copyright (C) 1989-2011 by the Free Software Foundation, Inc.
DESCRIPTION
 Bash is a sh-compatible command language interpreter that executes commands read from the standard input or from a file. Bash also incorporates useful features from the Korn and C shells (ksh and csh).
(...)
```

 Of course, there are [other command shells out there]. However, Bash is the default shell for most of the Linux systems (and Linux-based systems), including many Debian-based distributions and the Red Hat & Fedora & CentOS combo.


### Functions in Bash

The interesting stuff comes from the fact that Bash is also a scripting language, with the ability to define functions. This is super useful when you are writing scripts. For example, ```hello.sh```:
```sh
#!/bin/bash
function hello {
 echo Hello!
}
hello
```
which can be called as:
```sh
$ chmod a+x hello.sh
$ ./hello.sh
Hello!
```

A function may be compacted into a single line. You just need to choose a name and put a ```()``` after it. Everything inside ```{}``` will belong to the scope of your function.

For example, we can create a function ```bashiscool``` that uses ```echo``` to display message on the standard output:

```sh
$ bashiscool() { echo "Bash is actually Fun"; }
$ bashiscool
Bash is actually Fun
```


### Child Processes and the ```export``` command

We can make things even more interesting. The statement ```bash -c ``` can be used to execute a new instance of Bash, as a subprocess, to run new commands (```-c``` passes a string with a command). The catch is that the child process does not inherit the functions or variables that we defined in the parent:
```sh
$ bash -c bashiscool # spawn nested shell
bash: bashiscool: command not found
```


So before executing a new instance of Bash, we need to export the **environment variables** to the child. That's why we need the ```export``` command. In the example below, the flag ```-f``` means *read key bindings from filename*:
```sh
$ export -f bashiscool
$ bash -c bashiscool # spawn nested shell
Bash is actually Fun
```



In other words, first, the ```export``` command creates a **regular environment variable** containing the function definition. Then, the second shell reads the environment. If it sees a variable that looks like a function, it evaluates this function!


### A Simple Example of an Environment Variable


Let's see how environment variables work examining some *builtin* Bash command. For instance, a very popular one, ```grep```, is used to search for pattern in files (or the standard input).

Running ```grep``` in a file that contains the word 'fun' will return the line where this word is. Running ```grep``` with a flag ```-v``` will return the non-matching lines, *i.e.,* the lines where the word 'fun' does not appear:
```sh
$ echo 'bash can be super fun' > file.txt
$ echo 'bash can be dangerous' >> file.txt
$ cat file.txt
 bash can be super fun
 bash can be dangerous
$ grep fun file.txt
 bash can be super fun
$ grep -v fun file.txt
 bash can be dangerous
```

The ```grep``` command uses an environment variable called **GREP_OPTIONS** to set default options. This variable is usually set to:
```sh
$ echo $GREP_OPTIONS
--color=auto
```

 To update or create a new environment variable, it is not enough to use the Bash syntax ```GREP_OPTIONS='-v'```, but instead we need to call the *builtin* ```export```:

```sh
$ GREP_OPTIONS='-v'
$ grep fun file.txt
 bash can be super fun
$ export GREP_OPTIONS='-v'
$ grep fun file.txt
 bash can be dangerous
```

### The ```env``` command

Another Bash *builtin*, the ```env``` prints the environment variables. But it can also be used to run a single command with an exported variable (or variables) given to that command. In this case, ```env``` starts a new process, then it modifies the environment, and then it calls the command that was provided as an argument (the ```env``` process is replaced by the command process).

In practice, to use ```env``` to run commands, we:

 1. set the environment variable value with env,
 2. spawn a new shell using bash -c,
 3. pass the command/function we want to run (for example, grep fun file.txt).

For example:
```sh
$ env GREP_OPTIONS='-v' | grep fun file.txt # this does not work, we need another shell
bash can be super fun
$ env GREP_OPTIONS='-v' bash -c 'grep fun file.txt' # here we go
bash can be dangerous

```

### Facing the Shellshock Vulnerability



What if we pass some function to the variable definition?
```sh
$ env GREP_OPTIONS='() { :;};' bash -c 'grep fun file.txt'
grep: {: No such file or directory
grep: :;};: No such file or directory
grep: fun: No such file or directory
```
Since the things we added are strange when parsed to the command ```grep```, it won't understand them.

What if we add stuff *after* the function? Things start to get weirder:
```sh
$ env GREP_OPTIONS='-v () { :;}; echo NOOOOOOOOOOOOOOO!' bash -c 'grep fun file.txt'
grep: {: No such file or directory
grep: :;};: No such file or directory
grep: echo: No such file or directory
grep: NOOOOOOOOOOOOOOO!: No such file or directory
grep: fun: No such file or directory
file.txt:bash can be super fun
file.txt:bash can be dangerous

```

Did you notice the confusion? *Both* matches and non-matches were printed! It means that some stuff was parsed well! When in doubt, Bash appears to do *everything*?

Now, what if we just keep the function, taking out the only thing that makes sense, ```-v```?
```sh
$ env GREP_OPTIONS='() { :;}; echo NOOOOOOOOOOOOOOO!' bash -c 'grep fun file.txt'
NOOOOOOOOOOOOOOO!
grep: {: No such file or directory
grep: :: No such file or directory
grep: }: No such file or directory
grep: fun: No such file or directory
```
Did you notice that ```echo NOOOOOOOOOOOOOOO!``` was executed normally? **This is the (first) Shellshock bug!**

This works because when the new shell sees an environment variable beginning with ```()```, it gets the variable name and executes the string following it. This includes running anything after the function, *i.e*, the evaluation does not stop when the end of the function definition is reached!

Remember that ```echo``` is not the only thing we can do. The possibilities are unlimited! For example, we can issue any ```/bin``` command:
```sh
$ env GREP_OPTIONS='() { :;}; /bin/ls' bash -c 'grep fun file.txt'
anaconda certificates file.txt IPython
(...)
```

WOW.

Worse, we actually don't need to use a system environment variable nor even call a real command:
```sh
$ env test='() { :;}; echo STILL NOOOOOOOO!!!!' bash -c :
STILL NOOOOOOOO!!!!
```


In the example above, ```env``` runs a command with an arbitrary variable (test) set to some function (in this case is just a single ```:```, a Bash command defined as doing nothing). The semi-colon signals the end of the function definition. Again, the bug is in the fact that there's nothing stopping the parsing of what is after the semi-colon!


Now it's easy to see if your system is vulnerable, all you need to do is run:
```sh
$ env x='() { :;}; echo The system is vulnerable!' bash -c :
```

That simple.



[GNU Bourne Again shell (BASH)]: http://www.gnu.org/software/bash/
[Unix shell]: http://en.wikipedia.org/wiki/Bash_(Unix_shell)
[Brian Fox]: http://en.wikipedia.org/wiki/Brian_Fox_(computer_programmer)
[GNU Project]: http://www.gnu.org/gnu/thegnuproject.html
[Bourne shell]: http://en.wikipedia.org/wiki/Bourne_shell
[command language interpreter]:http://en.wikipedia.org/wiki/Command-line_interface
[other command shells out there]: http://en.wikipedia.org/wiki/Comparison_of_command_shells


----

## There is more than one!
The Shellshock vulnerability is an example of an [arbitrary code execution] (ACE) vulnerability, which is executed on running programs. An attacker will use an ACE vulnerability to run a program that gives her a simple way of controlling the targeted machine. This is nicely achieved by running a Shell such as Bash.

It is not surprising that right after a patch for [CVE-2014-6271] was released, several new issues were opened:

[arbitrary code execution]: http://en.wikipedia.org/wiki/Arbitrary_code_execution


* [CVE-2014-7169]: Right after the first bug was disclosed, a [tweet] from [Tavis Ormandy] showed a *further parser error* that became the second vulnerability:
```sh
$ env X='() { (a)=>\' bash -c "echo vulnerable"; bash -c "echo Bug CVE-2014-7169 patched"
vulnerable
```

* [CVE-2014-7186] and [CVE-2014-7187]: A little after the second bug, two other bugs were found by [Florian Weimer]. One concerning *out of bound memory read error* in [redir_stack] and the other an *off-by-one error in nested loops*. You can check these vulnerabilities in your system [with this script].

* [CVE 2014-6277] and [CVE 2014-6278]: A couple of days ago, these new bugs were found by [Michal Zalewski].

What do you think, is Shellshock [just a blip]?

[just a blip]: http://blog.erratasec.com/2014/09/the-shockingly-bad-code-of-bash.html
[with this script]: https://github.com/hannob/bashcheck
[Florian Weimer]: http://www.enyo.de/fw/

[Michal Zalewski]:http://lcamtuf.blogspot.de/2014/09/bash-bug-apply-unofficial-patch-now.html

[CVE 2014-6277]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6277

[CVE 2014-6278]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6278
[tweet]: https://twitter.com/taviso/status/514887394294652929

[Tavis Ormandy]: http://taviso.decsystem.org/

[redir_stack]: http://tools.cisco.com/security/center/viewAlert.x?alertId=35860

----
## Suggestions to Protect Your System


Several patches have been released since the Shellshock vulnerabilities were found. Although at this point they [seem to solve most of the problem], below are some recommendations to keep your system safer:

[seem to solve most of the problem]: https://securityblog.redhat.com/2014/09/24/bash-specially-crafted-environment-variables-code-injection-attack/


- Update your system! And keep updating it... Many Linux distributions have released new Bash software versions, so follow the instructions of your distribution. In most of the cases, a simple ```yum update``` or ```apt-get update``` or similar will do it. If you have several servers, the script below can be helpful:
```sh
#!/bin/bash
servers=(
120.120.120.120
10.10.10.10
22.22.22.22
)
for server in ${servers[@]}
do
ssh $server 'yum -y update bash'
done
```




- Update firmware on your router or any other web-enabled devices, as soon as they become available. Remember to only download patches from reputable sites (only HTTPS please!), since scammers will likely try to take advantage of Shellshock reports.


- Keep an eye on all of your accounts for signs of unusual activity. Consider changing important passwords.




- HTTP requests to CGI scripts have been identified as the major attack vector. Disable any scripts that call on the shell (however, it does not fully mitigate the vulnerability). To check if your system is vulnerable, you can use [this online scanner]. Consider [mod_security] if you're not already using it.



- Because the HTTP requests used by Shellshock exploits are quite unique, monitor logs with keywords such as ```grep '() {' access_log```or ```cat access_log |grep "{ :;};"```. Some common places for http logs are: ```cPanel: /usr/local/apache/domlogs/```, ```Debian/Apache: /var/log/apache2/```, or ```CentOS: /var/log/httpd/```.

- [Firewall and network filters] can be set to block requests that contain a signature for the attack, *i.e* ```“() {“```.

- If case of an attack, publish the attacker's information! You can use [awk] and [uniq] (where *print $1* means print the first column) to get her IP, for example:
```sh
$ cat log_file |grep "{ :;};" | awk '{print $1}'|uniq
```



- If you are on a managed hosting subscription, check your company's status. For example: [Acquia], [Heroku], [Mediatemple], and [Rackspace].

- Update your Docker containers and AWS instances.


- If you are running production systems that don't need exported functions at all, take a look at [this wrapper] that refuses to run bash if any environment variable's value starts with a left-parent.


[Firewall and network filters]: https://access.redhat.com/articles/1212303
[this wrapper]: https://github.com/dlitz/bash-shellshock
[this online scanner]: http://milankragujevic.com/projects/shellshock/
[mod_security]: https://access.redhat.com/articles/1212303
[CVE-2014-6278]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6278
[CVE-2014-6277]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6277
[CVE-2014-7187]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7187
[CVE-2014-7186]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7186
[CVE-2014-7169]:https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169
[CVE-2014-6271]:https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
[Acquia]: https://docs.acquia.com/articles/september-2014-gnu-bash-upstream-security-vulnerability
[Rackspace]: https://status.rackspace.com/
[Mediatemple]: http://status.mediatemple.net/
[Heroku]: https://status.heroku.com/incidents/665
[awk]: http://www.grymoire.com/Unix/Awk.html
[uniq]: http://en.wikipedia.org/wiki/Uniq




---

## Further References


#### Reviews

[http://stephane.chazelas.free.fr/](http://stephane.chazelas.free.fr)

[https://securityblog.redhat.com/2014/09/24/bash-specially-crafted-environment-variables-code-injection-attack](https://securityblog.redhat.com/2014/09/24/bash-specially-crafted-environment-variables-code-injection-attack)

[http://lcamtuf.blogspot.co.uk/2014/09/quick-notes-about-bash-bug-its-impact.html](http://lcamtuf.blogspot.co.uk/2014/09/quick-notes-about-bash-bug-its-impact.html)

[http://www.openwall.com/lists/oss-security/2014/09/24/11](http://www.openwall.com/lists/oss-security/2014/09/24/11)

[http://blog.erratasec.com/2014/09/bash-bug-as-big-as-heartbleed.html#.VCNbefmSx8G](http://blog.erratasec.com/2014/09/bash-bug-as-big-as-heartbleed.html#.VCNbefmSx8G)

[http://seclists.org/oss-sec/2014/q3/649](http://seclists.org/oss-sec/2014/q3/649)

[http://www.circl.lu/pub/tr-27/#recommendations](http://www.circl.lu/pub/tr-27/#recommendations)

[http://www.troyhunt.com/2014/09/everything-you-need-to-know-about.html](http://www.troyhunt.com/2014/09/everything-you-need-to-know-about.html)

[http://lcamtuf.blogspot.com/2014/09/quick-notes-about-bash-bug-its-impact.html](http://lcamtuf.blogspot.com/2014/09/quick-notes-about-bash-bug-its-impact.html)

#### Bugs Description

[http://ftp.gnu.org/gnu/bash/bash-4.3-patches/bash43-025](http://ftp.gnu.org/gnu/bash/bash-4.3-patches/bash43-025)

[http://ftp.gnu.org/gnu/bash/bash-4.3-patches/bash43-026](http://ftp.gnu.org/gnu/bash/bash-4.3-patches/bash43-026)

[http://ftp.gnu.org/gnu/bash/bash-4.3-patches/bash43-027](http://ftp.gnu.org/gnu/bash/bash-4.3-patches/bash43-027)

[http://blog.cloudflare.com/inside-shellshock/](http://blog.cloudflare.com/inside-shellshock/)




#### Proof-of-Concept Attacks

[https://github.com/mubix/shellshocker-pocs]:https://github.com/mubix/shellshocker-pocs

[http://research.zscaler.com/2014/09/shellshock-attacks-spotted-in-wild.html](http://research.zscaler.com/2014/09/shellshock-attacks-spotted-in-wild.html)

[http://www.clevcode.org/cve-2014-6271-shellshock/](http://www.clevcode.org/cve-2014-6271-shellshock/)

[https://www.invisiblethreat.ca/2014/09/cve-2014-6271/](https://www.invisiblethreat.ca/2014/09/cve-2014-6271/)

[http://marc.info/?l=qmail&m=141183309314366&w=2](http://marc.info/?l=qmail&m=141183309314366&w=2)

[https://www.dfranke.us/posts/2014-09-27-shell-shock-exploitation-vectors.html](https://www.dfranke.us/posts/2014-09-27-shell-shock-exploitation-vectors.html)

[https://www.trustedsec.com/september-2014/shellshock-dhcp-rce-proof-concept/](https://www.trustedsec.com/september-2014/shellshock-dhcp-rce-proof-concept/)

[https://www.invisiblethreat.ca/2014/09/cve-2014-6271/](https://www.invisiblethreat.ca/2014/09/cve-2014-6271/)

[http://pastebin.com/VyMs3rRd](http://pastebin.com/VyMs3rRd)

[http://infosecnirvana.com/shellshock-hello-honeypot/](http://infosecnirvana.com/shellshock-hello-honeypot/)

[https://isc.sans.edu/forums/diary/Shellshock+A+Collection+of+Exploits+seen+in+the+wild/18725](https://isc.sans.edu/forums/diary/Shellshock+A+Collection+of+Exploits+seen+in+the+wild/18725)

