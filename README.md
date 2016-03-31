# QMF - a matrix factorization library

[![Build Status](https://travis-ci.org/quora/qmf.svg?branch=master)](https://travis-ci.org/quora/qmf)
[![Hex.pm](https://img.shields.io/hexpm/l/plug.svg)](LICENSE)

## Introduction

QMF is a fast and scalable C++ library for implicit-feedback matrix factorization models. The current implementation supports two main algorithms:

* **Weighted ALS** [1]. This model optimizes a weighted squared loss, and thus allows you to specify different weights on each positive example. The algorithm is based on alternating minimization on user and item factors matrices. QMF uses efficient parallelization to perform these minimizations.
* **BPR** [2]. This model (approximately) optimizes average per-user AUC using stochastic gradient descent (SGD) on randomly sampled (user, positive item, negative item) triplets. Asynchronous, parallel Hogwild! [3] updates are supported in QMF to achieve near-linear speedup in the number of processors (when the dataset is sparse enough).

For evaluation, QMF supports various ranking-based metrics that are computed per-user on test data, in addition to training or test objective values.

## Building QMF

QMF requires gcc 4.9+, as it uses the C++14 standard, and CMake version 2.8+. It also depends on glog, gflags and lapack libraries.

### Ubuntu

To install libraries dependencies:
```
sudo apt-get install libgoogle-glog-dev libgflags-dev liblapack-dev
```

To build the binaries:
```
cmake .
make
```
To run tests:

```
make test
```

Output binaries will be under the `bin/` folder.

## Usage

Here's a basic example of usage:
```
# to train a WALS model
./wals \
    --train_dataset=<train_dataset> \
    --user_factors=<user_factors_file> \
    --item_factors=<item_factors_file> \
    --regularization_lambda=0.05 \
    --confidence_weight=40 \
    --nepochs=10 \
    --nfactors=30 \
    --nthreads=4

# to train a BPR model
./bpr \
    --train_dataset=<train_dataset> \
    --test_dataset=<test_dataset> \
    --user_factors=<user_factors_file> \
    --item_factors=<item_factors_file> \
    --nepochs=10 \
    --nfactors=30 \
    --num_hogwild_threads=4 \
    --nthreads=4
```
The input dataset files should adhere to the following format:
```
<user_id1> <item_id1> <weight1>
<user_id2> <item_id2> <weight2>
...
```
where `weight` is always `1` in BPR, but can be any integer in WALS (`r_ui` in the paper [1]).

The output files will be in the following format:
```
<{user|item}_id> [<bias>] <factor_0> <factor_1> ... <factor_k-1>
...
```
where the bias term will only be present for BPR item factors when the `--use_biases` option is specified.

In order to compute test ranking metrics (averaged per-user), you can add the following parameters to either binary:
* `--test_avg_metrics=<metric1[,metric2,...]>` specifies the metrics, which include `auc` (area under the ROC curve), `ap` (average precision), `p@k` (e.g. `p@10` for precision at 10), `r@k` (recall at k)
* `--num_test_users=<nusers>` specifies the number of users to consider when computing test metrics (by default 0 = all users). Computing these metrics requires computing predicted scores for all items and test users, which can be slow as the number of user gets big. The users are picked uniformely at random with a fixed seed (which can be specified with `--eval_seed`)
* `--test_always` will compute these metrics after each epoch (by default they're computed only after the last epoch)

In the case of BPR, a dataset of (user, positive item, negative item) triplets is sampled during initialization for both training and test sets (with a fixed seed, or as given by `--eval_seed`), and is used to evaluate an estimated loss after each epoch.

## Credits

This library was built at Quora by [Denis Yarats](https://github.com/1nadequacy) and [Alberto Bietti](https://github.com/albietz).

## License
QMF is released under the [Apache 2.0 Licence](https://github.com/quora/qmf/blob/master/LICENSE).

## References

[1] Koren et al. “Collaborative Filtering for Implicit Feedback Datasets”.  ICDM 2008.

[2] Rendle et al. “BPR: Bayesian Personalized Ranking from Implicit Feedback”. UAI 2009.

[3] Recht et al. "Hogwild!: A Lock-Free Approach to Parallelizing Stochastic Gradient Descent". NIPS 2011.
