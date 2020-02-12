# End-to-End Encryption

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.1 or higher.
* [Quick dev start](../quick-start-for-developers.md) tutorial.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](scone-framework.md#scone-framework) framework.
* [First part](create-your-first-sgx-app.md) of the confidential computing tutorial.
* [Second part](sgx-encrypted-dataset.md) of the confidential computing tutorial.
{% endhint %}

In the first part of this tutorial, we created a hello world application that runs securely inside an [enclave](intel-sgx-technology.md#enclave). In the second part, we deployed an encrypted dataset on iExec and pushed the secret to be stored safely in the [SMS](scone-framework.md#secret-management-service-sms).

In this section we will see how to combine both. Our next application will read an encrypted dataset and write its content in another file.

Go ahead and clone the tutorial Github repository and jump in the folder `scone/hello-world-app-with-dataset`.



Draft:

Once you deployed your hw dataset, let's make the app we deployed previously &lt;link&gt; read it.

change code

then return to tutorial.













