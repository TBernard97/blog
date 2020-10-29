---
layout: post
title:  "Lazy Scan Script Part 1"
date:   2020-10-28 11:05:18 -0400
categories: Programming
---
## Why am I doing this?
I am lazy. There I said it (probably not something prospecting employers will find desirable). However, I am very curious and like to build things that do mundane crap for me. So I decided to script out large portions of my enumeration process using bash. 


I know that there are other scripts that do this, but like I said I am curious, and also, I have weird quirks in my enumeration process (which we will get to in detail in the next part). 

Basically I like to take a breadth first approach and then improvise on details later. Because of this I will occassionally miss key information so I like to have as many detailed references from as many tools as possible in my working directory. That way my ADD will kick in and force me to look at everything that could possibly be looked at. Essentially, it prevents me from being a *complete* moron.

![House_Moron](/blog/assets/LazyScan/House_Moron.gif)



Yes I know. Thank you Gregory House.

## Initial requirements

There were a few initial requirements that came to mind, one of the obvious ones being nmap scans. However, the scan had to be done in a way where popular ports were scanned first and quickly just to see if it was open. After that, version scans could be completed. Then I thought "what if port 80, or 443, or 8080 were open? Should we gobuster?". So that became a required flag as well. Also smbmap if 445 was included. So we already have a pretty good set of requirements for initial enumeration.

* Always perform -Pn scan to determine open ports
* Use version scan on open ports *if* specified via a flag
* Use gobuster *if* certain HTTP related ports (80, 443, 8080) are open *if* specified via a flag
* Use smbmap on 445 *if* 445 is open and specified

With these base level requirements we are ready to move on to actually building this thing.


## Nmap Ping Only
The first step of the script which will always be done no matter what flags are submitted is to perform an nmap ping only scan against the host.

{% highlight bash %}
nmap -Pn -oX nmap.xml $NMAP_TIME $host
{% endhighlight %}

The $NMAP_TIME and $host variables are for the scan speed and host name or IP address repecitvely. Next step is to parse the output for key information, priority being ports. I found this neat tool that parses nmap output from an XML file and gets key information out of it like ports, banners, etc. For more info click [here.](https://github.com/ernw/nmap-parse-output)

{% highlight bash %}
OPEN_PORTS=$(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml ports)
{% endhighlight %}

{% highlight bash %}
echo "[!] Open ports found: ${OPEN_PORTS}."
{% endhighlight %}

Nmap-parse-output will get the open ports on the machine and assign it to a variable. That variable can then be used to do all sorts of neat stuff. 

## Version Scans
Now we are getting to the heart of any wrapper script, flags. Using getopts we can designate the characters we want the user to enter in order to perform a certain action.
Lets start with the option for version scans.

{% highlight bash %}
while getopts h:v flag
do
        case "${flag}" in
        h ) host=${OPTARG};;
        v ) version=1;;
        \?) echo "[?] Invalid flag"
    esac
done

shift $((OPTIND-1))
{% endhighlight %}


So essentially we are checking to make sure that the host flag is set as that should  always be given to the script. This uses the OPTARG variable to hold the input. If -v is specified the script will perform a version scan against each of the open ports found.

{% highlight bash%}
version_scan(){

    echo "[+] Doing version scan on open ports"
    nmap -sS -sV -p$OPEN_PORTS -oN $NMAP_OUTPUT -oX nmap.xml $host
    
    if test -f $NMAP_PARSE_OUTPUT; then
        rm $NMAP_PARSE_OUTPUT
    fi

    echo "###################### PORT INFO #########################################" >> $NMAP_PARSE_OUTPUT
    IFS=',' read -r -a array <<< $OPEN_PORTS

    for port in ${array[@]}
    do
        echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml port-info ${port}) >> $NMAP_PARSE_OUTPUT
    done
    echo "##########################################################################" >> $NMAP_PARSE_OUTPUT

    echo $'\n' >> $NMAP_PARSE_OUTPUT

    echo "###################### HTTP PORTS #########################################" >> $NMAP_PARSE_OUTPUT
    echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml http-ports) >> $NMAP_PARSE_OUTPUT
    echo "##########################################################################" >> $NMAP_PARSE_OUTPUT

    echo $'\n' >> $NMAP_PARSE_OUTPUT

    echo "###################### SERVICE NAMES ######################################" >> $NMAP_PARSE_OUTPUT
    echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml service-names) >> $NMAP_PARSE_OUTPUT
    echo "###########################################################################" >> $NMAP_PARSE_OUTPUT

    echo $'\n' >> $NMAP_PARSE_OUTPUT

    echo "###################### PRODUCTS ###########################################" >> $NMAP_PARSE_OUTPUT
    echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml product) >> $NMAP_PARSE_OUTPUT
    echo "###########################################################################" >> $NMAP_PARSE_OUTPUT

}

