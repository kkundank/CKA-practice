Security Primitives:
!
!
- Root based disabled
- Password based authentication disabled
- kube-apiserver first line of defence, controlling access to api-server itself

Authentication:
who can access the api-server? 
- Username and password stored in file
- username and tokens 
- cert
- LDAP
- Service accounts

Authorisation:
What can they do
- RBAC Authorisation
- ABAC Authorisation
- Node Authorisation 
- Webhook Mode

- All components talking to Kube-Api Server is secured via tls encryption 
- communicaton between application, pod within the cluster, can access all other pods within the cluster
  can restrict using network policies 
  
  Authentication:
  
  Access to K8s cluster:
  Admin - Users
  Developers - Users
  End users
  Bots - Service accounts since this is 3rd party
  
  - k8s does not manage users account nativlely 
  - cannot create users on clusters
  
  - Can create service accounts however for bots etc
  
  user access:
  
  - All user access is managed by the kubernetes api-server 
  - Api servers authenticate users before processing
  - kube-apiserver authenticates 
    - static password file = create user/passwords in csv file, can assign groups also 
    - static token file    = 
    - certificates
    - identity services
  
 Article on Setting up Basic Authentication
Setup basic authentication on Kubernetes (Deprecated in 1.19)
Note: This is not recommended in a production environment. This is only for learning purposes. Also note that this approach is deprecated in Kubernetes version 1.19 and is no longer available in later releases

Follow the below instructions to configure basic authentication in a kubeadm setup.

Create a file with user details locally at /tmp/users/user-details.csv

# User File Contents
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
password123,user4,u0004
password123,user5,u0005


Edit the kube-apiserver static pod configured by kubeadm to pass in the user details. The file is located at /etc/kubernetes/manifests/kube-apiserver.yaml



apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
      <content-hidden>
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
    volumeMounts:
    - mountPath: /tmp/users
      name: usr-details
      readOnly: true
  volumes:
  - hostPath:
      path: /tmp/users
      type: DirectoryOrCreate
    name: usr-details


Modify the kube-apiserver startup options to include the basic-auth file



apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
      <content-hidden>
    - --basic-auth-file=/tmp/users/user-details.csv
Create the necessary roles and role bindings for these users:



---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
 
---
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
Once created, you may authenticate into the kube-api server using the users credentials

