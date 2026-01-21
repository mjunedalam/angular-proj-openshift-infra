apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: frontend-build-pipeline
spec:
  params:
    - name: git-revision
      type: string
      description: git revision
    - name: repo-url
      type: string
      description: repo url
    - name: repo-name
      type: string
      description: repo name
    - name: branch-name
      type: string
      description: branch name
  tasks:
    - name: create-resources
      taskRef:
        kind: Task
        name: openshift-client
      params:
        - name: SCRIPT
          value: |
            cat <<-EOF | oc apply -f -
            kind: PersistentVolumeClaim
            apiVersion: v1
            metadata:
              name: $(params.repo-name)-npm-pvc
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 2Gi
              storageClassName: ocs-storagecluster-ceph-rbd
              volumeMode: Filesystem
            EOF
            cat <<-EOF | oc apply -f -
            kind: PersistentVolumeClaim
            apiVersion: v1
            metadata:
              name: $(params.repo-name)-source-pvc
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 5Gi
              storageClassName: ocs-storagecluster-ceph-rbd
              volumeMode: Filesystem
            EOF
            cat <<-EOF | oc apply -f -
            apiVersion: image.openshift.io/v1
            kind: ImageStream
            metadata:
              name: $(params.repo-name)
            spec:
              labels:
                app: $(params.repo-name)
                app.kubernetes.io/name: $(params.repo-name)
                app.openshift.io/runtime: quarkus
              lookupPolicy:
                local: false
            status:
              dockerImageRepository: >-
                image-registry.openshift-image-registry.svc:5000/easd-agwa-pipeline/$(params.repo-name)
              publicDockerImageRepository: >-
                default-route-openshift-image-registry.apps.ocpb.enp.aramco.com.sa/easd-agwa-pipeline/$(params.repo-name)
            EOF

    - name: git-clone
      runAfter:
        - create-resources
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.git-revision)
        - name: sslVerify
          value: 'false'
        - name: gitInitImage
          value: >-
            registry.redhat.io/openshift-pipelines/pipelines-git-init-rhel8:v1.14.5-1
      taskRef:
        kind: Task
        name: git-clone
      workspaces:
        - name: output
          workspace: source

    - name: set-parameters
      runAfter:
        - git-clone
      taskRef:
        kind: Task
        name: set-frontend-parameters-task
      params:
        - name: BRANCH_NAME
          value: $(params.branch-name)
      workspaces:
        - name: source
          workspace: source

    - name: install
      params:
        - name: IMAGE
          value: 'registry.access.redhat.com/ubi9/nodejs-20:latest'
        - name: ARGS
          value:
            - install
            - --force
      runAfter:
        - set-parameters
      taskRef:
        kind: Task
        name: npm-task
      workspaces:
        - name: source
          workspace: source
        - name: npm-cache
          workspace: npm-cache
        - name: npmrc
          workspace: npmrc

    - name: lint
      params:
        - name: IMAGE
          value: 'registry.access.redhat.com/ubi9/nodejs-20:latest'
        - name: ARGS
          value:
            - run
            - lint
      runAfter:
        - install
      taskRef:
        kind: Task
        name: npm-task
      workspaces:
        - name: source
          workspace: source
        - name: npm-cache
          workspace: npm-cache
        - name: npmrc
          workspace: npmrc

    - name: test
      params:
        - name: IMAGE
          value: 'registry.access.redhat.com/ubi9/nodejs-20:latest'
        - name: ARGS
          value:
            - test
      runAfter:
        - lint
      taskRef:
        kind: Task
        name: npm-task
      workspaces:
        - name: source
          workspace: source
        - name: npm-cache
          workspace: npm-cache
        - name: npmrc
          workspace: npmrc

    - name: build
      params:
        - name: IMAGE
          value: 'registry.access.redhat.com/ubi9/nodejs-20:latest'
        - name: ARGS
          value:
            - run
            - build
      runAfter:
        - test
      taskRef:
        kind: Task
        name: npm-task
      workspaces:
        - name: source
          workspace: source
        - name: npm-cache
          workspace: npm-cache
        - name: npmrc
          workspace: npmrc

    - name: buildah
      params:
        - name: IMAGE
          value: >-
            image-registry.openshift-image-registry.svc:5000/easd-agwa-pipeline/$(params.repo-name):$(tasks.set-parameters.results.image-tag)-SNAPSHOT
        - name: BUILDER_IMAGE
          value: registry.redhat.io/rhel8/buildah:8.10
        - name: DOCKERFILE
          value: ./container/Containerfile
        - name: TLSVERIFY
          value: 'false'
      runAfter:
        - build
      taskRef:
        kind: Task
        name: buildah
      workspaces:
        - name: source
          workspace: source
    
    - name: scan
      params:
        - name: fortify-project-name
          value: $(tasks.set-parameters.results.fortify-project-name)
        - name: frontend-project-name
          value: $(tasks.set-parameters.results.frontend-project-name)
        - name: repo-name
          value: $(params.repo-name)
        - name: nexus-iq-project-name
          value: $(tasks.set-parameters.results.nexus-iq-project-name)
      runAfter:
        - git-cli
      taskRef:
        kind: Task
        name: frontend-security-scan-task
      workspaces:
        - name: source
          workspace: source
    
    - name: git-cli
      runAfter:
        - buildah
      taskRef:
        kind: Task
        name: git-cli
      params:
        - name: url
          value: $(params.repo-url)-resources
        - name: revision
          value: master
        - name: git-script
          value: |
            cd $(params.repo-name)-resources/base
            sed -i "/^\([[:space:]]*digest: \).*/s//\1 \"$(tasks.buildah.results.IMAGE_DIGEST)\"/" kustomization.yaml
            echo $(cat kustomization.yaml)
        - name: subdirectory
          value: resources
        - name: deleteExisting
          value: 'true'
        - name: verbose
          value: 'true'
      workspaces:
        - name: output
          workspace: source

  workspaces:
    - name: source
    - name: npm-cache
    - name: npmrc
