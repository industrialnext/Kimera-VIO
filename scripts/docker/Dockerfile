# Use an official Python runtime as a parent image
FROM ubuntu:18.04 as base

LABEL maintainer='Antoni Rosinol "arosinol@mit.edu"'

RUN apt-get update && apt-get -y install zip unzip

# To avoid tzdata asking for geographic location...
ENV DEBIAN_frontend noninteractive

# Set the working directory to /root
ENV DIRPATH /root
WORKDIR $DIRPATH
ENV DEBIAN_FRONTEND=noninteractive
#Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends apt-utils

FROM base as dev
RUN apt-get update && apt-get install -y git build-essential cmake unzip pkg-config

# Install xvfb to provide a display to container for GUI related testing.
RUN apt-get update && apt-get install -y xvfb

FROM dev as dev-boost
# Install GTSAM
RUN apt-get update && apt-get install -y libboost-all-dev
FROM dev-boost as gtsam-code
ADD https://api.github.com/repos/borglab/gtsam/git/refs/heads/master version.json
RUN git clone https://github.com/borglab/gtsam.git
RUN cd gtsam && \
    git fetch && \
    git reset --hard e5866799dff48239573cdd84964180867e50edd2
FROM gtsam-code as gtsam
RUN cd gtsam && \
    mkdir build && \
    cd build && \
    cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DGTSAM_BUILD_TESTS=OFF -DGTSAM_BUILD_EXAMPLES_ALWAYS=OFF -DCMAKE_BUILD_TYPE=Release -DGTSAM_BUILD_UNSTABLE=ON -DGTSAM_TANGENT_PREINTEGRATION=OFF .. && \
    make -j$(nproc) install

FROM dev as opencv
# Install OpenCV for Ubuntu 18.04
RUN apt-get update && apt-get install -y \
      libjpeg-dev libpng-dev libtiff-dev \
      libvtk6-dev \
      libgtk-3-dev \
      libatlas-base-dev gfortran

RUN git clone https://github.com/opencv/opencv.git
RUN cd opencv && \
      git checkout tags/3.3.1 && \
      mkdir build

RUN git clone https://github.com/opencv/opencv_contrib.git
RUN cd opencv_contrib && \
      git checkout tags/3.3.1

RUN cd opencv/build && \
      cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -D BUILD_opencv_python=OFF \
      -D BUILD_opencv_python2=OFF \
      -D BUILD_opencv_python3=OFF \
      -DOPENCV_EXTRA_MODULES_PATH=$DIRPATH/opencv_contrib/modules .. && \
      make -j$(nproc) install

# Install Open_GV
FROM dev as opengv
COPY --from=gtsam-code $DIRPATH/gtsam $DIRPATH/gtsam
RUN git clone https://github.com/laurentkneip/opengv
RUN cd opengv && \
      mkdir build
RUN cd opengv/build && \
      cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DEIGEN_INCLUDE_DIRS=$DIRPATH/gtsam/gtsam/3rdparty/Eigen \
      -DEIGEN_INCLUDE_DIR=$DIRPATH/gtsam/gtsam/3rdparty/Eigen .. && \
      make -j$(nproc) install

FROM opencv as dbcow2-code
# Install DBoW2
RUN git clone https://github.com/dorian3d/DBoW2.git
FROM dbcow2-code as dbcow2
RUN cd DBoW2 && \
      mkdir build && \
      cd build && \
      cmake .. && \
      make -j1 install

FROM dev-boost AS pgo
# Install RobustPGO
COPY --from=gtsam /usr/local /usr/local
ADD https://api.github.com/repos/MIT-SPARK/Kimera-RPGO/git/refs/heads/master version.json
RUN git clone https://github.com/MIT-SPARK/Kimera-RPGO.git
RUN cd Kimera-RPGO && \
      mkdir build && \
      cd build && \
      cmake .. && \
      make -j$1 install

FROM dev-boost as python3
## [Optional] Install Kimera-VIO-Evaluation from PyPI
RUN apt-get update && \
    apt-get install software-properties-common -y
