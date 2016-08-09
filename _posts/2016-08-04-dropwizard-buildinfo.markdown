---
layout: post
title:  "Exposing build metadata in your Dropwizard webservice"
date:   2016-08-04 00:18:23 
categories: java dropwizard
---
# Exposing metadata from the jar manifest in Web Service

In my current job we develop as a continuous delivery process - meaning every build is a artifact that can potentially be deployed to production. Therefore we do not version our software anymore - we stick to the snapshots. The question arises - how do we keep track of what is deployed to production? Deployment happens a lot (i.e. daily), so doing this manually is cumbersome and prone to errors. 

I know that Jenkins has a fingerprint ability (you can upload a jar file and it will tell you which build it is) but this seems like a lot of work. We have almost a dozen microservices running. What I was looking for was some kind of dashboard that tells me for every service what version it is.

So my approach is this:

1. Embed metadata in each build (jar) that tracks the Git revision number and other potentially usefull information
2. Expose this information via an HTTP endpoint

This is not rocket science, but it took me a couple of hours of googling and experimenting before I got everything to work. Here is the result, in the hope that it is useful.

## 1. Embedding metadata

Each jar contains the MANIFEST.MF file, which is a dictionary of information. This seems like the right place to keep the information. We use Maven as our build tool, so we add the following plugins to our pom.xml:

<!-- enable JGit plugin -->
			<plugin>
				<groupId>ru.concerteza.buildnumber</groupId>
				<artifactId>maven-jgit-buildnumber-plugin</artifactId>
				<version>1.2.9</version>
				<executions>
					<execution>
						<id>git-buildnumber</id>
						<goals>
							<goal>extract-buildnumber</goal>
						</goals>
						<phase>prepare-package</phase>
					</execution>
				</executions>
			</plugin>
			<!-- write to manifest -->

			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jar-plugin</artifactId>
				<version>2.3.2</version>
				<configuration>
					<archive>
						<manifest>
							<addDefaultImplementationEntries>true</addDefaultImplementationEntries>
						</manifest>
						<manifestEntries>
							<Specification-Title>${project.name}</Specification-Title>
							<Specification-Version>${project.version}</Specification-Version>
							<Specification-Vendor>${project.specification_vendor}</Specification-Vendor>
							<Implementation-Title>${project.groupId}.${project.artifactId}</Implementation-Title>
							<Implementation-Vendor>${project.implementation_vendor}</Implementation-Vendor>
							<Implementation-Version>${git.revision}</Implementation-Version>
							<X-Git-Branch>${git.branch}</X-Git-Branch>
							<X-Git-Tag>${git.tag}</X-Git-Tag>
							<X-Git-Commits-Count>${git.commitsCount}</X-Git-Commits-Count>
							<Build-Time>${maven.build.timestamp}</Build-Time>
						</manifestEntries>
					</archive>
				</configuration>
			</plugin>
			