curl -v -k https://localhost:6443/api/v1/pods -u "user1:password123"
  
 TLS Introduction:
 TLS Certificates PRE-REQ:
 
 - User access web server, comminication between user and server is encrypted this is TLS
 - Encrypt data using encrption keys, data is sent to server from client, cause this is encrypted
   it cannot be deencrypted by attacker however the key can since it is symmatric key and not assemtric
   since the client sends the key to the server. Same key to encrypt and decrypt 
   
 - Assmmetric, different keys to encrypt and dencrypt
 - Public lock can only be unlocked with private key 
 - When the user first accessing the https webserver
   - he gets the public key from the server
   - the users browser than encrypts the symmetric key provided by the servers public key
   - Private key is never sent over transit
   - Private key is used to dencrypt 
   
   - Ensure integrity by using Certificates
   - Certificate contains, who the cert is issued too, location 
     - Name on it / Subject 
     - if you assign a certificate, it is called a self signed certificate
     - Cert authority can assign certs
       - Generate CSR 
         - Using open ssl for example
     - Validate information provided 
     - sign and send certificate
     - if hacker tried to validate his certificate, it would fail and be rejected
     
     - How do you know, cert signed by Symantec etc, 
       - They have their own public and private keys pairs
         - private keys to sign certificates
         - public keys hosted in browsers built in, using public key of CA if actually
           signed by CA
           
    - For Internal sites, private links, Private CAs can be hosted on prem, to validate
      private on-prem connections. 
      
    1. To encrypt messages use assymmetric keys
    2. Admin uses pair of keys to connect to servers via ssh
    3. Server uses pair of keys to secure https traffic
    4. server first sends CSR to CA, 
    5. The CA uses its private keys to sign CSR
    6. The signing Certificate is than sent back to the server, 
    7. server than configures web application with new certitificate 
    8. To communicate with client, server first sends its certificate with its public key 
    9. Users browser reads the certificate and uses the CAs public key to validate and retrieve
       the servers public key
   10. Than generates a symmetric key that it wishes to use going forward for communication 
   11. The symmetric key is encrypted using the servers public key and sent back to the server 
        
    12. The server uses its private key to decrypt the message and return the symmetric key
        the symmetric key is used for communication 
    
    How does the server trust the client:
    
    1. The server can request the clients certificate
    2. Client generates pair of keys and signed certificate from a verified CA
    3. Client than sends the client certificate to the server for it to verify the client is 
       who they are
  
    
   Security Kubernetes with TLS:
    
   - Previously we saw a CA, have their own public and private key-pairs to sign server certificates
     this is called a 'root certificate' 
     
   - A server can request a client certificate using client certificates
   - 3 types of certificates Server certificate, Client Certificate, Root certificates 
   - *crt or * pem = public key
   - *key or *key.pem = private key
   
   - K8s cluster containes set of master and worker nodes, communication between all needs to be secure and -
     encrypted 
     
   - Client must secure tls connection to components.
   
   - Kube-API-Server exposes https service, it is a server and request certificate to secure the service.
     generate key pair, called apiserver.crt and apiserver.key
     
   - etcd requires pairs of certificate and key also etcdserver.crt
   - kubelet also requires cert and key pair kubelet.crt
   
   - client or admin requires cert and key pair to authenticate to kube-apiserver
   - kube scheduler talks to the kube-apiserver to look for pods which requires scheduling and get the 
     kube api server to schedule the pods to nodes. 
   - The scheduler is a client act as the kube-apiserver from the kube-apiserver perspective the kube scheduler
     is just another client
     
   - The kube-scheduler needs to valides its identiy using client certificate and key scheduler.crt and key 
   
   - The kube controler manager also requires a client cert and key to validate identity 
   
   - Kube-proxy also requires a cert and key to authenticate to kube-api server 
   
   - Kube-APIServer communicates with the ETCD server is the only one which speaks to the ETCD Server 
     in this case the kube-api-server is the client can either use the existing cert and key or specifically 
     new ones just for communication to the ETCD. 
     
    - The kube-apiserver also communicates with the kubelet thats how it monitors each worker nodes.
      can use orignal cert key or new ones 
   
   - Generate certificate by using a CA, at least 1 for your k8s cluster and 1 specifically for ETCD, signed by 
     ETCD CA. 
     
   - Usually require 2 certificates, 1 for client certificates and 1 for server side certificates
     - All componenets speaking to API-Server = Client cert and Key
     - API-Server Speaking to ETCD = Client Cert and key 
     - API-SERVER & ETCD Acting as Server 
     
   Certification creation:
   
   Many ways in generating certificates for kubernetes cluster
   - Tools such as 
     - EASYRSA
     - OPENSSL
     - CFSSL
   
    - Main focus is OPENSSL
    1. Starting with CA Certificates
    2. Create private key using command openssl genrsa -out ca key 2048
    3. CSR Request = openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
    4. Sign cert by this command, openssl x509 -req -in ca.csr -signkey ca
    5. The CA now has its private key and root certificate
    6. now creating a client certificate using openssl genrsa -out admin.key 2048
    7. CSR Request = openssl req -new -key admin.key -subj "/CN=kube-admin/o=system:masters" -out admin.csr
    8. Sign cert by this command, openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
       specifiy the CA certificate and CA key makes it valid certificate and outputting admin.crt file
    
    Kube-Scheduler / kube-controller must have name system in the certificate 
    
    To use the above certificates such as Admin certificates
    1. Can call an API Call, specify key 
    2. Or put it in a Kubeconfig, all of the config 
    
    - For clients to validate certificates sent by the server and vise verse they all need copy CA Public 
      certificate usually installed in web browser to communicate you must specify in all components root -
      certificate 
      
      Server side certificate:
      
      - if you have multiple ETCD clusters, must specify peering certificates 
      - Kube-APIServer
        1. Create private key using command openssl genrsa -out apiserver.key 2048
        2. CSR Request = openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr
        3. Sign cert by this command, openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out 
        apiserver.crt
        
   Kubelet server:
   
   1. Require key cert pair for each node in the cluster
   2. name each node after the node of the names
   3. once cert created, use in kubelet config file 
   4. must be done by each node in the cluster 
   5. Client certificates to communicate with the kube-api server 
    
     
   TLS Certificates: Viewing certs
   
   1. Hardway = generate cert to yourself
   2. Kubeadm = easier way 
   3. Cluster deployed by Kubeadm 
   - cat /etc/kubernetes/manifests/kube-apiserver.yaml
     
   Practice test: View certificate details
   
   1. Identify the certificate file used for the kube-api server = 
      cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep tls-cert
       - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
   2. Identify the Certificate file used to authenticate kube-apiserver as a client to ETCD Server
      cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-cert
       - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
   3. Identify the ETCD Server Certificate used to host ETCD server = 
   
   4. Identify the ETCD Server Certificate used to host ETCD server
      root@controlplane /etc/kubernetes/pki ➜  cat /etc/kubernetes/manifests/etcd.yaml | grep cert-file    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
   
   5. Identify the ETCD Server CA Root Certificate used to serve ETCD Server
      ETCD can have its own CA. So this may be a different CA certificate than the one used by kube-api server.
      Look for CA Certificate (trusted-ca-file) in file /etc/kubernetes/manifests/etcd.yaml
      
   6. What is the Common Name (CN) configured on the Kube API Server Certificate?
      OpenSSL Syntax: openssl x509 -in file-path.crt -text -noout 
      = Run the command openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text and look for Subject CN.
      
   7. What is the name of the CA who issued the Kube API Server Certificate?
      = kubernetes
      
   8. Which of the below alternate names is not configured on the Kube API Server Certificate?
      = Run the command openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text and look at Alternative Names
      
   9. What is the Common Name (CN) configured on the ETCD Server certificate?
      = Run the command openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text and look for Subject CN.
      
   10. How long, from the issued date, is the Kube-API Server Certificate valid for?
       File: /etc/kubernetes/pki/apiserver.crt
       = Run the command openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text and check on the Expiry date.
       
   11. How long, from the issued date, is the Root CA Certificate valid for?
       File: /etc/kubernetes/pki/ca.crt
       = openssl x509 -in /etc/kubernetes/pki/ca.crt -text
       
   12. Kubectl suddenly stops responding to your commands. Check it out! Someone recently modified the /etc/kubernetes/manifests/etcd.yaml file
       You are asked to investigate and fix the issue. Once you fix the issue wait for sometime for kubectl to respond. Check the logs of the ETCD container.
       The certificate file used here is incorrect. It is set to /etc/kubernetes/pki/etcd/server-certificate.crt which does not exist. As we saw in the previous questions the correct path should be /etc/kubernetes/pki/etcd/server.crt.

       root@controlplane:~# ls -l /etc/kubernetes/pki/etcd/server* | grep .crt
       -rw-r--r-- 1 root root 1188 May 20 00:41 /etc/kubernetes/pki/etcd/server.crt
       root@controlplane:~# 
       Update the YAML file with the correct certificate path and wait for the ETCD pod to be recreated. wait for the kube-apiserver to get to a Ready state.

       NOTE: It may take a few minutes for the kubectl commands to work again so please be patient.
       
   13. The kube-api server stopped again! Check it out. Inspect the kube-api server logs and identify the root cause and fix the issue.
       Run docker ps -a command to identify the kube-api server container. Run docker logs container-id command to view the logs.
      
       If we inspect the kube-apiserver container on the controlplane, we can see that it is frequently exiting.

      root@controlplane:~# docker ps -a | grep kube-apiserver
      8af74bd23540        ca9843d3b545           "kube-apiserver --ad…"   39 seconds ago      Exited (1) 17 seconds ago                          k8s_kube-apiserver_kube-apiserver-controlplane_kube-system_f320fbaff7813586592d245912262076_4
      c9dc4df82f9d        k8s.gcr.io/pause:3.2   "/pause"                 3 minutes ago       Up 3 minutes                                       k8s_POD_kube-apiserve-controlplane_kube-system_f320fbaff7813586592d245912262076_1
      root@controlplane:~# 
      If we now inspect the logs of this exited container, we would see the following errors:

      root@controlplane:~# docker logs 8af74bd23540  --tail=2
      W0520 01:57:23.333002       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...
      Error: context deadline exceeded
      root@controlplane:~# 
      This indicates an issue with the ETCD CA certificate used by the kube-apiserver. Correct it to use the file /etc/kubernetes/pki/etcd/ca.crt.

      Once the YAML file has been saved, wait for the kube-apiserver pod to be Ready. This can take a couple of minutes.
   
     
 Certificate API:
 
 - A new admin comes in to and wants access to the cluster, requires cert key pair to access the -
   cluster, 1. creates own private key, 2. generates CSR request and sends to admin, 3. take cert to ca server signed by ca private key and root cert
   and sends back to user. 
   
 - Access does expire.
 
 - CA, is just a pair key and cert files generated, whoever has acces to these files can create
   as many users as possible.
  
 - Files need to be protected, placed in a server this is called a CA.
 - kubeadm does the same thing, it creates CA pair of files and stores in master node itself 
 
 - So far we have been creating certificates manually, however when users grow, you want to automate this process
   kubernetes has its own certificate API.
   
 - Now you send certitificate request to kubernetes api through an api call, this time when the admin recieves the request
   instead of logging into the CA Manually via the master node now creates a kubernetes API object called -
   certificates signing request, once object is created this can be seen by all admin users of th cluster and reviewed 
   and approved, can than be extracted and shared. 
   1. User create key and gene CSR request with Key
   2. Send request to admin 
   3. admin takes key and creates certficate signing request
   4. created using manifest file
   5. request field is where you the csr sent by users encoded using base64 command and put this in the request field
   6. submit request
   7. now can be seen by admin using kubectl get csr command 
   8. can approve cert by kubeclt certificate approve command 
   9. generates new cert for user 
   10. kubectl grt csr jane -o yaml
   11. Controller manager looks after all the CSR request etc 
 
  
  Kubernetes test certificates API quiz:
  
  1. A new member akshay joined our team. He requires access to our cluster. The Certificate Signing Request is at the /root location.
     
  2. Create a CertificateSigningRequest object with the name akshay with the contents of the akshay.csr file
     As of kubernetes 1.19, the API to use for CSR is certificates.k8s.io/v1.
     Please note that an additional field called signerName should also be added when creating CSR. For client authentication to the API server we will use the built-in signer kubernetes.io/kube-apiserver-client.
     = 
     Use this command to generate the base64 encoded format as following: -

     cat akshay.csr | base64 -w 0
     and finally, create a CSR name akshay as follows: -

     ---
     apiVersion: certificates.k8s.io/v1
     kind: CertificateSigningRequest
     metadata:
     name: akshay
     spec:
     groups:
     - system:authenticated
     request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZV3R6YUdGNU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQXY4azZTTE9HVzcrV3JwUUhITnI2TGFROTJhVmQ1blNLajR6UEhsNUlJYVdlCmJ4RU9JYkNmRkhKKzlIOE1RaS9hbCswcEkwR2xpYnlmTXozL2lGSWF3eGVXNFA3bDJjK1g0L0lqOXZQVC9jU3UKMDAya2ZvV0xUUkpQbWtKaVVuQTRpSGxZNDdmYkpQZDhIRGFuWHM3bnFoenVvTnZLbWhwL2twZUVvaHd5MFRVMAo5bzdvcjJWb1hWZTVyUnNoMms4dzV2TlVPL3BBdEk4VkRydUhCYzRxaHM3MDI1ZTZTUXFDeHUyOHNhTDh1blJQCkR6V2ZsNVpLcTVpdlJNeFQrcUo0UGpBL2pHV2d6QVliL1hDQXRrRVJyNlMwak9XaEw1Q0ErVU1BQmd5a1c5emQKTmlXbnJZUEdqVWh1WjZBeWJ1VzMxMjRqdlFvbndRRUprNEdoayt2SU53SURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBQi94dDZ2d2EweWZHZFpKZ1k2ZDRUZEFtN2ZiTHRqUE15OHByTi9WZEdxN25oVDNUUE5zCjEwRFFaVGN6T21hTjVTZmpTaVAvaDRZQzQ0QjhFMll5Szg4Z2lDaUVEWDNlaDFYZnB3bnlJMVBDVE1mYys3cWUKMkJZTGJWSitRY040MDU4YituK24wMy9oVkN4L1VRRFhvc2w4Z2hOaHhGck9zRUtuVExiWHRsK29jQ0RtN3I3UwpUYTFkbWtFWCtWUnFJYXFGSDd1dDJveHgxcHdCdnJEeGUvV2cybXNqdHJZUXJ3eDJmQnErQ2Z1dm1sVS9rME4rCml3MEFjbVJsMy9veTdqR3ptMXdqdTJvNG4zSDNKQ25SbE41SnIyQkZTcFVQU3dCL1lUZ1ZobHVMNmwwRERxS3MKNTdYcEYxcjZWdmJmbTRldkhDNnJCSnNiZmI2ZU1KejZPMUU9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
     signerName: kubernetes.io/kube-apiserver-client
     usages:
     - client auth
     
     
     
  3. What is the Condition of the newly created Certificate Signing Request object?
     kubectl get csr
     pending
  
  4. Approve the CSR Request
     kubectl certificate approve akshay 
  
  5. How many CSR requests are available on the cluster?
     Including approved and pending = 2
  
  6. During a routine check you realized that there is a new CSR request in place. What is the name of this request?
     = agent smith
     
  7. Hmmm.. You are not aware of a request coming in. What groups is this CSR requesting access to?
     Check the details about the request. Preferebly in YAML.
     Run the command kubectl get csr agent-smith -o yaml
     system::masters
  
  8. That doesn't look very right. Reject that request.
     = kubectl certificate deny agent-smith
  
  9. Let's get rid of it. Delete the new CSR object
     kubectl delete csr agent-smith
  
 Kube-Config:
 
 - moving certificate config file to KubeConfig file
   kubectl get pods 
      --kubeconfig config
   
 - kubeconfig file is in a specific format
 - 3 sections 
   - clusters: various k8s clusters need access too, 
     development, production, google. 
   - contexts: users which have access to account, which user account to be used for which cluster
   - users: existing users for existing privelieges to access what cluster they are accessing
   
   - Servers specifis go into cluster section
   - admin user keys / certificate go into users sections.
   - than create context use mykube admin users to access mykube cluster
   
   Kube-Config Quiz:
   
   1. Where is the default kubeconfig file located in the current environment?
      Find the current home directory by looking at the HOME environment variable.
      = /root/.kube  
   2. How many clusters are defined in the default kubeconfig file? kubectl config view 1 = kubernetes
   3. How many Users are defined in the default kubeconfig file? = 1 kubernetes-admin
   4. How many context are defined in the default kubeconfig file? = 1 kubernetes-admin@kubernetes
   5. What is the user configured in the current context? = kubernetes-admin
   6. What is the name of the cluster configured in the default kubeconfig file?
      = kubernetes
   7. A new kubeconfig file named my-kube-config is created. It is placed in the /root directory. How many clusters are defined in that kubeconfig file?
      = 4
   8. How many contexts are configured in the my-kube-config file? 4
   9. What user is configured in the research context? dev-user
   10. What is the name of the client-certificate file configured for the aws-user?
       client-certificate: /etc/kubernetes/pki/users/aws-user/aws-user.crt
   11. What is the current context set to in the my-kube-config file? kubectl config current-context --kubeconfig my-kube-config
       = test-user@development
   12. I would like to use the dev-user to access test-cluster-1. Set the current context to the right one so I can do that.
       Once the right context is identified, use the kubectl config use-context command.
       = To use that context, run the command: kubectl config --kubeconfig=/root/my-kube-config use-context research
        To know the current context, run the command: kubectl config --kubeconfig=/root/my-kube-config current-context
   13. We don't want to have to specify the kubeconfig file option on each command. Make the my-kube-config file the default kubeconfig.
       Replace the contents in the default kubeconfig file with the content from my-kube-config file.
   14. With the current-context set to research, we are trying to access the cluster. However something seems to be wrong. Identify and fix the issue.
       Try running the kubectl get pods command and look for the error. All users certificates are stored at /etc/kubernetes/pki/users.
       = The path to certificate is incorrect in the kubeconfig file. Correct the certificate name which is available at /etc/kubernetes/pki/users/.
       cert should be dev-user not developer-user. 
   
