# Build trusted applications

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 10.12.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 5.0.0 or higher.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
{% endhint %}

{% hint style="warning" %}
Please make sure you have already checked the [quickstart](../your-first-app.md) and [Your first application](../your-first-app.md) tutorials before building your trusted application on iExec.
{% endhint %}

After understanding the fundamentals of Confidential Computing and explaining the technologies behind it, it is time to roll up our sleeves and get hands-on with [enclaves](intel-sgx-technology.md#enclave). In this guide, we will focus on protecting an application - that is already compatible with the iExec platform - using SGX, and without changing the source code. That means we will use the same code we [previously](../your-first-app.md#build-your-app) deployed for a basic iExec application.

## Prepare your application:

Create a directory tree for your application in `~/iexec-projects/`.

```bash
cd ~/iexec-projects
mkdir my-tee-hello-world-app && cd my-tee-hello-world-app
mkdir src
touch Dockerfile
touch protect-fs.sh
```

In the folder `src/` create the file `app.js` \(or `app.py` if you want to use Python\) then copy [this](../your-first-app.md#write-the-app-shell-script-example) code inside.

As we mentioned earlier, the advantage of using **SCONE** is the ability to make the application **Intel® SGX-enabled** without changing the source code. The only thing we are going to modify is the `Dockerfile`. First, we need to change the base image from the official `node` \(or `python`\) to the one provided by SCONE: `sconecuratedimages/apps:node-8.9.4-alpine-scone3.0`or `sconecuratedimages/apps:python-3.7.3-alpine3.10-scone3.0`. Those base docker images contain a `nodejs` / `python` interpreters that run inside enclaves.

{% tabs %}
{% tab title="Javascript" %}
{% code title="Dockerfile" %}
```bash
FROM sconecuratedimages/sconecli:alpine3.7-scone3.0 AS scone

FROM sconecuratedimages/apps:node-8.9.4-alpine-scone3.0

COPY --from=scone   /opt/scone/scone-cli    /opt/scone/scone-cli
COPY --from=scone   /usr/local/bin/scone    /usr/local/bin/scone
COPY --from=scone   /opt/scone/bin          /opt/scone/bin

### install dependencies you need
RUN apk add bash nodejs-npm
RUN SCONE_MODE=sim npm install figlet

COPY ./src /app

###  protect file system with Scone
COPY ./protect-fs.sh ./Dockerfile /build/
RUN sh /build/protect-fs.sh /app

ENTRYPOINT [ "node", "/app/app.js"]
```
{% endcode %}
{% endtab %}

{% tab title="Python" %}
{% code title="Dockefile" %}
```bash
FROM sconecuratedimages/apps:python-3.7.3-alpine3.10-scone3.0

### install python3 dependencies you need
RUN SCONE_MODE=sim pip3 install pyfiglet

COPY ./src /app

###  protect file system with Scone
COPY ./tee/protect-fs.sh ./tee/Dockerfile /build/
RUN sh /build/protect-fs.sh /app

ENTRYPOINT ["python", "/app/app.py"]
```
{% endcode %}
{% endtab %}
{% endtabs %}

In the above [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/), we copy some needed SCONE CLI binaries from the image `sconecuratedimages/sconecli:alpine3.7-scone3.0` \(we do not do that for the python image since it already has those files\) then we install the dependencies of the application. The last section is the answer to the legitimate question: **how would the enclave verify the integrity of the code?**

The short answer is: the application is protected by taking a snapshot of the file system's state. The script `protect-fs.sh` uses the famous [fspf](intel-sgx-technology.md#fspf-file-system-protection-file) feature of SCONE to authenticate the file system directories that would be used by the application \(/bin, /lib...\) as well as the code itself. It takes a snapshot of their state that will be later shared with the worker \(via the Blockchain\) to make sure everything is under control. If we change one bit of one of the authenticated files, the file system's state changes completely and the enclave will refuse to boot since it considers it as a possible attack.

The content of `protect-fs.sh` should look like this:

{% code title="protect-fs.sh" %}
```bash
#!/bin/sh

cd $(dirname $0)

if [ ! -e Dockerfile ]
then
    printf "\nFailed to parse Dockerfile ENTRYPOINT\n"
    printf "Did you forget to add your Dockerfile in your build?\n"
    printf "COPY ./tee/Dockerfile /build/\n\n"
    exit 1
fi

ENTRYPOINT_ARSG=$(grep ENTRYPOINT ./Dockerfile | tail -1 |  grep -o '"[^"]\+"' | tr -d '"')
echo $ENTRYPOINT_ARSG > ./entrypoint

if [ -z "$ENTRYPOINT_ARSG" ]
then
    printf "\nFailed to parse Dockerfile ENTRYPOINT\n"
    printf "Did you forget to add an ENTRYPOINT to your Dockerfile?\n"
    printf "ENTRYPOINT [\"executable\", \"param1\", \"param2\"]\n\n"
    exit 1
fi

INTERPRETER=$(awk '{print $1}' ./entrypoint) # python
ENTRYPOINT=$(cat ./entrypoint) # /python /app/app.py

export SCONE_MODE=sim
export SCONE_HEAP=1G

APP_FOLDER=$1

printf "\n### Starting file system protection ...\n\n"

scone fspf create /fspf.pb
scone fspf addr /fspf.pb /          --not-protected --kernel /
scone fspf addr /fspf.pb /usr       --authenticated --kernel /usr
scone fspf addf /fspf.pb /usr       /usr
scone fspf addr /fspf.pb /bin       --authenticated --kernel /bin
scone fspf addf /fspf.pb /bin       /bin
scone fspf addr /fspf.pb /lib       --authenticated --kernel /lib
scone fspf addf /fspf.pb /lib       /lib
scone fspf addr /fspf.pb /etc/ssl   --authenticated --kernel /etc/ssl
scone fspf addf /fspf.pb /etc/ssl   /etc/ssl
scone fspf addr /fspf.pb /sbin      --authenticated --kernel /sbin
scone fspf addf /fspf.pb /sbin      /sbin
printf "\n### Protecting code found in folder \"$APP_FOLDER\"\n\n"
scone fspf addr /fspf.pb $APP_FOLDER --authenticated --kernel $APP_FOLDER
scone fspf addf /fspf.pb $APP_FOLDER $APP_FOLDER

scone fspf encrypt /fspf.pb > ./keytag

MRENCLAVE="$(SCONE_HASH=1 $INTERPRETER)"
FSPF_TAG=$(cat ./keytag | awk '{print $9}')
FSPF_KEY=$(cat ./keytag | awk '{print $11}')
FINGERPRINT="$FSPF_KEY|$FSPF_TAG|$MRENCLAVE|$ENTRYPOINT"
echo $FINGERPRINT > ./fingerprint

printf "\n\n"
printf "Your application fingerprint (mrenclave) is ready:\n"
printf "#####################################################################\n"
printf "iexec.json:\n\n"
printf "%s\n" "\"app\": { " " \"owner\" : ... " " \"name\": ... " "  ..." " \"mrenclave\": \"$FINGERPRINT\"" "}"
printf "#####################################################################\n"
printf "Hint: Replace 'mrenclave' before doing 'iexec app deploy' step.\n"
printf "\n\n"
```
{% endcode %}

{% hint style="info" %}
All dependencies and files must be added to the image before invoking the **protect-fs.sh** script \(see below\).
{% endhint %}

{% hint style="warning" %}
It is important to carefully choose files to authenticate. It can be tricky to consider including enough files to protect the application without being more general than we should. For instance, if we authenticate the entire **/etc** directory the enclave will fail to boot because the content of /etc/hosts is modified at runtime by Docker.
{% endhint %}

{% hint style="warning" %}
That's why we do not simply authenticate "/" for example!
{% endhint %}

In the end, your folder's structure should be like this:

```bash
.
├── Dockerfile
├── protect-fs.sh
└── src
    └── app.js
```

## Build the docker image:

Once everything is ready we proceed to build the image. Make sure you are inside the right directory and run the following command in your terminal \(replace all occurrences of `<dockerusername>` with your Dockerhub username\):

```bash
docker image build . --tag my-tee-hello-world
```

At the end of the build's output, you should see this message \(with different a hash of course\):

```bash
Your application fingerprint (mrenclave) is ready:
#####################################################################
iexec.json:

"app": { 
 "owner" : ... 
 "name": ... 
  ...
 "mrenclave": "8a2e59370e47425ebaad0ba72ab06beb49ddf53aa1575c0de9a32dc82687d20c|695e1fd6bb78cc6745786d9941dda921|a8e434c81b82012c19d028ab3e7ef3adecb7786c10e5739422a7f7444e2d323c|node /app/app.js"
}
#####################################################################
Hint: Replace 'mrenclave' before doing 'iexec app deploy' step.
```

That alphanumeric string is the [fingerprint](intel-sgx-technology.md#applications-fingerprint) of your application. It allows the verification of its integrity.

## Test your app on iExec

At this stage, your application is ready to be tested on iExec. The process is similar to testing any type of application on the platform, with these minor exceptions:

* When you tag your Docker image, use `my-tee-hello-world` instead of `my-hello-world`.
* Do not forget the put the application's fingerprint inside `iexec.json` \(`"mrenclave"` attribute\).
* Add the option `--tag tee` to the command `iexec app run`.

Please go ahead and follow [these steps](../your-first-app.md#test-your-app-on-iexec) to run your first enclave-protected application on iExec.

## Next step?

In this tutorial, you learned how to leverage your application with the power of Trusted Execution Environments using iExec. But according to your use case, you may need to use some confidential data to get the full potential of the **Confidential Computing** paradigm. Check out the next chapter to see how.

