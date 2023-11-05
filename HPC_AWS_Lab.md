# HPC: Azure-Cycle-Cloud-vs-AWS-Parallel-Cluster

## Overview
In this tutorial, you will go through the basics of using AWS Parallel Cluster. This tutorial will walk you through how to create and connect to a HPC cluster in AWS and a use case on how to submit an HPC job. We will be using the problem of finding the smallest prime factor as an example

### Step 0: Before You Start 
* Sign up for a free AWS account here [AWS Account Sign Up](https://aws.amazon.com/free/?trk=7d839240-0f22-461a-924a-bd9f4c9f3138&sc_channel=ps&s_kwcid=AL!4422!10!71399763847723!71400284985686&ef_id=bc458f06349b10643121eb2a92f9869c:G:s&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all)

### Step 1: Deploy ParallelCluster UI - 20 minutes to deploy
* Create UI stack [here](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=parallelcluster-ui&templateURL=https://parallelcluster-ui-release-artifacts-us-east-1.s3.us-east-1.amazonaws.com/parallelcluster-ui.yaml)
* Update the field AdminUserEmail with a valid email to receive a temporary password in order to connect to the ParallelCluster UI GUI
![](parallelClusterUI-addEmail.png)
* Leave the other default values and proceed through the next few pages till you are at stage 4
* At stage 4 click the two tick boxes to create new IAM resources
* Then click **Create stack **
![](ParallelClusterUI_Aknowledgement.png)
* Wait about 20 minutes for it deploy!
### Step 2: Connect to ParallelCluster UI 
* Go to the AWS Console, log in and in the search box search for AWS CloudFormation and click on that service: [link to console](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks?filteringText=&filteringStatus=active&viewNested=true)
* You’ll see a stack named **parallelcluster-ui**, click on that **stack > Outputs Tab**
* Then click on the **ParallelClusterUI URL** to connect
![](ParallelClusterUI-Connect.png)
* You will see a prompt to enter your email and password
![](password.png)
    * During deployment you should have received an email titled [AWS ParallelCluster UI] "Welcome to ParallelCluster UI, please verify your account" **Copy** the password from that email.
    * Enter the credentials using the email you used when deploying the stack and the temporary password from the email above
    * You will be asked to provide a new password. Enter a new password to complete signup
### Step 3: Get to the AWS ParallelCluster page - Repeat first 3 bullet points of step 2 and refer to image above
* Go to the AWS Console, log in and in the search box search for AWS CloudFormation and click on that service: [link to console](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks?filteringText=&filteringStatus=active&viewNested=true)
* You’ll see a stack named **parallelcluster-ui,** click on that **stack > Outputs Tab**
* Then click on the **ParallelClusterUI URL** to connect
### Step 4: Create your Cluster - 10-15 minutes to deploy
* **Save this template**: https://www.hpcworkshops.com/template/cluster-config-hpc6a.yml
* Click **Create Cluster** Button and **with a template** and then open the template above
![](CreateCluster_ClickTemplate.png)
* **Cluster properties** page
  * Add **name**: hpc-cluster
  * Select **VPC** and click whatever is available
  * Click **Next**
![](ClusterProperties.png)
* **Head Node** page - screenshot slightly different than current page but same inputs required
  * Pick the **subnet id** from the Availability Zone ID **use2-az2**
  * Click **Key pair** and choose None (make sure to click None or it might not work)
  * Click **Next**
![](createCluster_HeadNode.png)
* **Queues**
  * At **subnet IDs** click the same subnet id you chose above
  * Click **Next**
![](Queues.png)
* **Storage**
   * Click **Next**
* **Create**
  * Click **Dry run** and make sure it passes that
  * Click **create**!
![](CreateCluster_Create.png)
* Wait about 10-15 minutes for the cluster to go into **CREATE COMPLETE** and the compute fleet status to be running!

### Step 5: Connect to the cluster using DCV
* Once the cluster goes into **CREATE COMPLETE** we can connect to the head node
* Click on **DCV**
![](ConnectCluster_DCVpt1.png)
* As a one time step since DCV uses self-signed certificates you'll need to click on **Advanced> Proceed to Unsafe**
![](ConnectCluster_DCVpt2.png)
* Next to Launch a terminal we'll click **activities** and then **Terminal**
![](ConnectCluster_DCVpt3.png)

### Step 6: Submit HPC job using control code for smallest prime factor
1. To demonstrate ParallelCluster's HPC capabilities, we will first write a program to calculate the smallest prime factor of a large integer.
It will run the function `calcMinPrimeFactor` on the same input 4 times within a single ParellelCluster process.

Within the ***DCV terminal***, copy-paste and run the following to save the program to the file `mpi_control_least_prime_factor.c`
```
cat > mpi_control_least_prime_factor.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <unistd.h>
#include <time.h>

// Function declarations
unsigned long long int calcMinPrimeFactor(unsigned long long int n);

int main(int argc, char* argv[]) {
    // MPI initialization
    int process_rank,size_of_comm;

    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &size_of_comm);
    MPI_Comm_rank(MPI_COMM_WORLD, &process_rank);

    unsigned long long int taskInputs[] = { 18848997157, 18848997157, 18848997157, 18848997157 };
    unsigned long long int taskOutputs[4];
    int i;

    // Benchamrk runtime
    clock_t t;
    t = clock();

    for (i = 0; i < 4; i++) {
        taskOutputs[i] = calcMinPrimeFactor(taskInputs[i]);
    }

    t = clock() - t;
    double time_taken = 1000000 * ((double) t) / CLOCKS_PER_SEC; // calculate the elapsed time

    // Print output and runtime
    for (i = 0; i < 4; i++) {
        printf("Minimum prime factor of %d is %d\n", taskInputs[i], taskOutputs[i]);
    }
    printf("Time taken: %.2f micro-seconds\n", time_taken);// calculate the elapsed time

    // MPI finalization
    MPI_Finalize();

    return 0;
}

unsigned long long int calcMinPrimeFactor(unsigned long long int n) {
    // Function implementation
    unsigned long long int i;

    // Iterate from 2 to n/2
    for (i = 2; i <= n/2; i++) {
        // Check if i is a factor of n
        if (n % i == 0) {
            // Check if i is prime
            unsigned long long int j, is_prime = 1;
            for (j = 2; j <= i/2; j++) {
                if (i % j == 0) {
                    is_prime = 0;
                    break;
                }
            }
            if (is_prime) {
                // Found min prime factor
                return i;
            }
        }
    }

    // If no prime factor was found, n is prime
    return n;
}
EOF
```

2. Then, run the following to load the required modules and compile the application
```
module load intelmpi
mpicc mpi_control_least_prime_factor.c -o mpi_control_least_prime_factor
```

3. Execute the application on the head node using a single process (specified by `-n 1`)
```
mpirun -n 1 ./mpi_control_least_prime_factor
```

This should print out the computed least prime factor, and give you a runtime of the process as follows:
```
<CODE OUTPUT GOES HERE>
```

### Step 7: Submit HPC job on finding minimum prime factor
1. Now, we'll do the same using MPI code designed to run in parallel across multiple concurrent processes.
It will run the function `calcMinPrimeFactor` on the 4 inputs scattered across 4 concurrent parallel processes.

Within the ***DCV terminal***, copy-paste and run the following to save the program to the file `mpi_parallel_least_prime_factor.c`
```
cat > mpi_parallel_least_prime_factor.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <unistd.h>
#include <time.h>

// Function declarations
unsigned long long int calcMinPrimeFactor(unsigned long long int n);

int main(int argc, char* argv[]) {
    // MPI initialization
   
    int process_rank, size_of_comm;
    unsigned long long int taskInputs[4] = {18848997157, 18848997157, 18848997157, 18848997157};
    unsigned long long int subtaskInput;

    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &size_of_comm);
    MPI_Comm_rank(MPI_COMM_WORLD, &process_rank);

    MPI_Scatter(&taskInputs, 1, MPI_UNSIGNED_LONG_LONG, &subtaskInput, 1, MPI_UNSIGNED_LONG_LONG, 0, MPI_COMM_WORLD);

    // Call your functions within the MPI program
    clock_t t;
    t = clock(); // start timer
    unsigned long long int minPrimeFactor = calcMinPrimeFactor(subtaskInput);
    t = clock() - t;
    double  time_taken =1000000 * ((double) t) / CLOCKS_PER_SEC; // calculate the elapsed time
    printf("Process %d: Time taken: %.2f micro-seconds\n", process_rank, time_taken);// calculate the elapsed time
    printf("Process %d: Minimum prime factor of %d is %d\n", process_rank, subtaskInput, minPrimeFactor);

    // MPI finalization
    MPI_Finalize();

    return 0;
}

unsigned long long int calcMinPrimeFactor(unsigned long long int n) {
  // Function implementation
  // You can use MPI features in your function if needed
  unsigned long long int i;

  // Iterate from 2 to n/2
  for (i = 2; i <= n/2; i++) {
      // Check if i is a factor of n
      if (n % i == 0) {
          // Check if i is prime
          unsigned long long int j, is_prime = 1;
          for (j = 2; j <= i/2; j++) {
              if (i % j == 0) {
                  is_prime = 0;
                  break;
              }
          }
          if (is_prime) {
              // Found min prime factor
              return i;
          }
      }
  }

  // If no prime factor was found, n is prime
  return n;
}
EOF
```

2. Then, run the following to load the required modules and compile the application
```
module load intelmpi
mpicc mpi_parallel_least_prime_factor.c -o mpi_parallel_least_prime_factor
```

3. Execute the application on the head node using 4 processes (specified by `-n 4`)
```
mpirun -n 4 ./mpi_parallel_least_prime_factor
```

This should print out the computed least prime factor, and give you a runtime of each of the processes as follows:

```
<CODE OUTPUT GOES HERE>
```

### Step 8: Compare your two runtimes!
* The runtime to find the smallest prime factor for four numbers using parallel computing should take signifcantly shorter amount of time than the runtime to find the same problem without it.

### Step 9: Clean up!
* Make sure to go back to the AWs Parallel Cluster page and hit the delete button and delete your node!
* The cluster and all its resources will be deleted by CloudFormation. You can check the status in the Stack Events tab.
![](TerminateCluster.png)
