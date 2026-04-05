pipeline {
    agent any
 
    environment {
        IMAGE_NAME        = "coding-platform"
        CONTAINER_NAME    = "coding-platform-container"
        APP_PORT          = "5173"
        SONAR_PROJECT_KEY = "coding-platform"
        SONAR_HOST_URL    = "http://localhost:9000"
        // Add SONAR_TOKEN in Jenkins Credentials as a Secret Text with id: 'sonar-token'
    }
 
    stages {
 
        // ─────────────────────────────────────────────
        // STAGE 1 — Clone Repository
        // ─────────────────────────────────────────────
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/nikhilesh440/Coding-platform.git'
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 2 — Install Dependencies
        // ─────────────────────────────────────────────
        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 3 — Verify Project Structure
        // ─────────────────────────────────────────────
        stage('Test: Verify Project Structure') {
            steps {
                bat '''
                    IF NOT EXIST src (
                        echo ERROR: src folder missing!
                        exit /b 1
                    )
                    IF NOT EXIST index.html (
                        echo ERROR: index.html missing!
                        exit /b 1
                    )
                    IF NOT EXIST vite.config.js (
                        echo ERROR: vite.config.js missing!
                        exit /b 1
                    )
                    IF NOT EXIST src\\pages\\Login.jsx (
                        echo ERROR: Login.jsx missing!
                        exit /b 1
                    )
                    IF NOT EXIST src\\pages\\Leaderboard.jsx (
                        echo ERROR: Leaderboard.jsx missing!
                        exit /b 1
                    )
                    IF NOT EXIST src\\pages\\Signup.jsx (
                        echo ERROR: Signup.jsx missing!
                        exit /b 1
                    )
                    echo All required project files are present.
                '''
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 4 — LOGIN TEST CASES
        //   TC01: Empty username + password → should alert
        //   TC02: Empty username only       → should alert
        //   TC03: Empty password only       → should alert
        //   TC04: Wrong password            → should alert "Incorrect password"
        //   TC05: Non-existent username     → should alert "No account found"
        //   TC06: Correct credentials       → login success, no alert
        // ─────────────────────────────────────────────
        stage('Test: Login Validation') {
            steps {
                // Copy the test file into src/pages so Jest picks it up
                bat 'copy /Y Login.test.js src\\pages\\Login.test.js'
                bat 'npx jest src/pages/Login.test.js --no-coverage --verbose'
            }
            post {
                failure {
                    echo "LOGIN TESTS FAILED — check empty-field or wrong-password logic in Login.jsx"
                }
                success {
                    echo "LOGIN TESTS PASSED — empty-field and wrong-password cases verified."
                }
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 5 — LEADERBOARD TEST CASES
        //   TC07: Users sorted by stars descending
        //   TC08: Missing stars treated as 0
        //   TC09: Empty localStorage returns empty list
        //   TC10: Single user shown as rank 1
        //   TC11: Equal stars handled without crash
        // ─────────────────────────────────────────────
        stage('Test: Leaderboard Order') {
            steps {
                bat 'copy /Y Leaderboard.test.js src\\pages\\Leaderboard.test.js'
                bat 'npx jest src/pages/Leaderboard.test.js --no-coverage --verbose'
            }
            post {
                failure {
                    echo "LEADERBOARD TESTS FAILED — check sort logic (b.stars - a.stars) in Leaderboard.jsx"
                }
                success {
                    echo "LEADERBOARD TESTS PASSED — star sorting and edge cases verified."
                }
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 6 — FULL JEST TEST SUITE + COVERAGE
        //   Runs all *.test.js files and generates
        //   a coverage/lcov.info for SonarQube
        // ─────────────────────────────────────────────
        stage('Test: Full Suite + Coverage Report') {
            steps {
                bat 'npx jest --coverage --coverageReporters=lcov --coverageReporters=text --verbose'
            }
            post {
                always {
                    // Archive coverage report for Jenkins UI
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Jest Coverage Report'
                    ])
                }
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 7 — SONARQUBE ANALYSIS
        //   Checks:
        //   ✔ Security vulnerabilities in Login.jsx & Signup.jsx
        //   ✔ Code smells & duplications across all components
        //   ✔ Uses Jest lcov coverage from Stage 6
        //   ✔ Quality Gate — fails build if score too low
        // ─────────────────────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    bat """
                        npx sonar-scanner ^
                          -Dsonar.projectKey=%SONAR_PROJECT_KEY% ^
                          -Dsonar.projectName="Coding Platform" ^
                          -Dsonar.sources=src ^
                          -Dsonar.inclusions=**/*.jsx,**/*.js,**/*.json ^
                          -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/*.test.js ^
                          -Dsonar.tests=src ^
                          -Dsonar.test.inclusions=**/*.test.js ^
                          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info ^
                          -Dsonar.host.url=%SONAR_HOST_URL% ^
                          -Dsonar.token=%SONAR_TOKEN% ^
                          -Dsonar.sourceEncoding=UTF-8
                    """
                }
            }
            post {
                failure {
                    echo "SONARQUBE ANALYSIS FAILED — check sonar-scanner installation and SONAR_TOKEN credential."
                }
                success {
                    echo "SONARQUBE ANALYSIS COMPLETE — check http://localhost:9000/dashboard?id=coding-platform"
                }
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 8 — SONARQUBE QUALITY GATE
        //   Blocks the pipeline if SonarQube detects:
        //   - Security vulnerabilities (Login/Signup)
        //   - Too many code smells or duplications
        // ─────────────────────────────────────────────
        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                failure {
                    echo "QUALITY GATE FAILED — Fix security issues in Login.jsx/Signup.jsx or reduce code smells."
                }
                success {
                    echo "QUALITY GATE PASSED — Code is clean and secure!"
                }
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 9 — Build Verification
        // ─────────────────────────────────────────────
        stage('Test: Build Verification') {
            steps {
                bat 'npm run build'
                bat '''
                    IF NOT EXIST dist (
                        echo ERROR: Build failed - dist folder not created!
                        exit /b 1
                    )
                    echo Build successful - dist folder exists.
                '''
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 10 — Build Docker Image
        // ─────────────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME% .'
            }
        }
 
        // ─────────────────────────────────────────────
        // STAGE 11 — Run Container
        // ─────────────────────────────────────────────
        stage('Run Container') {
            steps {
                bat '''
                    docker stop %CONTAINER_NAME% 2>nul
                    docker rm %CONTAINER_NAME% 2>nul
                    FOR /F "tokens=5" %%a IN ('netstat -ano ^| findstr :%APP_PORT%') DO taskkill /PID %%a /F 2>nul
                    docker run -d -p 5173:80 --name %CONTAINER_NAME% %IMAGE_NAME%
                '''
            }
        }
    }
 
    post {
        success {
            echo """
            ============================================================
             Coding Platform deployed at: http://localhost:5173
             SonarQube Report:            http://localhost:9000
             Jest Coverage:               See 'Jest Coverage Report' tab
            ============================================================
            """
        }
        failure {
            echo "Pipeline FAILED. Check console output above for the failing stage."
        }
    }
}