API Groups:

- To access API: Can do this: 
  Curl https://kube-master:6443/version = To see version
  Curl https://kube-master:6443/api/v1/pods = List of pods
  
- APIs are grouped into several groups:
  /metrics
  /healthz
  /version
  
  /api:
  v1/namespaces/pods/rc/events/endpoints/nodes/bindings/PV/PVC/configmaps/secrets
  
  /services:
  
  /apis:
  Apps/ = deploymnents / replicasets / statefulsets
  extensions/ = 
  networking.k8s.io = network policies
  /storage.k8s.io = 
  /authentication.k8s.io =
  /certifices.k8s.io = certificatessigningrequests
  
  /logs
  
  - not allowed access to kube-api as you have not specified any authentication methods
    have to authenticate using the following:
    curl http://localhost:6443 -k 
         --key admin.key
         --cert admin.crt
         --cacert ca.crt
         
  - Alternative option is to use or start a 'kubeproxy' client which uses ports 8001 locally. 
    uses cresendials and certificates from kube-config files to access the cluster.
    command = kubectl proxy 
    curl http://localhost:8001 -k
    
 - kube proxy = enables connectivity between pods and services across different nodes in cluster
 - kubectl proxy = http proxy services created by kubectl to access kube-api server. 
 
 Authorization:
 
 - Depending who is accessing what, needs to restrict users i.e with namespaces etc
 - Authorization Mechanisms:
   - Node: Kube-API is accessed by users for mgmt purposes aswell as kubelet.
     The Kueblet acts as the api server to read information about services/endpoints/nodes/pods
     also reports information to kube-api about node status/pod status/events.
     request handled by special node authorizer 
   - ABAC: Groups of users with set of permissions, for example 'Dev-User' can create pod / view pods and delete pods
   - RBAC: define a role, for developers for example with set of permissions required for developers
     then associate all developers to that role 
   - WEBHOOK: If you want to manage an authorisation externally and built in mechanisms. Example 
     Open Policy Agent, users can make a call to the Kube Api with info about users and access requirements
     and have the Open Policy agent decide to permit or not. 
   - AlwaysAllow - Allows all request without performing any auth checks
   - AlwaysDeny - denies the request
   
  - Mode are set using the auth mode option on the kube-api server, if you dont change by default,
    it is always 'AllowsAllow' 
    
 - Can change to multiple modes for example Node/RBAC/WEBHOOK 
   goes in order i.e Node.
   Everytime a module denies a request, it simply goes to the next one in the chain order 
   as soon as request is accepted, that is it. 
     
