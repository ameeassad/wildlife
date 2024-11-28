```python exec="true" session="run"
import contextlib, io

def run(str):
    f = io.StringIO()
    with contextlib.redirect_stdout(f):
        eval(str)
    output = f.getvalue()
    return output
```

```python exec="true" session="run"
from wildlife_datasets import datasets, splits
import pandas as pd

df = pd.read_csv('docs/csv/MacaqueFaces.csv')
df = df.drop('Unnamed: 0', axis=1)
dataset = datasets.MacaqueFaces('.', df)
df = dataset.df
```


# How to use splitting functions

The crucial part of machine learning is training a method on a training set and evaluating it on a separate testing set. We provide a default split for each dataset. However, it is possible to create additional custom splits. The following splits are implemented:

- **closed-set split**. The default split where the testing set does not contain any new individuals. For each sample from the testing set, the goal is to assign some individual from the training set. The name follows from the fact that the population is closed.
- **open-set split**. The split where the testing set contains new individuals. For each sample from the testing set, the goal is to assign some individual from the training set or predict that the sample contain a new individual. The name follows from the fact that the population is open.
- **disjoint-set split**. The split where there are no individuals in both the training and testing sets. For each sample from the testing set, the goal is to assign some individual from the testing set. The name follows from the fact that the two populations are disjoint.

The splits are usually created randomly but in situations where the dataset contains timestamps, it is also possible to create the split based on timestamps. Due to our representation of the [random number generator](../reference_splits#lcg), the splits are both machine- and system-independent. We followed the presentation from [this paper](https://arxiv.org/abs/2211.10307).

We assume that we have already [downloaded](../tutorial_datasets#downloading-datasets) the MacaqueFaces dataset. Then we load the dataset and the dataframe.

```python
from wildlife_datasets import datasets, splits

dataset = datasets.MacaqueFaces('data/MacaqueFaces')
df = dataset.df
```

## Splits based on identities

Splits on identities perform the split for each individual separetely. All these splits are random.

### Closed-set split

The most common split is the closed-set split, where each individual has samples in both the training and testing sets.

```python exec="true" source="above" result="console" session="run"
splitter = splits.ClosedSetSplit(0.8)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

This code generates a split, where the training set contains approximately 80% of all samples. Even though this split contains precisely 80%, since the split is done separately for each individual, the real ratio of the training set may be different. The outputs of the spliller are labels, not indices, and, therefore, we access the training and testing sets by

```python
df_train, df_test = df.loc[idx_train], df.loc[idx_test]
```

### Open-set split

In the open-set split, there are some individuals with all their samples only in the testing set. For the remaining individuals, the closed-set split is performed.

```python exec="true" source="above" result="console" session="run"
splitter = splits.OpenSetSplit(0.8, 0.1)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

This code generates a split, where approximately 10% of samples are put directly into the testing set. It also specifies that the training set should contain 50% of all samples. As in the previous (and all following) cases, the numbers are only approximate with the actual ratios being 8.92% and 80%. The other possibility to create this split is to prescribe the number of individuals (instead of ratio of samples) which go directly into the testing set.

```python exec="true" source="above" result="console" session="run"
splitter = splits.OpenSetSplit(0.8, n_class_test=5)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

### Disjoint-set split

For the disjoint-set split, each individual has all samples either in the training or testing set but never in both. Similarly as in the open-set split, we can create the split either by

```python exec="true" source="above" result="console" session="run"
splitter = splits.DisjointSetSplit(0.2)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

or

```python exec="true" source="above" result="console" session="run"
splitter = splits.DisjointSetSplit(n_class_test=10)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

The first method put approximately 20% of the samples to the testing set, while the second method puts 10 classes to the testing set.


## Splits based on time

Splits based on time create some cutoff time and put everything before the cutoff time into the training set and everything after the cutoff time into the training set. Therefore, this splits are not random but deterministic. These splits also ignore all samples without timestamps.

### Time-proportion split

Time-proportion split counts on how many days was each individual observed. Then it puts all samples corresponding to the first half of the observation days to the training set and all remaining to the testing set. It ignores all individuals observed only on one day. Since all individuals are both in the training and testing set, it leads to the closed-set split.

```python exec="true" source="above" result="console" session="run"
splitter = splits.TimeProportionSplit()
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

Even though the split is non-random, it still required the seed because of the [random resplit](#random-resplit).

### Time-cutoff split

While the time-proportion day selected a different cutoff day for each individual, the time-cutoff split creates one cutoff year for the whole dataset. All samples taken before the cutoff year go the training set, while all samples taken during the cutoff year go to the testing set. Since some individuals may be present only in the testing set, this split is usually an open-set split.

```python exec="true" source="above" result="console" session="run"
splitter = splits.TimeCutoffSplit(2015)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

It is also possible to place all samples taken during or after the cutoff year to the testing set by

```python exec="true" source="above" result="console" session="run"
splitter = splits.TimeCutoffSplit(2015, test_one_year_only=False)
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

It is also possible to create all possible time-cutoff splits for different years by

```python exec="true" source="above" result="console" session="run"
splitter = splits.TimeCutoffSplitAll()
for idx_train, idx_test in splitter.split(df):
    splits.analyze_split(df, idx_train, idx_test)
    print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

### Random resplit

Since the splits based on time are not random, it is also possible to create similar random splits by 

```python exec="true" source="above" result="console" session="run"
idx_train, idx_test = splitter.resplit_random(df, idx_train, idx_test)
splits.analyze_split(df, idx_train, idx_test)
print(run('splits.analyze_split(df, idx_train, idx_test)')) # markdown-exec: hide
```

For each individual the number of samples in the training set will be the same for the original and new splits. Since the new splits are random, they do not utilize the time at all.