# Get python3
RUN apt-get update && \
    add-apt-repository ppa:deadsnakes/ppa
RUN apt-get update && \
    apt-get install -y python3.9 python3.9-dev python-pip python3-pip  python3.9-distutils 
RUN python3.9 -m pip install --upgrade pip wheel setuptools requests

FROM python3 as python3-ui
RUN apt-get update && \
    apt-get install -y python-tk python3-tk qt5-default qtbase5-dev python3-pyqt5 qttools5-dev-tools
RUN qmake --version
# RUN python3.9 -m pip install PyQt5==5.15.7

FROM python3-ui as evo1
# Install evo-1 for evaluation
# Hack to avoid Docker's cache when evo-1 master branch is updated.
ADD https://api.github.com/repos/ToniRV/evo-1/git/refs/heads/master version.json
RUN git clone https://github.com/ToniRV/evo-1.git
RUN cd evo-1 && python3.9 $(which pip3) install .

FROM evo1 as kimera-build

COPY --from=gtsam /usr/local /usr/local
COPY --from=gtsam-code $DIRPATH/gtsam $DIRPATH/gtsam
COPY --from=opencv /usr/local /usr/local
COPY --from=opengv /usr/local /usr/local
COPY --from=dbcow2 /usr/local /usr/local
COPY --from=pgo /usr/local /usr/local

# Install spark_vio_evaluation
RUN python3.9 $(which pip3) install ipython prompt_toolkit
# Hack to avoid Docker's cache when spark_vio_evaluation master branch is updated.
ADD https://api.github.com/repos/ToniRV/Kimera-VIO-Evaluation/git/refs/heads/master version.json
RUN git clone https://github.com/ToniRV/Kimera-VIO-Evaluation.git
# We use `pip3 install -e .` so that Jinja2 has access to the webiste template...
RUN cd Kimera-VIO-Evaluation && python3.9 $(which pip3) install -e .

# Add credentials on build
ARG SSH_PRIVATE_KEY
RUN mkdir /root/.ssh/
RUN echo "${SSH_PRIVATE_KEY}" > /root/.ssh/id_rsa
RUN chmod 600 ~/.ssh/id_rsa

# Make sure your domain is accepted
RUN touch /root/.ssh/known_hosts
RUN ssh-keyscan -t rsa github.com >> /root/.ssh/known_hosts

# Install glog, gflags
RUN apt-get update && apt-get install -y libgflags2.2 libgflags-dev libgoogle-glog0v5 libgoogle-glog-dev


# Install Kimera-VIO
# RUN git clone https://github.com/MIT-SPARK/Kimera-VIO.git
COPY . Kimera-VIO/
RUN cd Kimera-VIO \
    && mkdir build \
    && cd build \
    && cmake .. -DCMAKE_CXX_FLAGS='\
    -Wno-comment \
    -Wno-maybe-uninitialized \
    -Wno-parentheses \
    -Wno-pragma-once-outside-header \
    -Wno-reorder \
    -Wno-return-type \
    -Wno-sign-compare \
    -Wno-unused-but-set-variable \
    -Wno-unused-function \
    -Wno-unused-parameter \
    -Wno-unused-value \
    -Wno-unused-variable' \
    && make -j1

FROM base as euroc
# Download and extract EuRoC dataset.
RUN apt-get update && apt-get install -y wget
RUN wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/vicon_room1/V1_01_easy/V1_01_easy.zip
RUN mkdir -p $DIRPATH/euroc && unzip V1_01_easy.zip -d $DIRPATH/euroc

# Yamelize euroc dataset
#RUN bash $DIRPATH/Kimera-VIO/scripts/euroc/yamelize.bash -p $DIRPATH/euroc

#CMD xvfb-run $DIRPATH/Kimera-VIO/scripts/stereoVIOEuroc.bash -p $DIRPATH/euroc -r
#CMD $DIRPATH/Kimera-VIO/scripts/stereoVIOEuroc.bash -p $DIRPATH/euroc -r

FROM base as build
COPY --from=kimera-build /usr/local /usr/local
COPY --from=euroc $DIRPATH/euroc $DIRPATH/euroc

