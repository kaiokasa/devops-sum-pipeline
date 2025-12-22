pipeline {
    agent any

    environment {
        // Required by the assignment
        CONTAINER_ID = ""

        // Jenkins workspace paths
        SUM_PY_PATH = "${WORKSPACE}\\sum.py"
        DIR_PATH = "${WORKSPACE}"
        TEST_FILE_PATH = "${WORKSPACE}\\test_variables.txt"

        // Docker settings
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

                    // Run container in background
                    bat "docker run -d --name %CONTAINER_NAME% %IMAGE_NAME%"

                    // Store container ID (not strictly needed, but required by assignment)
                    def idOut = bat(
                        script: "docker ps -q -f name=%CONTAINER_NAME%",
                        returnStdout: true
                    ).trim()

                    env.CONTAINER_ID = idOut
                    echo "Container started: name=%CONTAINER_NAME%, id=${env.CONTAINER_ID}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Read test file
                    def lines = readFile("${TEST_FILE_PATH}").trim().split("\\r?\\n")

                    for (def line : lines) {
                        line = line.trim()
                        if (line == "") continue

                        def parts = line.split("\\s+")
                        def a = parts[0]
                        def b = parts[1]
                        def expected = Double.parseDouble(parts[2])

                        // Run Python script inside container
                        def raw = bat(
                            script: "docker exec %CONTAINER_NAME% python /app/sum.py ${a} ${b}",
                            returnStdout: true
                        ).trim()

                        // Extract last non-empty line (sandbox-safe)
                        def outLines = raw.split("\\r?\\n")
                        def lastNonEmpty = ""
                        for (int i = 0; i < outLines.length; i++) {
                            def t = outLines[i].trim()
                            if (t != "") {
                                lastNonEmpty = t
                            }
                        }

                        double result = Double.parseDouble(lastNonEmpty)

                        // Floating-point tolerance
                        double eps = 1e-6

                        if (Math.abs(result - expected) < eps) {
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
                    // NOTE: docker login must already be configured on Jenkins machine
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
                // Always clean up container
                bat "docker rm -f %CONTAINER_NAME% || exit 0"
            }
        }
    }
}
