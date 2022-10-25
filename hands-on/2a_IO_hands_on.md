# Scientific Computing with Kubernetes Tutorial

Dealing with data\
Hands on session

## Simple config files

Let's start with creating a simple test file and import it into a pod.

Create a local file named `hello.txt` with any content you fancy.

Let's now import this file into a configmap (replace username, as before):
```
kubectl create configmap config1-<username> --from-file=hello.txt
```

Can you see it?
```
kubectl get configmap
```

You can also look at its content with
```
kubectl get configmap -o yaml config1-<username>
```

Import that file into a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c1-<username>
spec:
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 100Mi
        cpu: 100m
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: hello
      mountPath: /tmp/hello.txt
      subPath: hello.txt
  volumes:
  - name: hello
    configMap:
      name: config1-<username>
      items:
      - key: hello.txt
        path: hello.txt
```

Create the pod and once it has started, login using kubectl exec and check if the file is indeed in the /tmp directory.

Inside the pod, make some changes to 
```
/tmp/hello.txt
```

Did that affect the content of the configmap?

Once you are done exploring, delete the pod and the configmap:
```
kubectl delete pod c1-<username>
kubectl delete configmap config1-<username> 
```

## Importing a whole directory

Create an additional local file named `world.txt` with any content you fancy.

Let's now import this file into a configmap (replace username, as before):
```
kubectl create configmap config2-<username> --from-file=hello.txt --from-file=world.txt
```

Check its content, as before.

Let's now import the whole configmap into a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c2-<username>
spec:
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: cfg
      mountPath: /tmp/myconfig
  volumes:
  - name: cfg
    configMap:
      name: config2-<username>
```

Create the pod and once it has started, login using kubectl exec and check if the file is indeed in the /tmp/myconfig directory.

Once again, try to modify their content.

Log out, and update the content of either hello.txt or world.txt.

Let's now update the configmap:

```
kubectl create configmap config2-<username> --from-file=hello.txt --from-file=world.txt --dry-run=client -o yaml |kubectl replace -f -
```

Log back into the node, and check the content of the files in /tmp/myconfig.

Wait a minute (or so), and check again. The changes should propagate into the running pod. 

Once you are done exploring, delete the pod and the configmap:
```
kubectl delete pod c2-<username>
kubectl delete configmap config2-<username> 
```

## Importing code

