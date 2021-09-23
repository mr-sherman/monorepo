# How to use CodeQL Code Scanning in a Monorepo

## The Scenario
In this example repository, we want to be able to build, scan, test, and deploy each of these sub repositories independently of the other.  In such a scenario, each sub repository doesn't know anything about the directory structure of the monorepo.  As such, when we build ```broken```, it is done entirely within its own directory, not from the root of the monorepo.  

## The Problem
There are two potential problems with scanning each sub-directory independently of the other.  The first is that when you upload the SARIF results file to the GitHub repository, the file only contains the results of that sub-repo's scan.  So, you end up having a .sarif file for ```broken``` and one for ```govwa```.  If we upload the results of a scan of ```broken```, and then upload the results of a scan of ```govwa```, then the code scanning alerts from ```broken``` will get automatically closed.  This is because the results of the ```govwa``` scan don't contain entirely different results, and when you upload the .sarif file, the security alerts logic will assume that all of those vulnerabilities that had existed for ```broken``` have been remedied, and will create new alerts, only from the scan of ```govwa```.

The second problem is that since the scan was done from the ```broken``` or ```govwa``` root directory and not from the root of the monorepo, the results' previews will not render correctly in the security alert.  The .sarif file contains pointers to source files with a line and column number so the alert dashboard can render a preview of the source, but since the .sarif file has file locations starting at the root of the subrepo and not at the root of the mono repo, they will not render appropriately.

## The Solution
In this example, we will use the CodeQL CLI.  You can do the same thing with CodeQL Runner, and I'll call out the specific flags you need to set in order to solve these problems:

First, we want to build the CodeQL databases:
```
cd broken
/codeql-2.6.1/codeql database create --language=javascript db
cd ..
cd govwa
/codeql-2.6.1/codeql database create --language=go db
```

Now we want to analyze each, but we need to make use of the ```--sarif-category``` option to solve problem #1 mentioned above.  Otherwise, each .sarif upload will clobber the other.
```
cd ..
cd broken
/codeql-2.6.1/codeql database analyze --format=sarif-latest --output=results-before.sarif --sarif-category=broken  db /codeql-2.6.1/qlpacks/codeql-javascript/codeql-suites/javascript-security-extended.qls
cd ..
cd govwa
/codeql-2.6.1/codeql database analyze --format=sarif-latest --output=results-before.sarif --sarif-category=govwa  db /codeql-2.6.1/qlpacks/codeql-go/codeql-suites/go-security-extended.qls
```

How we have two .sarif files whose results' locations are based off of the root being the root of the sub-repo.  We need to change that so the results' locations are based off the monorepo.  Since .sarif files are just JSON-formatted data, we can use ```jq``` to modify the results and create a new .sarif from the command line:
```
cd ..
cd broken
cat results-before.sarif | jq '(..|.uri? | select( . != null)) |= "broken/"+. ' > results.sarif  
cd ..
cd govwa
cat results-before.sarif | jq '(..|.uri? | select( . != null)) |= "govwa/"+. ' > results.sarif 
```

Now we can upload each of those results.sarif files independently using the ```codeql github upload``` command.
You can see how the results are rendered by clicking on the "Security" tab above and viewing the code scanning alerts.

## Considerations for the CodeQL Runner. 
If you're using the CodeQL Runner, you'll want to use the ```--category``` flag in the runner's ```analyze``` step.  Use this in the same manner as the ```sarif-category``` flag of the CLI.  
Another thing you can't do is analyze and upload the results to GitHub in the same step.   You need to specify the ```--no-upload``` flag in the ```analyze``` command, and then use the ```upload``` command to upload the sarif files in a separate step. 
Documentation on these fields are here:
https://docs.github.com/en/code-security/code-scanning/using-codeql-code-scanning-with-your-existing-ci-system/configuring-codeql-runner-in-your-ci-system

