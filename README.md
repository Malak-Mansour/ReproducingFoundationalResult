# Reproducing a foundational result to find underlying methodological issues 

###### Malak Mansour
###### August 11th, 2023


The paper's primary purpose is to 
1. Derive a model for TCP to predict BandWidth (BW)

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/7d07f62f-2ea2-4631-b82b-b0563469dea3)

2. Validate this foundational result through three primary network environments: Queueless Random Packet Loss, Environments with Queueing (not random), Effect of TCP Implementation.


It should take about 30 minutes to run this experiment.

You can run this experiment on [Fabric](https://teaching-on-testbeds.github.io/hello-fabric/ ). The steps in the hyperlink should be done.


# Background
We are reproducing research from the paper titled _The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm_
(Mathis, M., Semke, J., & Mahdavi, J. (1997). The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm. ACM SIGCOMM Computer Communication Review, 27(3), 67-82. ) in order to better understand underlying methodology issues by running the experiments from the paper and matching its settings as closely as possible.
In this experiment, we are only focusing on reproducing results from the first environment, Queueless Random Packet Loss. This environment validates the foundational result, which shows a linear relationship between TCP bandwidth and three parameters that we vary: Delay/RTT, MSS, packet loss (p). 

<img width="272" alt="image" src="https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/e04ea6f0-1a89-4e6b-94ee-d68323bbe4dd">

As part of validating the foundational result, we are going to try reproducing figure 3 below.

<img width="641" alt="image" src="https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/8ef680f3-0051-4911-8654-846739068e0b">

## Choices for setting up the experiment 
We are using this part from the paper to extract information about what parameter values we are going to use in our trials.

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/d07f59d1-143e-4b94-9a71-d8d47229e2af)


### MSS
There are 3 MSS values: 526 bytes (B), 1460 B, 4312 B.
### RTT/delay
There are 5 RTT values between 3 and 300ms, but no further details are given. One approach is to choose 5 values that are equally apart: 3, 77.25, 151.5, 225.75, and 300 ms. 
### Packet loss (p)
Packet loss values are uniformly distributed in log(p) between  0.00003 and  0.3. 
The total number of points in Figure 3 is approximately 60, this means that we have 60 BW values, so 60 trials in total. With 3 MSS values and 5 RTT values, 60/(3*5)=4, this leaves 4 points for p. Figure 3 has packet loss on its x-axis, so p is not just 4 discrete values. For every combination of MSS and delay, we should generate 4 uniformly distributed values between 0.00003 and 0.3
### Bottleneck link rate
We don't want a queue to form since this is a queueless random packet loss experiment. Therefore, the bottleneck link rate must be greater than the maximum model BW. The maximum BW in Figure 3 of the paper is around 2e8 bits/s, so we will set the bottleneck link rate to 1Gbit.


## Experiment Procedure 
The straightforward implementation of the experiment may not behave as expected. By validating at each step, we can identify the issues and resolve them. This is the flowchart of the experiment procedure.

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/86b9af16-e1f2-47ec-b182-894286b0e161)

# Results 
This is how our results will look like as we discover the methodological issues

<img width="473" alt="image" src="https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/351116f5-e37e-4bb2-a2c0-bc48050a2bd3">




# Run My Experiment 
Run setup.ipynb (in this repository) until the end of 'Exercise: Log in to resources' to setup the line network. Do not run 'Turn segment offloading off'. We will address this later!

Open 4 new terminals on Jupyter where you paste the SSH command for 2 romeo terminals, 1 juliet terminal, and 1 router terminal
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/0218e560-2bfe-4098-9b8d-b43d3315b422)


## Let's run one experiment example according to the flowchart experiment procedure.
### 1. Setup and Validate
For each experiment, we are going to setup the requested parameters using tc qdisc. 


For example, let's setup the following parameters: mss = 1460, p = 0.1566283969, rtt = 6ms. 

#### Setup
Router: (1Gbit bottleneck link rate [3], 0.1GB buffer on both sides [3], delay and loss [1] change depending on trial settings) 

<pre>
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 151.5ms 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate <b>1Gbit</b>
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem <b>delay 151.5ms loss 0.1097164529% limit 100MB</b>
</pre>


#### Validate this experiment setup using ping:
##### Romeo:

Sending 1000 packets

<pre>
ping juliet -c <b> 1000 </b> -i 0.2
</pre>

