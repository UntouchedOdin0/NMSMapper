# NMSMapper

[![NMS Mapper CI](https://github.com/ScreamingSandals/NMSMapper/actions/workflows/build.yml/badge.svg)](https://github.com/ScreamingSandals/NMSMapper/actions/workflows/build.yml)
<!--[![Repository version](https://img.shields.io/nexus/r/org.screamingsandals.nms/NMSMapper?server=https%3A%2F%2Frepo.screamingsandals.org)](https://repo.screamingsandals.org/#browse/browse:maven-releases:org%2Fscreamingsandals%2Fnms%2FNMSMapper)-->
[![Discord](https://img.shields.io/discord/582271436845219842?logo=discord)](https://discord.gg/4xB54Ts)

A Gradle plugin for generating multi-version NMS accessors.

> Mappings generated by this library can be browsed [here](https://nms.screamingsandals.org/)  
>
> These mappings are provided "AS-IS", with no warranty, so mistakes are possible. We are only solving issues in classes, that we are actively using in ScreamingSandals plugins. If you want to fix anything, feel free to open a pull request or contact us on our Discord server. NMSMapper is made for servers! Many client-side mappings are missing.
>
> NMSMapper is made for servers! Client-only mappings are missing.

## Features and planned features

- [X] Reading Mojang obfuscation maps
- [X] Reading Spigot mappings
- [X] Reading MCP mappings
- [ ] Reading Yarn mappings
- [ ] Reading Fabric Intermediate mappings
- [X] Exporting result for each version to separated json file
- [X] Support for pre-obfuscation map versions
- [X] Generating website where you can easily browse through mapping (just like javadocs)
- [X] Getting extra information about classes (modifiers, superclass, implemented interfaces) - server jar needed
- [X] Getting extra information about fields (type)
- [X] Getting extra information about fields (modifiers) - server jar needed
- [X] Getting extra information about methods (return type)
- [X] Getting extra information about methods (modifiers) - server jar needed
- [X] Comparing between versions, saving result to file (file will be then used by slib to generate reflection)
- [ ] Caching for json files to speed up building
- [X] Using NMSMapper to generate accessors

## Usage

### Gradle
> This project requires Gradle >= 7.0. Maven is not supported. At least JDK 11 is needed for compiling, however the compiled classes use only Java 8 methods.

settings.gradle file:
```groovy
pluginManagement {
    repositories {
        maven {
            url = "https://repo.screamingsandals.org/public/"
        }

        gradlePluginPortal()
    }
}

rootProject.name = 'YourProject'
```

Build.gradle file:
```groovy
plugins {
    id 'org.screamingsandals.nms-mapper' version 'LATEST_VERSION_HERE'
}
```

### Standalone

Download the latest file from releases and save it in your project.  
Create a groovy file (example: `nms.groovy`). Contents of this file will be similar like when using Gradle, however this time only nmsGen section and its content is present.   
Run the generation using:  
`java -jar nms-mapper-standalone.jar -b nms.groovy`

## Examples

### Basic setup

```groovy
/* First add a new source set. Don't use your main source set for generated stuff. */
sourceSets.main.java.srcDirs = ['src/generated/java', 'src/main/java']

/* All other things will be set inside the nmsGen method, */
nmsGen {
    basePackage = "com.example.nms.accessors" // All generated classes will be in this package.
    sourceSet = "src/generated/java" // All generated classes will be part of this source set.
  
    /* This means that the folder will be cleared before generation. 
     *
     * If this value is false, old and no longer used classes won't be removed.
     */
    cleanOnRebuild = true 
  
    /* Here we can define the classes */
}
```
### Defining objects for generation
We want to access the `net.minecraft.core.Rotations` in our plugin. The following method generates a new class, named `RotationsAccessor`, which you can use to retrieve the type.

```groovy
nmsGen {
  /* Setup, see chapter before */
  
  reqClass('net.minecraft.core.Rotations')
}
 ```
The generated code looks like this:
```java
public class RotationsAccessor {

  public static Class<?> getType() {
    return AccessorUtils.getType(RotationsAccessor.class, mapper -> {
        mapper.map("spigot", "1.9.4", "net.minecraft.server.${V}.Vector3f");
        mapper.map("spigot", "1.17", "net.minecraft.core.Vector3f");
        mapper.map("mcp", "1.9.4", "net.minecraft.util.math.Rotations");
        mapper.map("mcp", "1.17", "net.minecraft.src.C_4709_");
      });
  }
  
}
```
We can see we got a new static method `getType()` which returns a class based on the version and platform (Spigot and Forge are supported)
  
Okay, we have the class. But classes are not all, we also need to access some fields, methods or even constructors.
```groovy
nmsGen {
  /* Setup, see chapter before */

  reqClass('net.minecraft.core.Rotations')
          .reqConstructor(float, float, float)
          .reqField('x')
          .reqField('y')
          .reqField('z')
          .reqMethod('getX')
          .reqMethod('getY')
          .reqMethod('getZ')
}
```
This will generate access methods for one constructor, three fields and three methods.
```java
public class RotationsAccessor {
  public static Class<?> getType() {
      // Some generated code
  }

  public static Field getFieldX() {
    // Some generated code
  }

  public static Field getFieldY() {
    // Some generated code
  }

  public static Field getFieldZ() {
    // Some generated code
  }

  public static Constructor<?> getConstructor0() {
    // Some generated code
  }

  public static Method getMethodGetX1() {
    // Some generated code
  }

  public static Method getMethodGetY1() {
    // Some generated code
  }

  public static Method getMethodGetZ1() {
    // Some generated code
  }
}
```
A generated access method for a field will always be called `getField<Name>` and will always return `Field`. A generated access method for a constructor will always be called `getConstructor<Index>` and will always return `Constructor<?>`. This index is generated from the specified order in `build.gradle`.

A generated access method for a method will always be called `getMethod<Name><Index>` and will always return `Method`. Here the index is present, because multiple methods can have other parameters, but the same name.
 
> Note: If a class, a field, a method or a constructor is not found, null is returned.

Maybe you are asking: How to define parameters to methods? It's actually pretty easy, and the same applies to constructors:
```groovy
nmsGen {
  /* Setup, see chapter before */

  var Level = reqClass('net.minecraft.world.level.Level')

  reqClass('net.minecraft.world.entity.decoration.ArmorStand')
          .reqConstructor(Level, double, double, double)
          .reqMethod('setSmall', boolean)
}
```
Parameters can be classes (e.g. `String.class`, in groovy you don't have to specify the .class suffix), strings (`java.lang.String`) or a requested class (in this example it's Level).

> You can also specify another nms class as a parameter without requesting it, in this case you will have to add a prefix `&`:  
> `.reqConstructor('&net.minecraft.world.level.Level', double, double, double)`  
> However the accessor for Level will be generated anyways.

If the class is enum, and we want to retrieve its enum value, we can simply use `reqEnumField` method:
```groovy
nmsGen {
  /* Setup, see chapter before */

  reqClass('net.minecraft.network.protocol.game.ServerboundClientCommandPacket$Action')
          .reqEnumField('PERFORM_RESPAWN')
}
 ```
In this case, the `getField` method will be generated again, however it will return directly the `Object` instead of `Field`.  
```java
public static Object getFieldPERFORM_RESPAWN() {
  // Some generated code
}
```
* For generating accessor classes, you will have to execute `generateNmsComponents` task

### Using alternative mappings

#### Classes

For classes we can use the spigot mappings instead of the official one. We do not use package in this case:
```groovy
reqClass('spigot:PacketPlayInClientCommand$EnumClientCommand')
```
Generated accessor will be called `PacketPlayInClientCommand_i_EnumClientCommandAccessor` instead of `ServerboundClientCommandPacket_i_Action`

#### Methods, Fields, Enum values

For these symbols we can use any mapping supported by NMSMapper (`mojang`, `searge`, `spigot`, `obfuscated`)  
If the mapping is not specified, then Mojang mapping is used.
```groovy
reqClass('spigot:EntityLiving')
    .reqMethod('spigot:getAttributeInstance', IAttribute)
    .reqMethod('spigot:getAttributeMap')
    .reqMethod('spigot:getCombatTracker')
```

Method generated with an alternative mapping will also use the mapping in its name.

> You can also specify which version of mappings has to be used for finding a field, a method or an enum value.
> 
> ```groovy
>    reqClass('spigot:World')
>       .reqField('spigot:methodProfiler:1.9.4')
> ```

## Contributing

### How to add new minecraft version or update its info.json

```
$ ./gradlew generateNmsConfig -PminecraftVersion=1.19
```

or for pre-releases and release-candidates (you have to rename it because it doesn't know how to deal with snapshots yet)
```
$ ./gradlew generateNmsConfig -PminecraftVersion=1.19-rc1 -PcustomVersionString=1.19
```
