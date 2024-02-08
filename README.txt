KARGO		https://kargo.akuity.io/quickstart

CCI		as of 1/31:
		-----------
		Client Version: v0.3.1
		Server Version: v0.3.2

		points of interest: 
		-------------------
		main branch 		- kargo/base/deploy.yaml
		stage/prod branch	- kargo/base/deploy.yaml
					- kargo/stages/prod/kustomization.yaml
		stage/uat branch	- kargo/base/deploy.yaml
					- kargo/stages/uat/kustomization.yaml
		stage/test branch	- kargo/base/deploy.yaml
					- kargo/stages/test/kustomization.yaml

IMAGE		cci-repo.cci-dev.com:5000/gin-qa-solutions	#NOT https
		docker.io/ccipearl/gin-qa-solutions
                WHERE latest Dockerfile is, as follows:		#the command "tail -f /dev/null" will keep the container alive
		--------------------------------
		#
		# Base the image on the latest version of Robot
		FROM robotframework/rfdocker:latest
		#
		# Identify yourself as the image maintainer (where EMAIL is your email address)
		LABEL maintainer="pearl@customercaresolutions.com"
		#
		# Install Robot-Json
		RUN pip install robotframework-jsonlibrary
		#
		# Install Robot-Requests
		RUN pip install robotframework-requests
		#
		# Command to execute
		ENTRYPOINT ["sh", "-c", "robot -d /usr/share/gin-qa-solutions/html/kargo/Results /usr/share/gin-qa-solutions/html/kargo/Tests/Main.robot; tail -f /dev/null"]

SET-UP

1) Make sure gin server (referenced in robot tests) is running
2) Configure 172.31.27.186:5000 in /etc/rancher/k3s/registries.yaml (if using cci-repo)
3) Follow Kargo QuickStart instructions, up to creating credentials for gitops repo
4) Create credentials for image repo, as follows:

$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cci-image-repo
  namespace: kargo-demo
  labels:
    kargo.akuity.io/secret-type: repository
stringData:
  type: image
  url: docker.io/ccipearl/gin-qa-solutions
  username: 'ccipearl'
  password: '<PASSWORD>'
EOF

5) Create credentials for test repo, as follows:

$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cci-test-repo
  namespace: kargo-demo
  labels:
    kargo.akuity.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/pc085n/gin-test
  username: 'pc085n'
  password: '<PASSWORD>'
EOF

6) Define pv (persistent volume), as follows:

$ kubectl apply -f pv.yaml

where pv.yaml:
--------------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kargo-pv
  labels:
    type: local
spec:
  storageClassName: local-path
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 2Gi
  hostPath:
    path: "/home/ubuntu/shared"

7) Define CCI-specific warehouse and stage resources (e.g., docker.io), as follows:

$ cat <<EOF | kargo apply -f -
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: kargo-demo
  namespace: kargo-demo
spec:
  subscriptions:
  - image:
      repoURL: docker.io/ccipearl/gin-qa-solutions
      semverConstraint: ^1.0.0
  - git:
      repoURL: https://github.com/pc085n/gin-test
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: test
  namespace: kargo-demo
spec:
  subscriptions:
    warehouse: kargo-demo
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/test
      kustomize:
        images:
        - image: docker.io/ccipearl/gin-qa-solutions
          path: stages/test
    argoCDAppUpdates:
    - appName: kargo-demo-test
      appNamespace: argocd
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: uat
  namespace: kargo-demo
spec:
  subscriptions:
    upstreamStages:
    - name: test
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/uat
      kustomize:
        images:
        - image: docker.io/ccipearl/gin-qa-solutions
          path: stages/uat
    argoCDAppUpdates:
    - appName: kargo-demo-uat
      appNamespace: argocd
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: prod
  namespace: kargo-demo
spec:
  subscriptions:
    upstreamStages:
    - name: uat
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/prod
      kustomize:
        images:
        - image: docker.io/ccipearl/gin-qa-solutions
          path: stages/prod
    argoCDAppUpdates:
    - appName: kargo-demo-prod
      appNamespace: argocd
EOF

8) Sync up all stages, manually via argocd gui
9) Define pvc for each stage, as follows: 

$ kubectl apply -f pvc-test.yaml -n kargo-demo-test

where pvc-test.yaml:
--------------------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kargo-pvc
  namespace: kargo-demo-test
spec:
  storageClassName: local-path
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

$ kubectl apply -f pvc-uat.yaml -n kargo-demo-uat

where pvc-uat.yaml:
-------------------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kargo-pvc
  namespace: kargo-demo-uat
spec:
  storageClassName: local-path
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

$ kubectl apply -f pvc-prod.yaml -n kargo-demo-prod

where pvc-prod.yaml:
--------------------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kargo-pvc
  namespace: kargo-demo-prod
spec:
  storageClassName: local-path
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

10) Define secret for each stage, as follows:

$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo-test
  labels:
    kargo.akuity.io/secret-type: repository
stringData:
  type: git
  project: default
  url: ${GITOPS_REPO_URL}
  username: ${GITHUB_USERNAME}
  password: ${GITHUB_PAT}
EOF

$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo-uat
  labels:
    kargo.akuity.io/secret-type: repository
stringData:
  type: git
  project: default
  url: ${GITOPS_REPO_URL}
  username: ${GITHUB_USERNAME}
  password: ${GITHUB_PAT}
EOF

$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo-prod
  labels:
    kargo.akuity.io/secret-type: repository
stringData:
  type: git
  project: default
  url: ${GITOPS_REPO_URL}
  username: ${GITHUB_USERNAME}
  password: ${GITHUB_PAT}
EOF

11) Look up kargo and argocd gui