##### Output:

<pre>
--- juliet ping statistics ---
1000 packets transmitted, 829 received, <b>17.1% packet loss</b>, time 200593ms
<b>rtt min/avg</b>/max/mdev = <b>6.088/6.131</b>/6.395/0.026 ms
</pre>

### 2. Execute and Validate
Execute experiment using iperf

Look at ss-output to validate,

#### mss

#### rtt

#### queueless environment

#### packet loss

(show output as code block and bold the necessary parts)


### 3. Identify Methodology Issues

#### Issue #1: Interpreting the packet loss parameter
##### Experiment settings to run:

(parameters and link back to "Run My Experiment", sections 1 and 2)
Trial 5

Experiment setup: RTT = 6ms, p=0.1566283969, MSS = 1460B

Command to implement setup: 
<pre>
netem delay 151.5ms <b>loss 0.1566283969%</b> limit 100MB
</pre>

Validating execution: Model BW = 4.92 Mbps, Experiment BW = 207 Mbps

<pre> 
—- juliet ping statistics —-
1000 packets transmitted, 999 received, <b> 0.1% packet loss </b> time 200174ms
</pre> 
___________________________________
Command to fix issue: 
<pre> 
netem delay 151.5ms <b>loss 15.66283969% </b> limit 100MB 
</pre>

Validating execution: Model BW = 4.92 Mbps, Experiment BW = 1.68 Mbps


<pre> 
—- juliet ping statistics —-
1000 packets transmitted, 838 received, <b> 16.2% packet loss </b> time 200549ms
</pre> 

##### Validate setup, what is wrong with the output?
(highlight miustakes in outout code block)

##### How to fix it: 
The "loss" in netem is expressed as a percent, in the paper p is expressed as a ratio.

##### Rerun experiment with fix 
(insert example of fixed result)

#### Issue #2: Setting a sufficient experiment duration
##### Experiment settings to run:
(parameters and link back to "Run My Experiment", sections 1 and 2)

##### Validate setup, what is wrong with the output?
(highlight miustakes in outout code block)

##### How to fix it: 
For small values of p, packet loss is a rare event - we need a longer experiment duration to approximate the "requested" packet loss.


##### Rerun experiment with fix 
(insert example of fixed result)

#### Issue #3: Experiencing a queueless environment
##### Experiment settings to run:
(parameters and link back to "Run My Experiment", sections 1 and 2)

##### Validate setup, what is wrong with the output?
(highlight miustakes in outout code block)

##### How to fix it: 
Although the model does not predict BW exceeding 1Gbps, we observe this in practice, so we must increase the bottleneck rate to avoid queuing.


##### Rerun experiment with fix 
(insert example of fixed result)

#### Issue #4: Getting the requested MSS (1460B case)
##### Experiment settings to run:
(parameters and link back to "Run My Experiment", sections 1 and 2)

##### Validate setup, what is wrong with the output?
(highlight mistakes in output code block)

##### How to fix it: 
Hidden TCP timestamps option takes up 12 bytes from what we set mss to, so we must turn it off to get 1460B mss value.


##### Rerun experiment with fix 
(insert example of fixed result)

#### Issue #5: Getting the requested MSS (4312B case)
(parameters and link back to "Run My Experiment", sections 1 and 2)

##### Validate setup, what is wrong with the output?
(highlight miustakes in outout code block)

##### How to fix it: 
Default Ethernet MTU is 1500B, to have MSS > 1460 we need to increase MTU also.


##### Rerun experiment with fix 
(insert example of fixed result)

#### Issue #6: Getting the requested MSS (all cases)
##### Experiment settings to run:
(parameters and link back to "Run My Experiment", sections 1 and 2)

##### Validate setup, what is wrong with the output?
(highlight miustakes in outout code block)

##### How to fix it: 
NIC has a segment offload feature, combines segments en route to same destination - unless we turn it off, "effective MSS" is higher than what TCP thinks it is.

Try 'tcpdump' on the router while the normal iperf is running.

Router: [6]
```
sudo tcpdump -i ens7
```

When it is done scroll up and notice how the packet size is double, triple, or even quadruple the mss value that you set (try it with 536, 1460, and 4312). Because of the direct proportionality between BW and MSS in the model, when MSS is very high (aggregated), it results in the high BW values that we were getting. So we can turn off this segment offloading feature to prevent segments from aggregating and get reasonable experimental BW values.


