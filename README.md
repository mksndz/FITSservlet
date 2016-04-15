#FITS Service
The FITS Service is a project that allows [FITS](http://fitstool.org) to be deployed as a service on either Tomcat or JBoss. The project has been built with Java 8.
(It has been tested on Tomcat 7, Tomcat 8, and minimally tested on JBoss 7.1.)

* <a href="#servlet-usage">FITS Web Service Usage Notes</a>
* <a href="#tomcat">Deploying to Tomcat</a>
* <a href="#jboss">Deploying to JBoss</a>
* <a href="#ide-notes">IDE Notes</a>
* <a href="#test-client">Test Client</a>

## <a name="servlet-usage"></a>FITS Web Service Usage Notes
**This project requires the installation of [FITS](http://fitstool.org) minimum release v.0.9.0.**

Download FITS from the [download](http://fitstool.org/downloads) page and unpack the zip file to a directory on the server. Take note of this directory.

In order to run the FITS Web Service on a server it’s necessary to modify the server’s classpath configuration to add FITS JAR files. Essentially, this mean adding the FITS home directory to the server’s classpath since FITS can (and should) be deployed to a location outside the server. See below for how to do this in Tomcat and JBoss.

The WAR file can be built from the source code using Ant. Alternatively it is available on the [FITS](http://fitstool.org/downloads#fits-servlet) website. Since the WAR file contains the version, it may be desirable to shorten this file name by removing this version number or just renaming the path to the application within the application server used to deploy the application. Note: The FITS Service version is contained within the WAR file's manifest file.
Here is an example of the base URL for accessing this application without modification to the WAR file:
    `http://yourserver.yourdomain.com:<port>/fits/<endpoint>`
Note: If you deploy the WAR file as fits-1.1.0.war then the URL examples contained here will need to contain `fits-1.1.0` instead of `fits`.
The `<endpoint>` is one of the endpoints available within the FITS Service plus parameters to access the service.

### Endpoints
There are currently two services provided by the web applciation.
1. __/examine__ -- Examining a file and returning corresponding metadata containing both FITS output and standard schema output in XML format. (See [FITS](http://fitstool.org) for more information.)
    Substitute 'examine' for `<endpoint>` (see above) plus add a 'file' parameter name with the path to the input file for a GET request or submit a POST request with form data with a 'file' parameter name containing the contents of the file as its payload.
Examples:
        GET (could be from a browser or using curl) `http://yourserver.yourdomain.com:<port>/fits/examine?file=path/to/file`
        POST `curl -i --data-urlencode file=path/to/file http://yourserver.yourdomain.com:<port>/fits/examine`
2. __/version__ -- Obtaining the version of FITS being used to examine input files returned in plain text format.
Example:
        GET (could be from a browser or using curl) `http://yourserver.yourdomain.com:<port>/<project name>/version`
        
See <a href="#test-client">below</a> for a Java test client example.

## <a name="tomcat"></a>Deploying to Tomcat 7 and Tomcat 8
### Add Entries to catalina.properties
It’s necessary to add the location of the FITS directory to the file `$CATALINA_BASE/conf/catalina.properties` then add the FITS lib folder JAR files. (See example below.) 
1. Add the “fits.home” environment variable.
2. Add all “fits.home”/lib/ JAR files to the shared class loader classpath with a wildcard ‘*’ and the `${fits.home}` property substitution.
**Note: Do NOT add any JAR files that are contained in any of the FITS lib/ subdirectories to this classpath entry. They are added programmatically at runtime by the application.**
3. (optional) Rather than using the default log4j.properties file located within the WAR file (which logs to a file within the Tomcat directory structure) it's possible to set up logging to point to an external log4j.properties file. Add a "log4j.configuration" property to catalina.properties pointing to this file. It can be either a full path or have the `file:` protocol at the beginning of the entry. This is managed by the class `edu.harvard.hul.ois.fits.service.listeners.LoggingConfigurator.java`
#### catalina.properties example
Add the following to the bottom of the file:
`fits.home=path/to/fits/home'` (note: no final slash in path)
`shared.loader=${fits.home}/lib/*.jar
log4j.configuration=/Users/dan179/Documents/FITS/log4j.properties`
or
`log4j.configuration=file:/Users/dan179/Documents/FITS/log4j.properties`
#### Additional Information:
**Class loading:** Within the WAR file’s META-INF directory is a Tomcat-specific file, context.xml. This file indicates to the Tomcat server to modify the Tomcat default class loader scheme for this application. The result is that, rather than load the WAR’s classes and JAR files first, classes on Tomcat’s shared classpath will be loaded first. This is critical given the nature of the custom class loaders used in FITS. (This file will be ignored if deploying to JBoss.)

## <a name="jboss"></a>Deploying to JBoss 7.1
Setting up the FITS Web Service within JBoss is somewhat more involved due to the more complex nature of JBoss class isolation.
### Setting Environment Variable ‘fits.home’
This can be one in one of two ways:
1. Set environment variable on command line.
In script that calls `<JBOSS-HOME>/bin/standalone.sh` add `fits_home_dir='/path/to/fits/home'`.
Then start the server with `./standalone.sh -Dfits.home=$fits_home_dir`
2. Set an environment variable in `<JBOSS-HOME>/standalone/configuration/standalone.xml`
Note: This MUST be after the closing element `</extensions>`
```
<system-properties>
  <property name="fits.home" value="/path/to/fits/home"/>
</system-properties>
```
### Adding FITS to classpath
1. `$ mkdir <JBOSS-HOME>/modules/fits/main`
This is where the module.xml file and symlink to “fits.home” will reside.
2. Create module and place in `<JBOSS-HOME>/modules/fits/main`

It’s necessary to add the full list of JAR files in <fits-home>/lib. Adding “*.jar” will **not** work for JBoss nor will ${fits.home} as substitution value from the environment variable (unfortunate for both of these).
As a result of this, if the FITS lib/ directory has changed then this file needs to be changed accordingly. **Do NOT add any JAR files that are contained in any lib/ subdirectories. They are added programmatically at runtime by the application.**

    <?xml version="1.0" encoding="UTF-8"?>
    <module xmlns="urn:jboss:module:1.1" name="fits">
        <resources>
            <resource-root path="fits-0.9.0/lib/fits.jar"/>
            <resource-root path="fits-0.9.0/lib/AES31.jar"/>
            ...
            (you MUST list every JAR files in FITS/lib)
        </resources>
        <dependencies>
            <module name="javax.api"/> <!-- for xerces -->
        </dependencies>
    </module>

Create the symlink to fits.home directory (adjusting paths accordingly):
`$ ln -s <fits.home dir> <JBOSS-HOME>/modules/fits/main`
Add a reference to the module in standalone.xml
```
<profile>
...
    <subsystem xmlns="urn:jboss:domain:ee:1.0">
        <global-modules>
            <module name="fits"/>
        </global-modules>
    </subsystem>
</profile>
```

#### Additional Information:
Within the WAR file’s WEB-INF directory is a JBoss-specific file, jboss-deployment-structure.xml. This file indicates to the JBoss server the location of the JBoss module for FITS. (This file will be ignored if deploying to Tomcat.)

Additional JBoss information can be found here: https://developer.jboss.org/thread/219956?tstart=0

## <a name="ide-notes"></a>IDE Notes
For Eclipse and other IDE's it will be necessary to resolve reference to classes in the FITS project. When adding this project to an IDE for development the FITS project should also be added and referenced. The Ant `build.xml` file will attempt to resolve an environment variable configured location for fits.jar, `fits_jar_location` or fallback to a default location as configured in `build.properties`. One of these two must be satisfied for a successful build. See the `build.xml` file for more details.
At runtime in a server environment these references will be resolved via the Tomcat shared classpath or the JBoss module additions.

## <a name="test-client"></a>Test Client
There is a stand-alone test application within the "test" section of the source code for uploading files via HTTP Post:
`edu.harvard.hul.ois.fits.clients.FormFileUploaderClientApplication.java`
This test can be used directly from within an Eclipse test environment. Its one requirement is the configuration of the input file for processing. In Eclipse go to Run Configuration... > FormFileUploaderClientApplication > Arguments. Then in the Program Arguments section add a file system path to the input file. Optionally, as a second argument add the URL of the server instance running server to override the value
`private static String serverUrl = "http://localhost:8080/fits-1.1.0/examine";`
as set up in the test class.

There is also a shell script in the project, `run-test-client.sh`, to exercise this test class. It takes the same input parameter(s).
