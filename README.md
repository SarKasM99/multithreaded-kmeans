# Purpose
This project solves k-means clustering problem by using the Lloyd's algorithm. It was implemented as part of a university course (LEPL1503 - Projet 3) in C language and it uses multithreading to solve a computational problem. The project was designed for a Unix-like system (more specifically for a **Raspberry Pi OS**) and uses GCC as its compiler. 

# Project overview

The program first reads the data from a binary file, where: 
* The first number is an **unsigned 32 bits int** that represents the dimension of the points used 
* The second number is an **unsigned 64 bits int** that represents the number of points in the data 
* The rest is a succession of **signed 64 bits int** that gives the coordinates of each point 
> The data in the binary file needs to be encoded in big endian order. The program will later convert to host byte order. 

After the binary file has been read, the program applies the Lloyd's algorithm and tests every possible combinations of initialisation of the clusters among the n initialisation of points that is specified by the user (see **Usage**). 

After solving the clustering problem with Lloyd's algorithm, the program stores the result in a *CSV* file of each initialisation. Note that the program does not store in any particular order between the lines since the program uses threads in order to solve the problem faster. Here is an example of how an initialisation is stored: 
* First line is always as follows: 
 `initialization centroids,distortion,centroids,clusters` 
* And the rest are the results, where each quotation represents the initialization centroids, distortion, centroids and data in each cluster (respectively to the order of centroids): \
`"[(1, 1), (2, 2)]",11,"[(1, 1), (4, 5)]","[[(1, 1), (2, 2)], [(3, 4), (5, 7), (3, 5), (5, 5), (4, 5)]]"`

Those binary files can be generated by a python script written by one of the university assitants. This script can be found [here](https://github.com/louisna/lepl1503-2022-pyfec). 

# Requirements
The project requires the Pthread API library to run and the CUnit library for its unit tests.

# Usage
The makefile has the following commands:
* `make kmeans` - compiles and creates an executable file to solve the clustering problem 
* `make tests` - does kmeans, then compiles our units tests and executes them 
* `make clean` - removes all executables and object files 
* `make memcheck` - launches valgrind memory checks with helgrind

Here are the parameters that can be used with the executable file (./kmeans): \
**-q** if used, the last column (the data points belonging to the cluster) in the csv file will not be shown \
**-k n_clusters** (default: 2) specifies the numbers of clusters  \
**-p n_combinations** (default: 2) specifies the n first points that needs to be tested for the clustering problem. 
> For example, if n_clusters = 2 and n_combinations = 7, then the program will test C(n_combinations,n_clusters) combinations.

**-n threads** (default: 4) specifies the number of threads that applies Lloyd's algorithm \
**-d distance_metric** (default: "manhattan") it's either "manhattan" or "euclidean". Specifies the distance formula that will be used \
**-f output_file** (default: *stdout*) the path of the csv file where the results will be stored \
**input_filename** (default: *stdin*) the path of the binary file that contains the data that needs to be processed. 

### Usage example:
`./kmeans -n 4 -p 10 -k 2 -f example.csv test/example_dim3.bin`


# Multithreading architecture 
The solution is based on the 'consumer and producer' strategy. The producer (which is the main thread) generates combinations of points (C(n,k) where
n is the number of points of initialisation and k is the number of clusters). The consumers (pthreads) takes those points from the buffer, applies Lloyd's algorithm and writes the resutls in the requested output file. 

The producer and the consumers communicates via a semaphor in order to give signals when the buffer is either full or empty. The producer will not stop producing until all the points are generated. Each consumer thread will keep computing until there are no points left in the buffer. 

The concurrency between producers are controlled by two mutex: \
`pthread_mutex_t mutex;` \
`pthread_mutex_t write;`


Mutex `write` locks the process of writting in the output file. Therefore only one thread at a time can write in this file. 

Mutex `mutex` locks the process of taking a point from the buffer. First it locks and checks if there is any elements left in the buffer 
(the number of items in the buffer is calculated by applying a combination algorithm and then decremented each time a thread takes a point). 

If there still an item to take, the consumer will communicate with the producer (with `sem_wait(&empty)`) to check if the set of points that the consumer wishes to obtain is ready. After the consumer got his points, he unlocks the mutex and signals the producer once he finished copying his set of points. 

If there is no item left to take, the consumer unlocks the mutex and exits the while loop.

# Perfomance
Here are some performance tests to show the speed-up of the threads. Those test were done on a **Raspberry Pi** with a binary file that contains 200 points.

Tests -k 2 -p 25 -d euclidian:
- 1 thread  : 0.1752s
- 2 threads : 0.0966s
- 3 threads : 0.065s
- 4 threads : 0.0562

Tests -k 2 -p 85 -d euclidian:
- 1 thread  : 2.0136s
- 2 threads : 1.0176s
- 3 threads : 0.7242s
- 4 threads : 0.5342s

Tests -k 2 -p 200 -d euclidian:
- 1 thread  : 11.0078s
- 2 threads : 5.6064s
- 3 threads : 3.7928s
- 4 threads : 2.9104s


