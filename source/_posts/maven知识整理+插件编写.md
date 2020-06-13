---
title: maven知识整理+插件编写
categories: 
    - maven 

tags: 
    - maven
---
我们都知道，maven可以帮助我们管理依赖，否则我们就需要采用古老的方式，单独从其他地方找到对应的jar，然后放到classpath中，这样很麻烦也很容易出现兼容问题
<!-- more -->

<a name="9xWL1"></a>
### refer

- [https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html) (仲裁机制+scope等知识)
- [https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html) (lifeCycle+phase+goal)
- [https://maven.apache.org/guides/plugin/guide-java-plugin-development.html](https://maven.apache.org/guides/plugin/guide-java-plugin-development.html) （插件编写）
- [https://maven.apache.org/guides/mini/guide-configuring-plugins.html#Using_the_executions_Tag](https://maven.apache.org/guides/mini/guide-configuring-plugins.html#Using_the_executions_Tag) (插件参数的配置)


<br />
<br />我们都知道，maven可以帮助我们管理依赖，否则我们就需要采用古老的方式，单独从其他地方找到对应的jar，然后放到classpath中，这样很麻烦也很容易出现兼容问题<br />

<a name="AkmXp"></a>
### maven的一些知识整理
<a name="zuSd4"></a>
#### <br />
<a name="l0Npl"></a>
#### 1. maven的仲裁机制
所谓仲裁，就是出现冲突时会怎么抉择。maven会将项目的依赖信息，做成一棵树的形状，如果有两个一样的jar，但他们的版本不一样

- 离树根越近的那个会被选中;
- 如果他们两个离树根的距离相等，那么就看谁是第一个被发现(依靠他们在pom中的顺序)

比如A项目中有两个D的jar(他们分别是被不同的jar带进来的)，但是版本不一样, 路径如下:
```
A->B->D1.0
A->C->E->D2.0
```
那么D1.0会被选中

<a name="zs5en"></a>
#### 2.  exclude+optional的区别
在依赖某个dependency的时候，我们可能不想要某个dependency，那么我们可以显式的通过exclude将其剔除掉，比如:
```xml
 <dependency>
     <groupId>com.101tec</groupId>
     <artifactId>zkclient</artifactId>
     <version>0.8</version>
     <exclusions>
       <exclusion>
         <artifactId>slf4j-log4j12</artifactId>
         <groupId>org.slf4j</groupId>
       </exclusion>
     </exclusions>
</dependency>
```


其实还有一种方式，就是optional方式，optional的意思相当于"exclude by default", 即默认被排除掉. 比如我们在开发项目的时候，会觉得我们依赖的某个jar并不是 依赖我们所必须的，这时候就可以将其标记位optional。假设

- A项目依赖了`org.yaml:snakeyaml:1.20`但又不希望将它传递给依赖自己的B，那么可以在pom中作以下的dependency声明
- 那么B引用A的时候， `org.yaml:snakeyaml:1.20` 是不会加到B中来的。如果需要加进来，B自己可以显式地在自己的pom中进行定义
```xml
            <dependency>
                <groupId>org.yaml</groupId>
                <artifactId>snakeyaml</artifactId>
                <version>1.20</version>
                <optional>true</optional>
            </dependency>
```
<br />
<a name="HYXQc"></a>
#### 3. 一个dependency的表示
```xml
<project>
  ...
  <dependencies>
    <dependency>
      <groupId>group-c</groupId>
      <artifactId>artifact-b</artifactId>
      <!-- This is not a jar dependency, so we must specify type. -->
      <type>war</type>
		  <classifier>stage</classifier>
      <version>1.0</version>
    </dependency>
 
    <dependency>
      <groupId>group-a</groupId>
      <artifactId>artifact-b</artifactId>
      <!-- This is not a jar dependency, so we must specify type. -->
      <type>bar</type>
      <version>1.1</version>
    </dependency>
  </dependencies>
</project>
```
一般的我们引入一个dependency的时候，我们只要定义其GAV(groupId, artifactId, version)即可。其实一个dependency完整的表示应该包含: GAV+type+classifier

- type: 表示当前是什么类型. 因为默认type=jar，所以很多时候我们不定义. 但是如果对应的dependency不是jar则需要显式声明
- classifier: 表示当前是那种用途的，该值默认为null。比如同一个依赖(即GAV+type一样)，但是需要适应于不同的业务场景，比如一个是给客户A用的，一个是给客户B用的，那么就可以使用classifier来区分了。比如json-lib可能需要同时提供给jdk13和jdk15

![image.png](m-1.png)<br />

```xml
<dependency>
   <groupId>net.sf.json-lib</groupId>
   <artifactId>json-lib</artifactId>
   <version>2.2.2</version>
   <classifier>jdk15</classifier> 
  </dependency>
```


<a name="zuglD"></a>
#### 4. dependency的scope及其传递性
dependency的scope表示其作用范围，同时也确定了其传递性，有6个scope<br />

| scope | 特性 | 是否会传递给依赖项 |
| --- | --- | --- |
| compile | 默认的scope | 是 |
| provided | 跟compile类似. 不同之处在于:它期望在运行时JDK或者Container会提供该dependency. <br /> | 否 |
| runtime | 在compile阶段不是必须的，是给运行时准备的 | 否 |
| test | 测试时使用的 | 否 |
| system | 跟provided类似，但是它不会区从仓库中查找对应的dependency，而是通过指定的路径区找 | 否 |
| import | 将另一个pom引入进来 | 是 |

<br />
<a name="tvvmD"></a>
### 插件相关


<a name="ZjN36"></a>
#### 1. 生命周期简单介绍

<br />maven是基于build lifecycle的概念设计出来的，有三个生命周期:

- clean, 清理上次build的数据
- default，负责编译打包部署
- site，负责文档的生成

其中default+clean是我们平时接触比较多的。而每个生命周期都是由多个phase组合而成的，一个phase对应一个plugin，它可能会包含多个goal. 其中goal就是一个具体的执行任务了，大致结构如下<br />![生命周期.jpg](lifecycle-structure.jpg)<br />
<br />其中一个phase由0个或者多个goal组成，如果该phase中没有一个goal，那么它不会执行；但如果它有多个goal，那么没有特别指定执行那个goal的情况下，它下面的所有goal都会执行。有一点需要注意的是，执行一个phase时，在它之前的所有phase都会被先执行。phase可以直接被执行，就像我们平时的 mvn clean install 其实都是直接执行phase的，只是里面的goal会被全部执行而已
```shell
mvn clean myPhase:myGoal package
```
考虑以上命令，其中clean和package是一个phase，而myPhase:myGoal是一个goal。那么这条命令会如下执行

- 执行clean phase之前的所有phase
- 执行clean phase
- 执行 myPhase:myGoal
- 执行package之前的所有phase
- 执行package phase


<br />
<br />default build lifecycle 会包含以下phase

- validate - 检查项目的正确性，比如是否有东西缺失
- compile - 编译项目的源代码
- test - 对编译好的代码进行测试. 可以通过 `-Dmaven.test.skip=true` 参数跳过该阶段
- package - 对编译好的代码进行打包，通过该phase可以生成jar，所以平时我们会执行` mvn clean package` 进行打包动作
- verify - 对测试结果进行验证，以确保达到了预期的标准
- install -将打好的包安装到本地仓库，以方便本地的其他项目可以使用
- deploy - 将打好的包上传到远程仓库，方便其他项目可以下载使用.


<br />

<a name="mIuQO"></a>
#### 2. 插件编写
其实插件的编写就是一个写Mojo的过程. 在写好之后，如果要单独执行对应的goal需要执行如下命令
```java
mvn {groupId}:{artifactId}:{version}:{goalName}
```
这个命令太长了，我们可以通过配置做做一些精简

1. 在pom中通过引入maven-plugin-plugin插件增加goalPrefix
```xml
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-plugin-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <goalPrefix>duplicate</goalPrefix>
                </configuration>
            </plugin>
```

2. 同时在maven的settings.xm文件中配置如下信息. settings.xml文件的位置:
   1. 自己的: `${user.home}/.m2/settings.xml`
   1. 全局的(大家都可以看到的): `${maven.home}/conf/settings.xml`



```xml
<pluginGroups>
        <pluginGroup>com.demo</pluginGroup>
    </pluginGroups>
```

<br />

<a name="xjKBE"></a>
##### 一个插件的具体的例子


1. pom.xml文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.demo</groupId>
    <artifactId>duplicate-check</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>maven-plugin</packaging>

    <properties>
        <java.version>1.8</java.version>
        <java.encoding>UTF-8</java.encoding>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-plugin-api</artifactId>
            <version>3.6.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugin-tools</groupId>
            <artifactId>maven-plugin-annotations</artifactId>
            <version>3.6.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-project</artifactId>
            <version>2.2.1</version>
            <exclusions>
                <exclusion>
                    <groupId>backport-util-concurrent</groupId>
                    <artifactId>backport-util-concurrent</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-core</artifactId>
            <version>3.6.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.shared</groupId>
            <artifactId>maven-dependency-tree</artifactId>
            <version>3.0.1</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${java.encoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-plugin-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <goalPrefix>duplicate</goalPrefix>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>3.1.1</version>
                <executions>
                    <execution>
                        <phase>initialize</phase>
                        <goals>
                            <goal>properties</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
   

</project>
```


2. DuplicateClassCheckMojo.java 文件
```java
package com.demo;

import com.google.common.collect.Maps;
import com.google.common.collect.Sets;
import org.apache.commons.lang3.StringUtils;
import org.apache.maven.artifact.Artifact;
import org.apache.maven.execution.MavenSession;
import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;
import org.apache.maven.plugin.MojoFailureException;
import org.apache.maven.plugins.annotations.Component;
import org.apache.maven.plugins.annotations.LifecyclePhase;
import org.apache.maven.plugins.annotations.Mojo;
import org.apache.maven.plugins.annotations.Parameter;
import org.apache.maven.plugins.annotations.ResolutionScope;
import org.apache.maven.project.DefaultProjectBuildingRequest;
import org.apache.maven.project.MavenProject;
import org.apache.maven.project.ProjectBuildingRequest;
import org.apache.maven.shared.dependency.graph.DependencyGraphBuilder;
import org.apache.maven.shared.dependency.graph.DependencyGraphBuilderException;
import org.apache.maven.shared.dependency.graph.DependencyNode;

import java.io.File;
import java.util.Collection;
import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Consumer;

// requiresDependencyResolution的含义: 依赖包之间的关系是否需要已经存在? 如果有的依赖包不存在或者下载不下来 需不需要报错？ 这个默认是runtime, 即只会检验scope=compile+runtime的依赖
@Mojo(name = "class-check", defaultPhase = LifecyclePhase.VALIDATE, requiresDependencyResolution = ResolutionScope.TEST, requiresDependencyCollection = ResolutionScope.TEST)
public class DuplicateClassCheckMojo extends AbstractMojo {

    @Parameter(defaultValue = "${project}", readonly = true, required = true)
    private MavenProject project;

    @Parameter(defaultValue = "${session}", readonly = true, required = true)
    private MavenSession session;

    @Component(hint = "default")
    private DependencyGraphBuilder dependencyGraphBuilder;

    /**
     * 当有duplicate的时候，是否需要失败
     */
    @Parameter(defaultValue = "true")
    private boolean failOnDuplicate;

    /**
     * 不需要检查的groupIds
     */
    @Parameter
    private String[] excludeGroupIds;

    /**
     * 不需要检查的 artifactIds
     */
    @Parameter
    private String[] excludeArtifactIds;

    /**
     * 不需要检查的 类名字(是类的全名, 比如: com.demo.DuplicateClassFoundMojo)的前缀
     */
    @Parameter
    private String[] excludeClassPrefixes;


    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {

        DependencyNode dependencyNode = getDependencyNode();
        if (dependencyNode == null) {
            getLog().info(DuplicateClassConstants.LOG_TAG + "dependencyNode is null");
            return;
        }
        Set<Artifact> totalArtifactIds = getArtifacts(dependencyNode);
        resolveDependencies(totalArtifactIds);
    }

    private DependencyNode getDependencyNode() {
        ProjectBuildingRequest buildingRequest = new DefaultProjectBuildingRequest(session.getProjectBuildingRequest());
        buildingRequest.setProject(project);
        try {
            return dependencyGraphBuilder.buildDependencyGraph(buildingRequest, null);
        } catch (DependencyGraphBuilderException e) {
            getLog().info(DuplicateClassConstants.LOG_TAG + "get dependency tree info failed: " + e.getMessage());
            return null;
        }

    }

    private Set<Artifact> getArtifacts(DependencyNode dependencyNode) {

        final Artifact artifact = dependencyNode.getArtifact();
        if (!isAllowedArtifact(artifact)) {
            return Collections.emptySet();
        }

        final List<DependencyNode> children = dependencyNode.getChildren();
        Set<Artifact> totalArtifacts = Sets.newHashSetWithExpectedSize(children != null ? children.size() + 1 : 1);
        totalArtifacts.add(artifact);
        if (children != null && !children.isEmpty()) {
            for (DependencyNode child : children) {
                Artifact childArtifact = child.getArtifact();
                if (!isAllowedArtifact(childArtifact)) {
                    continue;
                }
                totalArtifacts.add(childArtifact);
                List<DependencyNode> myChildren = child.getChildren();
                if (isCollectionEmpty(myChildren)) {
                    continue;
                }
                for (DependencyNode myChild : myChildren) {
                    Set<Artifact> artifactFromChild = getArtifacts(myChild);
                    if (!isCollectionEmpty(artifactFromChild)) {
                        totalArtifacts.addAll(artifactFromChild);
                    }
                }
            }
        }
        return totalArtifacts;
    }

    private boolean isAllowedArtifact(Artifact artifact) {
        return artifact != null && !Artifact.SCOPE_TEST.equals(artifact.getScope());
    }

    private boolean isCollectionEmpty(Collection myChildren) {
        return myChildren == null || myChildren.isEmpty();
    }

    private void resolveDependencies(Set<Artifact> totalArtifactIds) throws MojoFailureException, MojoExecutionException {
        if (totalArtifactIds == null || totalArtifactIds.isEmpty()) {
            return;
        }
        Set<String> skipCheckGroupIds = excludeGroupIds == null ? Collections.emptySet() : Sets.newHashSet(excludeGroupIds);
        Set<String> skipArtifactIds = excludeArtifactIds == null ? Collections.emptySet() : Sets.newHashSet(excludeArtifactIds);
        Set<String> skipClassPrefixes = excludeClassPrefixes == null ? Collections.emptySet() : Sets.newHashSet(excludeClassPrefixes);

        Map<String, String> classArtifactIdMap = Maps.newHashMapWithExpectedSize(1024);
        Map<String, Set<String>> classArtifactIdListMap = Maps.newHashMapWithExpectedSize(1024);
        Map<String, Set<String>> classDistinctArtifactIdCountMap = Maps.newHashMapWithExpectedSize(1024);
        totalArtifactIds.forEach(item -> {
            File file = item.getFile();
            if (file == null) {
                return;
            }
            final String groupId = item.getGroupId();
            final String artifactId = item.getArtifactId();
            if (skipArtifactIds.contains(artifactId) || skipCheckGroupIds.contains(groupId)) {
                return;
            }

            String gaItem = groupId + DuplicateClassConstants.COMMA_STR + artifactId;
            String gavItem = gaItem + DuplicateClassConstants.COMMA_STR + item.getVersion();
            List<String> classes = FileRetriever.getClassesFromPath(file);
            for (String foundClassItem : classes) {
                if (isClassShouldSkip(foundClassItem, skipClassPrefixes)) {
                    continue;
                }
                if (classArtifactIdMap.containsKey(foundClassItem)) {
                    final String lastGavItem = classArtifactIdMap.get(foundClassItem);
                    if (lastGavItem.equals(gavItem)) {
                        continue;
                    }

                    Set<String> existingGavItems = classArtifactIdListMap.getOrDefault(foundClassItem, new HashSet<>());
                    if (existingGavItems.contains(gavItem)) {
                        continue;
                    }
                    existingGavItems.add(lastGavItem);
                    existingGavItems.add(gavItem);
                    classArtifactIdListMap.put(foundClassItem, existingGavItems);


                    // 同样的artifact但是version不一样的, 比如都是 fastjson, 可能有1.0 和 2.0
                    String lastGaItem = getGaInfo(lastGavItem);
                    Set<String> existingGaSet = classDistinctArtifactIdCountMap.getOrDefault(foundClassItem, new HashSet<>());
                    existingGaSet.add(gaItem);
                    existingGaSet.add(lastGaItem);
                    classDistinctArtifactIdCountMap.put(foundClassItem, existingGaSet);
                }
                classArtifactIdMap.put(foundClassItem, gavItem);
            }
        });
        log(classArtifactIdListMap, classDistinctArtifactIdCountMap);
        if (!classArtifactIdListMap.isEmpty() && failOnDuplicate) {
            throw new MojoFailureException(DuplicateClassConstants.LOG_TAG + "there are classes existing in multiple artifact, please check!");
        }
    }

    private boolean isClassShouldSkip(String foundClassItem, Set<String> skipClassPrefixes) {
        if (isCollectionEmpty(skipClassPrefixes)) {
            return false;
        }
        return skipClassPrefixes.stream().filter(StringUtils::isNotBlank).anyMatch(foundClassItem::startsWith);
    }

    private void log(Map<String, Set<String>> classArtifactIdListMap, Map<String, Set<String>> classDistinctArtifactIdCountMap) {
        if (classArtifactIdListMap.isEmpty()) {
            return;
        }
        getLog().warn(DuplicateClassConstants.LOG_TAG + "duplicate class size: " + classArtifactIdListMap.keySet().size());
        classArtifactIdListMap.forEach((classItem, artifactIds) -> {
            Set<String> distinctGaItems = classDistinctArtifactIdCountMap.get(classItem);
            if (distinctGaItems.size() > 1) {
                Consumer<String> printFun = failOnDuplicate ? getLog()::error : getLog()::warn;
                printDuplicateClassInfo(printFun, classItem, artifactIds);
            } else {
                printDuplicateClassInfo(getLog()::warn, classItem, artifactIds);
            }
        });
    }

    private String getGaInfo(String lastGavItem) {
        int lastIndex = lastGavItem.lastIndexOf(DuplicateClassConstants.COMMA_STR);
        return lastGavItem.substring(0, lastIndex);
    }

    private void printDuplicateClassInfo(Consumer<String> consumer, String classItem, Set<String> artifactIds) {
        final String title = DuplicateClassConstants.LOG_TAG + "class[" + classItem + "] exist in following artifact:";
        consumer.accept(title);
        for (String artifactId : artifactIds) {
            final String content = "------------>: " + artifactId;
            consumer.accept(content);
        }
    }
}

```


3. FileRetriever.java文件
```java
package com.demo;

import java.io.File;
import java.io.FilenameFilter;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Enumeration;
import java.util.List;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

public class FileRetriever {


    public static List<String> getClassesFromPath(File path) {
        if (path.isDirectory()) {
            return getClassesFromDirectory(path);
        } else {
            return getClassesFromJarFile(path);
        }
    }

    /**
     * 将 'com/demo/DuplicateClassFoundMojo' 转换成 "com.demo.DuplicateClassFoundMojo"
     *
     * @param fileName
     * @return
     */
    private static String fromFileToClassName(final String fileName) {
        return fileName.replaceAll("[/\\\\]", "\\.");
    }

    private static List<String> getClassesFromJarFile(File path) {

        try {
            if (path.canRead()) {
                List<String> classes = new ArrayList<>();
                JarFile jar = new JarFile(path);
                Enumeration<JarEntry> en = jar.entries();
                while (en.hasMoreElements()) {
                    JarEntry entry = en.nextElement();
                    final String name = entry.getName();
                    if (name.endsWith(DuplicateClassConstants.CLASS_FILE_SUFFIX) && !name.endsWith(DuplicateClassConstants.MODULE_INFO_CLASS_SUFFIX)) {
                        String className = fromFileToClassName(name);
                        classes.add(className);
                    }
                }
                return classes;
            }
        } catch (Exception e) {
            throw new RuntimeException(DuplicateClassConstants.LOG_TAG + "Failed to read classes from jar file: " + path, e);
        }
        return Collections.emptyList();
    }

    private static List<String> getClassesFromDirectory(File path) {
        List<String> classes = new ArrayList<>();
        // get jar files from top-level directory
        List<File> jarFiles = listFiles(path, (dir, name) -> name.endsWith(DuplicateClassConstants.JAR_FILE_SUFFIX), false);
        for (File file : jarFiles) {
            classes.addAll(getClassesFromJarFile(file));
        }

        // get all class-files
        List<File> classFiles = listFiles(path, (dir, name) -> name.endsWith(DuplicateClassConstants.CLASS_FILE_SUFFIX), true);
        int substringBeginIndex = path.getAbsolutePath().length() + 1;
        for (File classfile : classFiles) {
            String className = classfile.getAbsolutePath().substring(substringBeginIndex);
            className = fromFileToClassName(className);
            try {
                classes.add(className);
            } catch (Throwable e) {
                e.printStackTrace();
            }
        }
        return classes;
    }

    private static List<File> listFiles(File directory, FilenameFilter filter, boolean recurse) {

        File[] entries = directory.listFiles();
        if (entries == null || entries.length == 0) {
            return Collections.emptyList();
        }

        List<File> files = new ArrayList<>();
        // Go over entries
        for (File entry : entries) {
            // If there is no filter or the filter accepts the
            // file / directory, add it to the list
            if (filter == null || filter.accept(directory, entry.getName())) {
                files.add(entry);
            }

            // If the file is a directory and the recurse flag
            // is set, recurse into the directory
            if (recurse && entry.isDirectory()) {
                files.addAll(listFiles(entry, filter, true));
            }
        }
        // Return collection of files
        return files;
    }

}

```

<br />
<br />
<br />插件编写，以及源码

- 如何缩短单独运行的命令行<br />如何运行的
- <br />



