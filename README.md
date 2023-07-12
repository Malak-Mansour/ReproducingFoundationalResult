# ReproducingFoundationalResult
Reproducing a foundational result to find underlying methodological issues 

# Setup
Create an account on Fabric and familiarize yourself with it on https://teaching-on-testbeds.github.io/hello-fabric/ \\
Run the following Fabric file until the end of 'Exercise: Log in to resources': https://jupyter.fabric-testbed.net/hub/user-redirect/lab/tree/CongestionAvoidance/setup.ipynb 
Open 4 new terminals on Jupyter where you paste the SSH command for 2 romeo terminals, 1 juliet terminal, and 1 router terminal
![image](https://github.com/Malak-Mansour/ReproducingFoundationalResult/assets/73076958/0218e560-2bfe-4098-9b8d-b43d3315b422)

# Experiment explanation
In this document, we are running the first environment of the following paper https://cseweb.ucsd.edu/classes/wi01/cse222/papers/mathis-tcpmodel-ccr97.pdf. 
The paper's main purpose is to 
1. Derive a model for TCP to predict BandWidth (BW)
2. Validate this foundational result through three main network environemnts.

If you read the paper, you will notice that the first environment that they use to validate the BW-predicting model is through a queueless random packet loss experiment. In other words, they assume no queue formed, and emulate random packet losses.











# Experiment run
1Gbit bottleneck link rate [3], 0.1GB buffer on both sides [3], delay and loss [1] change depending on trial settings