Let's now do something (semi) useful. Let's [compute pi](https://www.geeksforgeeks.org/calculate-pi-with-python/).

###### pi.py:

```
import os

itr=1000
if 'PI_STEPS' in os.environ:
  itr=int(os.environ['PI_STEPS'])


# compute pi
k = 1
pi = 0

print("Computing pi using %i steps"%itr)
for i in range(itr):
    pi += (1 if ((i % 2)==0) else -1) * 4/k
    k += 2
     
print(pi)
```
As before, let's put it in a configmap:
```
kubectl create configmap config3-<username> --from-file=pi.py
```

Now import that file into a pod and compute pi:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: c3-<username>
spec:
  completions: 1
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: tensorflow/tensorflow:2.10.0
        resources:
          limits:
            memory: 1.5Gi
            cpu: 1
          requests:
            memory: 500Mi
            cpu: 1
        command: ["sh", "-c", "python3 /opt/code/pi.py; echo Done"]
        volumeMounts:
        - name: pi
          mountPath: /opt/code/pi.py
          subPath: pi.py
      volumes:
      - name: pi
        configMap:
          name: config3-<username>
          items:
          - key: pi.py
            path: pi.py
```

Once the pod terminated, check if the result is correct:
```
kubectl logs c3-<usernam>-<hash>
```

You may have noticed that we used the default 1000 steps. Let's get more precision, and use 100000 steps.
Our script can be configured by passing the appropriate environment variable (and no other changes are needed):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: c3m-<username>
spec:
  completions: 1
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: tensorflow/tensorflow:2.10.0
        resources:
          limits:
            memory: 1.5Gi
            cpu: 1
          requests:
            memory: 500Mi
            cpu: 1
        command: ["sh", "-c", "python3 /opt/code/pi.py; echo Done"]
        env:
        - name: PI_STEPS
          value: "100000"
        volumeMounts:
        - name: pi
          mountPath: /opt/code/pi.py
          subPath: pi.py
      volumes:
      - name: pi
        configMap:
          name: config3-<username>
          items:
          - key: pi.py
            path: pi.py
```

Once the pod terminated, check the result with:
```
kubectl logs c3m-<usernam>-<hash>
```

You should see a result that is slightly different than the before.

You can now delete the pods and the configmap:
```
kubectl delete job c3m-<username>
kubectl delete job c3-<username>
kubectl delete configmap config3-<username> 
```

## Using a secret

Secrets are very similar to configmaps, but provide a little additional protecton for sensitive information.

Create a local file named `mysecret.txt` with any content you fancy.

Let's now import this file into a secret object (replace username, as before):
```
kubectl create secret generic secret1-<username> --from-file=mysecret.txt
```

Can you see it?
```
kubectl get secret
```

You can also look at its content with
```
kubectl get secret -o yaml secret1-<username>
```

How does it differ from the configmap?


Import that file into a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c4-<username>
spec:
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 100Mi
        cpu: 0.1
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: s1
      mountPath: /etc/secret/mysecret.txt
      subPath: mysecret.txt
  volumes:
  - name: s1
    secret:
      secretName: secret1-<username>
      defaultMode: 384
      items:
      - key: mysecret.txt
        path: mysecret.txt
```

Create the pod and once it has started, login using kubectl exec and check if the file is indeed in the /etc/secret directory.

Can you see its content?

Do you think you could access the secret of any other user in the namespace that way, too?

You can now delete the pod and the configmap:
```
kubectl delete pod c4-<username>
kubectl delete secret config4-<username> 
```

## Using ephemeral storage

In the past exercizes You have seen that you can write in pretty much any part of the filesystem inside a pod.
But the amount of space you have at your disposal is limited.

If you need access to a larger (and often faster) local area, you should use the so-called ephemeral storage using emptyDir.

Note that you can request either a disk-based or a memory-based partition. We will do both below.

You can copy-and-paste the lines below, but please do replace “username” with your own id;\
As mentioned before, all the participants in this hands-on session share the same namespace, so you will get name collisions if you don’t.

###### s1.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s1-<username>
spec:
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 8Gi
        cpu: 100m
        ephemeral-storage: 10Gi
      requests:
        memory: 4Gi
        cpu: 100m
        ephemeral-storage: 1Gi
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: scratch1
      mountPath: /mnt/myscratch
    - name: ramdisk1
      mountPath: /mytmp
  volumes:
  - name: scratch1
    emptyDir: {}
  - name: ramdisk1
    emptyDir:
      medium: Memory
```

Create the pod and once it has started, log into it using kubectl exec.

Look at the mounted filesystems:

```
df -H / /mnt/myscratch /tmp /mytmp /dev/shm
```

As you can see, / and /tmp are on the same filesystem, but /mnt/myscratch is a completely different filesystem.

You should also notice that /dev/shm is tiny; the real ramdisk is /mytmp.

*Note:* You can mount the ramdisk as /dev/shm (or /tmp). that way your applications will find it where they expect it to be.

Once you are done exploring, please delete the pod.

## Communicating between containers

Kubernetes allows you to run multiple containers as part of a single pod. 
While there are many uses for such a setup, for batch oriented workloads the most useful concept is the initialization pod.

Container images come with a pre-defined setup, and while they may have everything for your main application, 
there may be some tools that are missing for your job initialization. To avoid creating a dedicated image
we can instead use a separate image for the initialization phase, and pass the results using the emphemeral partition.

As an example, let's build an application from source, and import it into the main container that does not contain any compilers:

###### hello.c:

```
#include <stdio.h>

int main() {
  printf("Hello world\n");
  return 0;
}
```

Put it in a configmap:
```
kubectl create configmap configs2-<username> --from-file=hello.c
```

And launch a two container pod:

###### s2.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s2-<username>
spec:
  initContainers:
  - name: myinit
    image: gcc:9.5.0
    resources:
      limits:
        memory: 500Mi
        cpu: 1
        ephemeral-storage: 10Gi
      requests:
        memory: 100Mi
        cpu: 1
        ephemeral-storage: 1Gi
    command: ["sh", "-c", "gcc -static -o /opt/my/bins/my_hello /opt/code/hello.c"]
    volumeMounts:
    - name: bins
      mountPath: /opt/my/bins
    - name: hello
      mountPath: /opt/code/hello.c
      subPath: hello.c
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 100Mi
        cpu: 100m
        ephemeral-storage: 10Gi
      requests:
        memory: 100Mi
        cpu: 100m
        ephemeral-storage: 1Gi
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: bins
      mountPath: /opt/my/bins
  volumes:
  - name: hello
    configMap:
      name: configs2-<username>
      items:
      - key: hello.c
        path: hello.c
  - name: bins
    emptyDir: {}
```

Create the pod and monitor what's going on with:

```
kubectl get events --sort-by=.metadata.creationTimestamp |grep s2-<username>
```

Once the main container has started Running, log into it using kubectl exec.

You should now be able to execute the newly built binary:
```
/opt/my/bins/my_hello
```

Can you find the source code anywhere?

*Hint:* Did you notice any warning messages when you logged in?

Once you are done exploring, please delete the pod.

## Using persistent storage

Everything we have done so far has been temporary in nature.

The moment you delete the pod, everything that was computed is gone.

Most applications will however need access to long term data for either/both input and output.

In the Kubernetes cluster you are using we have a distributed filesystem, which allows using it for real data persistence.

To get storage, we need to create an object called PersistentVolumeClaim.

By doing that we "Claim" some storage space from a "Persistent Volume".

There will actually be PersistentVolume created, but it's a cluster-wide resource which you can not see.

Create the file (replace username as always):

###### pvc.yaml:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vol-<username>
spec:
  storageClassName: rook-cephfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

We're creating a 1GB volume.

Look at it's status with 

```
kubectl get pvc vol-<username>
```
(replace username). 

The `STATUS` field should be equals to `Bound` - this indicates successful allocation.

Note that it may take a few seconds for the system to get there, so be patient.
You can check the progress with

```
kubectl get events --sort-by=.metadata.creationTimestamp --field-selector involvedObject.name=vol-<username>
```

Now we can attach it to our pod. Create one with (replacing `username`):

###### s3.yaml:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: s3-<username>
spec:
  completionMode: Indexed
  completions: 10
  parallelism: 10
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: centos:centos8
        resources:
           limits:
             memory: 100Mi
             cpu: 0.1
           requests:
             memory: 100Mi
             cpu: 0.1
        command: ["sh", "-c", "let s=2*$JOB_COMPLETION_INDEX; d=`date +%s`; date; sleep $s; (echo Job $JOB_COMPLETION_INDEX; ls -l /mnt/mylogs/)  > /mnt/mylogs/log.$d.$JOB_COMPLETION_INDEX; sleep 1000"]
        volumeMounts:
        - name: mydata
          mountPath: /mnt/mylogs
      volumes:
      - name: mydata
        persistentVolumeClaim:
          claimName: vol-<username>
```

Create the job and once any of the pods has started, log into it using kubectl exec.

Check the content of /mnt/mylogs
```
ls -l /mnt/mylogs
```

Try to create a file in there, with any content you like.

Now, delete the job, and create another one (with the same name).

Once one of the new pods start, log into it using kubectl exec.

What do you see in /mnt/mylogs ?

Once you are done exploring, please delete the pod.

If you have time, try to do the same exercise but using emptyDir. And verify that the logs indeed do not get preserved between pod restarts.

## Attaching existing storage

Sometimes you just need to attach to existing storage that is shared between multiple users.

In this cluster we have one CVMFS mountpoint available.

Let's mount it in our pod:

###### s4.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s4-<username>
spec:
  containers:
  - name: mypod
    image: centos:centos7
    resources:
      limits:
        memory: 1Gi
        cpu: 1
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: cvmfs
      mountPath: /cvmfs/oasis.opensciencegrid.org
      readOnly: true
  volumes:
  - name: cvmfs
    persistentVolumeClaim:
      claimName: cvmfs-oasis
```


Now let’s start the pod and log into the created pod.

Try to use git:
```
git
```

You will notice it is not present in the image.

So let's use the one on CVMFS, under `osg-software/osg-wn-client`:
```
source /cvmfs/oasis.opensciencegrid.org/osg-software/osg-wn-client/current/el7-x86_64/setup.sh 
git
```

Feel free to explore the rest of the mounted filesystem.

When you are done, delete the pod.

## Explicitly moving files in

Most science users have large input data.
If someone has not already pre-loaded them on a PVC, you will have to fetch them yourself.

You can use any tool you are used to, from curl to ssh. You can either pre-load it to a PVC or fetch the data just-in-time, whatever you feel is more appropriate.

We do not have an explicit hands-on tutorial, but feel free to try out your favorite tool using what you have learned so far.

## Handling output data

Unlinke most batch systems, there is no shared filesystem between the submit host (aka your laptop) and the execution nodes.

You are responsible for explicit movement of the output files to a location that is useful for you.

The easiest option is to keep the output files on a persistent PVC and do the follow-up analysis inside the kuberenetes cluster.

But when you want any data to be exported outside of the kubernetes cluster, you will have to do it explicitly.
You can use any (authenticated) file transfer tool from ssh to globus. Just remember to inject the necessary creatials into to pod, ideally using a secret.

For *small files*, you can also use the `kubectl cp` command.
It is similar to `scp`, but routes all traffic through a central kubernetes server, making it very slow.
Good enough for testing and debugging, but not much else. 

Again, we do not have an explict hands-on tutorial, and we discourage the uploading of any sensitive cretentials to this shared kubernetes setup.

## End

**Please make sure you did not leave any running pods. Jobs and associated completed pods are OK.**