{% endhighlight %}


This function will perform a version scan, output the scan to an XML file and parse for very specific things *I* am interested in. You should modify the script to output things you are interested in when performing enumeration to really get the most out of the tool.

To activate the function an if statement is used.

{% highlight bash %}

if [[ "${version}" -eq 1 ]]; then
    version_scan
fi

{% endhighlight %}

Oh and we need to check to make sure that the host flag is given. If not we give the user a usage report.

{% highlight bash %}

usage(){
    echo "[?] Usage $0 -h [host] [OTHER FLAGS]."
    echo "[?] -h: Host or IP."
    echo "[?] -v: nmap version scan."
    echo "[?] -g: gobuster if 80, 443, or 8080 open."
    echo "[?] -w: wordlist for gobuster."
    echo "[?] -s: smbmap if 445 open."
    exit 1
}

{% endhighlight %}

{% highlight bash %}

if [  -z "$host" ]; then
    usage
fi

{% endhighlight %}


## Gobuster Script
So now we are going to use the open ports to check and see if a gobuster might be viable.

{% highlight bash %}

gobust(){
    IFS=',' read -r -a array <<< $OPEN_PORTS

    if [[ " ${array[@]} " =~ "80" ||  " ${array[@]} " =~ "443" || " ${array[@]} " =~ "8080" ]]; then
        if [ ! -z "$wordlist" ] 
            then
            gobuster dir -u http://${host} -w $wordlist -o $GOBUSTER_OUTPUT
        elif [  -z "$wordlist" ]  
            then
                echo "[ERROR] Need a wordlist for gobuster. Use -w flag."
        fi
        else
            echo "[ERROR] No Standard HTTP ports found."
    
    fi
}

{% endhighlight %}

Because the array output from nmap-parse-output is delimted by commas we will use the IFS statement combined with the read command to assign the $OPEN_PORTS variable to an array. An if statement is used to check if the array contains certain ports. 

The "${array[@]}" selects all the ports in the array. Each =~ checks to see if the array contains a certain port number. If any of these ports are open gobuster will be run. If not it will error out and tell the user that *no standard HTTP ports are open*. Mind you this is what is standard to *my* knowledge and *my* experience. It might be wise to experiment with the filter if new discoveries are made.

### An additional word or so on gobuster...
Sometimes some kind of anti-DDOS software will be enabled on the target. If there is even a chance that this can ban your IP it might be wise to omit this flag, especially if you are in a situation where the *"client"* (hopefully this is a client or CTF) cannot take you off the blacklist. Also a lot of medium rated HTB machines have a fail2ban software installed.

## Smbmap Script
For smbmap we essentially do the same thing with port 445.

{% highlight bash%}

smap(){
    IFS=',' read -r -a array <<< $OPEN_PORTS

    if [[ " ${array[@]} " =~ "445" ]]; then
            smbmap -H $host
        else
            echo "[ERROR] No Standard SMB ports found."
    fi
}

{% endhighlight %}

## Completed Script

