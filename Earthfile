VERSION 0.8
FROM openjdk:19
WORKDIR /sample-application-ci-cd-poc

build:
    COPY build.gradle ./ 
    COPY src src

    RUN apk update && apk add -y wget unzip

    # Install Gradle Manually
    RUN wget https://services.gradle.org/distributions/gradle-8.5-bin.zip \
        && unzip gradle-8.5-bin.zip -d /opt/ \
        && ln -s /opt/gradle-8.5/bin/gradle /usr/local/bin/gradle

    # Ensure proper permissions
    RUN mkdir -p /home/gradle/.gradle && chmod -R 777 /home/gradle/.gradle

    # Run Gradle build
    RUN gradle build --no-daemon --stacktrace

    SAVE ARTIFACT build/libs/sample-application-ci-cd-poc-3.3.0.jar /output/app.jar

docker:
    FROM harbor.operators.management.eks.us-west-2.aws.smarsh.cloud/smarsh/java-jre-17-alpine:1.0.10
    COPY +build/output/app.jar /opt/app/app.jar
    EXPOSE 8080
    CMD ["java" , "-jar" , "/opt/app/app.jar"]
    SAVE IMAGE sample-app:latest

export-tar:
    FROM earthly/dind:alpine-3.19-docker-25.0.5-r0
    WITH DOCKER --load sample-app:latest=+docker
        RUN docker save -o sample.tar sample-app:latest
    END
    SAVE ARTIFACT sample.tar

snyk-scan-push-harbor:
    FROM snyk/snyk:alpine
    COPY +export-tar/sample.tar sample-app.tar
    RUN --secret SNYK_TOKEN snyk container monitor --exclude-app-vulns docker-archive:sample-app.tar --org=8863bffa-2109-40c3-9794-37f0e7e7ac9b || true
    RUN --secret SNYK_TOKEN snyk container test --exclude-app-vulns oci-archive:sample-app.tar --org=8863bffa-2109-40c3-9794-37f0e7e7ac9b --severity-threshold=critical --fail-on=upgradable
    FROM +docker
    SAVE IMAGE  --push harbor.operators.management.eks.us-west-2.aws.smarsh.cloud/ep/sample-app:1.0.2

harbor-cosign-push:
    
    FROM harbor.operators.management.eks.us-west-2.aws.smarsh.cloud/smarsh/delivery-ci-all:1.0.2
    RUN --secret HARBOR_USERNAME --secret HARBOR_PASSWORD cosign login -u $HARBOR_USERNAME harbor.operators.management.eks.us-west-2.aws.smarsh.cloud -p $HARBOR_PASSWORD
    RUN --secret VAULT_TOKEN --secret VAULT_ADDR --secret VAULT_NAMESPACE cosign sign -a "org=smarsh" -y --key hashivault://smarsh-signing-key harbor.operators.management.eks.us-west-2.aws.smarsh.cloud/ep/sample-app:1.0.2
    
build-scan-and-publish:
    BUILD +build
    BUILD +docker
    BUILD +export-tar
    BUILD +snyk-scan-push-harbor
    
cosign-image:
    BUILD +harbor-cosign-push
