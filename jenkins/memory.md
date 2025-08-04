# Memory and CPU limits for a Jenkins container
    docker run \
      --name jenkins-lts \
      --detach \
      --memory 4g            \  # cap the container at 4 GiB RAM
      --memory-swap 4g       \  # disable swap (so RAM+swap = 4 GiB total)
      --cpus 2               \  # allow up to 2 CPU cores
      … all your other flags …
      jenkins/jenkins:2.479.3-lts-jdk17

# Tuning the JVM for Jenkins
    docker run \
      … \
      -e JAVA_OPTS='\
        -Xms1g \       # start heap at 1 GiB
        -Xmx3g \       # max heap at 3 GiB
        -XX:+UseG1GC \ # use the G1 garbage collector
        -XX:MaxGCPauseMillis=200 \
        -XX:+PrintGCDetails \
        -XX:+PrintGCDateStamps \
      ' \
      jenkins/jenkins:…
