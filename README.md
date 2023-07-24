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
In this document, we are focusing on reproducing results from the first environment: Queueless Random Packet Loss. In this environment, they validated the BW-predicting model by measuring BW under the assumption that no queue is formed and packet losses are random. The foundational result that we are validating involves varying three parameters (Delay/RTT, MSS, packet loss (p)) to produce BW. 

<img width="272" alt="image" src="https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/e04ea6f0-1a89-4e6b-94ee-d68323bbe4dd">


## Methodological issues
Let's run the experiment naively and discover the methodological issues by reproducing the research.


### 1. Trial combinations table
It is good to quantize and organize your data in a spreadsheet to know what trials you are running with what parameter settings. Since the first environment is quantized in Figures 3 and 4 of the paper, count the points in Figure 3 to find how many data points of BW (trials) we should have- it will be 60.


### 2. Parameter values
As mentioned before, each of the 60 trials is a combination of Delay/RTT, MSS, and packet loss (p) to calculate BW using the model and compare it to the experimental BW. 
###### MSS
There are 3 MSS values: 526 bytes (B), 1460 B, 4312 B.
###### RTT/delay
There are 5 RTT values between 3 and 300ms, but no further details are given. One approach is to choose 5 values that are equally apart: 3, 77.25, 151.5, 225.75, and 300 ms. 
###### Packet loss (p)
Packet loss values are uniformly distributed in log(p) between  0.00003 and  0.3. Since we have 60 points in total, 3 MSS values, and 5 RTT values, 60/(3*5)=4, this leaves 4 points for p. Figure 3 has packet loss on its x-axis, so p is not just 4 discrete values. For every combination of MSS and delay, we should generate 4 uniformly distributed values between 0.00003 and 0.3


Generate on Excel/Google Sheets 60 points with those 3 parameters, and number each trial. Calculate the model BW using the model (equation 3). To verify your choice of parameter values, you can plot the model BW against itself and confirm that the range of BW values on the x and y axes matches that from Figure 3 (min:5e4, max:2e8). Similarly, calculate BW*RTT/MSS vs p to confirm the range of values in figure 4 (min:0.0002, max:0.2). Plot in log scale. 
Here is a google sheet with my 60 trials for your reference: https://docs.google.com/spreadsheets/d/1vfvR07gic8oynpdxMrSt_JkWlNSyXOmwcM6QkL_JecY/edit 


### Network topology
This is the network topology

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/b300d43c-7884-468e-9782-6ec439dadae0)


### 3. Bottleneck link rate
We don't want a queue to form since this is a queueless random packet loss experiment. Therefore, the bottleneck link rate must be greater than the maximum model BW. Check the maximum BW in Figure 3 of the paper: it is around 2e8 bits/s, so we will set the bottleneck link rate to 1Gbit with 0.1GB buffer on both sides of the router (towards romeo and towards juliet).

Now run trial 7 by following the instructions in the section titled 'First environment runs' of this document. Add two columns to the excel sheet table called 'experiment BW' and 'experiment/model BW ratio', and record the experiment BW that you get from the run. 

### 4. Experimental BW Validation (hint: look at packet loss)
In the experiment BW from trial 7, did you notice it may have been higher than the model BW by 10, which is a lot. If that is the case, look again at what you set the packet loss to. The packet loss setting in the router takes p in %, meanwhile p that we have in our trial combinations is a scientific number. Now correct p in your parameter setting on JupyterLab so that when for example p=0.3, you set it as 30% in the router. You can add a column beside p in your excel table that has all p values multiplied by 100 to be used in the router settings directly.
After you've re-ran trial 7 again with the new higher p value, confirm that experimental/model BW ratio is much lower than 10. For trial 7 specifically, you will see that it goes up to nearly 3 (we will use this information to discover methodological issue #5). 

### 5. High Model BW could form a queue
Now run trial 8 by following the same instructions. Notice that the model BW is 10^8, which is close to what we set the bottleneck link rate to, 1Gbit. So it is possible that with a ratio that can go up to 3 like in trial 7, the experiment BW will exceed what you set the bottleneck link rate to, and a queue will form (queue is when rtt>>minrtt). So, what we can do is increase the bottleneck link rate to 3Gbit.

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

Romeo and Juliet:

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
1. min and avg rtt (that they match what you set delay to)

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
1. min and avg rtt (that they match what you set delay to)

2. packet loss (that it matches what you set it to- percentage value)


### Using iperf3 with continuous ss-output file
we will only be looking at the last line in the ss-output txt file since it summarizes all data transmitted in the experiment run the following simultaneously.

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

### Finding the name of your network interface (mine was ens7)
```
ifconfig -a
```

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