Run 'Turn segment offloading off' in setup.ipynb that has the following code:
```
for iface in slice.get_interfaces():
    iface_name = iface.get_device_name()
    n = iface.get_node()
    offloads = ["gro", "lro", "gso", "tso"]
    for offload in offloads:
        n.execute("sudo ethtool -K %s %s off" % (iface_name, offload))
```


##### Rerun experiment with fix 
(insert example of fixed result)


### 4. Generate figure 3 
Generate on Excel/Google Sheets 60 points with the 3 parameters, and number each trial. Calculate the model BW using the model (equation 3). To verify your choice of parameter values, you can plot the model BW against itself and confirm that the range of BW values on the x and y axes matches that from Figure 3 (min:5e4, max:2e8). Similarly, calculate BW*RTT/MSS vs p to confirm the range of values in figure 4 (min:0.0002, max:0.2). Plot in log scale. 
Here is a google sheet with my 60 trials for your reference: https://docs.google.com/spreadsheets/d/1vfvR07gic8oynpdxMrSt_JkWlNSyXOmwcM6QkL_JecY/edit 


#### Full code including all methodological issues

Router: (3Gbit bottleneck link rate [3], 0.1GB buffer on both sides [3], 3ms delay and 15.66283969% (0.1566283969) loss [1] - change depending on trial settings) 

```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 3ms
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate 3Gbit
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem delay 3ms loss 15.66283969% limit 100MB
```


##### Disable TCP timestamps option
Romeo: [4] 
```
sudo sysctl -w net.ipv4.tcp_timestamps=0  
```

##### 4312B only
###### You can check the current mtu value using
Romeo, Juliet, and/or Router:
```
ifconfig | grep mtu
```


###### Increase the MTU on the experiment interface of romeo and juliet

Romeo and Juliet: [5]
```
sudo ifconfig ens7 mtu 4352 up
```


######  Increase the MTU on both of the experiment interfaces on the router

Router: [5]

```
sudo ifconfig ens7 mtu 4352 up
sudo ifconfig ens8 mtu 4352 up
```

###### Confirm that mtu did increase using
Romeo, Juliet, and/or Router:
```
ifconfig | grep mtu
```


##### Validating results and measuring BW

##### Ping 

Romeo: (sending **5000** packets with 200ms in between each) [3] 
```
ping juliet -c 5000 -i 0.2
```

###### Data to look at:
1. min and avg rtt (that they match what you set delay to)

2. packet loss (that it matches what you set it to- percentage value)


##### Using iperf3 with continuous ss-output file
we will only be looking at the last line in the ss-output txt file since it summarizes all data transmitted in the experiment run the following simultaneously.

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
Romeo_2: [2] (240s duration, TCP reno, MSS **1460**)
```
iperf3 -c juliet -t 240 -C reno -M 1460
```

When the process in Romeo_2 is done, bash will also stop running, but the process will not close, so you need to close it manually using ctrl+C. Then paste the following to see the output with tcp reno, 
Romeo_1: [3]
```
cat sender-ss.txt | grep "reno"
```

###### Data to look at:
iperf:

BandWidth: compare it to model BW, you can find the ratio of experimental BW to model BW


ss-output:

1. min and avg rtt (that avg rtt is not much larger than min rtt, otherwise a queue would have formed)

2. retrans and data_segs_out: packet loss=retrans/data_segs_out (that it matches what you set it to- not the percentage value)



#### Analyzing results
To analyze results, we are plotting the experimental BW (from iperf in JupyterLab) against the model BW (from the calculations using the model), and comparing it to figure 3 in the paper. 

# Resources
[1] https://www.cs.unm.edu/~crandall/netsfall13/TCtutorial.pdf 

[2] https://manpages.ubuntu.com/manpages/xenial/man1/iperf3.1.html 

[3] https://witestlab.poly.edu/blog/tcp-congestion-control-basics/#setupexperiment 

[4] https://docs.vmware.com/en/vRealize-Operations/8.10/com.vmware.vcom.scg.doc/GUID-DAC867BC-8C5F-4A5E-BB55-36FC25555696.html 

[5] https://linuxhint.com/how-to-change-mtu-size-in-linux/ 

