# Perform CICD with GitHub Actions

This project utilizes GitHub Actions for CI/CD of a three-tier Java web application on Amazon ECS. The workflow incorporates testing with Maven, Checkstyle checks, SonarQube code analysis, and actively monitors the health of the ECS cluster using Amazon CloudWatch. It further automates Docker image builds, publishes to Amazon ECR, and deploys updates to ECS task definitions, ensuring a robust and monitored development and deployment pipeline.

![Architecture](images/architecture.png)

## Flow of Execution
![Tasks](images/tasks.png)

### CI/CD Workflow

```sh
name: Hprofile Actions
on: [push, workflow_dispatch]
env:
  AWS_REGION: us-east-1
  ECS_REPOSITORY: actapp
  ECS_SERVICE: vproapp-act-svc
  ECS_CLUSTER: vproapp-act
  ECS_TASK_DEFINITION: aws-files/taskdeffile.json
  CONTAINER_NAME: vproapp
jobs:
  Testing:
    runs-on: ubuntu-latest
    steps:
      - name: Code checkout
        uses: actions/checkout@v4

      - name: Maven test
        run: mvn test

      - name: Checkstyle
        run: mvn checkstyle:checkstyle

      # Setup java 11 to be default (sonar-scanner requirement as of 5.x)
      - name: Set Java 11
        uses: actions/setup-java@v3
        with:
         distribution: 'temurin' # See 'Supported distributions' for available options
         java-version: '11'

    # Setup sonar-scanner
      - name: Setup SonarQube
        uses: warchant/setup-sonar-scanner@v7
   
    # Run sonar-scanner
      - name: SonarQube Scan
        run: sonar-scanner
           -Dsonar.host.url=${{ secrets.SONAR_URL }}
           -Dsonar.login=${{ secrets.SONAR_TOKEN }}
           -Dsonar.organization=${{ secrets.SONAR_ORGANIZATION }}
           -Dsonar.projectKey=${{ secrets.SONAR_PROJECT_KEY }}
           -Dsonar.sources=src/
           -Dsonar.junit.reportsPath=target/surefire-reports/ 
           -Dsonar.jacoco.reportsPath=target/jacoco.exec 
           -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
           -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/

      # Check the Quality Gate status.
      - name: SonarQube Quality Gate check
        id: sonarqube-quality-gate-check
        uses: sonarsource/sonarqube-quality-gate-action@master
        # Force to fail step after specific time.
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_URL }} #OPTIONAL   

  BUILD_AND_PUBLISH:
    needs: Testing
    runs-on: ubuntu-latest
    steps:
      - name: Code checkout
        uses: actions/checkout@v4

      - name: Debug Secrets
        run: |
          echo "RDS_USER: ${{ secrets.RDS_USER }}"
          echo "RDS_PASS: ${{ secrets.RDS_PASS }}"
          echo "RDS_ENDPOINT: ${{ secrets.RDS_ENDPOINT }}"

      - name: Update application.properties file
        run: |
          awk -v var="${{ secrets.RDS_USER }}" '/^jdbc.username/ {$0="jdbc.username=" var} 1' src/main/resources/application.properties > tmpfile && mv tmpfile src/main/resources/application.properties
          awk -v var="${{ secrets.RDS_PASS }}" '/^jdbc.password/ {$0="jdbc.password=" var} 1' src/main/resources/application.properties > tmpfile && mv tmpfile src/main/resources/application.properties
          awk -v var="${{ secrets.RDS_ENDPOINT }}" '{gsub(/db01/, var)} 1' src/main/resources/application.properties > tmpfile && mv tmpfile src/main/resources/application.properties
              
      - name: Build & Upload image to ECR
        uses: appleboy/docker-ecr-action@master
        with:
         access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
         secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
         registry: ${{ secrets.REGISTRY }}
         repo: actapp
         region: ${{ env.AWS_REGION }}
         tags: latest,${{ github.run_number }}
         daemon_off: false
         dockerfile: ./Dockerfile
         context: ./


  Deploy:
    needs: BUILD_AND_PUBLISH
    runs-on: ubuntu-latest
    steps:
      - name: Code checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME}}
          image: ${{ secrets.REGISTRY }}/${{ env.ECS_REPOSITORY }}:${{ github.run_number }}
    
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true 
```

### Deployment Validation

![app](images/login1.png)
![app-db](images/logged-in.png)




