## day 49

codes:

code quality.yml

# linting SAST 
#- lint flake-8 SAST- Bandit
name: Code Quality

on:
    workflow_call:

jobs:
    validate:
        runs-on: ubuntu-latest
        strategy:
            fail-fast: false
            matrix:
                python-version: ["3.11", "3.12", "3.13"]
        steps:
            - name: checkout code
              uses: actions/checkout@v4

            - name: setup python ${{ matrix.python-version}}
              uses: actions/setup-python@v5
              with:
                python-version: ${{ matrix.python-version}}
                
            - name: install dependencies
              run: pip install -r requirements.txt

            - name: run linter
              run: flake8 app.py 

            - name: run SAST
              run: bandit -r app.py

dependency-scan.yml

name: Dependency Scan

on:
    workflow_call:

jobs:
    dependency-check:
        runs-on: ubuntu-latest
        steps:
            - name: Code Checkout
              uses: actions/checkout@v4
            
            - name: setup python 3.12
              uses: actions/setup-python@v5
              with:
                python-version: "3.12"

            - name: install Pip audit
              run: pip install pip-audit
            
            - name: Audit Dependencies
              run: pip-audit -r requirements.txt
docker-lint.yml
name: Docker Lint

on:
    workflow_call:

jobs:
    validate-dockerfile:
        runs-on: ubuntu-latest

        steps:
            - name: Checkout Code
              uses: actions/checkout@v4

            - name: Validate Dockerfile
              uses: hadolint/hadolint-action@v3.1.0
              with:
                dockerfile: Dockerfile

deploy-to-server.yml
name: Docker Lint

on:
    workflow_call:

jobs:
    validate-dockerfile:
        runs-on: ubuntu-latest

        steps:
            - name: Checkout Code
              uses: actions/checkout@v4

            - name: Validate Dockerfile
              uses: hadolint/hadolint-action@v3.1.0
              with:
                dockerfile: Dockerfile
docker-build-push.yml
name: Docker Build & Push

on: 
    workflow_call:

jobs:
    build-and-push:
        runs-on: ubuntu-latest

        steps:

            - name: Code Checkout
              uses: actions/checkout@v4

            - name: Login to Docker Hub
              uses: docker/login-action@v3
              with:
                username: ${{ vars.DOCKERHUB_USER }}
                password: ${{ secrets.DOCKERHUB_TOKEN }}
                
            - name: Build & Push to Docker Hub
              uses: docker/build-push-action@v6
              with:
                context: .
                push: true
                tags: |
                    ${{ vars.DOCKERHUB_USER }}/github-actions-app:${{ github.ref_name }}
                    ${{ vars.DOCKERHUB_USER }}/github-actions-app:latest
                    ${{ vars.DOCKERHUB_USER }}/github-actions-app:${{ github.sha}}


image-scan.yml
name: Image Scanner

on:
    workflow_call:

jobs:
    image-scanner:
        runs-on: ubuntu-latest

        steps:
            - name: Checkout code
              uses: actions/checkout@v4

            - name: Login to Docker Hub
              uses: docker/login-action@v3
              with:
                username: ${{ vars.DOCKERHUB_USER }}
                password: ${{ secrets.DOCKERHUB_TOKEN }}

            - name: Trivy Scanner
              uses: aquasecurity/trivy-action@0.35.0
              with:
                image-ref: ${{ vars.DOCKERHUB_USER }}/github-actions-app:${{ github.sha}}
                severity: 'CRITICAL,HIGH'
                exit-code: '1'
                trivyignore: ${{ github.workspace}}/.trivyignore


  devsecops.yml
  name: Devsecops End To End Pipeline

on:
    workflow_dispatch:

jobs:   
        #CI
    code-quality:
        uses: ./.github/workflows/code-quality.yml

    secrets-scan:
        uses: ./.github/workflows/secrets-scan.yml

    dependency-scan:
        uses: ./.github/workflows/dependency-scan.yml

    docker-scan:
        uses: ./.github/workflows/docker-lint.yml
        
        # CD
    build:
        needs: [ code-quality ,secrets-scan, dependency-scan, docker-scan]
        uses: ./.github/workflows/docker-build-push.yml
        secrets: inherit

    trivy:
        needs: [build]
        uses: ./.github/workflows/image-scan.yml
        secrets: inherit

    deploy:
        needs: [trivy]
        uses: ./.github/workflows/deploy-to-server.yml
        secrets: inherit

        


<img width="1890" height="905" alt="image" src="https://github.com/user-attachments/assets/be9ec8ca-943a-41a2-88b5-06233d9895f9" />


<img width="1918" height="911" alt="image" src="https://github.com/user-attachments/assets/55ae37e2-7f15-4fee-a2e7-0327a0877d09" />
