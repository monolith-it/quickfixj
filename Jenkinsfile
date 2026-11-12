// Jenkinsfile – Monolith fork of QuickFIX/J (QFJ_2_3_2-Xetra, QFJ_2_3_2-mmts).
// Builds the fork and publishes it to Reposilite "releases", where the middleware builds find it.
// Releases can't be overwritten: to publish a change, first raise the version in every pom, for example
//   mvn versions:set -DnewVersion=2.3.2-XETRA-2 -DgenerateBackupFiles=false
// and update quickfixj.version in middleware. A build of an already published version only compiles.
pipeline {
    agent { label 'maven' }
    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }
    environment {
        JAVA_HOME = '/opt/java/openjdk'      // JDK 21; the fork still compiles for Java 8 (jdkLevel 1.8)
        MAVEN_OPTS = '-Xmx3g'                // upstream CI uses 3 GB for the message code generation
        RELEASES = 'http://reposilite:8080/releases'
    }
    stages {
        stage('Build and publish') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'reposilite-ci',
                        usernameVariable: 'REPOSILITE_USER', passwordVariable: 'REPOSILITE_TOKEN')]) {
                    sh '''
                        VERSION=$(mvn -B -q -N help:evaluate -Dexpression=project.version -DforceStdout)
                        PARENT_POM="$RELEASES/org/quickfixj/quickfixj-parent/$VERSION/quickfixj-parent-$VERSION.pom"
                        if [ -n "${CHANGE_ID:-}" ]; then
                            echo "Pull request: build only"
                            GOAL=verify
                        elif curl -fsS -o /dev/null -u "$REPOSILITE_USER:$REPOSILITE_TOKEN" "$PARENT_POM"; then
                            echo "$VERSION is already in Reposilite: build only. Raise the version to publish a change."
                            GOAL=verify
                        else
                            echo "Publishing $VERSION to Reposilite"
                            GOAL=deploy
                        fi
                        # Upstream's test suite is long and timing-sensitive; the fork changes only FIX dictionaries.
                        # Javadoc 2.10.4 fails on JDK 21, and no build needs the javadoc jars.
                        # quickfixj-distribution (zip archives) is not used by any build.
                        mvn -B -ntp -DskipTests -Dmaven.javadoc.skip=true -pl '!quickfixj-distribution' $GOAL \
                            -DaltDeploymentRepository=reposilite-releases::default::$RELEASES
                    '''
                }
            }
        }
    }
}
