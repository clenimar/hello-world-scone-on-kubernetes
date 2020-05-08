# SCONE on Kubernetes

This tutorial shows how to deploy confidential applications with the help of SCONE in a  Kubernetes cluster.

The tutorial is organized as follows:

1.  [Before you begin](#before-you-begin)
2.  [Hello World!](#hello-world)
3.  [Hello World! with remote attestation](#run-with-remote-attestation)
4.  [Hello World! with TLS certificates auto-generated by CAS](#tls-with-certificates-auto-generated-by-cas)
5.  [Encrypt your source code](#encrypt-your-source-code)

An executable version of this code can be cloned as follows:

```bash
git clone https://github.com/scontain/hello-world-scone-on-kubernetes.git
```

!!!info "Note that you need to get access to the images"
        [Send us an email](mailto:info@scontain.com) to get access.

You can run this demo by executing:

```bash
cd hello-world-scone-on-kubernetes
./run_hello_world.sh
```

### Before you begin

##### A Kubernetes cluster and a separate namespace

This tutorial assumes that you have access to a Kubernetes cluster (through `kubectl`), and that you are deploying everything to a separate namespace, called `hello-scone`. Namespaces are a good way to separate resources, not to mention how it makes the clean-up process a lot easier at the end:

```bash
kubectl create namespace hello-scone
```

Don't forget to specify it by adding `-n hello-scone` to your `kubectl` commands.


##### Set yourself a workspace

A lot of files are created through this tutorial, and they have to be somewhere. Choose whatever directory you want to run the commands below, but it might be a good idea to create a temporary one:

```bash
cd $(mktemp -d)
```

We use the following base image:

export BASE_IMAGE=sconecuratedimages/apps:python-3.7.3-alpine3.10

Have fun!

##### Set a Configuration and Attestation Service (CAS)

When remote attestation enters the scene, you will need a CAS to provide attestation, configuration and secrets. Export now the address of your CAS. If you don't have your own CAS, use our public one at scone.ml:

```bash
export SCONE_CAS_ADDR=scone-cas.cf
```

Now, generate client certificates (needed to submit new sessions to CAS):

```bash
mkdir -p conf
if [[ ! -f conf/client.crt || ! -f conf/client-key.key  ]] ; then
    openssl req -x509 -newkey rsa:4096 -out conf/client.crt -keyout conf/client-key.key  -days 31 -nodes -sha256 -subj "/C=US/ST=Dresden/L=Saxony/O=Scontain/OU=Org/CN=www.scontain.com" -reqexts SAN -extensions SAN -config <(cat /etc/ssl/openssl.cnf \
<(printf '[SAN]\nsubjectAltName=DNS:www.scontain.com'))
fi
```

### Hello World!

Let's start by writing a simple Python webserver: it only returns the message `Hello World!` and the value of the environment variable `GREETING`.

```bash
mkdir -p app
cat > app/server.py << EOF
from http.server import HTTPServer
from http.server import BaseHTTPRequestHandler
import os


class HTTPHelloWorldHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        """say "Hello World!" and the value of \`GREETING\` env. variable."""
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'Hello World!\n\$GREETING is: %s\n' % (os.getenv('GREETING', 'no greeting :(').encode()))


httpd = HTTPServer(('0.0.0.0', 8080), HTTPHelloWorldHandler)


httpd.serve_forever()
EOF
```

To build this application with SCONE, simply use our Python 3.7 curated image as base. The Python interpreter will run inside of an enclave!

```bash
cat > Dockerfile << EOF
FROM $BASE_IMAGE
EXPOSE 8080
COPY app /app
CMD [ "python", "/app/server.py" ]
EOF
```

Set the name of the image. If you already have an image, skip the building process.

```bash
export IMAGE=sconecuratedimages/kubernetes:hello-k8s-scone0.1
docker build --pull . -t $IMAGE && docker push $IMAGE
```

Deploying the application with Kubernetes is also simple: we just need a deployment and a service exposing it. 

Write the Kubernetes manifests:

```bash
cat > app.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  selector:
    matchLabels:
      run: hello-world
  replicas: 1
  template:
    metadata:
      labels:
        run: hello-world
    spec:
      containers:
      - name: hello-world
        image: $IMAGE
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: GREETING
          value: howdy!
        volumeMounts:
        - mountPath: /dev/isgx
          name: dev-isgx
        securityContext:
          privileged: true
      volumes:
      - name: dev-isgx
        hostPath:
          path: /dev/isgx
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  labels:
    run: hello-world
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    run: hello-world
EOF
```

Now, submit the manifests to the Kubernetes API, using `kubectl` command:

```bash
kubectl create -f app.yaml -n hello-scone
```

> **NOTE**: SGX applications need access to the Intel SGX driver, a char device located at `/dev/isgx` (in the host). If you are in a cluster context, it means that you need such device mounted into the container, and that's the reasoning behind the addition of the volume `dev-isgx` to the Kubernetes manifest. If you don't have access, or if you are not sure whether the underlying infrastructure has SGX installed, you can run in simulated mode, by adding `SCONE_MODE=sim` to the environment. This will emulate an enclave for you by encripting the main memory (Intel SGX security guarantees do not apply here).

> **NOTE**: You need special privileges to access a host device. Kubernetes does not allow a fine-grained access to such devices (like a Docker --device option does), so you need to run your container in privileged mode (i.e. your container has access to ALL host devices). This is defined in the container spec, as a `securityContext` policy (`privileged: true`).

Now that everything is deployed, you can access your Python app running on your cluster. Forward your local `8080` port to the service port:

```bash
kubectl port-forward svc/hello-world 8080:8080 -n hello-scone
```

The application will be available at your http://localhost:8080:

```bash
$ curl localhost:8080
Hello World! Environment GREETING is: howdy!
```

### Run with remote attestation

SCONE provides a remote attestation feature, so you make sure your application is running unmodified. It's also possible to have secrets and configuration delivered directly to the attested enclave!

The remote attestation is provided by two components: LAS (local attestation service, runs on the cluster) and CAS (a trusted service that runs elsewhere. We provide a public one in scone.ml).

You can deploy LAS to your cluster with the help of a DaemonSet, deploying one LAS instance per cluster node. As your application has to contact the LAS container running in the same host, we use the default Docker interface (172.17.0.1) as our LAS address.

```bash
cat > las.yaml << EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: local-attestation
  labels:
    k8s-app: local-attestation
spec:
  selector:
    matchLabels:
      k8s-app: local-attestation
  template:
    metadata:
      labels:
        k8s-app: local-attestation
    spec:
      hostNetwork: true
      volumes:
      - name: dev-isgx
        hostPath:
          path: /dev/isgx
      containers:
        - name: local-attestation
          image: sconecuratedimages/services:las
          volumeMounts:
          - mountPath: /dev/isgx
            name: dev-isgx
          securityContext:
            privileged: true
          ports:
          - containerPort: 18766
            hostPort: 18766
EOF
```

```bash
kubectl create -f las.yaml -n hello-scone
```

To setup remote attestation, you will need a session file and the `MRENCLAVE`, which is a unique signature of your application. Extract `MRENCLAVE` of your application by running its container with the environment variable `SCONE_HASH` set to `1`:

```bash
MRENCLAVE=`docker run -i --rm -e "SCONE_HASH=1" $IMAGE`
```

Then create a session file. Please note that we are also defining a secret `GREETING`. Only the `MRENCLAVES` registered in this session file will be allowed to see such secret.

```bash
cat > session.yaml << EOF
name: hello-k8s-scone
version: "0.2"

services:
   - name: application
     image_name: $IMAGE
     mrenclaves: [$MRENCLAVE]
     command: python /app/server.py
     pwd: /
     environment:
        GREETING: hello from SCONE!!!

images:
   - name: $IMAGE
     mrenclaves: [$MRENCLAVE]
     tags: [demo]
EOF
```

Now, [post the session file](https://sconedocs.github.io/cas_blender_example/) to your CAS of choice:

```bash
curl -v -k -s --cert conf/client.crt --key conf/client-key.key --data-binary @session.yaml -X POST https://$SCONE_CAS_ADDR:8081/v1/sessions
```

Once you submit the session file, you just need to inject the CAS address into your Deployment manifest. To showcase that, we're creating a new Deployment:

```bash
cat > attested-app.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attested-hello-world
spec:
  selector:
    matchLabels:
      run: attested-hello-world
  replicas: 1
  template:
    metadata:
      labels:
        run: attested-hello-world
    spec:
      containers:
      - name: attested-hello-world
        image: $IMAGE
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: SCONE_CAS_ADDR
          value: $SCONE_CAS_ADDR
        - name: SCONE_CONFIG_ID
          value: hello-k8s-scone/application
        - name: SCONE_LAS_ADDR
          value: 172.17.0.1:18766
        volumeMounts:
        - mountPath: /dev/isgx
          name: dev-isgx
        securityContext:
          privileged: true
      volumes:
      - name: dev-isgx
        hostPath:
          path: /dev/isgx
---
apiVersion: v1
kind: Service
metadata:
  name: attested-hello-world
  labels:
    run: attested-hello-world
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    run: attested-hello-world
EOF
```

**NOTE**: The environment defined in the Kubernetes manifest won't be considered once your app is attested. The environment will be retrieved from your session file.

Deploy your attested app:

```bash
kubectl create -f attested-app.yaml -n hello-scone
```

Once again, forward your local port 8081 to the service port:

```bash
kubectl port-forward svc/attested-hello-world 8081:8080 -n hello-scone
```

The attested application will be available at your http://localhost:8081:

```bash
$ curl localhost:8081
Hello World! Environment GREETING is: hello from SCONE!!!
```

### TLS with certificates auto-generated by CAS

The CAS service can also generate secrets and certificates automatically. Combined with access policies, it means that such auto-generated secrets and certificates will be seen only by certain applications. No human (e.g. a system operator) will ever see them: they only exist inside of SGX enclaves.

To showcase such feature, let's use the same application as last example. But now, we serve the traffic over TLS, and we let CAS generate the server certificates.

Start by rewriting our application to serve with TLS:

```bash
cat > app/server-tls.py << EOF
from http.server import HTTPServer
from http.server import BaseHTTPRequestHandler
import os
import socket
import ssl


class HTTPHelloWorldHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        """say "Hello World!" and the value of \`GREETING\` env. variable."""
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'Hello World!\n\$GREETING is: %s\n' % (os.getenv('GREETING', 'no greeting :(').encode()))


httpd = HTTPServer(('0.0.0.0', 4443), HTTPHelloWorldHandler)


httpd.socket = ssl.wrap_socket(httpd.socket,
                               keyfile="/app/key.pem",
                               certfile="/app/cert.pem",
                               server_side=True)


httpd.serve_forever()
EOF
```

Our updated Dockerfile looks like this:

```bash
cat > Dockerfile << EOF
FROM $BASE_IMAGE
EXPOSE 4443
COPY app /app
CMD [ "/usr/local/bin/python" ]
EOF
```

Build the updated image:

```bash
export IMAGE=sconecuratedimages/kubernetes:hello-k8s-scone0.2
docker build --pull . -t $IMAGE && docker push $IMAGE
```

The magic is done in the session file. First, extract the `MRENCLAVE` again:

```bash
MRENCLAVE=$(docker run -i --rm --device /dev/isgx -e "SCONE_HASH=1" $IMAGE)
```

Now, create a Session file to be submitted to CAS. The certificates are defined in the `secrets` field, and are injected into the filesystem through `images.injection_files`:

```bash
cat > session-tls-certs.yaml << EOF
name: hello-k8s-scone-tls-certs
version: "0.2"

services:
   - name: application
     image_name: $IMAGE
     mrenclaves: [$MRENCLAVE]
     command: python /app/server-tls.py
     pwd: /
     environment:
        GREETING: hello from SCONE with TLS and auto-generated certs!!!

images:
   - name: $IMAGE
     injection_files:
       - path:  /app/cert.pem
         content: \$\$SCONE::SERVER_CERT.crt\$\$
       - path: /app/key.pem
         content: \$\$SCONE::SERVER_CERT.key\$\$

secrets:
   - name: SERVER_CERT
     kind: x509
EOF
```

[Post the session file](https://sconedocs.github.io/cas_blender_example/) to your CAS of choice:

```bash
curl -k -s --cert conf/client.crt --key conf/client-key.key --data-binary @session-tls-certs.yaml -X POST https://$SCONE_CAS_ADDR:8081/session
```

The steps to run your app are similar to before: let's create a Kubernetes manifest and submit it.

```bash
cat > attested-app-tls-certs.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attested-hello-world-tls-certs
spec:
  selector:
    matchLabels:
      run: attested-hello-world-tls-certs
  replicas: 1
  template:
    metadata:
      labels:
        run: attested-hello-world-tls-certs
    spec:
      containers:
      - name: attested-hello-world-tls-certs
        image: $IMAGE
        imagePullPolicy: Always
        ports:
        - containerPort: 4443
        env:
        - name: SCONE_CAS_ADDR
          value: $SCONE_CAS_ADDR
        - name: SCONE_CONFIG_ID
          value: hello-k8s-scone-tls-certs/application
        - name: SCONE_LAS_ADDR
          value: "172.17.0.1"
        volumeMounts:
        - mountPath: /dev/isgx
          name: dev-isgx
        securityContext:
          privileged: true
      volumes:
      - name: dev-isgx
        hostPath:
          path: /dev/isgx
---
apiVersion: v1
kind: Service
metadata:
  name: attested-hello-world-tls-certs
  labels:
    run: attested-hello-world-tls-certs
spec:
  ports:
  - port: 4443
    protocol: TCP
  selector:
    run: attested-hello-world-tls-certs
EOF
```

```bash
kubectl create -f attested-app-tls-certs.yaml -n hello-scone
```

Now it's time to access your app. Again, forward the traffic to its service:

```bash
kubectl port-forward svc/attested-hello-world-tls-certs 8083:4443 -n hello-scone
```

And send a request:

```bash
$ curl -k https://localhost:8083
Hello World!
$GREETING is: hello from SCONE with TLS and auto-generated certs!!!
```

### Encrypt your source code

Moving further, you can run the exact same application, but now the server source code will be encrypted using SCONE's [file protection](https://sconedocs.github.io/SCONE_Fileshield/) feature, and only the Python interpreter that you register will be able to read them (after being attested by CAS).

##### Encrypt the source code and filesystem

Let's encrypt the source code using SCONE's Fileshield. Run a SCONE CLI container with access to the files:

```bash
docker run -it --rm --device /dev/isgx -v $PWD:/tutorial $BASE_IMAGE sh
```

Inside the container, create an encrypted region `/app` and add all the files in `/tutorial/app` (the plain files) to it. Lastly, encrypt the key itself.

```bash
cd /tutorial
rm -rf app_image && mkdir -p app_image/app
cd app_image
scone fspf create fspf.pb
scone fspf addr fspf.pb / --not-protected --kernel /
scone fspf addr fspf.pb /app --encrypted --kernel /app
scone fspf addf fspf.pb /app /tutorial/app /tutorial/app_image/app
scone fspf encrypt fspf.pb > /tutorial/app/keytag
```

The contents of `/tutorial/app_image/app` are now encrypted. Try to `cat` them from inside the container:

```bash
$ cd /tutorial/app_image/app && ls
server-tls.py
$ cat server-tls.py
$�ʇ(E@��
�����Id�Z���g3uc��mѪ�Z.$�__�!)ҹS��٠���(�� �X�Wϐ�{$���|�זH��Ǔ��!;2����>@z4-�h��3iݾs�t�7�<>H4:����(9=�a)�j?��p����q�ߧ3�}��Hظa�|���w-�q��96���o�9̃�Ev�v��*$����TU/���Ӕ��v�G���T�1<ڹ��C#p|i��AƢ˅_!6���F���w�@
                   �C��J���+81�8�
```

##### Build the new image

You can now exit the container. Let's build the server image with our files, now encrypted:

```bash
cat > Dockerfile << EOF
FROM $BASE_IMAGE
EXPOSE 4443
COPY app_image /
CMD [ "/usr/local/bin/python" ]
EOF
```

Choose the new image name and build it:

```bash
export IMAGE=sconecuratedimages/kubernetes:hello-k8s-scone-0.3
docker build --pull . -t $IMAGE && docker push $IMAGE
```

##### Run in Kubernetes

Time to deploy our server to Kubernetes. Let's setup the attestation first, so we make sure that only our application (identified by its `MRENCLAVE`) will have access to the secrets to read the encrypted filesystem, as well as our secret greeting.

Extract the `MRENCLAVE`:

```bash
MRENCLAVE=$(docker run -i --rm --device /dev/isgx -e "SCONE_HASH=1" $IMAGE)
```

Extract `SCONE_FSPF_KEY` and `SCONE_FSPF_TAG`:

```bash
export SCONE_FSPF_KEY=$(cat app/keytag | awk '{print $11}')
export SCONE_FSPF_TAG=$(cat app/keytag | awk '{print $9}')
export SCONE_FSPF=/fspf.pb
```

Create a session file:

```bash
cat > session-tls.yaml << EOF
name: hello-k8s-scone-tls
version: "0.2"

services:
   - name: application
     image_name: $IMAGE
     mrenclaves: [$MRENCLAVE]
     command: python /app/server-tls.py
     pwd: /
     environment:
        GREETING: hello from SCONE with encrypted source code and auto-generated certs!!!
     fspf_path: $SCONE_FSPF
     fspf_key: $SCONE_FSPF_KEY
     fspf_tag: $SCONE_FSPF_TAG

images:
   - name: $IMAGE
     injection_files:
       - path:  /app/cert.pem
         content: \$\$SCONE::SERVER_CERT.crt\$\$
       - path: /app/key.pem
         content: \$\$SCONE::SERVER_CERT.key\$\$

secrets:
   - name: SERVER_CERT
     kind: x509
EOF
```

[Post the session file](https://sconedocs.github.io/cas_blender_example/) to your CAS of choice:

```bash
curl -k -s --cert conf/client.crt --key conf/client-key.key --data-binary @session-tls.yaml -X POST https://$SCONE_CAS_ADDR:8081/session
```

Create the manifest and submit it to your Kubernetes cluster:

```bash
cat > attested-app-tls.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attested-hello-world-tls
spec:
  selector:
    matchLabels:
      run: attested-hello-world-tls
  replicas: 1
  template:
    metadata:
      labels:
        run: attested-hello-world-tls
    spec:
      containers:
      - name: attested-hello-world-tls
        image: $IMAGE
        imagePullPolicy: Always
        ports:
        - containerPort: 4443
        env:
        - name: SCONE_CAS_ADDR
          value: $SCONE_CAS_ADDR
        - name: SCONE_CONFIG_ID
          value: hello-k8s-scone-tls/application
        - name: SCONE_LAS_ADDR
          value: 172.17.0.1
        volumeMounts:
        - mountPath: /dev/isgx
          name: dev-isgx
        securityContext:
          privileged: true
      volumes:
      - name: dev-isgx
        hostPath:
          path: /dev/isgx
---
apiVersion: v1
kind: Service
metadata:
  name: attested-hello-world-tls
  labels:
    run: attested-hello-world-tls
spec:
  ports:
  - port: 4443
    protocol: TCP
  selector:
    run: attested-hello-world-tls
EOF
```

```bash
kubectl create -f attested-app-tls.yaml -n hello-scone
```

Time to access your app. Forward the traffic to its service:

```bash
kubectl port-forward svc/attested-hello-world-tls 8082:4443 -n hello-scone
```

And send a request:

```bash
$ curl -k https://localhost:8082
Hello World!
$GREETING is: hello from SCONE with encrypted source code and auto-generated certs!!!
```

### Clean up

To clean up all the resources created in this tutorial, simply delete the namespace:

```bash
kubectl delete namespace hello-scone
```

Author: Clenimar

&copy; [scontain.com](http://www.scontain.com), 2020. [Questions or Suggestions?](mailto:info@scontain.com)
