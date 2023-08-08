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


For example, let's setup the following parameters: RTT = 6ms, p = 0.08202874901, MSS = 536 bytes

#### Setup
Router: (1Gbit bottleneck link rate [3], 0.1GB buffer on both sides [3], delay and loss [1] change depending on trial settings) 

<pre>
iface_0=$(ip route get 10.10.1.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_0 root
sudo tc qdisc add dev $iface_0 root netem delay 3ms 
iface_1=$(ip route get 10.10.2.100 | grep -oP "(?<=dev )[^ ]+")
sudo tc qdisc del dev $iface_1 root
sudo tc qdisc add dev $iface_1 root handle 1: htb default 3
sudo tc class add dev $iface_1 parent 1: classid 1:3 htb rate <b>1Gbit</b>
sudo tc qdisc add dev $iface_1 parent 1:3 handle 3: netem <b>delay 3ms loss 8.202874901% limit 100MB</b>
</pre>


#### Validate this experiment setup using ping:
##### Sending 1000 packets

Romeo:
<pre>
ping juliet -c <b>1000</b> -i 0.2
</pre>

##### Output:

<pre>
--- juliet ping statistics ---
1000 packets transmitted, 921 received, <b>7.9% packet loss</b>, time 200338ms
<b>rtt min/avg</b>/max/mdev = <b> 6.107/6.139</b>/6.360/0.017 ms
</pre>

### 2. Execute and Validate

#### Execute experiment using iperf 

Juliet: [3]
```
iperf3 -s  -1  
```

While that is running, paste the following in one Romeo terminal, Romeo_1: [3]
```
wget -O ss-output.sh https://raw.githubusercontent.com/ffund/tcp-ip-essentials/gh-pages/scripts/ss-output.sh
bash ss-output.sh 10.10.2.100  
```

While that is running, paste the following in the other Romeo terminal, Romeo_2: [2] (240s duration, TCP reno, MSS 536) 
```
iperf3 -c juliet -t 240 -C reno -M 536
```

When the process in Romeo_2 is done, bash will also stop running, but the process will not close, so you need to close it manually using ctrl+C. Then paste the following to see the output with tcp reno,
Romeo_1: [3]
```
cat sender-ss.txt | grep "reno"
```

#### Validating setup 

##### iperf output 
<pre>
[ID] Interval /t Transfer /t Bitrate /t Retr
[5] 0.00-240.00 sec /t 49.7 MBytes /t 1.74 Mbits/sec /t 8838
[5] 0.00-240.01 sec /t 49.6 MBytes /t <b>1.73 Mbits/sec </b>
</pre>

Experiment BandWidth= 1.73 Mbits/sec 

##### ss-output
<pre> 
sack reno wscale: 7,7 to: 212 <b>rtt:8.022</b> /2.821 <b>mss:536 </b> pmtu:1500 rcvmss:536 advmss:53
6 cwnd:4 ssthresh:2 bytes_ sent:56761901 bytes_retrans:4737168 bytes_ acked:52023126 segs_ out:105902 segs_in:5
8321 <b> data segs out:105900 </b> send 2138120bps lastsnd:4 lastrev: 240008 lastack:4 pacing rate 2565544bps delivery rate 1404056bps delivered:97060 busy: 240000ms unacked:3 <b> retrans: </b> 0/<b> 8838 </b> rcv_space:5360 rcv_ssthresh:65000 no tsent: 80936 <b> minrtt:6.056 </b>

</pre>


#### mss
mss=536 bytes

#### rtt or delay
rtt= 8.022 ms, minrtt= 6.056 ms

#### queueless environment
A queue forms when rtt>>minrtt. But since rtt= 8.022 ms is close enough to minrtt= 6.056 ms, we know that a queue didn’t form

#### packet loss
Packet loss=retrans/data_segs_out= 8838/105900 = 0.08345609065 , p = 0.08202874901


### 3. Identify Methodology Issues
Refer to "Run My Experiment" above (sections 1 and 2) to know where to implement the changes accordingly

#### Issue #1: Interpreting the packet loss parameter
##### Experiment settings to run:
Trial 5: RTT = 6ms, p=0.1566283969, MSS = 1460B

Command to implement setup: 
<pre>
netem delay 151.5ms <b>loss 0.1566283969%</b> limit 100MB
</pre>

##### Validate setup, what is wrong with the output?
Model BW = 4.92 Mbps, <b>Experiment BW = 207 Mbps</b>

<pre> 
—- juliet ping statistics —-
1000 packets transmitted, 999 received, <b>0.1% packet loss</b> time 200174ms
</pre> 


##### How to fix it: 
Issue 1:
The "loss" in netem is expressed as a percent, in the paper p is expressed as a ratio.

Command to fix issue:
<pre> 
netem delay 151.5ms <b>loss 15.66283969%</b> limit 100MB 
</pre>

##### Fixed output
Validating execution: Model BW = 4.92 Mbps, <b>Experiment BW = 1.68 Mbps</b>

<pre> 
—- juliet ping statistics —-
1000 packets transmitted, 838 received, <b>16.2% packet loss</b> time 200549ms
</pre> 

#### Issue #2: Setting a sufficient experiment duration
##### Experiment settings to run:
Trial 53: RTT = 600ms, p = 0.0000559121270201603, MSS = 1460B

Command to implement setup:
<pre>
iperf3 -c juliet -t <b> 60 </b> -C reno -M 1460
</pre>

##### Validate setup, what is wrong with the output?
Validating execution: Model BW= 2.60 Mbps, Experiment BW = 31.4/23.1/32.1/31.8/32.0 Mbps


##### How to fix it: 
Issue 2:
For small values of p, packet loss is a rare event - we need a longer experiment duration to approximate the "requested" packet loss.

Command to fix issue: 
<pre>
iperf3 -c juliet -t <b> 240 </b> -C reno -M 1460
</pre>

##### Fixed output:
Validating execution: Model BW= 2.60 Mbps, Experiment BW= 41.0 Mbps


<pre> 
ts sack reno wscale:7,7 rto:804 rtt:600.139/0.027 ato:40 mss:1448 pmtu:1500 rcmss:536 advmss: 1448 cwnd: 216 ssthresh :123 bytes_ sent: 433389053 bytes_retrans:21720 bytes_acked:433054566 bytes_ received:1 segs out:299362 segs_ in:77613 <b> data segs_ out:299359 </b> send 4169
274bps lastsnd:124 lastrev: 241248 lastack:44 pacing_rate 5003120bps delivery_rate 4150000bps delivered: 299129 busy: 240644ms rwnd_limited: 2704ms
(1.1%) sndbuf limited: 14020ms (5.8%) unacked: 216 <b> retrans:0/15 </b> cv space:14480 rcv ssthresh:64088 notsent: 2074712 minrtt:600
</pre>

Packet loss=retrains/data_segs_out= 15/299359=0.00005010706

#### Issue #3: Experiencing a queueless environment
##### Experiment settings to run:
Trial 11: RTT = 6ms, p = 0.000233840647, MSS = 4312B

Command to implement setup: 
<pre> 
htb rate <b>1Gbit </b>
</pre>

##### Validate setup, what is wrong with the output?
Validating execution: Model BW= 376 Mbps, Experiment BW= 964 Mbps

<pre>
sack reno wscale:7,7 rto:224 <b>rtt:21.726</b> /
0.218 mss: 4312 pmtu: 4352 revms
S:536 advmss: 4312 cwnd:626 ssthresh: 173 bytes_sent :28918425429 bytes_retrans: 6502496 bytes_acke d:28909227934 segs_out:6706970 segs_in:665326 data segs out:6706968 send 993947160bps lastrov:2
40008 pacing_rate 1192722864bps delivery_ rate 972939840bps delivered: 6704836 busy: 239992ms rwnd _limited:10128ms (4.28) snabuf_limited:492ms(0.28) unacked:625 retrans:0/1508 rcv_space: 43120 Ic ssthresh:61224 notsent: 517056 <b> minrtt:6.079 </b>
</pre>


##### How to fix it: 
Issue 3: Although the model does not predict BW exceeding 1Gbps, we observe this in practice, so we must increase the bottleneck rate to avoid queuing

Command to fix issue: 
<pre>
htb rate <b> 3Gbit </b>
</pre>

##### Fixed output:
Validating execution: Model BW= 376 Mbps, Experiment BW= 1.57 Gbps

<pre>
sack reno scale: 7,7 rto: 208 <b>rtt:6.27 </b>, 3.149 m55:4312 pmt:4352 rcmss:536 advmss:4312 cwnd:111 ssthre
sh:91 bytes_sent: 47177703213 bytes_retrans: 10210816 bytes_acked: 47167039638 segs_out:10941338 segs_in: 1333797 data_segs_out:10 941336 send 610694737bps lastrcv:240004 pacing_rate 732804464bps delivery_rate 566746984bps delivered: 10938864 busy: 240000ms r wnd limited:1508ms (0.6%) sndbuf_limited: 48ms (0.0%) unacked: 105 retrans: 0/2368 cv_space: 43120 rcv_ssthresh:61224 notsent: 23069 10
<b> minrtt:6.073 </b>
</pre>

#### Issue #4: Getting the requested MSS (1460B case)
##### Experiment settings to run:
Trial 17: RTT = 6ms, p = 0.03774108797, MSS = 1460B

Command to implement setup: 
<pre>
iperf3 -c juliet-t 240 - C reno -M <b> 1460 </b>
</pre>

##### Validate setup, what is wrong with the output?

Validating execution: Model BW= 389 kbps, Experiment BW= 600 kbps

<pre>
ts sack reno wscale: 7,7 to:356 rtt:154.626/0.012 <b>mss:1448</b> pmtu:1500 revmss:536 advmss:1448 cwnd:20 ssthresh:6 bytes sent:18631453 bytes_retrans: 603816
bytes_acked:17998678 bytes_received:1 segs_out:12870 segs_in:5245 data_segs_out: 12868 5 end 1498325bps lastsnd:40 lastrcv:240352 lastack:40 pacing rate 1797984bps delivery rate 1347976bps delivered:12432 busy: 240196ms unacked:20 retrans:0/417 rv space: 14480 rcv_ssthresh:64088 notsent:105704 minrtt: 154
</pre>


##### How to fix it: 
Issue 4: Hidden TCP timestamps option takes up 12 bytes from what we set mss to, so we must turn it off to get 1460B mss value.

Command to fix issue: 
<pre>
sudo sysctl -w net.ipv4.<b>tcp_timestamps=0 </b>
</pre>

##### Fixed output:
Validating execution: Model BW= 389 kbps, Experiment BW= 561 kbps

<pre>

sack reno wscale: 7,7 to: 368 rtt:167.759/19.504 ato:40 <b>mss:1460</b> pmtu: 1500 rcmss:536 advmss:
1460 cwnd:5 ssthresh:5 bytes_sent:17492297 bytes_retrans:651160 bytes_acked: 16833838 bytes_received:1 segs_out: 11985 segs_in:5286 data_ segs_out:11982 send 348118bps lastsnd:92 lastrev: 240360 lastack:48 pacing_rate 417736bps delivery_r ate 289888bps delivered:11532 busy: 240204ms unacked:5 retrans:0/446 rev_space:14600 rev_ssthresh:64076 notsent: 2087801
minrtt:154.561

</pre>

#### Issue #5: Getting the requested MSS (4312B case)
##### Experiment settings to run:
Trial 9: RTT = 6ms, p = 0.006217314741, MSS = 4312B

Command to implement setup: 
<pre> 
iperf3 -c juliet - t 240 - C reno -M <b>4312 </b>
</pre>


##### Validate setup, what is wrong with the output?
Validating execution: mtu=1500B, Model BW= 72.9 Mbps, Experiment BW= 68.5 Mbps

<pre>
sack reno wcale:7,7 rto:208 rtt: 6.146/0.039
<b>mss:1460 </b> mtu: 1500 rcvmss: 536 ad
vmss:1460 cwnd:68 ssthresh:7 bytes_sent:2068078357 bytes_retrans: 12995460 bytes_acked: 2054989458 segs. out:1416496 Segs_in: 260893 data_segs_out:1416494 send 129228767bps lastrev: 240012 pacing_rate 15507452
bps delivery_rate 1181337686ps delivered: 1407530 busy: 240084ms rwnd_limited:8ms(0.0%) unacked:64 retr ans:0/8901 rev_space:14600 cv_ssthresh: 64076 notsent: 1220560 mint: 6.063
</pre>

##### How to fix it: 
Issue 5: Default Ethernet MTU is 1500B, to have MSS > 1460 we need to increase MTU also.

Command to fix issue: 
<pre> 
sudo ifconfig ens7 <b>mtu 4352 </b> up
</pre>

##### Fixed output:
Validating execution: mtu=4352B, Model BW= 72.9 Mbps, Experiment BW= 161 Mbps

<pre>
sack reno wscale:7,7 rto:208 rtt:6.487/0.63
<b>mss: 4312 </b> omtu: 4352 rcmss:536 advmss: 4312
cwnd: 10 ssthresh:5 bytes_sent: 4857045461 bytes_retrans:30501400 bytes_acked: 4826499254 segs_ou t:1126406 segs_in:311651 data_segs_out: 1126404 send 53177123bps lastrev:240008 pacing rate 6380
8856bps delivery_rate 45085440bps delivered:1119321 busy: 239984ms rwnd limited: 12ms (0.08) unack ed: 9 retrans: 0/7075 rev space: 43120 rcv ssthresh:61224 notsent:2922376 minrtt:6.068
</pre>

#### Issue #6: Getting the requested MSS (all cases)
##### Experiment settings to run:
Trial 3: RTT = 6ms, p = 0.001591398645, MSS = 536B

Command to implement setup: 
<pre>
iperf3 -c juliet-t 240 - C reno-M <b>536</b>
</pre>


Try 'tcpdump' on the router while the normal iperf is running.

Router: [6]
<pre>
sudo <b>tcpdump</b> -i ens7
</pre>

When it is done scroll up and notice how the packet size is double, triple, or even quadruple the mss value that you set (try it with 536, 1460, and 4312). 

##### Validate setup, what is wrong with the output?
Validating execution: Model BW= 17.9 Mbps, Experiment BW= 104 Mbps, Exp/Model BW=5.81

<pre>
19:35:25.382321 IP romeo.41376 > juliet.5201: Flags [P.], seg 52692552:52694160, ack 1, win 511, length <b>1608</b>
19:35:25.382321 IP romeo.41376 > juliet.5201: Flags [P.], seq 52694160:52695232, ack 1, win 511, length <b>1072</b>
19:35:25.388391 IP juliet.5201 > romeo.41376: Flags [.], ack 52682904, win 12392, options [nop, nop, sack 1 {52683976:52692552}], length 0
19:35:25.388444 IP romeo.41376 > juliet.5201: Flags [.], seq 52682904:52683440, ack 1, win 511, length <b>536</b>
</pre>


##### How to fix it: 
Issue 6: NIC has a segment offload feature, combines segments en route to same destination - unless we turn it off, "effective MSS" is higher than what TCP thinks it is.
Because of the direct proportionality between BW and MSS in the model, when MSS is very high (aggregated), it results in the high BW values that we were getting. So we can turn off this segment offloading feature to prevent segments from aggregating and get reasonable experimental BW values.


Command to fix issue:

Run 'Turn segment offloading off' in setup.ipynb that has the following code:
```
for iface in slice.get_interfaces():
    iface_name = iface.get_device_name()
    n = iface.get_node()
    offloads = ["gro", "lro", "gso", "tso"]
    for offload in offloads:
        n.execute("sudo ethtool -K %s %s off" % (iface_name, offload))
```


##### Fixed output:
Validating execution: Model BW= 17.9 Mbps, Experiment BW= 20.9 Mbps, Exp/Model BW=1.17

<pre>
19:35:25.511521 IP romeo.41376 › juliet.5201: Flags [.], seq 52792248:52792784, ack 1, win 511, length <b> 536 </b>
19:35:25.511611 IP romeo.41376 > juliet.5201: Flags [.], seq 52792784:52793320, ack 1, win 511, length <b> 536 </b>
19:35:25.517596 IP juliet.5201 > romeo. 41376: Flags [.], ack 52787960, win 12392, options [nop, nop, sack 1 {52788496:52792784}], length 0
19:35:25.517645 IP romeo.41376 > juliet.5201: Flags [.], seq 52787960:52788496, ack 1, win 511, length <b> 536 </b>
19:35:25.517664 IP juliet.5201 > romeo.41376: Flags [.], ack 52787960, win 12392, options [nop, nop, sack 1 {52788496:52793320}], length 0
19:35:25.517779 IP romeo.41376 › juliet.5201: Flags [.], seq 52793320:52793856, ack 1, win 511, length <b> 536 </b>
</pre>


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
To analyze results, we are plotting the experimental BW (from iperf) against the model BW (from the calculations using the model), and comparing it to figure 3 in the paper. The closer the set of points are to the x=y line, the better the results are. 


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


# Resources
[1] https://www.cs.unm.edu/~crandall/netsfall13/TCtutorial.pdf 

[2] https://manpages.ubuntu.com/manpages/xenial/man1/iperf3.1.html 

[3] https://witestlab.poly.edu/blog/tcp-congestion-control-basics/#setupexperiment 

[4] https://docs.vmware.com/en/vRealize-Operations/8.10/com.vmware.vcom.scg.doc/GUID-DAC867BC-8C5F-4A5E-BB55-36FC25555696.html 

[5] https://linuxhint.com/how-to-change-mtu-size-in-linux/ 

[6] https://ffund.github.io/tcp-ip-essentials/lab1/1-5-tcpdump-wireshark 

