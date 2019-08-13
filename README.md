# isolation-forest

This is a Scala/Spark implementation of the Isolation Forest unsupervised outlier detection
algorithm. This library was created by [James Verbus](https://www.linkedin.com/in/jamesverbus/) from
the LinkedIn Anti-Abuse AI team.

This library supports distributed training and scoring using Spark data structures. It inherits from
the ``Estimator`` and ``Model`` classes in [Spark's ML library](https://spark.apache.org/docs/2.3.0/ml-guide.html)
in order to take advantage of machinery such as ``Pipeline``s. Model persistence on HDFS is
supported.

## Copyright

Copyright 2019 LinkedIn Corporation
All Rights Reserved.

Licensed under the BSD 2-Clause License (the "License").
See [License](LICENSE) in the project root for license information.

## How to use

### Building the library

It is recommended to use Scala 2.11 and Spark 2.3. To build, run the following:

```bash
./gradlew build
```
This will produce a jar file in the ``./isolation-forest/build/libs/`` directory.

### Add an isolation-forest dependency to your project

Artifacts (Scala 2.11.8) for this project are [available on Bintray](https://bintray.com/beta/#/linkedin/maven/isolation-forest).

#### Gradle example

First, add the repository URL to the repositories block in the top-level build.gradle file.

```
repositories {
    maven {
        url "https://dl.bintray.com/linkedin/maven"
    }
}
```

Second, add the isolation-forest dependency to the module-level build.gradle file.

```
dependencies {
    compile 'com.linkedin.isolation-forest:isolation-forest_2.11:0.2.2'
}
```

### Model parameters

| Parameter     | Default Value    | Description                                                                                                                                                                                                          |
|---------------|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| numEstimators | 100              | The number of trees in the ensemble.                                                                                                                                                                                 |
| maxSamples    | 256              | The number of samples used to train each tree. If this value is between 0.0 and 1.0, then it is treated as a fraction. If it is >1.0, then it is treated as a count.                                                 |
| contamination | 0.0              | The fraction of outliers in the training data set. If this is set to 0.0, it speeds up the training and all predicted labels will be false. The model and outlier scores are otherwise unaffected by this parameter. |
| maxFeatures   | 1.0              | The number of features used to train each tree. If this value is between 0.0 and 1.0, then it is treated as a fraction. If it is >1.0, then it is treated as a count.                                                |
| bootstrap     | false            | If true, draw sample for each tree with replacement. If false, do not sample with replacement.                                                                                                                       |
| randomSeed    | 1                | The seed used for the random number generator.                                                                                                                                                                       |
| featuresCol   | "features"       | The feature vector. This column must exist in the input DataFrame for training and scoring.                                                                                                                          |
| predictionCol | "predictedLabel" | The predicted label. This column is appended to the input DataFrame upon scoring.                                                                                                                                    |
| scoreCol      | "outlierScore"   | The outlier score. This column is appended to the input DataFrame upon scoring.          

### Training and scoring

Here is an example demonstrating how to import the library, create a new ``IsolationForest``
instance, set the model hyperparameters, train the model, and then score the training data.``data``
is a Spark DataFrame with a column named ``features`` that contains a 
``org.apache.spark.ml.linalg.Vector`` of the attributes to use for training. In this example, the
DataFrame ``data`` also has a ``labels`` column; it is not used in the training process, but could
be useful for model evaluation.

```scala
import com.linkedin.relevance.isolationforest._
import org.apache.spark.ml.feature.VectorAssembler

/**
  * Load and prepare data
  */

// Dataset from http://odds.cs.stonybrook.edu/shuttle-dataset/
val rawData = spark.read
  .format("csv")
  .option("comment", "#")
  .option("header", "false")
  .option("inferSchema", "true")
  .load("isolation-forest/src/test/resources/shuttle.csv")

val cols = rawData.columns
val labelCol = cols.last
 
val assembler = new VectorAssembler()
  .setInputCols(cols.slice(0, cols.length - 1))
  .setOutputCol("features")
val data = assembler
  .transform(rawData)
  .select(col("features"), col(labelCol).as("label"))

// scala> data.printSchema
// root
//  |-- features: vector (nullable = true)
//  |-- label: integer (nullable = true)

/**
  * Train the model
  */
 
val isolationForest = new IsolationForest()
  .setNumEstimators(100)
  .setBootstrap(false)
  .setMaxSamples(256)
  .setMaxFeatures(1.0)
  .setFeaturesCol("features")
  .setPredictionCol("predictedLabel")
  .setScoreCol("outlierScore")
  .setContamination(0.1)
  .setRandomSeed(1)
 
val isolationForestModel = isolationForest.fit(data)
 
/**
  * Score the training data
  */
 
val dataWithScores = isolationForestModel.transform(data)

// scala> dataWithScores.printSchema
// root
//  |-- features: vector (nullable = true)
//  |-- label: integer (nullable = true)
//  |-- outlierScore: double (nullable = false)
//  |-- predictedLabel: double (nullable = false)
```

The output DataFrame, ``dataWithScores``, is identical to the input ``data`` DataFrame but has two
additional result columns appended with their names set via model parameters; in this case, these
are named ``predictedLabel`` and ``outlierScore``.

### Saving and loading a trained model

Once you've trained an ``isolationForestModel`` instance as per the instructions above, you can use the
following commands to save the model to HDFS and reload it as needed.

```scala
val path = "/user/testuser/isolationForestWriteTest"

/**
  * Persist the trained model on disk
  */

// You can ensure you don't overwrite an existing model by removing .overwrite from this command
isolationForestModel.write.overwrite.save(path)

/**
  * Load the saved model from disk
  */

val isolationForestModel2 = IsolationForestModel.load(path)
```

## Validation

The original 2008 "Isolation forest" paper by Liu et al. published the AUROC results obtained by
applying the algorithm to 12 benchmark outlier detection datasets. We applied our implementation of
the isolation forest algorithm to the same 12 datasets using the same model parameter values used in
the original paper. We used 10 trials per dataset each with a unique random seed and averaged the
result. The quoted uncertainty is the one-sigma error on the mean.

| Dataset                                                                            | Expected mean AUROC (from Liu et al.) | Observed mean AUROC (from this implementation) | Observed Std. Dev. AUROC (from this implementation) |
|------------------------------------------------------------------------------------|---------------------------------------|------------------------------------------------|-----------------------------------------------------|
| [Http (KDDCUP99)](http://odds.cs.stonybrook.edu/http-kddcup99-dataset/)            | 1.00                                  | 0.99973 &plusmn; 0.00007                       | 0.0002                                              |
| [ForestCover](http://odds.cs.stonybrook.edu/forestcovercovertype-dataset/)         | 0.88                                  | 0.903 &plusmn; 0.005                           | 0.017                                               |
| [Mulcross](https://www.openml.org/d/40897)                                         | 0.97                                  | 0.9926 &plusmn; 0.0006                         | 0.0019                                              |
| [Smtp (KDDCUP99)](http://odds.cs.stonybrook.edu/smtp-kddcup99-dataset/)            | 0.88                                  | 0.907 &plusmn; 0.001                           | 0.004                                               |
| [Shuttle](http://odds.cs.stonybrook.edu/shuttle-dataset/)                          | 1.00                                  | 0.9974 &plusmn; 0.0014                         | 0.004                                               |
| [Mammography](http://odds.cs.stonybrook.edu/mammography-dataset/)                  | 0.86                                  | 0.8636 &plusmn; 0.0015                         | 0.005                                               |
| [Annthyroid](http://odds.cs.stonybrook.edu/annthyroid-dataset/)                    | 0.82                                  | 0.815 &plusmn; 0.006                           | 0.019                                               |
| [Satellite](http://odds.cs.stonybrook.edu/satellite-dataset/)                      | 0.71                                  | 0.709 &plusmn; 0.004                           | 0.012                                               |
| [Pima](http://odds.cs.stonybrook.edu/pima-indians-diabetes-dataset/)               | 0.67                                  | 0.651 &plusmn; 0.003                           | 0.010                                               |
| [Breastw](http://odds.cs.stonybrook.edu/breast-cancer-wisconsin-original-dataset/) | 0.99                                  | 0.9862 &plusmn; 0.0003                         | 0.0010                                              |
| [Arrhythmia](http://odds.cs.stonybrook.edu/arrhythmia-dataset/)                    | 0.80                                  | 0.804 &plusmn; 0.002                           | 0.008                                               |
| [Ionosphere](http://odds.cs.stonybrook.edu/ionosphere-dataset/)                    | 0.85                                  | 0.8481 &plusmn; 0.0002                         | 0.0008                                              |

Our implementation provides AUROC values that are in very good agreement the results in the original
Liu et al. publication. There are a few very small discrepancies that are likely due the limited
precision of the AUROC values reported in Liu et al.

## Contributions

If you would like to contribute to this project, please review the instructions [here](CONTRIBUTING.md).

## References

* F. T. Liu, K. M. Ting, and Z.-H. Zhou, “Isolation forest,” in 2008 Eighth IEEE International Conference on Data Mining, 2008, pp. 413–422.
* F. T. Liu, K. M. Ting, and Z.-H. Zhou, “Isolation-based anomaly detection,” ACM Transactions on Knowledge Discovery from Data (TKDD), vol. 6, no. 1, p. 3, 2012.
* Shebuti Rayana (2016).  ODDS Library [http://odds.cs.stonybrook.edu]. Stony Brook, NY: Stony Brook University, Department of Computer Science.