RBAC:

How do we create a role?
- Create a role by created an RBAC object 
- Create a role definition file, with kind set to 'Role;

Developer role:
Can view PODS
Can create PODS
Can Delete PODS
Can Create ConfigMaps

1. 

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups [""]
  resources: ["pods"]
  verbs: ["list", "get', "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
  
kubectl create -f xxxxx-role.yaml

2. Next step is do perform 'role binding' links user objects to a role

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:  <<<<<-------------------------- Ths is where you tie user role above to this role binding.
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
  
3. View role
- kubectl get roles
- kubectl get rolebindings 
- kubectl describe role developer 
- kubectl describe role binding xxxx

4. Check Access
- kubectl auth can-i create deployments
- kubectl auth can-i delete nodes 
- kubectl auth can-i create pods --as dev-user
- kubectl auth can-i create deployments --as dev-user

Reasource name access: 

- if you want to grant user access to only certain pods in namespace for example:

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups [""]
  resources: ["pods"]
  verbs: ["list", "get', "create", "update", "delete"]
  resourceNames: ["blue", "orange"]

Practice test: Test roles 

1. Inspect the environment and identify the authorization modes configured on the cluster.
   Check the kube-apiserver settings. = kubectl describe pods -n kube-system kube-apiserver-controlplane
   = Node / RBAC
2. How many roles exist in the default namespace? = kubectl get roles
   No resources found in default namespace. = 0 
3. How many roles exist in all namespaces together? = 12 
4. What are the resources the kube-proxy role in the kube-system namespace is given access to?
   = kubectl describe role kube-proxy -n kube-system = configmaps
5. What actions can the kube-proxy role perform on configmaps? = Get
6. Which of the following statements are true? = kube-Proxy role can get details of configmap 
   object by the name kube-proxy only
7. Which account is the kube-proxy role assigned to? = 
   kubectl describe rolebinding  kube-proxy -n kube-system
   = system:bootstrappers:kubeadm:default-node-token
8. A user dev-user is created. User's details have been added to the kubeconfig file. Inspect the permissions granted to the user. Check if the user can list pods in the default namespace.
   Use the --as dev-user option with kubectl to run commands as the dev-user.
   = kubectl auth can-i get pods --as dev-user = no 
9. Create the necessary roles and role bindings required for the dev-user to create, list and delete pods in the default namespace.
   Use the given spec:
=

To create a Role:- kubectl create role developer --namespace=default --verb=list,create,delete --resource=pods
To create a RoleBinding:- kubectl create rolebinding dev-user-binding --namespace=default --role=developer --user=dev-user

OR

Solution manifest file to create a role and rolebinding in the default namespace:

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "create","delete"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-user-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
     
10. A set of new roles and role-bindings are created in the blue namespace for the dev-user. However, the dev-user is unable to get details of the dark-blue-app pod in the blue namespace. Investigate and fix the issue.
    We have created the required roles and rolebindings, but something seems to be wrong.
    = kubectl edit role developer -n blue
    = instead of blue-app, add dark-blue-app
11. Add a new rule in the existing role developer to grant the dev-user permissions to create deployments in the blue namespace.
    Remember to add api group "apps".
    = kubectl edit role developer -n blue
    apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: blue
rules:
- apiGroups:
  - apps
  resourceNames:
  - dark-blue-app
  resources:
  - pods
  verbs:
  - get
  - watch
  - create
  - delete
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - watch
  - create
  - delete

Service accounts:

- User accounts used by Humans, service accounts used by machines 
- user account could be admin using admin account to access the cluster
- service account used by an application to interact with the k8s cluster, example prometheus polls the kubernetes api for performance metrics
  or jenkins using service account to deploy applications on the kubernetes api.
- Example build a k8s dashboard build in python, build a list of pods querying k8s api, it requires to be authenticated using a service account
  kubectl create service account 
  kubernetes get service account
- When the service account is first created
  1. it first creates the object
  2. than generates a token for the service account
  3. then creates a secret object and stores that inside the secret object
  4. secret object is then linked to the service account
  5. So can create service account 
     1. assign correct permissions using RBAC
     2. Export service account tokens and use it to configure 3rd party application to authenticate to k8s api
  6. But what if your application is actually hosted on the kubernetes itself?
     1. This process can be made simple by automatically mounted the service token secret on the pod as a volume hosting the 3rd party application
     2. Default service account created on each namespace 
     3. When a new pod is created, the default service account and its token it mounted on the pod as volume mount. 
     4. If you want to use another service account instead othe default, modify the pod manifest with service account field and specify new service account
 
 Practice test: Service accounts
 
 1. How many Service Accounts exist in the default namespace? = kubectl get serviceaccounts = 1
 2. What is the secret token used by the default service account? = kubectl describe serviceaccounts default = default-token-66p9c
 3. We just deployed the Dashboard application. Inspect the deployment. What is the image used by the deployment? 
    = kubectl describe deployments web-dashboard = gcr.io/kodekloud/customimage/my-kubernetes-dashboard
 4. / 
 5. What is the state of the dashboard? Have the pod details loaded successfully? failed
 6. What type of account does the Dashboard application use to query the Kubernetes API? = Service accounts
 7. Which account does the Dashboard application use to query the Kubernetes API? = default
 8. Inspect the Dashboard Application POD and identify the Service Account mounted on it. = kubectl get pod web-dashboard-767bc588bc-xf2qp -o yaml
    default
 9. At what location is the ServiceAccount credentials available within the pod? = /var/run/secrets/kubernetes.io/serviceaccount
 10. The application needs a ServiceAccount with the Right permissions to be created to authenticate to Kubernetes. 
     The default ServiceAccount has limited access. Create a new ServiceAccount named dashboard-sa.
     = kubectl create serviceaccount dashboard-sa
 11. We just added additional permissions for the newly created dashboard-sa account using RBAC.
     If you are interested checkout the files used to configure RBAC at /var/rbac. We will discuss RBAC in a separate section.
     
 12. Enter the access token in the UI of the dashboard application. Click Load Dashboard button to load Dashboard
     Retrieve the Authorization token for the newly created service account , copy it and paste it into the token field of the UI.
     To do this, run kubectl describe against the secret created for the dashboard-sa service account, copy the token and paste it in the UI.
     = kubectl describe serviceaccounts dashboard-sa 
 13. You shouldn't have to copy and paste the token each time. The Dashboard application is programmed to read token from the secret mount location. However currently, the default service account is mounted. Update the deployment to use the newly created ServiceAccount
     Edit the deployment to change ServiceAccount from default to dashboard-sa
     = 
     apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dashboard
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      name: web-dashboard
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: web-dashboard
    spec:
      serviceAccountName: dashboard-sa
      containers:
      - image: gcr.io/kodekloud/customimage/my-kubernetes-dashboard
        imagePullPolicy: Always
        name: web-dashboard
        ports:
        - containerPort: 8080
          protocol: TCP           
 14. 
     
Image Security:
- Pod manifest file with 'nginx image' container.
- name is 'nginx'
- follow docker image convention, image is repository name it is actually -
  library - defauly account where docker images are stored.
- Images are stored and pulled from = Docker registery = docker.io this is where or by default
  is pulled from.
- Private repository = authenticated by credentials.
- to use image from private reg = in containers manifest file, you mention the name of the private repository.
  k8s gets the credentials to pull the private reg,
  1. need to create secret / docker registery 'regcred'
  2. server / user / pass / email
  3. specify in container manifest / imagepull secrets = name
      
    
Image security quiz:

1. What secret type must we choose for docker registry?
   docker-registry
2. We have an application running on our cluster. Let us explore it first. What image is the application using?
   nginx-alpine
3. We decided to use a modified version of the application from an internal private registry. Update the image of the deployment to use a new image from myprivateregistry.com:5000
   The registry is located at myprivateregistry.com:5000. Don't worry about the credentials for now. We will configure them in the upcoming steps.
   Use the kubectl edit deployment command to edit the image name to myprivateregistry.com:5000/nginx:alpine.
4. Are the new PODs created with the new images successfully running?
   = No
5. Create a secret object with the credentials required to access the registry.
Name: private-reg-cred
Username: dock_user
Password: dock_password
Server: myprivateregistry.com:5000
Email: dock_user@myprivateregistry.com
=
Run the command: kubectl create secret docker-registry private-reg-cred --docker-username=dock_user --docker-password=dock_password --docker-server=myprivateregistry.com:5000 --docker-email=dock_user@myprivateregistry.com
6. Configure the deployment to use credentials from the new secret to pull images from the private registry
   = 
    spec:
      containers:
      - image: myprivateregistry.com:5000/nginx:alpine
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      imagePullSecrets:  <<<-------------------
      - name: private-reg-cred
      dnsPolicy: ClusterFirst

Docker Security:

example
- Host with docker installed on it / docker deamon / ssh process 
- run ubuntu docker container runs a process sleeps for an hour
- containers and host share the same kernal
- containers are isolated using namespaces in linux
- all the process are run in the host itself 
- docker container in its own namespace and can only see its own processes
- different process id in different namespaces [process isoloation]
- by default docker runs processes in the root user 
  - if you want to change user for process, can simply do this
    docker run --user=1000 ubuntu sleep 3600
- can change or enforce define docker image, time of creation, set the user to 1000
- docker and host root users is different, not the same
- by default docker run root user with limited capapilities 
- can use the --cap-add command to run additonal priviledges 

Security Context:
- k8s containers encapsulated in pods 
- security setting at pod level ,setting carries to all containers
- if configured pod and continers, setting in the containrs overrides the pod
- when configuring pod manifest with container security context
  add section under containers called 'securityContext'
  capabilities add to the pod. 

Security context quiz:

1. What is the user used to execute the sleep process within the ubuntu-sleeper pod?
   = root
2. Edit the pod ubuntu-sleeper to run the sleep process with user ID 1010.
   Note: Only make the necessary changes. Do not modify the name or image of the pod.
   = 
   ---
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
  namespace: default
spec:
  securityContext:
    runAsUser: 1010
  containers:
  - command:
    - sleep
    - "4800"
    image: ubuntu
    name: ubuntu-sleeper
3. A Pod definition file named multi-pod.yaml is given. With what user are the processes in the web container started?
   The pod is created with multiple containers and security contexts defined at the Pod and Container level.
   = 1002
     containers overide pods 
     
4. With what user are the processes in the sidecar container started?
   The pod is created with multiple containers and security contexts defined at the Pod and Container level.
   = The User ID defined in the securityContext of the POD is carried over to all the containers in the Pod.
   = 1001
5. Update pod ubuntu-sleeper to run as Root user and with the SYS_TIME capability.
   Note: Only make the necessary changes. Do not modify the name of the pod.
   = 
   ---
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
  namespace: default
spec:
  containers:
  - command:
    - sleep
    - "4800"
    image: ubuntu
    name: ubuntu-sleeper
    securityContext:
      capabilities:
        add: ["SYS_TIME"] <<<<< change system time, this is usually for root of the host onlu 
6. Now update the pod to also make use of the NET_ADMIN capability.
   Note: Only make the necessary changes. Do not modify the name of the pod.
   = 
   apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
  namespace: default
spec:
  containers:
  - command:
    - sleep
    - "4800"
    image: ubuntu
    name: ubuntu-sleeper
    securityContext:
      capabilities:
        add: ["SYS_TIME", "NET_ADMIN"]


Network Policies:
- ingress and egress
- traffic to web server is ingress
- traffic away or back to user is egress
- ingress is always originated from user
  egress is always internal traffic egress traffic such as api to database server
- all pods are in a virtual private network, all by defaults pods, can reach each other
- default rules pod to pod within cluster all allowed by default
- What if we didnt want pods to talk to each other and restrict access,
- network policy is another object within the k8 namespace, like pod, replicsets
  link network policy with one or more pods.
- labels and selectors are used to create network policies
  podSelector:
    matchLabels: 
       role: db
  labels:
    role: db
- policyTypes:
  - Ingress
  ingress
  - from:
    - podSelector:
        matchLabels:
           name: api-pod
    ports:
    - protocol: TCP
      port: 3306
- 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
   name: db-policy
spec: 
   podSelector:
      matchLabels:
         role: db
   policyTypes:
   - Ingress
   ingress:
   - from:
     - podSelector:
         matchLabels:
           name: api-pod
    ports: 
    - protocol: TCP
      port: 3306

Developing network policies:

- 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
   name: db-policy
spec: 
   podSelector:
      matchLabels:
         role: db
   policyTypes:
   - Ingress
   ingress:
   - from:
     - podSelector:
         matchLabels:
           name: api-pod  <<<<<----------label of the pod
     - namespaceSelector: <<<<<<-----can separate namespace + labels
           matchLabels:
             name: prod
      - ipBlock:
           cidr: 192.168.5.10/32
    ports: 
    - protocol: TCP
      port: 3306
      
- stateful by default

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
   name: db-policy
spec: 
   podSelector:
      matchLabels:
         role: db
   policyTypes:
   - ingress
   - Egress
   ingress:
   - from:
     - podSelector:
         matchLabels:
           name: api-pod  <<<<<----------label of the pod
     ports:
     - protocol: TCP
       -port: 3306
    egress: 
    - to:
      - ipBlock:
           cidr: 192.168.5.10/32
    ports: 
    - protocol: TCP
      port: 80

Network Policy Quiz:

1. How many network policies do you see in the environment?
We have deployed few web applications, services and network policies. Inspect the environment.
= 1

2. What is the name of the Network Policy?
   = payroll-policy

3. Which pod is the Network Policy applied on?
   = kubectl get pods --selector=name=payroll
   = payroll

4. What type of traffic is this Network Policy configured to handle?
   = ingress
   
5. What is the impact of the rule configured on this Network Policy?
   = traffic from internal to port 8080 is allowed
   
6. What is the impact of the rule configured on this Network Policy?
   = internal pods can access 8080 on payroll pod 

8. Perform a connectivity test using the User Interface in these Applications to access the payroll-service at port 8080.   
   = only internal apps can access payroll services. 
 
9. Perform a connectivity test using the User Interface of the Internal Application to access the external-service at port 8080.
   = success
   
10. Create a network policy to allow traffic from the Internal application only to the payroll-service and db-service.
Use the spec given below. You might want to enable ingress traffic to the pod to test your rules in the UI.
= 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  - Ingress
  ingress:
    - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306

  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080

  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
      
 Note: We have also allowed Egress traffic to TCP and UDP port. This has been added to ensure that the internal DNS resolution works from the internal pod. Remember: The kube-dns service is exposed on port 53:
