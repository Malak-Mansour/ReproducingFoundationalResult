# ReproducingFoundationalResult
Reproducing a foundational result to find underlying methodological issues 

# Setup on Fabric
Create an account on Fabric and familiarize yourself with it on https://teaching-on-testbeds.github.io/hello-fabric/ 

Run setup.ipynb (Fabric file in this repository) until the end of 'Exercise: Log in to resources'

Open 4 new terminals on Jupyter where you paste the SSH command for 2 romeo terminals, 1 juliet terminal, and 1 router terminal
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/0218e560-2bfe-4098-9b8d-b43d3315b422)

# Our purpose
Our main goal is to reproduce research from the paper titled The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm:
https://cseweb.ucsd.edu/classes/wi01/cse222/papers/mathis-tcpmodel-ccr97.pdf. We are reproducing research to create educational material for students and professors by helping them understand underlying methodological issues by running the experiments from the paper and matching its settings as closely as possible.
The paper's primary purpose is to 
1. Derive a model for TCP to predict BandWidth (BW)
2. Validate this foundational result through three primary network environments: Queueless Random Packet Loss, Environments with Queueing (not random), Effect of TCP Implementation.

# First Environment Setup
In this document, we are focusing on reproducing results from the first environment: Queueless Random Packet Loss. In this environment, they validated the BW-predicting model by measuring BW under the assumption that no queue is formed and packet losses are random. The foundational result that we are validating involves varying three parameters (Delay, MSS, packet loss (p)) to produce BW. 

<img width="272" alt="image" src="https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/e04ea6f0-1a89-4e6b-94ee-d68323bbe4dd">


## Methodological issues
Here are some methodological issues that were only discovered upon reproducing research:

1. The number of data points/trials to run

Solution: counted the points in Figure 3 to be 60

2. Choosing which five RTT values to take between 3 and 300ms since no details were given about the exact values

Solution: chose 5 values that are equally apart: 3, 77.25, 151.5, 225.75, and 300 ms

3. The randomness in the choice of packet loss values uniformly distributed in log(p) between  0.00003 and  0.3

Solution: 60 total points, 3 MSS, and 5 rtt, leaves 4 points for p. Also, figure 3 has packet loss on its x axis, so we have many values of p. Therefore, for every combination of MSS and delay, we generated 4 uniformly distributed values between 0.00003 and 0.3

4. Validating packet loss to make sure p-value matches the description of p in the paper: it didn't match the description, and that showed when our experimental BandWidth was off by 2 orders of magnitude.

Solution: Multiply p by 100 to express in % (eg. p=0.3 is 30%)

5. Selecting appropriate duration: no loss with low p because not enough duration

Solution: increase duration from 60s to 240s

6. For 1460 byte mss (and 4312 bytes), the actual mss is 12 bytes less than expected 

Solution: Disable TCP timestamps option with a sysctl command

7. For 4312 byte mss, we run into the limit of MTU

Solution: Need to increase interface MTU (e.g. with ifconfig)


8. Confusion arises about whether a wrong result (experimental BW being too different from model BW) is only an outlier or an experimental error, and if the trial should be run again.

Solution: We have to validate our experiments using ping (sends many packets) and generate an ss-output file to validate that our settings are correct by confirming the parameter values. 

If it is an experimental error and we have mistakenly set something wrong, then we can repeat the trial and set the correct parameters. 

Otherwise, if for example not enough packets have been sent to even see a packet loss or a queue formed, then we can increase the duration or number of packets for **all** trials to have a fixed setting for all! Don’t repeat experiments just because of unusual results because we would be “forcing results” and “throwing out data”. 

If both of these have been done and still the results are not exactly as expected, then it is probably an outlier, which is okay to exist!



## Parameter values
One-way delay takes the following values 3, 77.25, 151.5, 225.75, and 300 ms. MSS takes 536, 1460, and 4312 bytes. Sine Figure 3 has approximately 60 points in total, we conclude that the number of uniformly distributed values of packet loss (in log(p)) are 4: 5 delay * 3 MSS * 4 p = 60 BW points). For every combination of MSS and delay, generate 4 uniformly distributed values of p between 0.00003 and 0.3. Generate on excel 60 points with those parameters, and number each trial.

## This is the network topology

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/b300d43c-7884-468e-9782-6ec439dadae0)


## 60 trials table
Here is a google sheet with my 60 trials for your reference: https://docs.google.com/spreadsheets/d/1vfvR07gic8oynpdxMrSt_JkWlNSyXOmwcM6QkL_JecY/edit 

## Bottleneck link rate
First, we don't want a queue to form. Check the maximum BW in Figure 3 of the paper. If we don't want a queue to form, then the bottleneck link rate must be greater than the maximum possible BW. The maximum BW in the plot is around 2e8 bits/s, so we will set the bottleneck link rate to 1Gbit with  0.1GB buffer on both sides of the router (towards romeo and towards juliet)

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/25700a2f-5861-4e8e-b7dd-083d72e475d5)



# First environment runs
We now want to start running the 60 trials from the first environment. We will start with 20 trials corresponding to the 1460 Bytes MSS case (will do 4312 and 536 Bytes cases after). 


## Code that takes the methodological issues into consideration:

### Setup delay and loss on router
Router:

```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 3ms 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate 1Gbit
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem delay 3ms loss 15.66283969% limit 100MB
```


### Install iperf3 on Romeo and Juliet [3]

Romeo:

```
sudo apt-get update  
sudo apt-get -y install iperf3  
```

Install moreutils to use certain functions later on.
Romeo:
```
sudo apt-get -y install moreutils r-base-core r-cran-ggplot2 r-cran-littler
sudo sysctl -w net.ipv4.tcp_no_metrics_save=1  
```

Juliet:

```
sudo apt-get update  
sudo apt-get -y install iperf3  
```

## Hidden settings to set accurate MSS
Current MTU (Maximum Transmission Unit), which is the maximum size of the packet that can be transmitted from a network interface, is 1500 Bytes (B). 20B are consumed by IP header, 20B by TCP header, and 12B by the TCP timestamps option. That's why if you set MSS to any value greater than or equal to 1448B, you will always get MSS=1448B (MTU=1500B, 1500-20-20-12=1448B).

### If you are running the MSS=1460B or MSS=4312B case, disable TCP timestamps option, which consumes 12 bytes from what we set mss to

Romeo: [4] 
```
sudo sysctl -w net.ipv4.tcp_timestamps=0  
```

### If you are running the MSS=4312B case, increase mtu 
Considering the 40B consumed by headers and assuming disabled timestamps, we will need MTU=4352B (4312+40).

##### You can check the current mtu value using
Romeo, Juliet, and/or Router:
```
ifconfig | grep mtu
```


##### Increase the MTU on the experiment interface of romeo and juliet

Romeo and Juliet: [5]
```
sudo ifconfig ens7 mtu 4352 up
```


##### Increase the MTU on both of the experiment interfaces on the router

Router: [5]

```
sudo ifconfig ens7 mtu 4352 up
sudo ifconfig ens8 mtu 4352 up
```

##### Confirm that mtu did increase using
Romeo, Juliet, and/or Router:
```
ifconfig | grep mtu
```

## Validating results and measuring BW

### Ping 

Romeo: (sending 5000 packets with 200ms in between each) [3] 
```
ping juliet -c 5000 -i 0.2
```

##### Data to look at:
1. min and avg rtt (that avg rtt is not much larger than min rtt, otherwise a queue would have formed)

2. packet loss (that it matches what you set it to- percentage value)


### Using iperf3 with continuous ss-output file 
we will only be looking at the last line in the ss-output txt file since it summarizes all data transmitted in experiment run the following simultaneously.

Juliet: [3]
```
iperf3 -s  -1  
```

While that is running, paste the following in one Romeo terminal, 
Romeo_1: [3]
```
wget -O ss-output.sh https://raw.githubusercontent.com/ffund/tcp-ip-essentials/gh-pages/scripts/ss-output.sh
bash ss-output.sh 10.10.2.100  
```

While that is running, paste the following in the other Romeo terminal, 
Romeo_2: [2] (240s duration, TCP reno, MSS 1460)
```
iperf3 -c juliet -t 240 -C reno -M 1460
```

When the process in Romeo_2 is done, bash will also stop running, but the process will not close, so you need to close it manually using ctrl+C. Then paste the following to see the output with tcp reno, 
Romeo_1: [3]
```
cat sender-ss.txt | grep "reno"
```

##### Data to look at:
iperf:

BandWidth: compare it to model BW, you can find the ratio of experimental BW to model BW


ss-output:

1. min and avg rtt (that avg rtt is not much larger than min rtt, otherwise a queue would have formed)

