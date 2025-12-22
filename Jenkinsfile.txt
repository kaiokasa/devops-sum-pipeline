pipeline {
    agent any

    environment {
        CONTAINER_ID = ""

        SUM_PY_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\sum.py"
        DIR_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp"
        TEST_FILE_PATH = "C:\\Users\\hamoud\\Documents\\dev_fp\\test_variables.txt"
    }

    stages {
        stage('Build') {
            steps {
                dir("${DIR_PATH}") {
                    bat "docker build -t sum-app ."
                }
            }
        }

        stage('Run') {
            steps {
                script {
                    def output = bat(
                        script: "docker run -d sum-app",
                        returnStdout: true
                    ).trim()

                    CONTAINER_ID = output
                    echo "Container started with ID: ${CONTAINER_ID}"
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

                        def raw = bat(
                            script: "docker exec ${CONTAINER_ID} python /app/sum.py ${a} ${b}",
                            returnStdout: true
                        ).trim()

                        def outLines = raw.split("\\r?\\n")
                        def last = outLines.reverse().find { it.trim() != "" }.trim()
                        def result = last as BigDecimal

                        if (result == expected) {
                            echo " SUCCESS: ${a} + ${b} = ${result} (expected ${expected})"
                        } else {
                            error " ERROR: ${a} + ${b} returned ${result}, expected ${expected}"
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    bat "docker login"
                    bat "docker tag sum-app bouchentoufomar/sum-app:latest"
                    bat "docker push bouchentoufomar/sum-app:latest"
                }
            }
        }
    }

    post {
        always {
            script {
                bat "docker stop ${CONTAINER_ID} || exit 0"
                bat "docker rm ${CONTAINER_ID} || exit 0"
            }
        }
    }
}