[6] https://ffund.github.io/tcp-ip-essentials/lab1/1-5-tcpdump-wireshark 





------------------------------------------------------------------------------------------------------------------------------------------------------------------


Run setup.ipynb (Fabric file in this repository) until the end of 'Exercise: Log in to resources'

Open 4 new terminals on Jupyter where you paste the SSH command for 2 romeo terminals, 1 juliet terminal, and 1 router terminal
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/0218e560-2bfe-4098-9b8d-b43d3315b422)

# Our purpose
Our main goal is to reproduce research from the paper titled The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm:
https://cseweb.ucsd.edu/classes/wi01/cse222/papers/mathis-tcpmodel-ccr97.pdf. We are reproducing research to create educational material for students and professors by helping them understand underlying methodological issues by running the experiments from the paper and matching its settings as closely as possible.


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


#### Network topology
This is the network topology

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/b300d43c-7884-468e-9782-6ec439dadae0)


### 3. Bottleneck link rate
We don't want a queue to form since this is a queueless random packet loss experiment. Therefore, the bottleneck link rate must be greater than the maximum model BW. Check the maximum BW in Figure 3 of the paper: it is around 2e8 bits/s, so we will set the bottleneck link rate to 1Gbit with 0.1GB buffer on both sides of the router (towards romeo and towards juliet).

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/25700a2f-5861-4e8e-b7dd-083d72e475d5)

Now run trial 7 by following the instructions below. Add two columns to the excel sheet table called 'experiment BW' and 'experiment/model BW ratio', and record the experiment BW that you get from the run. 

##### Set parameters on router

Router:

```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 3ms 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate 1Gbit
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem delay 3ms loss 0.005730706937% limit 100MB
```


##### Install iperf3 on Romeo and Juliet [3]

Romeo and Juliet:

```
sudo apt-get update  
sudo apt-get -y install iperf3  
```

##### Install moreutils to use certain functions later on

Romeo:
```
sudo apt-get -y install moreutils r-base-core r-cran-ggplot2 r-cran-littler
sudo sysctl -w net.ipv4.tcp_no_metrics_save=1  
```


##### Validating results and measuring BW
To  validate our trial settings and results, we run two validation steps: ping and ss-output simultaneously with iperf.

###### 4. Ping packets 
To calculate how many ping packets we should send to see at least one packet loss and validate p, let's look at how the paper defines p. 

![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/7183a5b5-7ded-41eb-b122-65daecffc073)


Round up 1/p and multiply it by 3 or so. Packet loss is random, so we will not actually know exactly when a loss will happen, that's why we overestimate the packets to be sent. If you set a value for ping packets but the packet losses from the ping run are much lower than expected, you probably need to send more packets!

For trial 7, 1/p=1/0.005730706937=174.498541104, so we should send at least 500 packets.

Romeo: (sending 500 packets with 200ms in between each) [3] 
```
ping juliet -c 500 -i 0.2
```

###### Data to look at:
1. min and avg rtt (that they match what you set delay to)

2. packet loss (that it matches what you set it to- percentage value)


###### Using iperf3 with continuous ss-output file 
We will only be looking at the last line in the ss-output txt file since it summarizes all data transmitted in the experiment. Run the following instructions simultaneously.

###### 5. Duration of iperf
We should decide on the duration of iperf beforehand. Start with any value that makes sense to you initially. For example, check if 60 seconds is enough to send enough packets, see a packet loss, and validate the results that you are expecting- especially when you have a low enough packet loss value. If you do not see that 60s is enough, try again with 120s, and go up to 240s if needed, which is what we are using in the following code.


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

### 6. TCP timestamps option
Look at the mss value from the ss-output, notice how it is 1448 but you set it to 1460. This is because 12 bytes are consumed by the TCP timestamps option. So to disable it we can do the following.


Romeo: [4] 
```
sudo sysctl -w net.ipv4.tcp_timestamps=0  
```

### 7. Experimental BW Validation (hint: look at packet loss %)
Additionally, look at the experiment BW from iperf, did you notice it may have been higher than the model BW by 10, which is a lot? If that is the case, look again at what you set the packet loss to. The packet loss setting in the router takes p in %, meanwhile, p that we have in our trial combinations is a scientific number. Now correct p in your parameter setting on JupyterLab so that when for example p=0.3, you set it as 30% in the router. You can add a column beside p in your excel table that has all p values multiplied by 100 to be used in the router settings directly.

