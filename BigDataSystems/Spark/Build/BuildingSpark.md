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
	./sbt/sbt 'gen-idea no-classifiers no-sbt-classifiers'
	or  ./sbt/sbt gen-idea
	```
	
4. Import the projects to IDEA
	```shell
	File -> Project from Existing Sources -> SBT -> Use auto-import -> Finish
	```
    
	