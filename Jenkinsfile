pipeline {
    agent any

    environment {
        CONTAINER_ID = ""

        SUM_PY_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\sum.py"
        DIR_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp"
        TEST_FILE_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\test_variables.txt"

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
                    // clean up any old container
                    bat "docker rm -f %CONTAINER_NAME% || exit 0"

                    // PDF-recommended way: capture stdout, split lines, take last line as ID
                    def output = bat(script: "docker run -d --name %CONTAINER_NAME% %IMAGE_NAME%", returnStdout: true)
                    def lines = output.split("\\r?\\n")
                    CONTAINER_ID = lines[-1].trim()

                    echo "Container started: name=%CONTAINER_NAME%, id=${CONTAINER_ID}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    def testLines = readFile("${TEST_FILE_PATH}").trim().split("\\r?\\n")

                    for (def line : testLines) {
                        line = line.trim()
                        if (line == "") continue

                        def vars = line.split("\\s+")
                        def arg1 = vars[0]
                        def arg2 = vars[1]
                        def expected = vars[2]

                        // Run sum.py in the container
                        def output = bat(
                            script: "docker exec %CONTAINER_NAME% python /app/sum.py ${arg1} ${arg2}",
                            returnStdout: true
                        )
                        def outLines = output.split("\\r?\\n")
                        def result = outLines[-1].trim()

                        // Normalize comparison using Python (avoids Groovy float parsing + formatting issues)
                        def norm = bat(
                            script: """docker exec %CONTAINER_NAME% python -c "from decimal import Decimal; \
print(Decimal('${result}') == Decimal('${expected}'))" """,
                            returnStdout: true
                        ).trim()

                        if (norm.endsWith("True")) {
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
                bat "docker rm -f %CONTAINER_NAME% || exit 0"
            }
        }
    }
}

