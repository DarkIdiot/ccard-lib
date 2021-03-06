[![Stories in Ready](https://badge.waffle.io/chaoslawful/ccard-lib.png)](https://waffle.io/chaoslawful/ccard-lib)

[![Build Status](https://travis-ci.org/chaoslawful/ccard-lib.png)](https://travis-ci.org/chaoslawful/ccard-lib)

C library for estimating cardinality in data streams, in which case it is
infeasible to store all events in memory.

This library implements a series of cardinality estimating algorithms such as
Linear Counting, LogLog Counting, HyperLogLog Counting and Adaptive Counting.
For more information about these algorithms please read the
[Reference](#reference) section.

## Building

Building ccard-lib needs [scons](http://www.scons.org/). Please read [scons
user guide](http://www.scons.org/doc/production/HTML/scons-user/index.html) for
more information about it.

Building PHP extension of ccard-lib needs [SWIG](http://www.swig.org/) to be
installed. Running unit-tests needs
[googletest](https://github.com/google/googletest) to be installed.

### Building as Library

Assuming you have scons installed, just build ccard-lib like this:

```bash
scons install
```

Scons will build and install ccard-lib to your system.

You can also run unit-tests to make sure the library works as expected:

```bash
scons test
```

By default ccard-lib will be installed at `/usr/local/lib`, if you want to
change the install directory please replace the "libdir" setting in
`SConsturct` file with your target directory.

### Building as PHP Extension

The following command will build and install card-lib PHP extension:

```bash
scons install-php
```

[SWIG](http://www.swig.org) is used to generate PHP extension, please install
it before run this command.

### Uninstall

If you want to uninstall ccard-lib from your system, use the following
commands:

```bash
scons -c install-php
scons -c install
```

## Synopsis

### Estimate Cardinality

```c
#include "ccard_common.h"
#include "adaptive_counting.h"

int main(int argc, char **argv) {
    int64_t i, esti;

    /* construct context for cardinality estimator */
    /* use xxx_cnt_init to construct context */
    adp_cnt_ctx_t *ctx = adp_cnt_init(NULL, 16, CCARD_HASH_MURMUR);

    printf("Adaptive Counting with Murmurhash:\n");

    /* add 500,000 elements to set */
    for (i = 1; i <= 500000L; i++) {
        /* use xxx_cnt_offer to add new element to set */
        adp_cnt_offer(ctx, &i, sizeof(int64_t));

        /* print estimate result every 50,000 elements has been added */
        if (i % 50000 == 0) {
            /* use xxx_cnt_card to get estimate result */
            esti = adp_cnt_card(ctx);
            printf("actual: %9lu, estimated: %9lu, error: %+7.2f%%\n",
                   (long unsigned int)i, (long unsigned int)esti, (double)(esti - i) / i * 100);
        }
    }
    printf("\n");

    /* use xxx_cnt_fini to destory context */
    adp_cnt_fini(ctx);
}
```

### Merge Bitmaps
```c
#include "ccard_common.h"
#include "adaptive_counting.h"

int main(int argc, char **argv) {
    int64_t i, esti;

    /* for merging, contexts must have same length of bitmap and hash algorithm */
    adp_cnt_ctx_t *ctx = adp_cnt_init(NULL, 16, CCARD_HASH_LOOKUP3);
    adp_cnt_ctx_t *tbm1 = adp_cnt_init(NULL, 16, CCARD_HASH_LOOKUP3);
    adp_cnt_ctx_t *tbm2 = adp_cnt_init(NULL, 16, CCARD_HASH_LOOKUP3);

    int32_t m = 1 << 16;
    /* bitmaps */
    uint8_t buf1[m + 3], buf2[m + 3];
    uint32_t len1 = m + 3, len2 = m + 3;

    for (i = 1; i <= 20000L; i++) {
        adp_cnt_offer(ctx, &i, sizeof(uint64_t));
    }
    for (i = 10000L; i <= 30000L; i++) {
        adp_cnt_offer(tbm1, &i, sizeof(uint64_t));
    }
    /* use xxx_cnt_get_bytes to get bitmap from context */
    adp_cnt_get_bytes(tbm1, buf1, &len1);
    for (i = 20000L; i <= 40000L; i++) {
        adp_cnt_offer(tbm2, &i, sizeof(uint64_t));
    }
    adp_cnt_get_bytes(tbm2, buf2, &len2);

    /* use xxx_cnt_merge_bytes to merge bitmaps to context */
    adp_cnt_merge_bytes(ctx, buf1, len1, buf2, len2, NULL);
    esti = adp_cnt_card(ctx);

    printf("actual:40000, estimated: %9lu, error: %+7.2f%%\n",
           (long unsigned int)esti, (double)(esti - 40000) / 40000 * 100);

    adp_cnt_fini(tbm2);
    adp_cnt_fini(tbm1);
    adp_cnt_fini(ctx);
}
```

## For Developers

Source codes should always be formatted before committing by running script
`util/indent-src` in top-dir. It utilized
[astyle](http://astyle.sourceforge.net/) to do the job, so you probably want to
install it first. **Make sure you install astyle v2.03 or later**, as the
indenting result differs from previous versions ([see
 here](http://astyle.sourceforge.net/news.html) for details)

## Reference

### Linear Counting

 * K.-Y. Whang, B. T. Vander-Zanden, and H. M. Taylor. **[A Linear-Time
 Probabilistic Counting Algorithm for Database Applications]
 (http://dblab.kaist.ac.kr/Publication/pdf/ACM90_TODS_v15n2.pdf)**. ACM
 Transactions on Database Systems, 15(2):208-229, 1990.

### LogLog Counting and Adaptive Counting

 * Marianne Durand and Philippe Flajolet. **[LogLog counting of large
 cardinalities](http://algo.inria.fr/flajolet/Publications/DuFl03-LNCS.pdf)**.
 In ESA03, volume 2832 of LNCS, pages 605-617, 2003.
 * Min Cai, Jianping Pan, Yu K. Kwok, and Kai Hwang. **[Fast and accurate
 traffic matrix measurement using adaptive cardinality counting]
 (http://gridsec.usc.edu/files/tr/tr-2005-12.pdf)**. In
 MineNet '05: Proceedings of the 2005 ACM SIGCOMM workshop on
 Mining network data, pages 205-206, New York, NY, USA, 2005. ACM.

### HyperLogLog Counting and HyperLogLog++ Counting

 * P. Flajolet, E. Fusy, O. Gandouet, and F. Meunier.
 **[Hyperloglog: The analysis of a near-optimal cardinality
 estimation algorithm](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)**.
 Disc. Math. and Theor. Comp. Sci., AH:127-146, 2007.
 * Stefan Heule, Marc Nunkesser, Alex Hall. **[HyperLogLog in Practice:
 Algorithmic Engineering of a State of The Art Cardinality Estimation
 Algorithm](http://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/40671.pdf)**.
 In Proceedings of the EDBT 2013 Conference, ACM, Genoa, Italy.

The implemention refers [stream-lib](https://github.com/clearspring/stream-lib).

## Experiment

The following estimating results is calculated using bitmap with length of 2^16
(64k) bytes:

```bash
Linear Counting with Murmurhash:
actual: 50000,  estimated: 50062,  error: 0.12%
actual: 100000, estimated: 99924,  error: 0.08%
actual: 150000, estimated: 149865, error: 0.09%
actual: 200000, estimated: 199916, error: 0.04%
actual: 250000, estimated: 250123, error: 0.05%
actual: 300000, estimated: 299942, error: 0.02%
actual: 350000, estimated: 349801, error: 0.06%
actual: 400000, estimated: 400101, error: 0.03%
actual: 450000, estimated: 449955, error: 0.01%
actual: 500000, estimated: 500065, error: 0.01%

Linear Counting with Lookup3hash:
actual: 50000,  estimated: 49835,  error: 0.33%
actual: 100000, estimated: 99461,  error: 0.54%
actual: 150000, estimated: 149006, error: 0.66%
actual: 200000, estimated: 198501, error: 0.75%
actual: 250000, estimated: 248365, error: 0.65%
actual: 300000, estimated: 298065, error: 0.65%
actual: 350000, estimated: 347504, error: 0.71%
actual: 400000, estimated: 397292, error: 0.68%
actual: 450000, estimated: 446700, error: 0.73%
actual: 500000, estimated: 495944, error: 0.81%

Hyperloglog Counting with Murmurhash:
actual: 50000,  estimated: 50015,  error: 0.03%
actual: 100000, estimated: 100048, error: 0.05%
actual: 150000, estimated: 149709, error: 0.19%
actual: 200000, estimated: 201595, error: 0.80%
actual: 250000, estimated: 250168, error: 0.07%
actual: 300000, estimated: 299864, error: 0.05%
actual: 350000, estimated: 348571, error: 0.41%
actual: 400000, estimated: 398583, error: 0.35%
actual: 450000, estimated: 448632, error: 0.30%
actual: 500000, estimated: 498330, error: 0.33%

Hyperloglog Counting with Lookup3hash:
actual: 50000,  estimated: 49628,  error: 0.74%
actual: 100000, estimated: 99357,  error: 0.64%
actual: 150000, estimated: 148880, error: 0.75%
actual: 200000, estimated: 200475, error: 0.24%
actual: 250000, estimated: 249362, error: 0.26%
actual: 300000, estimated: 299119, error: 0.29%
actual: 350000, estimated: 349225, error: 0.22%
actual: 400000, estimated: 398805, error: 0.30%
actual: 450000, estimated: 448373, error: 0.36%
actual: 500000, estimated: 498183, error: 0.36%

Adaptive Counting with Murmurhash:
actual: 50000,  estimated: 50015,  error: 0.03%
actual: 100000, estimated: 100048, error: 0.05%
actual: 150000, estimated: 149709, error: 0.19%
actual: 200000, estimated: 201059, error: 0.53%
actual: 250000, estimated: 249991, error: 0.00%
actual: 300000, estimated: 300067, error: 0.02%
actual: 350000, estimated: 349610, error: 0.11%
actual: 400000, estimated: 399875, error: 0.03%
actual: 450000, estimated: 450348, error: 0.08%
actual: 500000, estimated: 500977, error: 0.20%

Adaptive Counting with Lookup3hash:
actual: 50000,  estimated: 49628,  error: 0.74%
actual: 100000, estimated: 99357,  error: 0.64%
actual: 150000, estimated: 148880, error: 0.75%
actual: 200000, estimated: 199895, error: 0.05%
actual: 250000, estimated: 249563, error: 0.17%
actual: 300000, estimated: 299047, error: 0.32%
actual: 350000, estimated: 348665, error: 0.38%
actual: 400000, estimated: 399266, error: 0.18%
actual: 450000, estimated: 450196, error: 0.04%
actual: 500000, estimated: 499516, error: 0.10%

Loglog Counting with Murmurhash:
actual: 50000,  estimated: 59857,  error: 19.71%
actual: 100000, estimated: 103108, error: 3.11%
actual: 150000, estimated: 150917, error: 0.61%
actual: 200000, estimated: 201059, error: 0.53%
actual: 250000, estimated: 249991, error: 0.00%
actual: 300000, estimated: 300067, error: 0.02%
actual: 350000, estimated: 349610, error: 0.11%
actual: 400000, estimated: 399875, error: 0.03%
actual: 450000, estimated: 450348, error: 0.08%
actual: 500000, estimated: 500977, error: 0.20%

Loglog Counting with Lookup3hash:
actual: 50000,  estimated: 59870,  error: 19.74%
actual: 100000, estimated: 103044, error: 3.04%
actual: 150000, estimated: 150435, error: 0.29%
actual: 200000, estimated: 199895, error: 0.05%
actual: 250000, estimated: 249563, error: 0.17%
actual: 300000, estimated: 299047, error: 0.32%
actual: 350000, estimated: 348665, error: 0.38%
actual: 400000, estimated: 399266, error: 0.18%
actual: 450000, estimated: 450196, error: 0.04%
actual: 500000, estimated: 499516, error: 0.10%

HyperloglogPlus Counting with Murmurhash 64bit:
actual: 50000,  estimated: 49801,  error: 0.40%
actual: 100000, estimated: 101098, error: 1.10%
actual: 150000, estimated: 151488, error: 0.99%
actual: 200000, estimated: 201337, error: 0.67%
actual: 250000, estimated: 252130, error: 0.85%
actual: 300000, estimated: 301995, error: 0.66%
actual: 350000, estimated: 352194, error: 0.63%
actual: 400000, estimated: 402413, error: 0.60%
actual: 450000, estimated: 454293, error: 0.95%
actual: 500000, estimated: 503228, error: 0.65%
```
