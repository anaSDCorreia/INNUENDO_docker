FROM centos:7
MAINTAINER Bruno Gonçalves <bfgoncalves@medicina.ulisboa.pt>

ARG SLURM_VERSION=17.02.11
ARG SLURM_DOWNLOAD_MD5=b32f4260a921d335a2d52950593f0a29
ARG SLURM_DOWNLOAD_URL=https://download.schedmd.com/slurm/slurm-17.02.11.tar.bz2

ARG GOSU_VERSION=1.10

# Set the locale
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

RUN yum makecache fast \
    && yum -y install epel-release \
    && yum -y install https://centos7.iuscommunity.org/ius-release.rpm \
    && yum -y install \
           wget \
           bzip2 \
           perl \
           gcc \
           gcc-c++\
           vim-enhanced \
           git \
           make \
           munge \
           munge-devel \
           python-devel \
           python-pip \
           mariadb-server \
           mariadb-devel \
           psmisc \
           bash-completion \
           python-psycopg2 \
           libpq-dev \
           libcurl4-openssl-dev \
           python-dev \
           libsasl2-dev \
           libldap2-dev \
           libssl-dev \
           libffi-dev \
           java-1.8.0-openjdk \
           python36u \
           python36u-devel \
           python36u-pip \
           build-essential \
           libmunge-dev \
    && yum clean all \
    && rm -rf /var/cache/yum

######################### Install process controller ##########################

# Create controller folder
WORKDIR /Controller/

# Install Nextflow
RUN wget -qO- https://get.nextflow.io | bash

# Move Nextflow to path
RUN mv nextflow /usr/local/bin

# Get INNUENDO_REST_API repository
RUN echo "GET INNUENDO"
RUN git clone https://github.com/bfrgoncalves/INNUENDO_PROCESS_CONTROLLER.git

# Change to repository
WORKDIR INNUENDO_PROCESS_CONTROLLER

# Install and initialize virtual env
RUN pip install virtualenv
RUN virtualenv flask

# Install repository dependencies
RUN flask/bin/pip install -r requirements.txt

# Install other dependencies
RUN flask/bin/pip install future requests

# Get allegro client
RUN wget https://github.com/bfrgoncalves/INNUENDO_files/releases/download/1.0.0/agraph-6.2.1-client-python.tar.gz

# Uncompress allegro client files
RUN tar zxf agraph-6.2.1-client-python.tar.gz

# Add allegro client to PYTHONPATH
ENV PYTHONPATH="/Controller/INNUENDO_PROCESS_CONTROLLER/agraph-6.2.1-client-python/src"

# Add symlink for python3
RUN ln -s /usr/bin/python3.6 /usr/bin/python3
RUN ln -s /usr/bin/pip3.6 /usr/bin/pip3


########################## Install SLURM #######################################

RUN pip install Cython nose \
    && pip3 install Cython nose

RUN set -x \
    && wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-amd64" \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true

RUN groupadd -r slurm --gid=995 && useradd -r -g slurm --uid=995 slurm

RUN set -x \
    && wget -O slurm.tar.bz2 "$SLURM_DOWNLOAD_URL" \
    && echo "$SLURM_DOWNLOAD_MD5" slurm.tar.bz2 | md5sum -c - \
    && mkdir /usr/local/src/slurm \
    && tar jxf slurm.tar.bz2 -C /usr/local/src/slurm --strip-components=1 \
    && rm slurm.tar.bz2 \
    && cd /usr/local/src/slurm \
    && ./configure --enable-debug --prefix=/usr --sysconfdir=/etc/slurm \
        --with-mysql_config=/usr/bin  --libdir=/usr/lib64 \
    && make install \
    && install -D -m644 etc/cgroup.conf.example /etc/slurm/cgroup.conf.example \
    && install -D -m644 etc/slurm.conf.example /etc/slurm/slurm.conf.example \
    && install -D -m644 etc/slurm.epilog.clean /etc/slurm/slurm.epilog.clean \
    && install -D -m644 etc/slurmdbd.conf.example /etc/slurm/slurmdbd.conf.example \
    && install -D -m644 contribs/slurm_completion_help/slurm_completion.sh /etc/profile.d/slurm_completion.sh \
    && cd \
    && rm -rf /usr/local/src/slurm \
    && mkdir /etc/sysconfig/slurm \
        /var/spool/slurmd \
        /var/run/slurmd \
        /var/run/slurmdbd \
        /var/lib/slurmd \
        /var/log/slurm \
        /data \
    && touch /var/lib/slurmd/node_state \
        /var/lib/slurmd/front_end_state \
        /var/lib/slurmd/job_state \
        /var/lib/slurmd/resv_state \
        /var/lib/slurmd/trigger_state \
        /var/lib/slurmd/assoc_mgr_state \
        /var/lib/slurmd/assoc_usage \
        /var/lib/slurmd/qos_usage \
        /var/lib/slurmd/fed_mgr_state \
    && chown -R slurm:slurm /var/*/slurm* \
    && /sbin/create-munge-key

WORKDIR /Controller

####################### Install Singularity ####################################

RUN yum -y install squashfs-tools

RUN VERSION=2.4.5 && wget https://github.com/singularityware/singularity/releases/download/$VERSION/singularity-$VERSION.tar.gz \
                  && tar xvf singularity-$VERSION.tar.gz \
                  && cd singularity-$VERSION \
                  && ./configure --prefix=/usr/local \
                  && make \
                  && make install

WORKDIR /Controller/

COPY slurm.conf /etc/slurm/slurm.conf
COPY slurmdbd.conf /etc/slurm/slurmdbd.conf

# Install aspera
RUN wget -qO- https://download.asperasoft.com/download/sw/connect/3.8.0/ibm-aspera-connect-3.8.0.158555-linux-g2.12-64.tar.gz | tar xvz
RUN chmod +x ibm-aspera-connect-3.8.0.158555-linux-g2.12-64.sh
RUN ./ibm-aspera-connect-3.8.0.158555-linux-g2.12-64.sh

ENV PATH $PATH:~/.aspera/connect/bin/

# Install Assemblerflow
WORKDIR /Controller/
RUN git clone https://github.com/assemblerflow/flowcraft.git
WORKDIR flowcraft/
RUN git pull
RUN git checkout dev
# && git submodule update --init --recursive
RUN python3 setup.py install

# Put all files inside bin executable
RUN chmod a+x flowcraft/bin/*

WORKDIR /Controller/INNUENDO_PROCESS_CONTROLLER

COPY get_profiles_and_training.sh executables/get_profiles_and_training.sh
ENV PATH="/Controller/INNUENDO_PROCESS_CONTROLLER/executables:${PATH}"

RUN chmod a+x executables/get_profiles_and_training.sh

COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

CMD ["slurmdbd"]

