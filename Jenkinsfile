pipeline {
    agent any

    environment {
        // Required by the PDF
        CONTAINER_ID = ""
        SUM_PY_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\sum.py"
        DIR_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp"
        TEST_FILE_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\test_variables.txt"

        // Project variables
        IMAGE_NAME = "sum-app"
        CONTAINER_NAME = "sum-container"
        DOCKERHUB_IMAGE = "bouchentoufomar/sum-app:latest"
    }

    stages {

        /* =======================
           STEP 2 — BUILD
        ======================== */
        stage('Build') {
            steps {
                dir("${DIR_PATH}") {
                    bat "docker build -t %IMAGE_NAME% ."
                }
            }
        }

        /* =======================
           STEP 3 — RUN
        ======================== */
        stage('Run') {
            steps {
                script {
                    // Remove old container if it exists
                    bat "docker rm -f %CONTAINER_NAME% || exit 0"

                    // Capture container ID (PDF-recommended way)
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

        /* =======================
           STEP 4 — TEST
        ======================== */
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

                        // Run sum.py inside container (for logs)
                        def output = bat(
                            script: "docker exec %CONTAINER_NAME% python /app/sum.py ${arg1} ${arg2}",
                            returnStdout: true
                        )
                        def outLines = output.split("\\r?\\n")
                        def result = outLines[-1].trim()

                        // Exact comparison using Decimal (avoids float precision issues)
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

        /* =======================
           STEP 6 — DEPLOY
        ======================== */
        stage('Deploy') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        bat """
                        echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                        docker tag %IMAGE_NAME% %DOCKERHUB_IMAGE%
                        docker push %DOCKERHUB_IMAGE%
                        """
                    }
                }
            }
        }
    }

    /* =======================
       STEP 5 — POST (CLEANUP)
    ======================== */
    post {
        always {
            script {
                bat "docker rm -f %CONTAINER_NAME% || exit 0"
            }
        }
    }
}


