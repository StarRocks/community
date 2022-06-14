# Setup StarRocks FE development environment on IDEA

## The development tool: IDEA

This article mainly focuses on IDEA's deployment function, which can be used to sync the local code automatically to the remote repository. Using IDEA to do the development will be more  convenient than vim whether for reading code or doing code jump.
* StarRocks FE can also be configured in the same way and FE can be debugged on IDEA remotely, while it will be complicated to debug remotely on IDEA for StarRocks BE.

## Code Import
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA1.png)

Choose the path to FE then do the setup step by step.
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA2.png)

After you finish the setup, there will still be some error messages in the project that you can't run UT. Because FE refers to thrift/protobuf which are some complied classes that will be only generated when you compile.
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA3.png)

## GenSrc（thrift/protobuf/build-in）

There are several ways you can build the thrift/protobuf. Two common methods are as follows:
1. Modify the POM and compile the FE locally.
2. Copy the compiled java class directly then add the dependencies manually.

Here is the detailed description of the 2nd method:
### Copy Gensrc
Firstly, compile the FE on the development machine, find the path : fe/fe-core/target/generated-sources and copy this directory to the local starrocks directory.
Then open File -> Project Structure as is shown is the following picture:
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA4.png)

Select Modules -> fe-core, then choose the "generated-sources" directory for "Add Content Root".
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA5.png)

Once added, you can see that the error on the fe-core has disappeared and now you can try to run again.
> Note: If the code related to thrift/protobuf/builtin changed, you need to copy again. Or you can do a remote sync configuration, to implement the file synchronization from the development machine periodically.

## Code Style settings

Find the option shown in the picture：
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA6.png)

Import the starrocks_intellij_style.xml file which is located under the FE path
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/IDEA7.png)

## Remote synchronisation
This part is familiar with the configuration of Clion,please refer to [Clion](https://github.com/StarRocks/community/blob/main/Contributors/guide/Clion.md)




