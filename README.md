# ReproducingFoundationalResult
Reproducing a foundational result to find underlying methodological issues 

# Setup on Fabric
Create an account on Fabric and familiarize yourself with it on https://teaching-on-testbeds.github.io/hello-fabric/ \\
Run the following Fabric file until the end of 'Exercise: Log in to resources': https://jupyter.fabric-testbed.net/hub/user-redirect/lab/tree/CongestionAvoidance/setup.ipynb 
Open 4 new terminals on Jupyter where you paste the SSH command for 2 romeo terminals, 1 juliet terminal, and 1 router terminal
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/0218e560-2bfe-4098-9b8d-b43d3315b422)

# Our purpose
Our main goal is to reproduce research from the paper titled The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm:
https://cseweb.ucsd.edu/classes/wi01/cse222/papers/mathis-tcpmodel-ccr97.pdf. We are reproducing research to create educational material for students and professors by helping them understand underlying methodological issues by running the experiments from the paper and matching its settings as closely as possible.
The paper's primary purpose is to 
1. Derive a model for TCP to predict BandWidth (BW)
2. Validate this foundational result through three primary network environments: Queueless Random Packet Loss, Environments with Queueing (not random), Effect of TCP Implementation.

# First Environment Setup
In this document, we are focusing on reproducing results from the first environment: Queueless Random Packet Loss. In this environment, they validate the BW-predicting model by measuring BW under the assumption that no queue is formed and packet losses are random. The foundational result that we are validating involves varying three parameters (Delay, MSS, packet loss (p)) to produce BW. One-way delay takes the following values 3, 77.25, 151.5, 225.75, and 300 ms. MSS takes 536, 1460, and 4312 bytes. Sine Figure 3 has approximately 60 points in total, we conclude that the number of uniformly distributed values of packet loss (in log(p)) are 4: 5 delay * 3 MSS * 4 p = 60 BW points). For every combination of MSS and delay, generate 4 uniformly distributed values of p between 0.00003 and 0.3.

This is the network topology
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/b300d43c-7884-468e-9782-6ec439dadae0)

#First, we don't want a queue to form. Check the maximum BW in Figure 3 of the paper. If we don't want a queue to form, then the bottleneck link rate must be greater than the maximum possible BW. The maximum BW in the plot is around 2e8 bits/s, so we will set the bottleneck link rate to 1Gbit with  0.1GB buffer on both sides of the router (towards romeo and towards juliet)![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/25700a2f-5861-4e8e-b7dd-083d72e475d5)
