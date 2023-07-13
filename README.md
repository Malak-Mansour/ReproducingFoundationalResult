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
In this document, we are focusing on reproducing results from the first environment: Queueless Random Packet Loss. In this environment, they validate the BW-predicting model by assuming that no queue is formed and setting random packet losses.










# Experiment run
1Gbit bottleneck link rate [3], 0.1GB buffer on both sides [3], delay and loss [1] change depending on trial settings
