FROM   registry.access.redhat.com/ubi8/ubi:8.0

# command line options to pass to the JVM
ENV	  JAVA_OPTIONS -Xmx512m


# Install the Java runtime, create a user for running the app, and set permissions
RUN   yum install -y --disableplugin=subscription-manager java-1.8.0-openjdk-headless && \
      yum clean all --disableplugin=subscription-manager -y && \
      mkdir -p /opt/app-root/bin

# Copy the runnable fat JAR to the container.
COPY  hello-java.jar /opt/app-root/bin/

COPY  run-app.sh /opt/app-root/bin/

RUN   chgrp -R 0 /opt/app-root && \
      chmod -R g=u /opt/app-root

RUN   chmod -x 755 /opt/app-root/run-app.sh

EXPOSE 8080

USER  1001

# Run the fat JAR
CMD   /opt/app-root/bin/run-app.sh