2. retrans and data_segs_out: packet loss=retrans/data_segs_out (that it matches what you set it to- not the percentage value)



# Code for remaining trials
For the each trial, do the following. Change the parameters in **bold** (or surrounded by 2 asterisks '** **') depending on each trial's settings.

Router:

```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay **3ms** 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate 1Gbit
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem delay **3ms** loss **15.66283969%** limit 100MB
```


## Validating results and measuring BW

### Ping 

Romeo: (sending **5000** packets with 200ms in between each) [3] 
```
ping juliet -c **5000** -i 0.2
```

##### Data to look at:
1. min and avg rtt (that avg rtt is not much larger than min rtt, otherwise a queue would have formed)

2. packet loss (that it matches what you set it to- percentage value)


### Using iperf3 with continuous ss-output file 
we will only be looking at the last line in the ss-output txt file since it summarizes all data transmitted in experiment run the following simultaneously.

Juliet: [3]
```
iperf3 -s  -1  
```

While that is running, paste the following in one Romeo terminal, 
Romeo_1: [3]
```
bash ss-output.sh 10.10.2.100  
```

While that is running, paste the following in the other Romeo terminal, 
Romeo_2: [2] (240s duration, TCP reno, MSS **1460**)
```
iperf3 -c juliet -t 240 -C reno -M **1460**
```

When the process in Romeo_2 is done, bash will also stop running, but the process will not close, so you need to close it manually using ctrl+C. Then paste the following to see the output with tcp reno, 
Romeo_1: [3]
```
cat sender-ss.txt | grep "reno"
```

##### Data to look at:
iperf:

BandWidth: compare it to model BW, you can find the ratio of experimental BW to model BW


ss-output:

1. min and avg rtt (that avg rtt is not much larger than min rtt, otherwise a queue would have formed)

2. retrans and data_segs_out: packet loss=retrans/data_segs_out (that it matches what you set it to- not the percentage value)



# Possible fixes
### If you're getting weird results in your validation steps (ping or ss-output), here are some things to do to check where the problem is. 

If you changed the delay, on the romeo or juliet ends, then your network topology would be wrong because this is how it should look like:
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/18bb6ce4-810b-42f8-81e0-34637d25755c)


Let's test that - if we 'ping' from iface0 of router to romeo, we should see 151.5ms of round trip delay. On the router, run
```
ping -c 10 10.10.1.100
```

also if we 'ping' from iface1 of router to juliet, we should see 151.5ms of round trip delay. So on the router, run
```
ping -c 10 10.10.2.100
```

If there is any extra delay on the romeo-router path, run this on the router: 
```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 0 
```

This sets 0ms delay on the iface0 interface of the router. Now repeat that router-romeo ping:
```
ping -c 10 10.10.1.100
```


If the delay is on packets leaving romeo, run this on romeo to see when you may have mistakenly added a delay:
```
tc -s qdisc show
sudo cat /var/log/auth.log | grep "qdisc"
```

Remove any additional delay or loss from iface0 on romeo
```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
```

After you do this, you should test again and make sure the delay is gone
```
ping -c 10 10.10.1.100
```

~Similarly, if there is any extra delay on the juliet-router path, run the same commands above but using iface_1 and 10.10.2.100

###  It is normal for the receiver BW to be slightly smaller than sender BW
The throughput is "data sent/time". 
The sender considers "time" to be "time from connection start, to time I sent last bit". 
The receiver considers "time" to be "time from connection start, to time I got the last bit"
The receiver gets the last bit a little later than the sender sends it, so the denominator in the receiver throughput is a tiny bit bigger, contributing to a slightly higher BW.


# Analyzing results
To analyze results, we are plotting the experimental BW (from iperf in JupyterLab) against the model BW (from the calculations using the model), and comparing it to figure 3 in the paper. 

# Resources
[1] https://www.cs.unm.edu/~crandall/netsfall13/TCtutorial.pdf 

[2] https://manpages.ubuntu.com/manpages/xenial/man1/iperf3.1.html 

[3] https://witestlab.poly.edu/blog/tcp-congestion-control-basics/#setupexperiment 

[4] https://docs.vmware.com/en/vRealize-Operations/8.10/com.vmware.vcom.scg.doc/GUID-DAC867BC-8C5F-4A5E-BB55-36FC25555696.html 

[5] https://linuxhint.com/how-to-change-mtu-size-in-linux/ 
