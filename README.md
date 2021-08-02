 # Description
 This document introduces a solution for connecting to the Dind server from the Dind pod running as a Jenkins agent which is defined in the Jenkinsfile.
 # Components
 * ##  Dind server.
    For our case Dind server must listen on port 2375. Here are K8S Dind pod, pv, pvc and service examples defined in ``yaml``.
 ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: dind-test
    namespace: test
    labels:  
      app: dind-test
  spec:  
    containers:
    - name: dind-test
      image: docker:18.05-dind

      volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
      securityContext:
        privileged: true
    volumes:
  - name: dind-storage
    persistentVolumeClaim:
      claimName: myclaim
---
 apiVersion: v1
 kind: PersistentVolume
 metadata:
  name: pv
  namespace: test
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /tmp/volume
--- 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: test
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  volumeName: pv 
---
apiVersion: v1
kind: Service
metadata:
  name: dind-svc
  namespace: test
spec:
  selector:
    app: dind-test
  ports:
    - protocol: TCP
      port: 2375
      targetPort: 2375
 ``` 
* ## Jenkinsfile
    Here is the Jenkinsfile example with Dind pod defined as an agent and executig a ``docker build`` command in the Dind server.

    **Note**: ``defaultContainer`` is set to `dind`, so if you want to run a command inside a ``jnlp`` container you must specify it.
    ```Jenskinsfile 
        pipeline {
            agent {
                kubernetes {
                    defaultContainer 'dind'
                    yaml """\
                      apiVersion: v1
                      kind: Pod
                      metadata:
                        labels:
                          jenkins/kube-default: "true"
                          app: jenkins
                          component: agent
                        namespace: jenkins
                      spec:
                        containers:
                          - name: jnlp
                            image: jenkinsci/jnlp-slave
                            imagePullPolicy: Always
                            env:
                            - name: POD_IP
                              valueFrom:
                                fieldRef:
                                  fieldPath: status.podIP
                          - name: dind
                            image: docker:18.05-dind
                            securityContext:
                            env:
                            - name: DOCKER_HOST
                              value: tcp://dind-svc.karen-test.svc.cluster.                    local:2375              
                              privileged: true
                            command:
                            - sh 
                            - -c
                            - tail -f /dev/null
        """.stripIndent()                    
                            }
                    }
    stages {
        stage('Testing some command in jnlp container') {
          steps { 
              container('jnlp') {
                  sh 'git --version'
                }                               
            }                  
        }
        stage("Tesing Dind pod") {
          steps {
              script {
                  docker.build(" test_image", "-f Dockerfile .")
                    }
                }
            }
        }
    }