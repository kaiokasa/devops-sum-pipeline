pipeline {
    agent any

    environment {
        // Required by the PDF
        CONTAINER_ID = ""

        // Use Jenkins workspace paths (NOT your local C:\Users\... paths)
        SUM_PY_PATH = "${WORKSPACE}\\sum.py"
        DIR_PATH = "${WORKSPACE}"
        TEST_FILE_PATH = "${WORKSPACE}\\test_variables.txt"

        // Convenience
        IMAGE_NAME = "sum-app"
        CONTAINER_NAME = "sum-container"
        DOCKERHUB_REPO = "bouchentoufomar/sum-app:latest"
    }

    stages {
        stage('Build') {
            steps {
                dir("${DIR_PATH}") {
                    bat "docker build -t %IMAGE_NAME% ."
                }
            }
        }

        stage('Run') {
            steps {
                script {
                    // Remove old container if it exists
                    bat "docker rm -f %CONTAINER_NAME% || exit 0"

                    // Start container
                    bat "docker run -d --name %CONTAINER_NAME% %IMAGE_NAME%"

                    // OPTIONAL: try to store container id (not required for the rest)
                    def idOut = bat(script: "docker ps -q -f name=%CONTAINER_NAME%", returnStdout: true).trim()
                    env.CONTAINER_ID = idOut
                    echo "Container started (name=%CONTAINER_NAME%), ID: ${env.CONTAINER_ID}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    def lines = readFile("${TEST_FILE_PATH}").trim().split("\\r?\\n")

                    for (def line : lines) {
                        line = line.trim()
                        if (line == "") continue

                        def parts = line.split("\\s+")
                        def a = parts[0]
                        def b = parts[1]
                        def expected = parts[2] as BigDecimal

                        // Run the script inside the container by NAME
                        def raw = bat(
                            script: "docker exec %CONTAINER_NAME% python /app/sum.py ${a} ${b}",
                            returnStdout: true
                        ).trim()

                        // Find last non-empty line WITHOUT reverse() (sandbox-safe)
                        def outLines = raw.split("\\r?\\n")
                        def lastNonEmpty = ""
                        for (int i = 0; i < outLines.length; i++) {
                            def t = outLines[i].trim()
                            if (t != "") {
                                lastNonEmpty = t
                            }
                        }

                        def result = lastNonEmpty as BigDecimal

                        if (result == expected) {
                            echo "✅ SUCCESS: ${a} + ${b} = ${result} (expected ${expected})"
                        } else {
                            error "❌ ERROR: ${a} + ${b} returned ${result}, expected ${expected}"
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // NOTE: docker login may hang in Jenkins if not configured (see note below)
                    bat "docker login"
                    bat "docker tag %IMAGE_NAME% %DOCKERHUB_REPO%"
                    bat "docker push %DOCKERHUB_REPO%"
                }
            }
        }
    }

    post {
        always {
            script {
                bat "docker rm -f %CONTAINER_NAME% || exit 0"
            }
        }
    }
}
