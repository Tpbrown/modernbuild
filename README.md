This script creates a [Buck](https://buckbuild.com) project from a Maven project.

Does support standard Java artifact types (jar,war).  Doesn't support Maven plugins.

Steps involved:

1. Have the entire project source tree you wish to convert in a single dirctory structure.
2. Build the existing project.  This ensures A) it's buildable, and B) the required dependencies have been resolved.
3. Create an [effective POM](https://maven.apache.org/plugins/maven-help-plugin/effective-pom-mojo.html) for the project.  It's structure dictates the order of processing. Must be named epom.xml
4. [Dump the dependencies](https://maven.apache.org/plugins/maven-dependency-plugin/resolve-mojo.html) of the project and all child projects.  This is used to capture transitive dependencies.  Effective POMs are pre-resolve and don't contain this information. Must be named tdeps.txt
5. Wait 10 seconds to modernize it.
6. Verify the Buck targets are valid
7. Force a resolve of external (binary) dependencies.  _Please_ consider committing these into your source tree instead of fetching.
8. Build and test the project.

or simply:
- mvn install
- mvn help:effective-pom -Doutput=epom.xml
- mvn dependency:resolve -DoutputFile=tdeps.txt
- ~/path/to/modernbuild .
- buck targets
- buck targets|xargs buck fetch
- buck test


##Assumptions
- A source tree will have a single top-level ('parent') POM.  Project folders not referenced in the parent heirarchy will be ignored
- The Group ID of the parent determines what you want to build from source.  Warnings will be emitted for binary dependencies with this group ID.

##Limitations
Many.

- There's an issue with classpaths when running some junit tests.  I'm working on that.
- It'll handle resources incorrectly if they're defined for both test and 'app'.  Might be the issue driving the above.
- It doesn't [filter resources](https://maven.apache.org/pom.html#Resources) yet.
- Occassionally the sha1 of the fetched dependency doesn't match what was discovered in the local Maven cache. This happened to me while building Maven.
- Artifacts with the same name but different group IDs will conflict as duplicate Buck rules.  This may be common with historical artifacts originating back in the m1 era... i.e. 'antlr' was an original org, but later became 'org.antlr' .. but perhaps without a relocation defined.

###example:
```
$cd ~
$git clone https://github.com/tpbrown/modernbuild
$git clone https://github.com/apache/maven
$cd maven
$mvn install
$mvn help:effective-pom -Doutput=epom.xml
$mvn dependency:resolve -DoutputFile=tdeps.txt
$~/modernbuild/modernbuild .
$buck targets
$buck targets|xarg buck fetch
```
[-] PARSING BUCK FILES...FINISHED 0.1s [100%]
Unable to download mvn:com.google.inject:guice:jar:4.0 (hashes do not match. **Expected 199b7acaa05b570bbccf31be998f013963e5e752, saw 0f990a43d3725781b6db7cd0acf0a8b62dfd1649**)
Unable to download mvn:org.apache.maven.wagon:wagon-http:jar:2.10 (hashes do not match. **Expected 4ef309c09abb5f8b2d0c6a4010205db185729cdc, saw b611dd343b14014af6201dd87d121b8d9a7eb5e1**)
BUILD FAILED: //externals:guice-jar failed with exit code -1:
curl
[-] PROCESSING BUCK FILES...FINISHED 0.2s
[-] DOWNLOADING... (0.00 B/S AVG, TOTAL: 0.00 B, 0 Artifacts)
[-] BUILDING...FINISHED 5.7s [100%] (37/39 JOBS, 35 UPDATED, 89.7% CACHE MISS)

```

$vim externals/BUCK  # Fix the SHA1 errors
$buck test
```
It'll still fail because we're not generating source with Modello, but you get the idea...
