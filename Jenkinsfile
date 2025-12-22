pipeline {
    agent any

    environment {
        // Required by the PDF
        CONTAINER_ID = ""

        // Use Jenkins workspace paths (NOT your local C:\Users\... paths)
        SUM_PY_PATH = "${WORKSPACE}\\sum.py"
        DIR_PATH = "${WORKSPACE}"
        TEST_FILE_PATH = "${WORKSPACE}\\test_variables.txt"

        // Convenience variables
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
                    // Remove old container if it exists (prevents conflicts)
                    bat "docker rm -f %CONTAINER_NAME% || exit 0"

                    // Start container in detached mode with a fixed name
                    bat "docker run -d --name %CONTAINER_NAME% %IMAGE_NAME%"

                    // Store the container ID (cleanly) to satisfy the requirement
                    def idOut = bat(
                        script: "docker inspect -f {{.Id}} %CONTAINER_NAME%",
                        returnStdout: true
                    ).trim()

                    env.CONTAINER_ID = idOut
                    echo "Container started with ID: ${env.CONTAINER_ID}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Read all test cases (each line: a b expected_sum)
                    def lines = readFile("${TEST_FILE_PATH}").trim().split("\\r?\\n")

                    for (def line : lines) {
                        line = line.trim()
                        if (line == "") continue

                        def parts = line.split("\\s+")
                        def a = parts[0]
                        def b = parts[1]
                        def expected = parts[2] as BigDecimal

                        // Execute inside container (use container NAME to avoid Windows parsing issues)
                        def raw = bat(
                            script: "docker exec %CONTAINER_NAME% python /app/sum.py ${a} ${b}",
                            returnStdout: true
                        ).trim()

                        // Get last non-empty line as result
                        def outLines = raw.split("\\r?\\n")
                        def last = outLines.reverse().find { it.trim() != "" }.trim()
                        def result = last as BigDecimal

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
                    // NOTE: docker login may require credentials setup in Jenkins
                    // If this hangs, use Jenkins credentials + password-stdin
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
                // Always stop/remove container
                bat "docker rm -f %CONTAINER_NAME% || exit 0"
            }
        }
    }
}

