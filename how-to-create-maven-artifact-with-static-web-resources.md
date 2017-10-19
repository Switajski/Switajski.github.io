## How to provide a Single-page application as maven dependency, host it on GitHub and add it to a Java EE backend project

Front end developers of a Single-page application can take a huge advantage of the new tools around node.js. They include testing, package managing and build tools, that are able to do their job easier than complicated Maven configurations with javascript, for which Maven hasn't been created. Using Maven to include Javascript ([WebJar](http://www.webjars.org/documentation)) breaks the frontend's developer responsiblity for the frontend. Nevertheless, for configuration, testing and development purposes, both worlds should integrate easily in Maven and be able to run on one application server. This article shows a possiblity how to achieve this.

## Create a pure HTML/Javascript project as a Maven dependency using GitHub as Maven repository 
A Javascript library like JQuery is already includable by a Maven dependency using [WebJar](http://www.webjars.org/documentation). But what about own javascript project, that includes a JavaScript framework in a `package.json` configured by npm?

Jürgen Kofler describes a [configuration](http://kofler.nonblocking.at/2014/05/bringing-the-java-and-javascript-world-together/), which creates a maven dependency with the web frontend out of a zip file. Then he includes that dependency in the backend. I followed these steps, but used 

* **jar** instead of zip, because of [Servlet 3 specification](https://blogs.oracle.com/alexismp/entry/web_inf_lib_jar_meta). Consequently unzipping is obsolete.
* **GitHub** as Maven repository, 
* **Travis-CI** instead of Jenkins for automation. 

Provided code example is [FlexibleOrdersGui](http://GitHub.com/Switajski/FlexibleOrdersGui) (**Frontend**). This repository was once part of [FlexibleOrders](http://GitHub.com/Switajski/FlexibleOrders) (**Backend**).

Here are the steps necessary to adopt this configuration: 

1. Frontend: Add a grunt task to zip all necessary files.
2. Frontend: Create a pom.xml for turning the jar file into a Maven artifact and upload it to GitHub into `mvn-repo` branch
3. Frontend (optional): Automate execution of grunt task and Maven deployemnt via Travis-CI
4. Backend: Configure Maven to include the GUI via `<dependency>` and let Maven unpack the webfrontend to a static folder provided by an application server

As a consequence I am able to develop the HTML / Javascript / CSS files with my favorite Javascript IDE.

### 1. Add a grunt task to zip all necessary files.

Grunt is used to automate repeated tasks that are necessary to build javascript projects. In our example we use grant to compress [(documentation)](http://grunt-tasks.com/grunt-contrib-compress/) all necessary files from the `app` folder to FlexibleOrdersGui.zip and rename it to a jar file.

![folder structure](http://wiki.switajski.de/gui-folder-structure.png)

[Gruntfile.js](https://GitHub.com/Switajski/FlexibleOrdersGui/blob/master/Gruntfile.js)

	module.exports = function (grunt) {
	    grunt.loadNpmTasks('grunt-contrib-compress');
	    grunt.loadNpmTasks('grunt-rename');
	    grunt.initConfig({
	        compress: {
	            main: {
	                options: {
	                    archive: 'deploy/FlexibleOrdersGui.zip'
	                },
	                files: [
	                    {
	                        expand: true,
	                        cwd: 'app/',
	                        src: ['**'],
	                        app: '/',
	                        dest: 'META-INF/resources'
	                    }
	                ]}
	        },
	        rename: {
	            toJar: {
	                src: 'deploy/FlexibleOrdersGui.zip',
	                dest: 'deploy/FlexibleOrdersGui.jar'
	            }
	        }
	
	    });
	
	    grunt.registerTask('default', [
	        'compress',
	        'rename'
	    ]);
	};
	
If we now run (node.js should be installed):

	npm install
	grunt
	
a jar file is created in the folder deploy.

	
### 2. Create a pom.xml for turning the jar file into a Maven artifact and upload it to GitHub into `mvn-repo` branch

Goal of this step is to make the just created jar file `FlexibleOrdersGui.jar` a Maven artifact. Therefore we configure Maven to have a `build-helper-maven-plugin` goal to turn the webfrontend jar file into a Maven artifact.

([The whole pom.xml is available here](https://GitHub.com/Switajski/FlexibleOrdersGui/blob/master/pom.xml))


			...
			<plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>build-helper-maven-plugin</artifactId>
                <version>1.8</version>
                <executions>
                    <execution>
                        <id>attach-artifacts</id>
                        <phase>package</phase>
                        <goals>
                            <goal>attach-artifact</goal>
                        </goals>
                        <configuration>
                            <artifacts>
                                <artifact>
                                    <file>deploy/FlexibleOrdersGui.jar</file>
                                    <type>jar</type>
                                </artifact>
                            </artifacts>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            ...

Next step is to upload this Maven artifact to GitHub to a branch like `mvn-repo`. `site-maven-plugin` does the job for us in the `deploy`(ment) phase:
	
	    <distributionManagement>
	        <repository>
	            <id>internal.repo</id>
	            <name>Temporary Staging Repository</name>
	            <url>file://${project.build.directory}/mvn-repo</url>
	        </repository>
	    </distributionManagement>
	    <properties>
	        <!-- GitHub server corresponds to entry in ~/.m2/settings.xml -->
	        <GitHub.global.server>GitHub</GitHub.global.server>
	    </properties>
	    <build>
	        <plugins>
	            <plugin>
	                <artifactId>maven-deploy-plugin</artifactId>
	                <version>2.8.1</version>
	                <configuration>
	                    <altDeploymentRepository>internal.repo::default::file://${project.build.directory}/mvn-repo
	                    </altDeploymentRepository>
	                </configuration>
	            </plugin>
	            <plugin>
	                <groupId>com.GitHub.GitHub</groupId>
	                <artifactId>site-maven-plugin</artifactId>
	                <version>0.11</version>
	                <configuration>
	                    <message>Maven artifacts for ${project.version}</message>  <!-- git commit message -->
	                    <noJekyll>true</noJekyll>                                  <!-- disable webpage processing -->
	                    <outputDirectory>${project.build.directory}/mvn-repo</outputDirectory> <!-- matches distribution management repository url above -->
	                    <branch>refs/heads/mvn-repo</branch>                       <!-- remote branch name -->
	                    <includes><include>**/*</include></includes>
	                    <repositoryName>FlexibleOrdersGui</repositoryName>      <!-- GitHub repo name -->
	                    <repositoryOwner>Switajski</repositoryOwner>    <!-- GitHub username  -->
	                </configuration>
	                <executions>
	                    <!-- run site-maven-plugin's 'site' target as part of the build's normal 'deploy' phase -->
	                    <execution>
	                        <goals>
	                            <goal>site</goal>
	                        </goals>
	                        <phase>deploy</phase>
	                    </execution>
	                </executions>
	            </plugin>
	            ...

Username and password of your GitHub repository is provided in your `~/.m2/settings.xml`:

	<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
	      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	      xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
	                          http://maven.apache.org/xsd/settings-1.0.0.xsd">
	 <servers>
	  <server>
	    <id>GitHub</id>
	    <username>[USERNAME]</username>
	    <password>[PASSWORD]</password>
	  </server>
	 </servers>
	</settings>


### 3. Automate execution of grunt task and Maven deployment via Travis-CI
based on [here](http://knowm.org/configure-travis-ci-to-deploy-snapshots/)

#### broken builds from mvn-repo
![failing build from mvn-repo](http://wiki.switajski.de/failing_build_mvn_repo.png)

### 4. Configure Maven to include the GUI via `<dependency>` and let Maven unpack the webfrontend to a static folder provided by an application server

Maven being able to download FlexibleOrdersGui.zip has to know about the location of the GitHub repository. Consequently we create a repository tag with a link to GitHub repository. 

	...
		<repository>
				<id>FlexibleOrdersName-mvn-repo</id>
				<url>https://raw.GitHub.com/Switajski/FlexibleOrdersGui/mvn-repo/</url>
				<snapshots>
					<enabled>true</enabled>
					<updatePolicy>always</updatePolicy>
				</snapshots>
			</repository>
	...

The dependency itself is written as usual:

	<dependency> 
		<groupId>de.switajski</groupId> 
		<artifactId>FlexibleOrdersGui</artifactId>
		<version>0.1.0-SNAPSHOT</version> 
	</dependency>
		

written by Marek Switajski, 7.1.2016
