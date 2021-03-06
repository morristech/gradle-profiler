# Gradle Profiler

A tool to automate the gathering of profiling and benchmarking information for Gradle builds. 

Profiling information can be captured using several different tools:

- Produce chrome trace output showing build operations such as project configuration and task execution, along with CPU and heap metrics.
- Using a [Gradle build scan](https://gradle.com)
- Java flight recorder built into the Oracle JVM
- Using [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html). 
- Using [YourKit](https://www.yourkit.com) profiler.
- Using [Honest Profiler](https://github.com/RichardWarburton/honest-profiler)

## Installing

First, build and install the `gradle-profiler` app using:

    > ./gradlew installDist
 
This will install the application into `./build/install/gradle-profiler`.

## Profiling a build using JFR

To run the `gradle-profiler` app to profile a build use:

    > ./build/install/gradle-profiler/bin/gradle-profiler --profile jfr --project-dir <root-dir-of-build> <task>...
    
Where `<root-dir-of-build>` is the directory containing the build to be profiled, and `<task>` is the name of the task to run,
exactly as you would use for the `gradle` command.

The profiler will run the build several times to warm up a daemon, then enable the flight recorder and run the build.
Once complete, the results will be written to a file called `profile-out/default/profile.jfr`.

When the profiler runs the build, it will use the tasks you specified. The profiler will use the default
Gradle version, Java installation and JVM args that have been specified for your build, if any.
This generally works the same way as if you were using the Gradle wrapper. For example, the profiler will use the values 
from your Gradle wrapper properties file, if present, to determine which Gradle version to run.

You can use the `--gradle-version` option to specify a Gradle version or installation to use to run the build, overriding the version specified in 
the Gradle wrapper properties file.

## Profiling a build using YourKit

To run the `gradle-profiler` app to capture CPU usage for a build using YourKit:

    > ./build/install/gradle-profiler/bin/gradle-profiler --profile yourkit --project-dir <root-dir-of-build> <task>...

## Benchmarking a build

Run the app using:

    > ./build/install/gradle-profiler/bin/gradle-profiler --benchmark --project-dir <root-dir-of-build> <task>...

Results will be written to a file called `profile-out/benchmark.csv`.

You can use the `--gradle-version` option to specify a Gradle version or installation to use to benchmark the build. You can specify multiple versions
and each of these is used to benchmark the build, allowing you to compare the behaviour of several different Gradle versions.

## Command line options

- `--project-dir`: Directory containing the build to run (required).
- `--benchmark`: Benchmark the build. Runs the builds more times and writes the results to a CSV file.
- `--profile <profiler>`: Profile the build using the specified profiler. Can be used with or without `--benchmark`.
    - `--profile buildscan`: Profile by collecting a build scan
    - `--profile chrome-trace`: Profile by generating chrome trace output. Note that using chrome-trace requires Gradle 3.3+.
    - `--profile jfr`: Profile using JFR
    - `--profile jprofiler`: Profile using JProfiler
    - `--profile hp`: Profile using Honest Profiler
    - `--profile yourkit`: Profile using YourKit. By default, uses CPU tracing and writes a snapshot into the result directory. 
        `--yourkit-memory`: Produce a memory allocation snapshot instead of CPU tracing snapshot.
- `--gradle-version <version>`: Specifies a Gradle version or installation to use to run the builds, overriding the default for the build. You can specify multiple versions.
- `--output-dir <dir>`: Directory to write results to.
- `--no-daemon`: Uses `gradle --no-daemon` to run the builds. The default is to use the Gradle tooling API and Gradle daemon.
- `-D<key>=<value>`: Defines a system property when running the build, overriding the default for the build.
- `--warmups`: Specifies the number of warm-up builds to run for each scenario. Defaults to 2 for profiling, 6 for benchmarking. 
- `--iterations`: Specifies the number of builds to run for each scenario. Defaults to 1 for profiling, 10 for benchmarking.
- `--buck`: Benchmark scenarios using Buck instead of Gradle. By default, only Gradle scenarios are run. You cannot profile a Buck build using this tool.

## Applying changes between build executions

A scenario can define changes that should be applied to the source before each build. You can use this to benchmark or profile an incremental build.
Mutations that are available:

- Add a public method to a Java source class. Each iteration adds a new method and removes the method added by the previous iteration.
- Change the body of a public method in a Java source class.
- Add an entry to a properties file. Each iteration adds a new entry and removes the entry added by the previous iteration.
- Add a string resource to an Android resource file. Each iteration adds a new resource and removes the resource added by the previous iteration.
- Change a string resource in an Android resource file.
- Add a permission to an Android manifest file.

## Scenario file

A scenario file can be provided to define scenarios to benchmark or profile. Use the `--scenario-file` option to provide this. The scenario file is defined in [Typesafe config](https://github.com/typesafehub/config) format.

The scenario file defines one or more scenarios. You can select which scenarios to run by specifying its name on the command-line when running `gradle-profiler`.

Here is an example:

    # Scenarios are run in alphabetical order
    assemble {
        tasks = ["assemble"]
    }
    clean_build {
        versions = ["3.1", "/Users/me/gradle"]
        tasks = ["clean", "build"]
        gradle-args = ["--parallel"]
        system-properties {
            key = "value"
        }
        cleanup-tasks = ["clean"]
        run-using = no-daemon // value can be "no-daemon" or "tooling-api"

        buck {
            targets = ["//thing/res_debug"]
            type = "android_binary" // can be a Buck build rule type or "all"
        }

        warm-ups = 10
        
        apply-abi-change-to = "src/main/java/MyThing.java"
        apply-non-abi-change-to = "src/main/java/MyThing.java"
        apply-property-resource-change-to = "src/main/resources/thing.properties"
        apply-android-resource-change-to = "src/main/res/value/strings.xml"
        apply-android-resource-value-change-to = "src/main/res/value/strings.xml"
        apply-android-manifest-change-to = "src/main/AndroidManifest.xml"
    }

Values are optional and default to the values provided on the command-line or defined in the build.