For trial 7 for example, this is what we can change to correct p (0.005730706937% will change to 0.5730706937%):

Router:

```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 3ms 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate 1Gbit
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem delay 3ms loss 0.5730706937% limit 100MB
```


This is the importance of the validation steps. They help us discover these unanticipated-for methodological issues.

After you've re-run trial 7 again with the disabled TCP timestamps option and the new higher p value (multiply scientific p by 100 to get it in %), confirm that the mss value is 1460 and that the experimental/model BW ratio is much lower than 10. For trial 7 specifically, you will see that it goes up to nearly 3 (we will use this information to discover methodological issue #5). 


### 8. High Model BW could form a queue
Now run trial 8 by following the same instructions. Notice that the model BW is 10^8, which is close to what we set the bottleneck link rate to, 1Gbit. So it is possible that with a ratio that can go up to 3 like in trial 7, the experiment BW will exceed what you set the bottleneck link rate to, and a queue will form (queue is when rtt>>minrtt). So, what we can do is increase the bottleneck link rate to 3Gbit.



Now re-run trial 8 again with a higher bottleneck link rate to avoid a queue from forming:

##### Setup delay and loss on router
Router:

```
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 3ms 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate 3Gbit
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem delay 3ms loss 0.0186147305% limit 100MB
```

##### Validating results and measuring BW

###### Ping 
roundup(1/p)=roundup(1/0.000186147305)=5373, so we can send 15000 ping packets

Romeo: (sending 15000 packets with 200ms in between each) [3] 
```
ping juliet -c 15000 -i 0.2
```

###### Data to look at:
1. min and avg rtt (that they match what you set delay to)

2. packet loss (that it matches what you set it to- percentage value)


###### Using iperf3 with continuous ss-output file 

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



Check that indeed a queue doesn't form- from ss-output, rtt is not much larger than minrtt, and experiment BW is not very close to the bottleneck link rate (3Gbit).


### 9. 4312 B cases
Now let's run trial 9, with 4312B mss value. Notice how mss comes out as 1460B in the ss-output even after you've disabled TCP timestamps option and set mss to 4312 when you ran iperf. This is because of the MTU.

Current MTU (Maximum Transmission Unit), which is the maximum size of the packet that can be transmitted from a network interface, is 1500 Bytes (B). 20B are consumed by IP header, 20B by TCP header, and 12B by the TCP timestamps option. That's why if you set MSS to any value greater than or equal to 1448B, you will always get MSS=1448B (MTU=1500B, 1500-20-20-12=1448B).


If you are running the MSS=4312B case, increase mtu.

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

Now re-run trial 9 again and confirm that mss does come out as 4312B in the ss-output.


### 10. 536B cases (tcpdump)
When you plot the experiment BW against model BW for the 4312 and 1460 B cases, you will notice that all the points are above the slope=1 line, meanwhile, in the paper, all the points are below the slope=1 line of Figure 3. Furthermore, when you try the 536B cases, you will notice that the experimental BW is much higher than model BW and the ratio of exp to model BW is much higher! Therefore, it seems like some packets are getting aggregated, and it only became significantly noticeable with the lower mss case (536B). 

If the NIC is aggregating TCP segments, it would make a bigger relative difference in the small MSS case than in the large MSS case.

Imagine the NIC is aggregating the TCP segments into e.g. 5000 B "chunks".  For the 4312 case, it's not very different than the segment size we "thought" we had. For the 536 case, it's very different. 

So we try 'tcpdump' on the router while the normal iperf is running.

Router: [6]
```
sudo tcpdump -i ens7
```

When it is done scroll up and notice how the packet size is double, triple, or even quadruple the mss value that you set (try it with 536, 1460, and 4312)


Now we'll do the following, to see what the effect of the segment offload is on the exp bw
First, do the experiment a bunch of times with the current setting, maybe 10 times. You'll get 10 "exp bw" values and 10 ratios of exp BW to model BW. 
Then, we'll turn off segment offloading, and do it another 10 times.

##### Segment offloading:
let's set up a second slice - make a copy of your notebook, in this copy, give it a different slice name, reserve and configure resources etc. At the end of configuring the resources (setting up the IP addresses and routes etc), in the "new" slice do

