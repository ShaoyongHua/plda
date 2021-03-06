#Thanks for reading README.

We improve plda on it's loglikelihood compution. We implement it as microsoft lightlda algorithm. The the motive for us to do this due to doc likelihood is decreasing by the iteration in origin code. We did many experience comfirmed it. You can test as much as you like.
###if you want to print out loglikehood, remember add "--compute_likelihood true" in command line.

There are explain of loglikelihood:
https://github.com/Microsoft/lightlda/issues/9

Let me try to make it more clear. Computing the total likelihood will need compute on all the data and model, which is prohibitively expensive. In a data-parallel distributed setting, the data is distributed across several workers and model is distributed stored across several servers.

To compute the likelihood efficiently, we decompose the total likelihood to three parts: doc-likelihood(only concern document, can compute without access of global shared model), word-likelihood(only concern about word-topic-table, can compute with only very small part of model) and normalized item in word-likelihood(only concern about the summary row).

Each worker computes the doc-likelihood on part of documents it holds, and word-likelihood of part of model it needs. Thanks to the decomposition, the computation on each worker can be also finished with multi-threads in a data-parallel fashion.

Then the total likelihood on the whole dataset should be the summation of : 1) all the doc-likelihood on each machines. 2) all the word-likelihood of each words in vocabulary. 3) the normalized likelihood.

Introduction
    plda+ must be run in linux environment with mpich2-1.5.  This is a beta version and you can report issues via https://code.google.com/p/plda/issues/list .
	
	plda+ is a new parallel algorithm for lda based on mpi, which is entirely different from plda. The detailed introduction to the algorithm of plda+ can be found in the paper: "PLDA+: Parallel Latent Dirichlet Allocation with Data Placement and Pipeline Processing. Zhiyuan Liu, Yuzhou Zhang, Edward Y. Chang, Maosong Sun. ACM Transactions on Intelligent Systems and Technology, special issue on Large Scale Machine Learning. 2011."

Installation
    * tar -xvfz plda+.tar.gz
    * make mpi_lda infer
    * You will see two binary files 'mpi_lda' and 'infer' in the folder

Prepare datafile
    * Data is stored in a sparse representation, with one document per line. Each line consists of the words and their counts. For example,
        <word1> <word1_count> <word2> <word2_count> ...
      Each word is a string without any space, newline or other special characters.

Training parallelly
    * First, make sure that every parallel node has the same copy of data file and executable file (i.e. mpi_lda).
    * Train: e.g.
        mpiexec -n 6 ./mpi_lda \
        --num_pw 2 \
        --num_topics 2 \
        --alpha 0.1 \
        --beta 0.01 \
        --training_data_file testdata/test_data.txt \
        --model_file testdata/lda_model \
        --total_iterations 150
    * After finish training, there will be num_pw model files (here named as lda_model_XXX, where XXX = 0,1,...,num_pw-1). Please collect all of the model files and concatenate them into a whole model file.

Inferring unseen documents
    * Prepare datafile in the same way of training datafile.
    * Infer: e.g.
        ./infer \
        --alpha 0.1 \
        --beta 0.01 \
        --inference_data_file testdata/test_data.txt \
        --inference_result_file testdata/inference_result.txt \
        --model_file testdata/lda_model.txt \
        --total_iterations 15 \
        --burn_in_iterations 10

Command-line flags
    Training flags:
        * num_pw: The number of pw processors, which should be greater than 0 and less than the total number of processors (here is 6). Suggested to be number_of_processors/3
        * num_topics: The total number of topics
        * alpha: Suggested to be 50/number_of_topics
        * beta: Suggested to be 0.01
        * total_iterations: The total number of GibbsSampling iterations
        * training_data_filel: The training data file
        * model_file: The output file of the trained model
    Inferring flags:
        * alpha and beta should be the same with training.
        * total_iterations: The total number of GibbsSampling iterations for an unseen document. This number does not need to be as much as training, usually tens of iterations is enough.
        * burn_in_iterations: For an unseen document, we will average the document-topic distribution of the last (total_iterations - burn_in_iterations) iterations as the final document-topic distribution.