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

The given project `OrderService` has no vulnerable packages given the current sources.

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

.editorconfig - not called out as its own numbered step but listed under "what to submit". Needed anyway since dotnet format --verify-no-changes checks against it - without it the gate just enforces the SDK's implicit defaults instead of something explicit.

===========

No dotnet test step in ci.yml. No test project exists in the repo (no xunit/nunit/mstest csproj), so running it as-is would fail on a missing target, not a real test failure. App is two one-line GET endpoints, already verified manually with curl. If this grows real logic later, next step is a proper OrderService.Tests project with WebApplicationFactory<Program> for integration tests, added as a step ahead of the docker build.

===========

Tested that the format gate actually blocks the pipeline - added bad indentation to Program.cs on purpose, pushed without running dotnet format locally first.

format-and-test failed on the format step:
Run dotnet format --verify-no-changes
/home/runner/work/tl-test/tl-test/OrderService/Program.cs(6,1): error WHITESPACE: Fix whitespace formatting. Delete 4 characters.
/home/runner/work/tl-test/tl-test/OrderService/Program.cs(7,1): error WHITESPACE: Fix whitespace formatting. Delete 4 characters.
Error: Process completed with exit code 2.

Non-zero exit code = step fails = job stops. build and dependency scan steps never ran after it. docker-build-and-scan job never triggered either since it needs: format-and-test - so one bad indent blocks the whole rest of the pipeline (docker build, scan, push, deploy). Fixed the indentation, reran dotnet format locally, pushed again, gate went back to green.