The first plugin ([JGit](https://github.com/alx3apps/jgit-buildnumber)) allows us to get the git revision number, and the second plugin (maven-jar-plugin) may already be in your pom file, in which case you just add the *manifestEntries* part.			

Our build is hierarchical, and we have a *company-parent* pom file that contains the base dependencies and version numbers. Once we add the plugins to this parent pom, all our build jars contain the build metadata in the manifest.

Let's try to read the information. A jar is just a zip file, so you can unpack the jar and view the manifest file. On OSx or Linux, you can read the file from the commandline without creating temporary files, like this:

```
unzip -q -c target/myservice.jar META-INF/MANIFEST.MF
```

I find this command so useful that I keep it in my bash_profile as a function: 

```
inspectjar() {
  unzip -q -c $1 META-INF/MANIFEST.MF
}
```

## 2. Exposing the manifest
Now that the .jar file contains the metadata, the next step is to read this manifest programatically. It appears that this is not trivial - your jar will also contain its dependent jar files. Each embedded jar will also contain a Manifest, and there seems to be no easy way to find the Manifest that we are looking for. So you end up looping through all manifests to find the right manifest. After some time spend on Stackoverflow [1](http://stackoverflow.com/questions/1272648/reading-my-own-jars-manifest) [2](http://stackoverflow.com/questions/7807782/is-there-any-way-to-access-the-name-value-pairs-of-a-manifest-mf-from-the-enclos?rq=1)
[3](http://stackoverflow.com/questions/37654184/programmatically-read-manifest-mf-from-jar-in-an-ear) I've settled on using [JCabi](http://manifests.jcabi.com) which is a small library that does exactly what I want. I suppose that under the hood it does roughly the same as the suggested solutions on Stackoverflow, but at least I didn't reinvent the wheel.

Using JCabi is easy:

`Manifests.DEFAULT` gives you a Map representing all the keys and values in the manifest. 
`Manifests.DEFAULT.read(key)` gives you a specific value.

It seemed reasonable (and more flexible) to me to just expose the entire Manifest, rather than some specific keys, so next I created a small REST resource to do just that:


{% highlight java %}
package com.mycompany.resources;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;

import com.jcabi.manifests.Manifests;

@Path("meta")
@Produces(MediaType.APPLICATION_JSON)
public class MetaResource {

    @GET
    public Response status() {
        return Response.ok(Manifests.DEFAULT).build();
    }

}
{% endhighlight %}

I'm a big fan of [Dropwizard](http://www.dropwizard.io) which makes it very easy to build REST api's. It embeds Jackson for Json serialization so we don't need to worry about that.

So lastly we register the resource in our main class:
{% highlight java %}
environment.jersey().register(new MetaResource());
{% endhighlight %}
and voila, we can read the manifest over HTTP.

## 3. Exposing the manifest on Dropwizard's admin api
However, if you create the REST resource like above, you end up exposing the information to the outside world. This may not be desirable, depending on whether your service is deployed on a secure network or on the public internet, and whether you have some security framework in place.

Dropwizard has a separate context path /admin which you can access via a different port. It makes more sense to register our Meta resource here.

Unfortunately, it turns out this is not so easy ([see Stackoverflow](http://stackoverflow.com/questions/32764337/dropwizard-new-admin-resource/34411900)). After some more googling I found [this page](https://spin.atomicobject.com/2015/03/30/jersey-servlets-dropwizard/) which explains how to create a new servlet container for our purpose, to which we can register our Meta resource.

At this point you can ask yourself whether you really need a full blown javax.ws resource, or whether a simple servlet will do. I decided that a servlet is just fine, and we can register that directly on the AdminEnvironment. We can reuse the Jackson ObjectMapper from the Dropwizard runtime for the Json serialization. The code for the servlet:

{% highlight java %}
package com.mycompany;

import java.io.IOException;
import java.io.PrintWriter;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.jcabi.manifests.Manifests;


public class MetaServlet extends HttpServlet {

    private final ObjectMapper objectMapper;

    public MetaServlet(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException
    {
        // Set response content type
        resp.setContentType("application/json");

        // Write response.
        PrintWriter out = resp.getWriter();
        String json = objectMapper.writeValueAsString(Manifests.DEFAULT);
        out.println(json);
    }

}
{% endhighlight %}

And it can be registered in our main class like this:

{% highlight java %}
environment.admin().addServlet("meta", new MetaServlet(environment.getObjectMapper()))
        .addMapping("/meta");
{% endhighlight %}

That's it. You could add code to cache the manifest (since it needs to read the file which is a fairly expensive operation) but since we will call it rarely, I've not found that worthwhile. There may be cases in which you want to do that, such as when you add metadata to a HTTP response header. 

The final result looks like this:

```
$ curl localhost:7575/admin/meta
{
  "Archiver-Version": "Plexus Archiver",
  "Implementation-Title": "com.mycompany.gateway-statecontroller-service",
  "Specification-Version": "1.0.0-SNAPSHOT",
  "Implementation-Version": "a66326471fd69b46d9dc8ba9beabf6278b1188de",
  "Manifest-Version": "1.0",
  "Created-By": "Apache Maven",
  "Implementation-Vendor-Id": "com.mycompany",
  "Build-Jdk": "1.8.0",
  "Specification-Vendor": "",
  "Implementation-Vendor": "",
  "Build-Time": "2016-08-08T09:03:49Z",
  "X-Git-Commits-Count": "3212",
  "Specification-Title": "gateway-statecontroller-service",
  "X-Git-Branch": "",
  "Built-By": "jenkins",
  "Main-Class": "com.mycompany.gsc.ApplicationModule"
}
```

Note that Implementation-Version contains the SHA hash that refers to the Git revision that lead to this build.


