# Setup StarRocks FE development environment on IDEA

## Requirements

### thrift compiler

Now, StarRocks needs thrift 0.13.x. So you need to install thrift-0.13.0 first.

#### Mac

On Mac you can install it with the following steps.

```
brew tap-new $USER/local-tap
brew extract --version='0.13.0' thrift $USER/local-tap
brew install thrift@0.13.0
```
After installing thrift successfully, you can check by executing following command in command-line.

```
$ thrift -version
Thrift version 0.13.0
```

### Maven Helper plugin

You'd better to install `Maven Helper` plugin in IDEA, in order to generate thrift code easily. You can find it through

```
Intellij IDEA -> Preferences... -> Plugins
```

Search `Maven Helper` in plugins marketplace, then hit `install`. After installing, you need to restart you IDE.

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
1. Generate code in IDE directlly.
2. Copy the compiled java class directly then add the dependencies manually.

### Generate local

#### thrift

Right click on `fe/fe-core/pom.xml`, then 'click' on `Run Maven -> Plugins -> maven-thrift-plugin -> thrift::compile`.
Then the thrift files will be compiled and the coresponding Java files will be generated.

#### proto

You should enter the directory `${STARROCKS_HOME}/fe/fe-core` and execute the following command to compile proto files into
Java sources.

```
mvn exec:exec@make-dir
mvn exec:exec@gensrc
mvn exec:exec@gen_proto
```

Then you can reload you project to include needed generated source files.


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




