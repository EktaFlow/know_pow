## Something is Wrong with the Deployments

#### Step 1: check out the pods
List out the basic info about the pods with: ```kubectl get pods -n dev```
```shell
NAME                                                        READY     STATUS             RESTARTS   AGE
dev-5b456c494b-000                                          2/3       CrashLoopBackOff   5          3m
cool-tiger-nginx-ingress-controller-546bb5bc56-00000        1/1       Running            0          84d
cool-tiger-nginx-ingress-default-backend-6dc4665cd8-00000   1/1       Running            0          84d
```
#### Check inside the pod that has a bad status
There are two ways to quickly check what is going on with a pod.
1. ```kubectl describe pod <pod_name> -n <namespace_name>```, will help you debug configuration erorrs, eg. k8s is trying to pull an image whose name doesn't appear in the repository.
2. ```kubectl logs pod <pod_name> -n <namespace_name> -c <container_name>```, will give you the logging info (stderr/out) from a running container. If the pod has only one container (which is the best practice, this particular example has multiple containers in a pod because its the dev envronment and technically requires less typing... however less typing is always a bad excuse, I should change this config) you don't need to provide the ```-c``` flag. This will allow you to debug application-level or container-level problems

Run: 
```kubectl logs -n dev dev-0000000000-00000 -c back```

output:
```shell
/user/src/back/src/resolvers.js:127
	var addedAnswer = await Answer.create(newAnswers);
	                  ^^^^^

SyntaxError: await is only valid in async function
    at createScript (vm.js:80:10)
    at Object.runInThisContext (vm.js:139:10)
    at Module._compile (module.js:616:28)
    at Object.Module._extensions..js (module.js:663:10)
    at Module.load (module.js:565:32)
    at tryModuleLoad (module.js:505:12)
    at Function.Module._load (module.js:497:3)
    at Module.require (module.js:596:17)
    at require (internal/module.js:11:18)
    at Object.<anonymous> (/user/src/back/index.js:3:19)
```
#### Find a good image to use in place of the broken one.
Ok, so we now know that there's a problem with the ```back``` container. We're not going to use a ```rollback``` because given the nature of the bug, we don't know that the previous rollout of the deployment was valid either. (Actually, I checked and it wasn't). Instead, we're going to go back to Jenkins and check for the last build that we know was successful. 

--> CAN'T TAKE SCREENSHOT IN KDE ;p;p;p;p

1. Go to Jenkins
2. Go to /back jobset
3. go to /dev repo jobset
4. Look for newest blue-running build on lefthand history. (Again, a blue-running build doesn't mean its performing as expected in the application layer, pick one you know is successful)
5. Get the console of the build
6. Find the name of the image tag and version, example back-dev:b46
7. Going to go into kubectl and use this good image as the base for the container in the dev deployment that is causing the problems.

#### Editing the deployment
``` kubectl edit deployments/dev -n dev``` will bring up the version of the config file that is currently in use. (Note: it will also display all of the default options explicitly, so you can learn ways that your written configuration file can be tweaked). 

Find the image definition that was causing the problems and replace it with the correct one. Make the edit directly in the text editor.
```yaml
 image: registry/back-dev:messed-up
 # change to:
 image: registry/back-dev:b46
```

exiting your text editor will automatically update the deployment... and you're back in business.