(This is all in the setup-offloadingOff.ipynb file in this repo)

```
for iface in slice.get_interfaces():
    iface_name = iface.get_device_name()
    n = iface.get_node()
    offloads = ["gro", "lro", "gso", "tso"]
    for offload in offloads:
        n.execute("sudo ethtool -K %s %s off" % (iface_name, offload))
```


then in the "new" slice you will test with tcpdump at the router, as we have just been doing

Router:
```
sudo tcpdump -i ens7
```

Make sure we don't see any evidence of segments being "combined". Then run the same trial (trial 3 for example) 10 times in the "new" slice and keep note of the ss-output results in a document.


# Full code incorporating all methodological issues
Run setup-offloadingOff.ipynb (Fabric file in this repository) until the end of 'Exercise: Log in to resources'. Open 4 terminals and paste the ssh output for romeo twice, router once, and juliet once.


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



##### Install iperf3 on Romeo and Juliet [3]

Romeo and Juliet:

```
sudo apt-get update  
sudo apt-get -y install iperf3  
```

##### Install moreutils to use certain functions later on

Romeo:
```
sudo apt-get -y install moreutils r-base-core r-cran-ggplot2 r-cran-littler
sudo sysctl -w net.ipv4.tcp_no_metrics_save=1  
```

##### Disable TCP timestamps option
Romeo: [4] 
```
sudo sysctl -w net.ipv4.tcp_timestamps=0  
```

##### 4312B only
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


#####  Increase the MTU on both of the experiment interfaces on the router

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
wget -O ss-output.sh https://raw.githubusercontent.com/ffund/tcp-ip-essentials/gh-pages/scripts/ss-output.sh
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




# Code for remaining trials
We now want to start running the 60 trials from the first environment. We will start with 20 trials corresponding to the 1460 Bytes MSS case, then do 4312 and 536 Bytes cases after. Remember to disable TCP timestamps option for the cases of 1460 and 4312 B MSS, and increase mtu for the 4312 B case. 

For each trial, do the following. Change the parameters in **bold** (or surrounded by 2 asterisks '** **') depending on each trial's settings.

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
We have to validate our experiments using ping (sends many packets) and generate an ss-output file to validate that our settings are correct by confirming the parameter values. 

If it is an experimental error and we have mistakenly set something wrong, then we can repeat the trial and set the correct parameters. 

Otherwise, if for example not enough packets have been sent to even see a packet loss or a queue formed, then we can increase the duration or number of packets for **all** trials to have a fixed setting for all! Don’t repeat experiments just because of unusual results because we would be “forcing results” and “throwing out data”. 

If both of these have been done and still the results are not exactly as expected, then it is probably an outlier, which is okay to exist!


### 1. Finding the name of your network interface (mine was ens7)
```
ifconfig -a
```

### 2. Experimental error (undo incorrect parameter settings)
If you're getting weird results in your validation steps (ping or ss-output), here are some things to do to check where the problem is. 

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

###  3. It is normal for the receiver BW to be slightly smaller than sender BW
The throughput is "data sent/time". 
The sender considers "time" to be "time from connection start, to time I sent last bit". 
The receiver considers "time" to be "time from connection start, to time I got the last bit"
The receiver gets the last bit a little later than the sender sends it, so the denominator in the receiver throughput is a tiny bit bigger, contributing to a slightly higher BW.


### 4. Check that you did indeed increase the number of cores (on slice)

On all nodes (romeo, juliet, and router)
```
lscpu
```

# Analyzing results
To analyze results, we are plotting the experimental BW (from iperf in JupyterLab) against the model BW (from the calculations using the model), and comparing it to figure 3 in the paper. 

# Resources
[1] https://www.cs.unm.edu/~crandall/netsfall13/TCtutorial.pdf 

[2] https://manpages.ubuntu.com/manpages/xenial/man1/iperf3.1.html 

[3] https://witestlab.poly.edu/blog/tcp-congestion-control-basics/#setupexperiment 

[4] https://docs.vmware.com/en/vRealize-Operations/8.10/com.vmware.vcom.scg.doc/GUID-DAC867BC-8C5F-4A5E-BB55-36FC25555696.html 

[5] https://linuxhint.com/how-to-change-mtu-size-in-linux/ 

[6] https://ffund.github.io/tcp-ip-essentials/lab1/1-5-tcpdump-wireshark 