Here is the full script. You can also find it on my github [here](https://github.com/TBernard97/lazy-scan-script) if you would like.


{% highlight bash%}

#!/bin/bash

####################################################################
############# INITIAL VARIABLES ####################################
PATH_TO_NMAP_PARSE_OUTPUT="/opt/nmap-parse-output/nmap-parse-output"
NMAP_TIME=-T4
GOBUSTER_OUTPUT="./gobuster.txt"
NMAP_OUTPUT="./nmap.txt"
NMAP_PARSE_OUTPUT="./nmap_parse.txt"
###################################################################
###################################################################

usage(){
    echo "[?] Usage $0 -h [host] [OTHER FLAGS]."
    echo "[?] -h: Host or IP."
    echo "[?] -v: nmap version scan."
    echo "[?] -g: gobuster if 80, 443, or 8080 open."
    echo "[?] -w: wordlist for gobuster."
    echo "[?] -s: smbmap if 445 open."
    exit 1
}
gobust(){
    IFS=',' read -r -a array <<< $OPEN_PORTS

    if [[ " ${array[@]} " =~ "80" ||  " ${array[@]} " =~ "443" || " ${array[@]} " =~ "8080" ]]; then
        if [ ! -z "$wordlist" ] 
            then
            gobuster dir -u http://${host} -w $wordlist -o $GOBUSTER_OUTPUT
        elif [  -z "$wordlist" ]  
            then
                echo "[ERROR] Need a wordlist for gobuster. Use -w flag."
        fi
        else
            echo "[ERROR] No Standard HTTP ports found."
    
    fi
}

smap(){
    IFS=',' read -r -a array <<< $OPEN_PORTS

    if [[ " ${array[@]} " =~ "445" ]]; then
            smbmap -H $host
        else
            echo "[ERROR] No Standard SMB ports found."
    fi
}

version_scan(){

    echo "[+] Doing version scan on open ports"
    nmap -sS -sV -p$OPEN_PORTS -oN $NMAP_OUTPUT -oX nmap.xml $host
    
    if test -f $NMAP_PARSE_OUTPUT; then
        rm $NMAP_PARSE_OUTPUT
    fi

    echo "###################### PORT INFO #########################################" >> $NMAP_PARSE_OUTPUT
    IFS=',' read -r -a array <<< $OPEN_PORTS

    for port in ${array[@]}
    do
        echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml port-info ${port}) >> $NMAP_PARSE_OUTPUT
    done
    echo "##########################################################################" >> $NMAP_PARSE_OUTPUT

    echo $'\n' >> $NMAP_PARSE_OUTPUT

    echo "###################### HTTP PORTS #########################################" >> $NMAP_PARSE_OUTPUT
    echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml http-ports) >> $NMAP_PARSE_OUTPUT
    echo "##########################################################################" >> $NMAP_PARSE_OUTPUT

    echo $'\n' >> $NMAP_PARSE_OUTPUT

    echo "###################### SERVICE NAMES ######################################" >> $NMAP_PARSE_OUTPUT
    echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml service-names) >> $NMAP_PARSE_OUTPUT
    echo "###########################################################################" >> $NMAP_PARSE_OUTPUT

    echo $'\n' >> $NMAP_PARSE_OUTPUT

    echo "###################### PRODUCTS ###########################################" >> $NMAP_PARSE_OUTPUT
    echo $(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml product) >> $NMAP_PARSE_OUTPUT
    echo "###########################################################################" >> $NMAP_PARSE_OUTPUT

}

while getopts h:vgw:s flag
do
    case "${flag}" in
        h ) host=${OPTARG};;
        v ) version=1;;
        g ) gobust=1;;
        w ) wordlist=${OPTARG};;
        s)  smbmap_flag=1;;
        \?) echo "[?] Invalid flag"
    esac
done

shift $((OPTIND-1))

if [  -z "$host" ]; then
    usage
fi

nmap -Pn -oX nmap.xml $NMAP_TIME $host
OPEN_PORTS=$(${PATH_TO_NMAP_PARSE_OUTPUT} nmap.xml ports)
echo "[!] Open ports found: ${OPEN_PORTS}."



if [[ "${version}" -eq 1 ]]; then
    version_scan
fi


if [[ "${gobust}" -eq 1 ]]; then
    gobust
fi

if [[ "${smbmap_flag}" -eq 1 ]]; then
    smap
fi




{% endhighlight %}


## Next Steps
Next steps are to add some more useful functions. The use of the cewl tool has been coming up a lot lately in CTF's so I might add that. Also Aquatone functions might be useful as having a pretty HTML report from a DNS flyover might be convienient. 