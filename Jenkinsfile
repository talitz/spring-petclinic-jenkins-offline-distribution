pipeline {
    agent any

    stages {
        stage ('Clone') {
            steps {
                git branch: 'master', url: "https://github.com/talitz/spring-petclinic-ci-cd-k8s-example.git"
            }
        }

        stage ('Artifactory Configuration') {
            steps {
                rtMavenResolver (
                    id: 'maven-resolver',
                    serverId: 'talyi-artifactory',
                    releaseRepo: 'libs-release',
                    snapshotRepo: 'libs-release'
                )  
                 
                rtMavenDeployer (
                    id: 'maven-deployer',
                    serverId: 'talyi-artifactory',
                    releaseRepo: 'libs-release-local',
                    snapshotRepo: 'libs-snapshot-local',
                    threads: 6,
                    properties: ['BinaryPurpose=Technical-BlogPost', 'Team=DevOps-Acceleration']
                )
            }
        }
        
        stage('Build Maven Project') {
            steps {
                rtMavenRun (
                    tool: 'Maven 3.3.9',
                    pom: 'pom.xml',
                    goals: '-U clean install',
                    deployerId: "maven-deployer",
                    resolverId: "maven-resolver"
                )
            }
        }

        stage ('Publish build info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "talyi-artifactory"
                )
            }
        }

        stage ('Xray Maven Scan') {
            steps {
                xrayScan (
                    serverId: "talyi-artifactory",
                    failBuild: true
                )
            }
        } 

        stage ('Promotion') {
            steps {
                rtPromote (
                    serverId: "talyi-artifactory",
                    targetRepo: 'libs-release-local',
                    comment: 'Passed Xray QualityGate',
                    sourceRepo: 'libs-snapshot-local',
                    status: 'Released',
                    includeDependencies: true,
                    failFast: true,
                    copy: false
                )
            }
        }

        stage ('Build Docker Image') {
            steps {
                script {
                    docker.build("talyi-docker.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}")
                }
            }
        }

        stage ('Push Image to Artifactory') {
            steps {
                rtDockerPush(
                    serverId: "talyi-artifactory",
                    image: "talyi-docker.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}",
                    targetRepo: 'docker',
                    properties: 'project-name=jfrog-blog-post;status=stable'
                )
            }
        }

        stage ('Publish Build Info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "talyi-artifactory"
                )
            }
        }

        stage ('Xray Docker Image Scan') {
            steps {
                xrayScan (
                    serverId: "talyi-artifactory",
                    failBuild: true
                )
            }
        }         
        
        stage ('Install JFrog CLI') {
            steps {
                 bash '''
                    curl -fL https://getcli.jfrog.io | sh
                 '''
            }
        }         

        stage ('Create & Sign Release Bundle') {
            steps {
                 bash '''
                    jfrog rt rbc --spec=RB-spec.json --sign EU-LISA-RB 1.0.${env.BUILD_NUMBER}
                 '''
            }
        }

        stage ('Export Release Bundle') {
            steps {
                 bash '''
                    curl -XPOST 'http://localhost:8082/distribution/api/v1/export/release_bundle/EU-LISA-RB/1.0.${env.BUILD_NUMBER}' -uadmin:password
                 '''
            }
        }
    }
}
