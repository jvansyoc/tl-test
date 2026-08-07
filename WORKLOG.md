Tried generating .net application, was notified that I did not have .NET SDK installed. 
Installed .NET SDK via Powershell command "winget install Microsoft.DotNet.SDK.8"
Reran command to build .NET app. "dotnet new webapi -n OrderService --use-controllers false"
Verified all the files were created in the folder.
Ran the app. "dotnet run"
Verified that "healthy" and "dev/1.0.0-test" reports from curl commands.

Verified clean list:
PS E:\Github Projects\Public\tl-test\OrderService> dotnet list package --vulnerable --include-transitive

The following sources were used:
   https://api.nuget.org/v3/index.json

The project `OrderService` has no vulnerable packages given the current sources.

==========

Retested to verify package installed, First attempts was me misspelling "Newon" instead of "Newton"

==========

PS E:\Github Projects\Public\tl-test\OrderService> dotnet list package --vulnerable --include-transitive

The following sources were used:
   https://api.nuget.org/v3/index.json

Project `OrderService` has the following vulnerable packages
   [net8.0]: 
   Top-level Package      Requested   Resolved   Severity   Advisory URL                                     
   > Newtonsoft.Json      9.0.1       9.0.1      High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

===========

Had to install Trivy locally for scanning docker containers. "winget install --id AquaSecurity.Trivy -e"

Complied the docker container "docker build -t orderservice:local ."

Ran Trivy Analysis "trivy image orderservice:local"
Had an issue with the screen showing too many finds, piped them to an output file for reasier reading "trivy-before"
Updated the Newtonsoft.Json package from 9.0.1 > 13.0.1
Rebuild the docker image
Reran Trivy, again piped to an output file. "no security findings detected"

===========

Ran docker container locally to verify appuser, then stopped and removed image.

PS E:\Github Projects\Public\tl-test\OrderService> docker run -d --name check orderservice:local
0de1146ee1ef4113a369e0684fbf9cb7bf9ac23a7f7cd5c089e8a7880b6f6bb3
PS E:\Github Projects\Public\tl-test\OrderService> docker exec check whoami
appuser
PS E:\Github Projects\Public\tl-test\OrderService> docker stop check
check
PS E:\Github Projects\Public\tl-test\OrderService> docker rm check
check

Why do we care if the container runs as root, and what did you have to change to make USER appuser actually work?
We care because root runs without guardrails, it can run any command and access any file or folder. Having a user or account that has explicit permissions can only run what is defined. In this case I did not need to change anything to make the USER command work as there is a previous step that creates the user prior to the Dockerfile calling it during the build.

===========

Added .editorconfig. This is needed since dotnet format "--verify-no-changes" checks against it and without the gate it will just enforce the SDK's implicit defaults instead of something explicit.

===========

Skipped dotnet test in ci.yml since there is no test project in the repo yet. It would just fail looking for something that doesn't exist. The app is only two, one-line, GET endpoints, and that is already checked those manually with curl. 

===========

Tested that the format gate actually blocks the pipeline by adding bad indentation to Program.cs on purpose, pushed without running dotnet format locally first.

format-and-test failed on the format step:
Run dotnet format --verify-no-changes
/home/runner/work/tl-test/tl-test/OrderService/Program.cs(6,1): error WHITESPACE: Fix whitespace formatting. Delete 4 characters.
/home/runner/work/tl-test/tl-test/OrderService/Program.cs(7,1): error WHITESPACE: Fix whitespace formatting. Delete 4 characters.
Error: Process completed with exit code 2.

Non-zero exit code = step fails = job stops. build and dependency scan steps never ran after it. 

Docker-build-and-scan job never triggered either since it needs: format-and-test, so one bad indent blocks the whole rest of the pipeline.
Fixed the indentation, reran dotnet format locally, pushed again, gate went back to green.

===========

Added push job to builds and push image to GHCR, gated to main only. Used built-in GITHUB_TOKEN instead of a custom token.

