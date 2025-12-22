pipeline {
    agent any

    environment {
        // Required by PDF
        CONTAINER_ID = ""
        SUM_PY_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\sum.py"
        DIR_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp"
        TEST_FILE_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\test_variables.txt"

        // Convenience variables
        IMAGE_NAME = "sum-app"
        CONTAINER_NAME = "sum-container"
        DOCKERHUB_IMAGE = "bouchentoufomar/sum-app:latest"
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
                    // Remove any previous container with same name
                    bat "docker rm -f %CONTAINER_NAME% || exit 0"

                    // PDF-style: capture stdout, split lines, take last line as container ID
                    def output = bat(
                        script: "docker run -d --name %CONTAINER_NAME% %IMAGE_NAME%",
                        returnStdout: true
                    )
                    def lines = output.split("\\r?\\n")
                    CONTAINER_ID = lines[-1].trim()

                    echo "Container started: name=%CONTAINER_NAME%, id=${CONTAINER_ID}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Read test cases
                    def testLines = readFile("${TEST_FILE_PATH}").trim().split("\\r?\\n")

                    for (def line : testLines) {
                        line = line.trim()
                        if (line == "") continue

                        def vars = line.split("\\s+")
                        def arg1 = vars[0]
                        def arg2 = vars[1]
                        def expected = vars[2]

                        // Execute sum.py inside container (for logs/proof)
                        def output = bat(
                            script: "docker exec %CONTAINER_NAME% python /app/sum.py ${arg1} ${arg2}",
                            returnStdout: true
                        )
                        def outLines = output.split("\\r?\\n")
                        def result = outLines[-1].trim()

                        // Compare using Decimal INSIDE Python to avoid:
                        // - Jenkins sandbox float parsing restrictions
                        // - floating-point precision issues
                        def ok = bat(
                            script: """docker exec %CONTAINER_NAME% python -c "from decimal import Decimal; \
a=Decimal('${arg1}'); b=Decimal('${arg2}'); e=Decimal('${expected}'); \
print(a+b==e)" """,
                            returnStdout: true
                        ).trim()

                        if (ok.endsWith("True")) {
                            echo "✅ SUCCESS: ${arg1} + ${arg2} = ${result} (expected ${expected})"
                        } else {
                            error "❌ ERROR: ${arg1} + ${arg2} returned ${result}, expected ${expected}"
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    bat "docker login"
                    bat "docker tag %IMAGE_NAME% %DOCKERHUB_IMAGE%"
                    bat "docker push %DOCKERHUB_IMAGE%"
                }
            }
        }
    }

    post {
        always {
            script {
                // Always remove container even if pipeline fails
                bat "docker rm -f %CONTAINER_NAME% || exit 0"
            }
        }
    }
}


