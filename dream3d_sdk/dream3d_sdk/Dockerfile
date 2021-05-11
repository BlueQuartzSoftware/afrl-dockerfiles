FROM library/ubuntu:18.04 AS build
LABEL Author="BlueQuartz Software <info@bluequartz.net>"

SHELL ["/bin/bash", "-c"]

RUN apt-get update && \
  apt-get -y upgrade && \
  apt-get -y install software-properties-common && \
  add-apt-repository ppa:beineri/opt-qt-5.15.2-bionic && \
  apt-get update && \
  apt-get -y install \
  qt515base \
  qt515x11extras \
  qt515svg \
  qt515tools

RUN apt-get -y install \
  wget \
  git \
  build-essential \
  fontconfig \
  dbus \
  libx11-6 \
  libx11-xcb1 \
  libgl1-mesa-dev \
  libpython3.5 \
  python3 \
  python3-distutils \
  python3-pip \
  ninja-build

RUN add-apt-repository ppa:ubuntu-toolchain-r/test && \
  apt-get update && \
  apt-get -y install gcc-9 g++-9


ENV CC=/usr/bin/gcc-9
ENV CXX=/usr/bin/g++-9

# Load up the mkdocs python modules
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
RUN /usr/bin/pip3 install mkdocs-material

# Define the DREAM3D_SDK directory
ENV DREAM3D_SDK=/opt/DREAM3D_SDK

# Define the version of CMake that we will be using
ENV CMAKE_VERSION=3.19
ENV CMAKE_PATCH=8
ENV CMAKE_NAME=cmake-$CMAKE_VERSION.$CMAKE_PATCH-Linux-x86_64

# define the version of Qt5 that we are going to use. This MUST match the values
# that are expected in the qt_headless.js file
ENV QT_MINOR=15
ENV QT_PATCH=0

# Define the remaining variables that will be needed during the build of the
# Docker image
ENV CMAKE_BUILD_TYPE=Release
ENV GIT_BRANCH=develop
ENV WORKSPACE=/opt/DREAM3D-Dev
ENV SDK_SOURCE_DIR=$DREAM3D_SDK/dream3dsuperbuild
ENV SDK_BUILD_DIR=$SDK_SOURCE_DIR/$CMAKE_BUILD_TYPE

# Create the DREAM3D_SDK Directory
RUN mkdir -p $DREAM3D_SDK && \
  chmod -R 0777 $DREAM3D_SDK && \
  # Install CMake 3.x
  cd $DREAM3D_SDK && \
  wget https://cmake.org/files/v$CMAKE_VERSION/$CMAKE_NAME.tar.gz && \
  tar -xzf $CMAKE_NAME.tar.gz && \
  rm -rf $CMAKE_NAME.tar.tgz

# Use CMake to configure the SDK build...
RUN echo "Cloning DREAM3DSuperbuild..." && \
  PATH=$PATH:/opt/qt515/bin && \
  # clone the DREAM3DSuperbuild repo
  cd $DREAM3D_SDK && \
  git clone -b develop https://github.com/bluequartzsoftware/dream3dsuperbuild && \
  # Create a directory for the CMake configuration of the DREAM3D SDK
  mkdir -p $SDK_BUILD_DIR && \
  cd $SDK_BUILD_DIR && \
  $DREAM3D_SDK/$CMAKE_NAME/bin/cmake \
  -DDREAM3D_SDK=$DREAM3D_SDK \
  -DINSTALL_QT5=OFF \
  -DBUILD_TBB=ON \
  -DQt5_QMAKE_EXECUTABLE=/opt/qt5${QT_MINOR}/bin/qmake \
  -Dqt5_version_full=5.$QT_MINOR.$QT_PATCH \
  -DQt5_DIR=/opt/qt515/lib/cmake/Qt5 \
  -DBUILD_ITK=ON \
  -DCMAKE_BUILD_TYPE=$CMAKE_BUILD_TYPE \
  $SDK_SOURCE_DIR

