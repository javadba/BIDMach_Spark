# BIDMach_Spark
Code to allow running BIDMach on Spark including HDFS integration and lightweight sparse model updates (Kylix). 

<h3>Dependencies</h3>

This repo depends on BIDMat, and also on lz4 and hadoop. Assuming you have hadoop installed and working, and that you've built a working BIDMat jar, copy these files into the lib directory of this repo. i.e.

<pre>cp BIDMat/BIDMat.jar BIDMach_Spark/lib
cp BIDMat/lib/lz4-*.*.jar BIDMach_Spark/lib</pre>

you'll also need the hadoop common library from your hadoop installation:

<pre>cp $HADOOP_HOME/share/hadoop/common/hadoop-common-*.*.jar BIDMach_Spark/lib</pre>

and then 

<pre>cd BIDMach_Spark
./sbt package</pre>

will build <code>BIDMatHDFS.jar</code>. Copy this back to the BIDMat lib directory:

<pre>cp BIDMatHDFS.jar ../BIDMat/lib</pre>

Make sure $HADOOP_HOME is set to the hadoop home directory (usually /use/local/hadoop), and make sure hdfs is running:
<pre>$HADOOP_HOME/sbin/start-dfs.sh</pre>
Then you should have HDFS access with BIDMat by invoking 
<pre>BIDMat/bidmath</pre>

<pre>saveFMat("hdfs://localhost:9000/filename.fmat")</pre> or
<pre>saveFMat("hdfs://filename.fmat")</pre>

<h3>Hadoop Config</h3>
The hadoop quickstart guides dont mention this but you need to set the hdfs config to point to a persistent set of directories to hold the HDFS data. Here's a typical hdfs-site.xml:

<pre> 
&lt;configuration&gt;
     &lt;property&gt;
         &lt;name&gt;dfs.replication&lt;/name&gt;
         &lt;value&gt;1&lt;/value&gt;
     &lt;/property&gt;
     &lt;property&gt;
         &lt;name&gt;dfs.name.dir&lt;/name&gt;
         &lt;value&gt;/data/hdfs/name&lt;/value&gt;
     &lt;/property&gt;
     &lt;property&gt;
         &lt;name>dfs.data.dir&lt;/name&gt;
         &lt;value>/data/hdfs/data&lt;/value&gt;
     &lt;/property&gt;
&lt;/configuration&gt;
</pre>


<h3>System properties to control the testing scripts</h3>

The **.ssc** files under **scripts/** directory are used to test the installation. Here are environment variables / System properties to configure them properly for your local environment:

- **bidmach.path**: path to the BIDMach installation   e.g. -Dbidmach.path=/git/BIDMach
- **hdfs.path**: path to the file saved to hdfs e.g. -Dhdfs.path=hdfs://sparkbook:8020/bidmach 
- **bidmach.merged.hdfs.path**: path to the final merged/combined lz4 output e.g. -Dbidmach.merged.hdfs.path=hdfs://sparkbook:8020/bidmach/BIDMach_MNIST/partsmerged.fmat.lz4 
- **spark.executors**: number of executors to use in processing e.g. -Dspark.executors=1

<h3>Additional steps and settings on Mac</h3>

**Protobuf**:

<pre>brew install homebrew/versions/protobuf250</pre>

**cmake and openssl**

brew install cmake
brew install openssl

**Install gnu sed**
<pre>brew install gsed</pre>

You will need to **compile native hadoop libraries** and install them to the hadoop directory. Reason? This package relies on **lz4** that is obtained by building the native libs.

**Install hadoop**
<pre>
brew install hadoop
</pre>

**Build Hadoop native libraries**
<pre>
git clone https://github.com/apache/hadoop
cd hadoop
git checkout branch-2.7.1
mvn package -Pdist,native -DskipTests -Dtar -Dmaven.javadoc.skip=true
# Add to your /etc/profile:
echo 'export HADOOP_HOME="/usr/local/Cellar/hadoop/2.7.2/libexec"' | sudo tee /etc/profile
source /etc/profile
# Need to update the hadoop-pipes ant build-main.xml script:
gsed -i  $HADOOP_HOME/git/hadoop/hadoop-tools/hadoop-pipes/src/ -DJVM_ARCH_DATA_MODEL=64 -DOPENSSL_ROOT_DIR=/usr/local/Cellar/openssl/1.0.2h_1 -DOPENSSL_LIBRARIES=/usr/local/Cellar/openssl/1.0.2h_1/lib"/>
</pre>

**Set up env variables**

Need to disable SIP to change the **DYLD_LIBRARY_PaTH** . Consult online resources for more details on how to do this. 
<pre>sudo csrutil disable</pre>
<pre>echo 'export DYLD_LIBRARY_PATH="/usr/local/Cellar/hadoop/2.7.2/libexec/share/hadoop/common/lib:$DYLD_LIBRARY_PATH' >> sudo tee /etc/profile
</pre>
Then you can reenable SIP: consult online resources for details.
<pre>sudo csrutil enable</pre>


**Updates to spark-env.sh**
<pre>
echo 'export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export SPARK_EXECUTOR_CORES=1  #, Number of cores for the executors (Default: 1).
export SPARK_MASTER_OPTS="-Dspark.deploy.defaultCores=1"
export SPARK_WORKER_MEMORY=6G
export SPARK_DRIVER_MEMORY=4G
export DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/usr/local/Cellar/hadoop/2.7.2/libexec/lib:/usr/local/Cellar/hadoop/2.7.2/libexec/share/hadoop/common:/usr/local/Cellar/hadoop/2.7.2/libexec/share/hadoop/common/lib
' >> $SPARK_HOME/conf/spark-env.sh
</pre>

**Updates to spark-default.conf**
<pre>
cp $SPARK_HOME/conf/spark-defaults.conf.template $SPARK_HOME/conf/spark-defaults.conf
echo '
spark.executor.memory	6g
' >> $SPARK_HOME/conf/spark-defaults.conf
</pre>

**Updates to spark-default.conf**
<pre>
cp $SPARK_HOME/conf/spark-defaults.conf.template $SPARK_HOME/conf/spark-defaults.conf
echo '
spark.executor.memory	6g
' >> $SPARK_HOME/conf/spark-defaults.conf
</pre>

**Testing on Mac**
I was unable to run the full 8million row MNIST.  For some smoke testing on Mac a 200K row file can be created using the following script:

<pre>scripts/append_mnist_200k.ssc</pre>


**Here is how the testing was performed on Mac**


<pre>
spark-shell --executor-memory 6g --total-executor-cores 1 --master spark://sparkbook:7077 --jars /git/BIDMach_Spark/BIDMatHDFS.jar,/git/BIDMach/lib/BIDMat.jar,/git/BIDMach/BIDMach.jar --driver-java-options "-Dbidmach.path=/git/BIDMach -Dbidmach.merged.hdfs.path=hdfs://sparkbook:8020/bidmach/BIDMach_MNIST/partsmerged.fmat.lz4 -Dhdfs.path=hdfs://sparkbook:8020/bidmach -Dspark.executors=1"
</pre>

Inside the spark-shell:

<pre>
:load /git/BIDMach_Spark/scripts/load_mnist.ssc
:load /git/BIDMach_Spark/scripts/append_mnist_200k.ssc
:load /git/BIDMach_Spark/scripts/KMeansLearner.ssc
</pre>