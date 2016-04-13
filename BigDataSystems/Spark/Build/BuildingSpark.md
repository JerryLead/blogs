# Building Spark

## Using SBT
To build spark-1.6.0 using SBT, we can 

1. git clone Spark-1.6.0
2. Modify the sbt version

	```shell
	cd Spark-1.6.0/projects
	modify build.properties (change sbt.version=0.13.7 to sbt.version=0.13.9)
	```
3. Generate idea modules 
	```shell
	cd Spark-1.6.0
	run ./sbt/sbt 'gen-idea no-classifiers no-sbt-classifiers' // faster
	or  ./sbt/sbt gen-idea
	```
	
4. Import the projects to IDEA
	```shell
	File -> Project from Existing Sources -> SBT -> Use auto-import -> Finish
	```
5. Select the profilers
	```shell
	Maven projects -> Select hadoop-1, maven-3, sbt, scala-2.10, unix
	```
6. Cancel SBT's auto-import
    ```shell
	File -> Setttings -> SBT -> Cancel Use auto-import
	```
	
When encountering some build errors, we can refer to:

1. http://stackoverflow.com/questions/25211071/compilation-errors-in-spark-datatypeconversions-scala-on-intellij-when-using-m
2. http://stackoverflow.com/questions/33311794/import-spark-source-code-into-intellj-build-error-not-found-type-sparkflumepr
3. http://blog.csdn.net/tanglizhe1105/article/details/50530104
4. http://www.iteblog.com/archives/1038