Checked repo Settings > Actions > General > Workflow permissions - set to "Read repository contents and packages permissions" (read-only default). Confirmed this doesn't block the push job since it declares its own permissions: packages: write at the job level, which overrides the repo-wide default for just that job. Left the repo setting as-is, didn't need to change it.

Pushed, job succeeded. Verified image actually landed in GHCR (not just trusting the green check) - package "orderservice" shows up under github.com/jvansyoc?tab=packages with a digest matching the commit SHA.

===========

Added deploy job - pull image, stop/rm any existing container, run new one, curl /version to verify.

First run failed:
Run docker pull $REGISTRY/$IMAGE_NAME:$GITHUB_SHA
Error: An error occurred trying to start process '/usr/bin/bash' with working directory '/home/runner/work/tl-test/tl-test/OrderService'. No such file or directory

Cause: workflow-wide working-directory: OrderService default applies to every run: step in every job. deploy doesn't run actions/checkout, so OrderService/ never exists on that job's runner.

Fix: added "working-directory" inside the deploy job specifically, overriding the workflow-level default back to root for just that job.

===========

Second run, pull/stop/rm/run all succeeded but curl step failed:
curl: (56) Recv failure: Connection reset by peer
Error: Process completed with exit code 56.

Cause: docker run -d returns immediately, doesn't wait for the app inside to finish starting. curl hit the port before the app had bound to it.

Fix: replaced the single curl with a retry loop polling /health every 2s up to 10 times before checking /version:
for i in {1..10}; do
  if curl --fail http://localhost:8080/health; then
    echo "App is up"
    break
  fi
  echo "Waiting for app to start... ($i/10)"
  sleep 2
done

Pushed again, all four jobs green.

===========

Manually tested the pushed image on a separate VM on my network. Pulled from GHCR, ran it, curled /health and /version, both correct. Confirms the image works standalone, not just inside the pipeline's own environment.

===========

Part 4

What's the actual risk of docker run with no restart policy and no health-based rollback, and what's the smallest change that reduces it?

Without a restart policy the container could fail during boot or crash later. The healthcheck in the Dockerfile marks the container unhealthy in docker inspect. To reduce risk you would want something to read that status, so it stops a broken container from being deployed or can roll back to the last working image.

===========

Part 5

Format check, test, security scan - if all three could run in parallel instead of sequentially, would you? What do you gain, what do you risk?

No, not fully. Format check is cheap and fast, you want it to fail first before spending time on the other steps. If it fails, no point having already paid for a build and a scan. Also the image scan can't actually run parallel with the build anyway, it needs the built image to exist first not just an ordering choice I made. I Would only consider parallelizing the format check against something else independent of it, not all three.

===========

Where do secrets live in your pipeline? What's the blast radius if one leaks into a log line?

Only secret in use is GITHUB_TOKEN, auto-generated per run by Actions. I didn't add a custom PAT. Actions auto-masks known secret values, the blast radius, if it, leaked anyway would be scoped to just this repo and not the whole account. Since the secret is short-lived the worst case is someone pushes a bad image to this repo's package, not account-wide access.

===========

Your Trivy gate uses --ignore-unfixed. A base-image CVE that had no fix gets one next month. Nothing in your pipeline catches that automatically. What's the gap, how do you close it?

Since the pipeline only scans on push/PR and if nobody touches the repo for a month, the image never gets rescanned even after a fix ships upstream. So a CVE that becomes fixable just sits there uncaught. The Fix would be to add a scheduled trigger that reruns the Trivy scan on a timer, not just on code changes, so a newly-fixed CVE gets caught even if the code hasn't moved.

===========

If this needed 3 replicas behind a load balancer instead of one docker run, what's the smallest realistic next step - and what would you explicitly not try to solve with a Dockerfile change alone?

The smalleet step would be to implement something that actually manages multiple containers and routes traffic between them, not more docker run commands. Docker Compose or an orchestrator with a load balancer in front and health checks wired to routing. A Dockerfile only defines what's inside the image, it has no concept of multiple running instances, load balancing, or traffic routing. No amount of editing it turns one docker run into 3 coordinated replicas.