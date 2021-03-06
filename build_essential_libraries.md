# Build essential libraries for RobotCarRPi
To run the lane detection application on Raspberry Pi (3b+, in my case), we need to build following libraries from source since there is no convenient pre-built package for C++ developers on the intenet.

### OpenCV c++ library v4.0.0: 
see installation guide [here](https://www.pyimagesearch.com/2018/09/26/install-opencv-4-on-your-raspberry-pi/), few things should be noted here:
- it's OK to skip python part since we won't run python code on target Raspberry Pi.
- in the guide above, swap size was changed to 2GB for opencv installation, please keep it for subsequent bazel/tensorflow compilation, then change CONF_SWAPSIZE back to 100MB after tensorflow shared library is successfully built.

### Tensorflow C++ library (v1.11 or 1.12): 
In my case I train the neural network model on Ubuntu 14.04 using tensorflow v1.11, save the model then load it on Raspbian 9 (Stretch), run prediction code using tensorflow v1.12.

Here are few general steps to build the tensorflow library which works for C++ application:

#### Build Bazel from source
We built bazel 0.17.0 for tensorflow v1.11, and built bazel 0.19.2 for tensorflow v1.12.
Be sure to download release version to avoid building issues, see all available release versions from [HERE](https://github.com/bazelbuild/bazel/releases).

Note that bazel supports only few CPU architecture like x86 and ARMv7-A, which means it does NOT support Raspberry PI model 1 because its CPU is based on ARMv6, please recheck the CPU architecture of your Raspberry Pi.

- In command-line interface, you have:
   ```Shell
   wget https://github.com/bazelbuild/bazel/releases/download/0.19.2/bazel-0.19.2-dist.zip
   unzip -qq bazel-0.19.2-dist.zip -d bazel-0.19.2
   cd bazel-0.19.2
   sudo chmod u+w ./* -R
   ```

- Following environment variable will be referred to Bazel when building from scratch, you can allocate more memory to java heap (1 GB, in my case) in order to avoid out-of-memory exception (see issue [HERE](https://github.com/bazelbuild/bazel/issues/1308)). If out-of-memory exception still turns up and you already added this option, maybe you can try increasing Java heap space e.g. -J-Xmx2g
  ```
  export BAZEL_JAVAC_OPT="-J-Xmx1g "
  ```

- then start building from source, it takes several hours.
  ```
  ./compile.sh
  ```
- after compilation finished, copy the binary to following path, type bazel to see if it works:
   ```
   sudo cp output/bazel /usr/local/bin/bazel
   ```


#### use compiled Bazel binary to build tensorflow C++ API from source
- check out tensorflow repository then roll back to previous release version v1.12 (commit ID: a6d8ffa)
   ```
   git clone https://github.com/tensorflow/tensorflow.git
   cd tensorflow
   git checkout r1.12
   ```
   you can recheck if you're already under v1.12 release branch using git branch or git log. (Note: see  other available releases from [HERE](https://github.com/tensorflow/tensorflow/releases) )


- According to [tensorflow issue #24372](https://github.com/tensorflow/tensorflow/issues/24372), if you compile tensorflow branch v1.12 right after previous step you will run into [linking error like THIS](build_essential_libraries.md#ERROR-1) , so we must apply 2 patches downloaded from [HERE](https://gist.github.com/fyhertz/4cef0b696b37d38964801d3ef21e8ce2) prior to subsequent steps.
  download & extract the zip file, then apply
  ```
  git am YOUR_PATCH_NAME.patch
  ```

- clean up previously built files
```
bazel clean --expunge
```

- run ./configure , here is my configuration:
```
./configure
You have bazel 0.19.2- (@non-git) installed.
Please specify the location of python. [Default is /usr/bin/python]:


Found possible Python library paths:
  /usr/local/lib/python2.7/dist-packages
  /usr/lib/python2.7/dist-packages
Please input the desired Python library path to use.  Default is [/usr/local/lib/python2.7/dist-packages]

Do you wish to build TensorFlow with XLA JIT support? [Y/n]: n
No XLA JIT support will be enabled for TensorFlow.

Do you wish to build TensorFlow with OpenCL SYCL support? [y/N]: n
No OpenCL SYCL support will be enabled for TensorFlow.

Do you wish to build TensorFlow with ROCm support? [y/N]: n
No ROCm support will be enabled for TensorFlow.

Do you wish to build TensorFlow with CUDA support? [y/N]: n
No CUDA support will be enabled for TensorFlow.

Do you wish to download a fresh release of clang? (Experimental) [y/N]: n
Clang will not be downloaded.

Do you wish to build TensorFlow with MPI support? [y/N]: n
No MPI support will be enabled for TensorFlow.

Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]:


Would you like to interactively configure ./WORKSPACE for Android builds? [y/N]: n
Not configuring the WORKSPACE for Android builds.

Preconfigured Bazel build configs. You can use any of the below by adding "--config=<>" to your build command. See .bazelrc for more details.
        --config=mkl            # Build with MKL support.
        --config=monolithic     # Config for mostly static monolithic build.
        --config=gdr            # Build with GDR support.
        --config=verbs          # Build with libverbs support.
        --config=ngraph         # Build with Intel nGraph support.
Preconfigured Bazel build configs to DISABLE default on features:
        --config=noaws          # Disable AWS S3 filesystem support.
        --config=nogcp          # Disable GCP support.
        --config=nohdfs         # Disable HDFS support.
        --config=noignite       # Disable Apacha Ignite support.
        --config=nokafka        # Disable Apache Kafka support.
Configuration finished
```

- According to [this page](https://www.tensorflow.org/install/source), users can run optional command **bazel test** prior to **bazel build <YOUR_OPTIONS>**, however it's also OK to skip **bazel test** then directly run **bazel build**.


- then start building (~12 hours). 
  Note that if you build tensorflow from source, be sure to add the options shown at the end of ./configure, to disable the function you don't need.
    ```
    bazel build -c opt --copt="-mfpu=neon-vfpv4" \
      --copt="-funsafe-math-optimizations" \
      --copt="-ftree-vectorize" \
      --copt="-fomit-frame-pointer" \
      --config=noaws \
      --define=grpc_no_ares=true \
      --config=monolithic \
      --copt=-DRASPBERRY_PI \
      --config=nogcp --config=nohdfs --config=noignite --config=nokafka \
      --local_resources 1024,1.0,1.0 \
      --verbose_failures \
      //tensorflow:libtensorflow_cc.so  2>&1 | tee ./tf_nativebuild__1.12.log
    ```
    
    Let's break down a little bit to explain some of important options :

    - ```--config=noaws```
    AWS is NOT enabled in our case, please add this option to disable AWS support & avoid [this linking error](build_essential_libraries.md#ERROR-1).
    - ```--define=grpc_no_ares=true```
    libc-ares-dev should be included when building from source, in newer version of bazel and tensorflow, you can simply add this option instead of manually download/install the package (e.g. by **apt-get install libc-ares-dev**)
    - ```--copt=-DRASPBERRY_PI```
    if you forgot to add this option, you will end up with [this linking error](build_essential_libraries.md#ERROR-2) at the end.
    - ```--local_resources 1024,1.0,1.0```
    1024 means 1024MB memory space (including swap space) for **bazel build**, you can increase the size for your requirement. If you forgot to add this option, compiler will report [out-of-memory error](build_essential_libraries.md#ERROR-3).

#### check the shared library
after previous step you will see libtensorflow_cc.so in ***<TENSORFLOW_SRC_PATH>/bazel-bin/tensorflow/***



### Build correct version of protobuf library (3.6.0)
* checkout the repository
  ```
  git clone https://github.com/protocolbuffers/protobuf.git
  ```
  * check out released branch v3.6.0 (see [all available releases](https://github.com/protocolbuffers/protobuf/releases))
  ```
  git checkout v3.6.0
  ```
  * you can recheck to see if your local repository is in correct release
  ```
  git branch
  ```
  * start building shared library (~1hr), after the step you will see libprotobuf.so.16.0.0 in ***<YOUR_PROTOBUF_REPO_PATH>/src/.lib***
  ```
  make
  ```


### build standalone tensorflow application
then copy essential C header files from your tensorflow source folder to destination application, please modify ***TENSORFLOW_SRC_PATH*** and **TARGET_APPS_PATH**  to fit your requirement.
```
mkdir -p TARGET_APPS_PATH/include/bazel-genfiles  \
         TARGET_APPS_PATH/include/third_party     \
         TARGET_APPS_PATH/include/tensorflow/cc   \
         TARGET_APPS_PATH/include/tensorflow/core

cp -r TENSORFLOW_SRC_PATH/bazel-genfiles/*   TARGET_APPS_PATH/include/bazel-genfiles 
cp -r TENSORFLOW_SRC_PATH/third_party/*      TARGET_APPS_PATH/include/third_party    
cp -r TENSORFLOW_SRC_PATH/tensorflow/cc/*    TARGET_APPS_PATH/include/tensorflow/cc  
cp -r TENSORFLOW_SRC_PATH/tensorflow/core/*  TARGET_APPS_PATH/include/tensorflow/core

// remove other types of files, we only need .h files
rm -rf | find TARGET_APPS_PATH/include -name "*.cc"
rm -rf | find TARGET_APPS_PATH/include -name "*.pb"
rm -rf | find TARGET_APPS_PATH/include -name "*.pbtxt"
rm -rf | find TARGET_APPS_PATH/include -name "BUILD"
```


write a simple test, verify if the shared library works well:
```
```




#### reference
https://gist.github.com/EKami/9869ae6347f68c592c5b5cd181a3b205



-------------------------------------------------------------------------
### Errors you may encounter during the build procedure.

###### ERROR 1
AWS support shouldn't be present in my case, however in the release branch r1.12 users cannot disable AWS support through ./configure , you would end up with linking error like following:
```
ERROR: /home/pi/open_src/tensorflow/1.12/tensorflow/tensorflow/BUILD:449:1: Linking of rule '//tensorflow:libtensorflow_cc.so' failed (Exit 1) gcc failed: error executing command
  (cd /home/pi/.cache/bazel/_bazel_pi/0b3a56398e8b5b41f08cb1aee4909967/execroot/org_tensorflow && \
  exec env - \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games \
    PWD=/proc/self/cwd \
    PYTHON_BIN_PATH=/usr/bin/python \
    PYTHON_LIB_PATH=/usr/local/lib/python2.7/dist-packages \
    TF_DOWNLOAD_CLANG=0 \
    TF_NEED_CUDA=0 \
    TF_NEED_OPENCL_SYCL=0 \
    TF_NEED_ROCM=0 \
  /usr/bin/gcc -shared -o bazel-out/arm-opt/bin/tensorflow/libtensorflow_cc.so -z defs -Wl,--version-script tensorflow/tf_version_script.lds '-Wl,-rpath,$ORIGIN/' -Wl,-soname,libtensorflow_cc.so -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread '-fuse-ld=gold' -Wl,-no-as-needed -Wl,-z,relro,-z,now -B/usr/bin -B/usr/bin -pass-exit-codes -Wl,--gc-sections -Wl,@bazel-out/arm-opt/bin/tensorflow/libtensorflow_cc.so-2.params)

.......................<skip other log messages>.......................

bazel-out/arm-opt/bin/external/aws/_objs/aws/AWSCredentialsProvider.pic.o:AWSCredentialsProvider.cpp:function Aws::Auth::EnvironmentAWSCredentialsProvider::GetAWSCredentials(): error: undefined reference to 'Aws::Environment::GetEnv[abi:cxx11](char const*)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/AWSCredentialsProvider.pic.o:AWSCredentialsProvider.cpp:function Aws::Auth::EnvironmentAWSCredentialsProvider::GetAWSCredentials(): error: undefined reference to 'Aws::Environment::GetEnv[abi:cxx11](char const*)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/AWSCredentialsProvider.pic.o:AWSCredentialsProvider.cpp:function Aws::Auth::EnvironmentAWSCredentialsProvider::GetAWSCredentials(): error: undefined reference to 'Aws::Environment::GetEnv[abi:cxx11](char const*)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/AWSCredentialsProvider.pic.o:AWSCredentialsProvider.cpp:function Aws::Auth::ProfileConfigFileAWSCredentialsProvider::GetConfigProfileFilename[abi:cxx11](): error: undefined reference to 'Aws::FileSystem::GetHomeDirectory[abi:cxx11]()'
bazel-out/arm-opt/bin/external/aws/_objs/aws/AWSCredentialsProvider.pic.o:AWSCredentialsProvider.cpp:function Aws::Auth::ProfileConfigFileAWSCredentialsProvider::GetCredentialsProfileFilename[abi:cxx11](): error: undefined reference to 'Aws::Environment::GetEnv[abi:cxx11](char const*)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/AWSCredentialsProvider.pic.o:AWSCredentialsProvider.cpp:function Aws::Auth::ProfileConfigFileAWSCredentialsProvider::GetCredentialsProfileFilename[abi:cxx11](): error: undefined reference to 'Aws::FileSystem::GetHomeDirectory[abi:cxx11]()'
bazel-out/arm-opt/bin/external/aws/_objs/aws/ClientConfiguration.pic.o:ClientConfiguration.cpp:function Aws::Client::ComputeUserAgentString(): error: undefined reference to 'Aws::OSVersionInfo::ComputeOSVersionString[abi:cxx11]()'
bazel-out/arm-opt/bin/external/aws/_objs/aws/DateTimeCommon.pic.o:DateTimeCommon.cpp:function Aws::Utils::DateTime::ConvertTimestampStringToTimePoint(char const*, Aws::Utils::DateFormat): error: undefined reference to 'Aws::Time::TimeGM(tm*)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/DateTimeCommon.pic.o:DateTimeCommon.cpp:function Aws::Utils::DateTime::ConvertTimestampToLocalTimeStruct() const: error: undefined reference to 'Aws::Time::LocalTime(tm*, long)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/DateTimeCommon.pic.o:DateTimeCommon.cpp:function Aws::Utils::DateTime::ConvertTimestampToGmtStruct() const: error: undefined reference to 'Aws::Time::GMTime(tm*, long)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/TempFile.pic.o:TempFile.cpp:function .LTHUNK4: error: undefined reference to 'Aws::FileSystem::RemoveFileIfExists(char const*)'
bazel-out/arm-opt/bin/external/aws/_objs/aws/TempFile.pic.o:TempFile.cpp:function Aws::Utils::ComputeTempFileName(char const*, char const*): error: undefined reference to 'Aws::FileSystem::CreateTempFilePath[abi:cxx11]()'

.......................<skip other log messages>.......................

collect2: error: ld returned 1 exit status
```


###### ERROR 2
Accroding to [the issue](https://github.com/tensorflow/tensorflow/issues/17790#issuecomment-392185433) and [this issue](https://github.com/tensorflow/serving/issues/852), you will get a following linking error without the define marco RASPBERRY_PI, which is referenced in tensorflow/core/platform/platform.h

```
ERROR: /home/pi/open_src/tensorflow/1.12/tensorflow/tensorflow/BUILD:477:1: Linking of rule '//tensorflow:libtensorflow_cc.so' failed (Exit 1): gcc failed: error executing command
  (cd /home/pi/.cache/bazel/_bazel_pi/0b3a56398e8b5b41f08cb1aee4909967/execroot/org_tensorflow && \
  exec env - \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games \
    PWD=/proc/self/cwd \
    PYTHON_BIN_PATH=/usr/bin/python \
    PYTHON_LIB_PATH=/usr/local/lib/python2.7/dist-packages \
    TF_DOWNLOAD_CLANG=0 \
    TF_NEED_CUDA=0 \
    TF_NEED_OPENCL_SYCL=0 \
    TF_NEED_ROCM=0 \
  /usr/bin/gcc -shared -o bazel-out/arm-opt/bin/tensorflow/libtensorflow_cc.so '-Wl,-rpath,$ORIGIN/../_solib_arm/_U_S_Stensorflow_Clibtensorflow_Ucc.so___Utensorflow' -Lbazel-out/arm-opt/bin/_solib_arm/_U_S_Stensorflow_Clibtensorflow_Ucc.so___Utensorflow -z defs -Wl,--version-script tensorflow/tf_version_script.lds '-Wl,-rpath,$ORIGIN/' -Wl,-soname,libtensorflow_cc.so -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread -pthread '-fuse-ld=gold' -Wl,-no-as-needed -Wl,-z,relro,-z,now -B/usr/bin -B/usr/bin -pass-exit-codes -Wl,--gc-sections -Wl,@bazel-out/arm-opt/bin/tensorflow/libtensorflow_cc.so-2.params)
bazel-out/arm-opt/bin/tensorflow/core/kernels/_objs/list_kernels/list_kernels.pic.o:list_kernels.cc:function tensorflow::TensorListStack<Eigen::ThreadPoolDevice, tensorflow::bfloat16>::Compute(tensorflow::OpKernelContext*): error: undefined reference to 'void tensorflow::ConcatCPU<tensorflow::bfloat16>(tensorflow::DeviceBase*, std::vector<std::unique_ptr<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix, std::default_delete<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix> >, std::allocator<std::unique_ptr<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix, std::default_delete<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix> > > > const&, tensorflow::TTypes<tensorflow::bfloat16, 2, int>::Matrix*)'
bazel-out/arm-opt/bin/tensorflow/core/kernels/_objs/list_kernels/list_kernels.pic.o:list_kernels.cc:function tensorflow::TensorListGather<Eigen::ThreadPoolDevice, tensorflow::bfloat16>::Compute(tensorflow::OpKernelContext*): error: undefined reference to 'void tensorflow::ConcatCPU<tensorflow::bfloat16>(tensorflow::DeviceBase*, std::vector<std::unique_ptr<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix, std::default_delete<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix> >, std::allocator<std::unique_ptr<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix, std::default_delete<tensorflow::TTypes<tensorflow::bfloat16, 2, int>::ConstMatrix> > > > const&, tensorflow::TTypes<tensorflow::bfloat16, 2, int>::Matrix*)'
collect2: error: ld returned 1 exit status
Target //tensorflow:libtensorflow_cc.so failed to build
INFO: Elapsed time: 28771.690s, Critical Path: 4467.15s, Remote (0.00% of the time): [queue: 0.00%, setup: 0.00%, process: 0.00%]
INFO: 3375 processes: 3375 local.
FAILED: Build did NOT complete successfully
```

###### ERROR 3
Failed to allocate memory when compiling source, chances are the memory limit is not configured in **bazel build**
```
ERROR: /home/pi/open_src/tensorflow/1.12/tensorflow-master/tensorflow/core/kernels/BUILD:2919:1: C++ compilation of rule '//tensorflow/core/kernels:matrix_square_root_op' failed (Exit 1): gcc failed: error executing command
  (cd /home/pi/.cache/bazel/_bazel_pi/fd3a3edf6b94d1539919e6f15e50731b/execroot/org_tensorflow && \
  exec env - \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games \
    PWD=/proc/self/cwd \
    PYTHON_BIN_PATH=/usr/bin/python \
    PYTHON_LIB_PATH=/usr/local/lib/python2.7/dist-packages \
    TF_DOWNLOAD_CLANG=0 \
    TF_NEED_CUDA=0 \
    TF_NEED_OPENCL_SYCL=0 \
    TF_NEED_ROCM=0 \
  /usr/bin/gcc -U_FORTIFY_SOURCE -fstack-protector -Wall -B/usr/bin -B/usr/bin -Wunused-but-set-parameter -Wno-free-nonheap-object -fno-omit-frame-pointer -g0 -O2 '-D_FORTIFY_SOURCE=1' -DNDEBUG -ffunction-sections -fdata-sections '-std=c++0x' -MD -MF bazel-out/arm-opt/bin/tensorflow/core/kernels/_objs/matrix_square_root_op/matrix_square_root_op.pic.d '-frandom-seed=bazel-out/arm-opt/bin/tensorflow/core/kernels/_objs/matrix_square_root_op/matrix_square_root_op.pic.o' -fPIC -D__CLANG_SUPPORT_DYN_ANNOTATION__ -DEIGEN_MPL2_ONLY '-DEIGEN_MAX_ALIGN_BYTES=64' '-DEIGEN_HAS_TYPE_TRAITS=0' -DTF_USE_SNAPPY -iquote . -iquote bazel-out/arm-opt/genfiles -iquote bazel-out/arm-opt/bin -iquote external/com_google_absl -iquote bazel-out/arm-opt/genfiles/external/com_google_absl -iquote bazel-out/arm-opt/bin/external/com_google_absl -iquote external/bazel_tools -iquote bazel-out/arm-opt/genfiles/external/bazel_tools -iquote bazel-out/arm-opt/bin/external/bazel_tools ................. -iquote external/zlib_archive -iquote bazel-out/arm-opt/genfiles/external/zlib_archive -iquote bazel-out/arm-opt/bin/external/zlib_archive -isystem external/eigen_archive -isystem bazel-out/arm-opt/genfiles/external/eigen_archive .............................. -isystem bazel-out/arm-opt/genfiles/external/zlib_archive -isystem bazel-out/arm-opt/bin/external/zlib_archive -funsafe-math-optimizations '-mfpu=neon-vfpv4' -ftree-vectorize -fomit-frame-pointer -DEIGEN_AVOID_STL_ARRAY -Iexternal/gemmlowp -Wno-sign-compare -fno-exceptions '-ftemplate-depth=900' -pthread -fno-canonical-system-headers -Wno-builtin-macro-redefined '-D__DATE__="redacted"' '-D__TIMESTAMP__="redacted"' '-D__TIME__="redacted"' -c tensorflow/core/kernels/matrix_square_root_op.cc -o bazel-out/arm-opt/bin/tensorflow/core/kernels/_objs/matrix_square_root_op/matrix_square_root_op.pic.o)
virtual memory exhausted: Cannot allocate memory
Target //tensorflow:libtensorflow_cc.so failed to build
INFO: Elapsed time: 5725.553s, Critical Path: 5711.34s, Remote (0.00% of the time): [queue: 0.00%, setup: 0.00%, process: 0.00%]
INFO: 182 processes: 182 local.
FAILED: Build did NOT complete successfully
```


