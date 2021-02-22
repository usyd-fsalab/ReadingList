# Application of BNN ANL (from Dr. Emani)- Low Latency Gravitational

[<u>1903.01998.pdf</u>](https://arxiv.org/pdf/1903.01998.pdf)

### Summary:

- Paper  uses Tensorflow as its framework

- Uses  two NN, one for measuring masses and one to measure spin and QNM since they are within the same range

- Design of this network then follows a “shared root component for all physical parameters and three leaf components for individual parameters”

- The root contains 7 convolutional layers shared by the leafs which follows the idea of **hierarchical  self decomposing CNNs** (essentially there is a class specific filter process that determines the impact of certain features, which can then be used to retrieve sub-networks from this output). This means that we can have a sub-network trained on a specific subset of the data.

- There is also a removal of the non-linear activation function in the 2nd last layer which is used to allow more neurons to be activated and pass to the final layer, which in turn smoothes the gradients allowing more information through.

- ![](/home/windows8/Documents/University/research/readings/img/2021-02-17-15-46-33-image.png)

#### Probablistic Model:

- Use prior and posterior functions on the last two layers of the leaf, which makes each of the leaves an independent probabilistic model.

- The root is seen as a method of feature extraction.

- **Two assumptions: likelihood is assumed to be Gaussian, and neural
   network weight distribution is independent Gaussians. This is designed in a way to simplify the process of VI and ensure it is tractable.**

- Uses the Variational Inference algorithm with a Gaussian Distribution.

- And SGD for estimating the value of $\theta$ for the mean field approximation, $q_\theta(w)$ through minimising the value of our cost function.

- The posterior distribution then follows:  

- ![](/home/windows8/Documents/University/research/readings/img/2021-02-17-15-48-04-image.png)

- #### Dataset Preparation:
  
  - **Batch size of 64,** $\alpha = 0.0008$ **and the total number of iterations =$120,000$**
  
  - **Dropout rate of $0.1$ for training and none for testing or validation.**

- They use an early stopping criterion with the relative error threshold =
   0.026 for masses and $0.0016$.
   Moreover they scale the mass model to make optimisation scale faster
   (I assume this is because the values are closer together and hence
   the threshold will be reached sooner.)

- #### Scaling
  
  - Probabilistic layers required twice the number of parameters to be optimised
  
  - Scaling involved running on Cray XC40 system, Theta at ANL. Used KNL which
     is an optimised version build of Tensorflow and Horovod for
     Xeon-Phi.
  
  - Uses one MPI rank per node and 128 hardware threads per node
  
  - Achieves approx 75% efficiency up to 1024 nodes.
  
  - **Since there are twice the parameters in BNN layers then there is a
     greater overhead of communication costs.**
  
  - Also run on HAL cluster using 4 NVIDIA V100 GPUs with batch size of 64.

![](/home/windows8/.config/marktext/images/2021-02-17-15-42-38-image.png)

- As shown above the ideal values as the nodes are scaled increase as due
   to the increased complexity of communication overhead.

### 
