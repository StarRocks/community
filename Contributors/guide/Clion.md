# The development configuration setup for StarRocks on Clion
The BE part of StarRocks is developed in C++ and the development platform that we can use is Clion.This article describes how to configure Clion and the Deployment function which is used to synchronise the code to a remote development machine.
## Use Clion to load StarRocks code
The construction of Clion's code tree relies on the CMakeLists file under the root path of the project. StarRocks itself is compiled via CMake, but the CMakeList contains various third party libraries, in order that the Clion cannot parse & compile CMakeLists properly. Here is a simple way to bypass：
1. Add the entire StarRocks root path in Clion，as shown in the following picture：
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/clion1.png)
2. Create a new CMakeLists.txt file under the root path with the following content, the header file of  gensrc and the third-party library  can be configured according to individual needs

```
cmake_minimum_required(VERSION 3.15)
project(StarRocks)

set(CMAKE_CXX_STANDARD 17)

include_directories(
        be/
        be/src
        be/test

        // thrift/protobuf files
        gensrc/build
        gensrc/build/common
        gensrc/build/gen_cpp

        // header files of third-parties
        ../deps/gperftools
        ../deps/include
)


add_executable(StarRocks
        be/src/service/starrocks_main.cpp)
  ```
  3. When finished you can do a code jump via Clion
  
  ## Synchronisation remotely 
  
 1. Free login setup for Server SSH : copy  ~/.ssh/rsa_id.pub file in the local machine as the correspond user on the remote server ~/.ssh/authorized_keys 

 2. Open Clion's Perferences, find the deployment under the build then add a remote machine configuration.
- Type: Choose SFTP 
- Host: Set the IP address of the development machine 
Port: Use SFTP port, the default value is 22
- Authentication : Choose "OpenSSH config and authentication agent" 
User name : Write your own account
Attention: The connection will fail if you do not set authentication and the user name
- Root path: The root path of your own project
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/clion2.png)
  
 3. Set the sync configuration in the Mapping tab according to your own usages
 ![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/clion3.png)
  
 4. There is some data which is no need to synchronize . You can set these paths in the Exclude paths tab, usually  for compiling, and some temporary path
  ![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/clion4.png)
 5. Choose Options tab of the deployment then you can set the format of the files which will be automatic filtered
- Attention:  Make sure that you have set the sync mechanism. It is recommended to choose Always option. In this case,if your laptop is MacBook,the synchronization will be triggered when Clion is on the main page (It will not synchronized when Clion is running in the background)
  ![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/clion5.png)
 6. Right click on the path after all the configuration has been done . You can see the options under Deployment. You can sync the local code to remote, or copy from the remote to the local. Also, you can do a manually sync diff which is recommended at the first time to check whether any configuration items are missing
  ![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/clion6.png)
  
  
  
  
  