# Build the SDK by compiling sources or downloading prebuilt binaries
# Be sure to untar the data directories so that the installation will work
RUN cd $SDK_BUILD_DIR && \
  export PATH=$PATH:/opt/qt515/bin && \
  make -j "$(nproc)" && \
  cd $DREAM3D_SDK/DREAM3D_Data/Data && \
  tar -xzf SmallIN100.tar.gz && \
  tar -xzf Image.tar.gz && \
  # Remove the entire DREAM3D_SDK Build directory
  rm $DREAM3D_SDK/cmake-$CMAKE_VERSION.$CMAKE_PATCH-Linux-x86_64.tar.gz && \
  rm -rf $DREAM3D_SDK/superbuild/CMP && \
  rm -rf $DREAM3D_SDK/superbuild/discount && \
  rm -rf $DREAM3D_SDK/superbuild/DREAM3D_Data && \
  rm -rf $DREAM3D_SDK/superbuild/Eigen && \
  rm -rf $DREAM3D_SDK/superbuild/haru && \
  rm -rf $DREAM3D_SDK/superbuild/hdf5 && \
  rm -rf $DREAM3D_SDK/superbuild/pybind11 && \
  rm -rf $DREAM3D_SDK/superbuild/qwt && \
  rm -rf $DREAM3D_SDK/superbuild/tbb


RUN mkdir -p $WORKSPACE && \
  cd $WORKSPACE && \
  git clone -b develop https://github.com/bluequartzsoftware/CMP && \
  git clone -b develop https://github.com/bluequartzsoftware/SIMPL && \
  git clone -b develop https://github.com/bluequartzsoftware/SIMPLView && \
  git clone -b develop https://github.com/bluequartzsoftware/H5Support && \
  git clone -b develop https://github.com/bluequartzsoftware/EbsdLib && \
  git clone -b develop https://github.com/bluequartzsoftware/DREAM3D && \
  mkdir -p $WORKSPACE/DREAM3D_Plugins && \
  cd $WORKSPACE/DREAM3D_Plugins && \
  git clone -b develop https://github.com/bluequartzsoftware/ITKImageProcessing && \
  git clone -b develop https://github.com/bluequartzsoftware/SimulationIO && \
  git clone -b develop https://github.com/dream3d/ImageProcessing && \
  git clone -b develop https://github.com/dream3d/DREAM3DReview && \
  git clone -b develop https://github.com/dream3d/UCSBUtilities


ENV WORKSPACE_BUILD=/opt/DREAM3D-Dev/Build

# Configure DREAM3D to build 
RUN mkdir -p $WORKSPACE_BUILD && \
  cd $WORKSPACE_BUILD && \
  $DREAM3D_SDK/$CMAKE_NAME/bin/cmake \
  -G Ninja \
  -DDREAM3D_SDK=$DREAM3D_SDK \
  -DQt5_DIR=/opt/qt515/lib/cmake/Qt5 \
  -DCMAKE_BUILD_TYPE=$CMAKE_BUILD_TYPE \
  -DSIMPL_EMBED_PYTHON=ON \
  -DSIMPL_WRAP_PYTHON=ON \
  $WORKSPACE/DREAM3D

# this section needs to either go away or be replaced with the actual code
RUN echo "#!/bin/bash" > build.sh && \
  echo "set -e" >> build.sh && \
  echo "$DREAM3D_SDK/cmake-$CMAKE_VERSION.$CMAKE_PATCH-Linux-x86_64/bin/ctest -VV -S /dream3d/Support/circleci/CircleContinuous.cmake" >> build.sh && \
  chmod u+x build.sh

ENTRYPOINT ["./build.sh"]

ARG BUILD_DATE
ARG IMAGE=dream3d/dream3d:dream3d_sdk
ARG VCS_REF
ARG VCS_URL
LABEL org.label-schema.build-date=$BUILD_DATE \
  org.label-schema.name=$IMAGE \
  org.label-schema.description="Image containing DREAM.3D environment to build and test DREAM3D" \
  org.label-schema.url="https://github.com/bluequartzsoftware/dream3d" \
  org.label-schema.vcs-ref=$VCS_REF \
  org.label-schema.vcs-url=$VCS_URL \
  org.label-schema.schema-version="1.0"

# Remove the entire DREAM3D_SDK Build directory
#RUN echo "DREAM3D_SDK: $DREAM3D_SDK" && \
#  echo "DREAM3DSuperbuild: $SDK_BUILD_DIR" && \
#  ls -laF $DREAM3D_SDK/

#RUN more  $DREAM3D_SDK/DREAM3D_SDK.cmake


