FROM jupyter/all-spark-notebook

# Set env vars for pydoop

ARG HADOOP_VERSION=2.7.2
ARG HDP_VERSION=2.5.0.0-1245

ENV HADOOP_VERSION=2.7.2
ENV HDP_VERSION=2.5.0.0-1245

ENV HADOOP_HOME /usr/local/hadoop-${HADOOP_VERSION}
ENV JAVA_HOME /usr/lib/jvm/java-8-openjdk-amd64
ENV HADOOP_CONF_HOME /usr/local/hadoop-${HADOOP_VERSION}/etc/hadoop
ENV HADOOP_CONF_DIR  /usr/local/hadoop-${HADOOP_VERSION}/etc/hadoop

USER root
# Add proper open-jdk-8 not just the jre, needed for pydoop
RUN gpg --keyserver pgpkeys.mit.edu --recv-key  8B48AD6246925553 && gpg -a --export 8B48AD6246925553 | apt-key add -
RUN echo 'deb http://cdn-fastly.deb.debian.org/debian jessie-backports main' > /etc/apt/sources.list.d/jessie-backports.list && \
    apt-get -y update && \
    apt-get install --no-install-recommends -t jessie-backports -y libjpeg62-turbo openjdk-8-jre-headless ca-certificates-java  && \
    rm /etc/apt/sources.list.d/jessie-backports.list && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/ && \
# Add hadoop binaries
    wget http://mirrors.ukfast.co.uk/sites/ftp.apache.org/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz && \
    tar -xvf hadoop-${HADOOP_VERSION}.tar.gz -C /usr/local && \
    chown -R $NB_USER:users /usr/local/hadoop-${HADOOP_VERSION} && \
    rm -f hadoop-${HADOOP_VERSION}.tar.gz && \
# Install os dependencies required for pydoop, pyhive
    apt-get update && \
    apt-get install --no-install-recommends -y build-essential python-dev libsasl2-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
# Remove the example hadoop configs and replace
# with those for our cluster.
# Alternatively this could be mounted as a volume
    rm -f /usr/local/hadoop-${HADOOP_VERSION}/etc/hadoop/*

# Download this from ambari / cloudera manager and copy here
COPY example-hadoop-conf/ /usr/local/hadoop-${HADOOP_VERSION}/etc/hadoop/

# Spark-Submit doesn't work unless I set the following
RUN echo "spark.driver.extraJavaOptions -Dhdp.version=${HDP_VERSION}" >> /usr/local/spark/conf/spark-defaults.conf  && \
    echo "spark.yarn.am.extraJavaOptions -Dhdp.version=${HDP_VERSION}" >> /usr/local/spark/conf/spark-defaults.conf && \
    echo "spark.master=yarn" >>  /usr/local/spark/conf/spark-defaults.conf && \
    echo "spark.hadoop.yarn.timeline-service.enabled=false" >> /usr/local/spark/conf/spark-defaults.conf && \
    chown -R $NB_USER:users /usr/local/spark/conf/spark-defaults.conf && \
    # Create an alternative HADOOP_CONF_HOME so we can mount as a volume and repoint
    # using ENV var if needed
    mkdir -p /etc/hadoop/conf/ && \
    chown $NB_USER:users /etc/hadoop/conf/

USER $NB_USER

# Install useful jupyter extensions and python libraries like :
# - Dashboards
# - PyDoop
# - PyHive
RUN pip install jupyter_dashboards faker && \
    jupyter dashboards quick-setup --sys-prefix && \
    pip2 install pyhive pydoop thrift sasl thrift_sasl faker

USER root
# Ensure we overwrite the kernel config so that toree connects to cluster
RUN jupyter toree install --sys-prefix --spark_opts="--master yarn --deploy-mode client --driver-memory 512m  --executor-memory 512m  --executor-cores 1 --driver-java-options -Dhdp.version=${HDP_VERSION} --conf spark.hadoop.yarn.timeline-service.enabled=false"
uSER $NB_USER

